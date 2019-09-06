```
 _____ ______   ________  _______   ________  ___       ________      ___    ___ _______   ________     
|\   _ \  _   \|\   __  \|\  ___ \ |\   __  \|\  \     |\   __  \    |\  \  /  /|\  ___ \ |\   __  \    
\ \  \\\__\ \  \ \  \|\  \ \   __/|\ \  \|\  \ \  \    \ \  \|\  \   \ \  \/  / | \   __/|\ \  \|\  \   
 \ \  \\|__| \  \ \   _  _\ \  \_|/_\ \   ____\ \  \    \ \   __  \   \ \    / / \ \  \_|/_\ \   _  _\  
  \ \  \    \ \  \ \  \\  \\ \  \_|\ \ \  \___|\ \  \____\ \  \ \  \   \/  /  /   \ \  \_|\ \ \  \\  \| 
   \ \__\    \ \__\ \__\\ _\\ \_______\ \__\    \ \_______\ \__\ \__\__/  / /      \ \_______\ \__\\ _\ 
    \|__|     \|__|\|__|\|__|\|_______|\|__|     \|_______|\|__|\|__|\___/ /        \|_______|\|__|\|__|
                                                                    \|___|/                             
```
## Purpose
Replaying recorded MySQL requests to a test environment database

## Recording Method
该工具读取tcpdump的产物，因此需要使用tcpdump进行流量录制，命令如下：
The tool reads the output of tcpdump, so you need to use tcpdump for traffic recording, as follows:
```sh
tcpdump -s 0 -i eth0 dst port 4000 and tcp -w /run/test.pcap
```
Where `-s 0` means to capture the entire packet, `-i eth0` means on interface eth0, `dst port 4000 and tcp` means to capture the TCP packets targeting port `4000`, which depends on the database instance, then `-w /run/test/pcap` means to save the data to `/run/test.pcap`.

If you can't remember so many parameters and often need to record, you can write the above command to a script `record.sh`:
```sh
#! /bin/sh

tcpdump -s 0 -i eth0 dst port 4000 and tcp -w /run/test.pcap
```

This is our tool for recording traffic! Of course, you can make some tweaks to the script to make it more efficient, such as adding the -B parameter to avoid the packet loss caused by tcpdump processing during bursty traffic.

## Playback Mode
### Preprocessing
To play back traffic, you need to parse the captured tcpdump first. Assuming that the capture file is `/run/test.pcap`, you need to first:
```sh
mysql-replayer prepare -i /run/test.pcap -o /tmp/test
```
This will analyze the pcap file and put the result in the /tmp/test directory (if the `/tmp/test` directory does not exist, it will be created automatically. If it is not a directory, it will report an error). 
After the analysis, the output will be a series of files, like this:
```
1550990789-194.32.77.196-56598-498081.rec
1550990789-194.32.77.196-57616-137694.rec
1550990790-194.32.77.196-57490-118496.rec
1550990789-194.32.77.196-56600-727887.rec
1550990789-194.32.77.196-57618-167221.rec
1550990792-194.32.77.196-57266-62854.rec
1550990789-194.32.77.196-56602-131847.rec
1550990789-194.32.77.196-57620-586574.rec  
1550990792-194.32.77.196-57268-60294.rec
```
The file name consists of `{TCP handshake time}-{client ip}-{client port}-{random number}.rec`
### Playback
Playback command documentation:
```
mysql-replayer bench -i input-dir [-h host] [-P port] [-u user] [-p passwd] [-s speed] [-c concurrent]:
        Bench mysql server with data from input-dir.
  -P string
        port number to use for connection (default "4000")
  -c int
        the concurrency, 0 or negative number means dynamic concurrency
  -h string
        connect to host (default "127.0.0.1")
  -i string
        the directory contains bench data
  -p string
        password to use when connecting to server
  -s int
        the bench speed (default 1)
  -u string
        user for login (default "root")
```
Suppose the previous step processes the pcap file into the directory `/tmp/test`, the database user is `root`, the database listens on the local port `4000`, the concurrency is `200`, and the speed is `2` times the actual speed during recording. (The measured results are not very accurate), then the command line would be:
```sh
mysql-replayer bench -i /tmp/test  -c 200 -s 2 # user and port are not required because they are default values
```
If the parameter `-c` is not used, the default is `0`, which means that the concurrency will be dynamically adjusted to follow the actual environment when recording the traffic. 
(For example, initially there may have been `200` concurrent sessions, but after a while it increased to `1000`, then the tool will automatically increase from `200` to `1000` sessions)

### Example Usage
Record 20 seconds of sysbench traffic （768 concurrent sessions). The output of sysbench is:
```
[ 10s ] thds: 768 tps: 0.00 qps: 260.12 (r/w/o: 213.19/0.00/46.93) lat (ms,95%): 0.00 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 768 tps: 0.00 qps: 243.40 (r/w/o: 225.36/0.00/18.04) lat (ms,95%): 0.00 err/s: 0.00 reconn/s: 0.00
```
Use mysql-replayer to play back the traffic using recorded sysbench traffic as the data source:
```
[root@vps mysql-replayer]# ./mysql-replayer bench -i test  -c 500
Processing...
Process 6273 querys in 32 seconds, QPS: 196

[root@vps mysql-replayer]# ./mysql-replayer bench -i test -c 500 -s 10
Processing...
Process 6273 querys in 8 seconds, QPS: 784

[root@vps mysql-replayer]# ./mysql-replayer bench -i test -c 2000 -s 10
Processing...
Process 6273 querys in 5 seconds, QPS: 1254
```
Actual results depend on the network environment at the time
