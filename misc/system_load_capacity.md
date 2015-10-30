# 系统负载能力

## 衡量参数与指标

* `Network Bandwidth(带宽)`
* `System Configuration(系统配置)`

### Network Bandwidth(带宽)
tools:

* netcat(nc)
* pv (allows a user to see the progress of data through a pipline.)
* iperf

#### netcat & pv
Server side
```bash
root@vagrant-ubuntu-trusty:~# nc -vvlnp 12345 > /dev/null
Listening on [0.0.0.0] (family 0, port 12345)
```
Client side
```bash
root@vagrant-ubuntu-trusty:~# dd if=/dev/zero bs=1M count=1K | nc -vvn 127.0.0.1  12345
Connection to 127.0.0.1 12345 port [tcp/*] succeeded!
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 4.43615 s, 242 MB/s
```
所以, server的带宽为242 MB/s.

或者, Server side
```bash
root@vagrant-ubuntu-trusty:~# nc -vvlnp 12345 | pv
Listening on [0.0.0.0] (family 0, port 12345)
Connection from [127.0.0.1] port 12345 [tcp/*] accepted (family 2, sport 50396)
   1GB 0:02:02 [8.37MB/s] [ <=>                                              ]
```
Client side
```bash
root@vagrant-ubuntu-trusty:~# dd if=/dev/zero bs=1M count=1K | nc -vvn 127.0.0.1  12345
Connection to 127.0.0.1 12345 port [tcp/*] succeeded!
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 115.727 s, 9.3 MB/s
```

#### iperf
Server side
```bash
root@vagrant-ubuntu-trusty:~# iperf -s
------------------------------------------------------------
Server listening on TCP port 5001
TCP window size: 85.3 KByte (default)
------------------------------------------------------------
[  4] local 127.0.0.1 port 5001 connected with 127.0.0.1 port 60144
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0-10.0 sec  24.5 GBytes  21.0 Gbits/sec
```
Client side
```bash
root@vagrant-ubuntu-trusty:~# iperf -c localhost
------------------------------------------------------------
Client connecting to localhost, TCP port 5001
TCP window size: 2.50 MByte (default)
------------------------------------------------------------
[  3] local 127.0.0.1 port 60144 connected with 127.0.0.1 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  24.5 GBytes  21.0 Gbits/sec
```

### System Configuration

* File Descriptors Limit(文静描述符限制): everything is a file. 主要针对sockets
* Process/Thread Number Limit(进程/线程数限制)
* The Kernel Configuration of TCP

#### File Descriptors Limit
* For whole system:
    * temporary modification
        ```bash
        echo 1000000 > /proc/sys/fs/file-max
        ```
    * permanent modification
        ```bash
        root@vagrant-ubuntu-trusty:~# vim /etc/sysctl.conf
    
        ############################
        # File Descriptors
        fs.file-max = 1000000
        ############################
    
        # reload sysctl variables
        root@vagrant-ubuntu-trusty:~# sysctl -p
        ```
* For one process:
    * `ulimit -n`进行查看和修改
    * 修改*/etc/security/limits.conf*中的nofile
