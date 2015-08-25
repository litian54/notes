# DevOps Troubleshooting: Linux Server Best Practices

## System Load
### 什么是system load
`load average` = `runnable`(using the CPU or waiting) 或者 `uninterruptible`(waiting for I/O)的进程平均数

### 查看load:

* `uptime`
    ```bash
    root@vagrant-ubuntu-trusty:~# uptime
    22:15:29 up 0 min,  1 user,  load average: 0.14, 0.04, 0.02
    ```

* `top`:
    ```
    top - 22:17:16 up 2 min,  1 user,  load average: 0.02, 0.03, 0.02
    Tasks:  76 total,   1 running,  75 sleeping,   0 stopped,   0 zombie
    %Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
    KiB Mem:    501832 total,   174892 used,   326940 free,    42184 buffers
    KiB Swap:   522236 total,        0 used,   522236 free.    87296 cached Mem
    ```

    * us: user CPU time，cpu跑用户进程的时间占cpu总时间的百分比。
    * sy: system CPU time，cpu跑kernel进程的时间占cpu时间的百分比。
    * ni: nice CPU time，cpu跑调整了nice优先级的用户进程的时间占cpu时间的百分比。nice优先级（-20 - 19)，数值越小优先级越大。
    * id: CPU idle time，cpu空闲时间占cpu总时间的百分比。如果系统慢，但是id很高，则说明不是high CPU load。如上，100% id，load 0.02(基本趋于0)，两个参数是相匹配的，说明cpu一直闲着。
    * wa: I/O wait，cpu等待I/O的时间占cpu总时间的百分比。如果系统慢，但是wa值低，则可以排除是因为disk或者network I/O问题引起的。
    * hi: hardware interrupts，cpu处理硬件中断的时间占cpu总时间的百分比。
    * si: software interrupts，cpu处理软件中断的时间占cpu总时间的百分比。
    * st: steal time，如果有virtual machines在跑，这是vm中执行的其他任务时间占cpu总时间的百分比。

用top查看系统慢的问题，一般顺序是:
1. wa(I/O wait)，查看是否是disk I/O的问题。
2. 如果wa低，查看id(idle)；如果wa高，则去查看disk和network I/O的问题。
3. 如果wa和id低，查看us，估计是用户进程(程序问题)；如果id高，说明cpu空闲，则有可能是其他如 MySQL query慢，或者network问题，或者web server问题等。

### High load Average:

* CPU-bound(processes waiting on CPU resources)
* RAM-bound(specifi cally, high RAM usage that has moved into swap)
* I/O-bound(processes fi ghting for disk or network I/O)

通常情况，CPU-bound load对系统影响较小，也就是说系统仍然有较好的响应。而RAM-bound load和I/O-bound load都因为对disk资源的消耗导致进程变慢甚至停止。  

load高低一般根据系统cpu core的数量来判断。如果`load average`大于`cpu core`的数量，则一般情况下load过高。
```bash
root@vagrant-ubuntu-trusty:~# cat /proc/cpuinfo | grep 'core id'
core id         : 0
core id         : 1
core id         : 2
```

#### High User Time
us高，wa低:
* 特定用户进程用了大量的cpu资源: 在`top`中，点'K'，然后输入PID来kill掉那个进程。
* 进程过多消耗cpu资源: 可以nice部分进程或者kill掉部分，然后在某个时间ugrade server或者将任务分散到多个servers上去。

#### Out of Memory
除了通过`top`查看memory，还可以用过`free`
```bash
root@vagrant-ubuntu-trusty:~# free -m
             total       used       free     shared    buffers     cached
Mem:           490        170        319          0         41         85
-/+ buffers/cache:         43        446
Swap:          509          0        509
```
`-m`: 代表单位是mb
一些关于memory的计算:
真正的 free = free + buffers + cached = 319 + 41 + 85 约等于446
真正的 used = used - (cached + buffer) = 170 - (85 + 41) 约等于43
所以第二行才是真正的used和free。

下面看`top`中关于memory的信息:
```bash
Mem: 1024176k total, 997408k used, 26768k free, 85520k buffers
Swap: 1004052k total, 4360k used, 999692k free, 286040k cached
```
swap被用了一点，没关系。因为"If a process becomes idle,
Linux will often page its memory to swap to free up RAM for other processes."

Memory有问题: used - cache 和 swap同时都高。  
在`top`中点'M'会按照memory usage排序

### High I/O Wait
`iostat`与常用参数:
* `-x`: displays extended statistics.
* `-m`: megabytes.
* `-d`: displays the device utilization report.
* `-t`: prints the time for each report displayed.
* 数字`n`: 每隔n秒刷新一次.
```bash
root@vagrant-ubuntu-trusty:~# iostat
Linux 3.13.0-24-generic (vagrant-ubuntu-trusty)         07/11/2015

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.07    0.00    0.22    0.14    0.00   99.57

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda               1.04        14.99         5.00     190779      63600
```
* tps: transfer per second to the device
'transfer' is I/O requests send to the device.
* kB_read/s: kB read from the device per second.
* kB_wrtn/s: kB write to the device per second.
* kB_read: total number of kB.
* kB_wrtn: total number of kB.

`iotop`: 可以查看每个进程对disk的读写。

`sar`命令：

* 默认显示cpu信息
* `-r`: RAM(memory)信息
* `-b`: disk信息
* `-A`: all
* `sar -s 20:00:00 -e 20:30:00`: star - end
* `sar -f /var/log/sysstat/sa06`: 指定某天

------

## The Linux Boot Process
### Linux boot process
```
    +------------+  boot loader task:
    |   BIOS     |  Basic Input Output System
    |            |  Intialize hardware and find the first device to boot
    +-----+------+  Usually reading the MBR(master boot record: the first
          |                                 512 bytes on a hard drive)
          v
    +------------+  To find what OS(and its kernel) can boot.
    |    GRUB    |  Stage 1: GRUB put a bit of code(446 bytes) into MBR,
    |            |           and execute it. The stage 1's code just enough
    +-----+------+           to locate the rest of the code on disk and
          |                  execute that.
          |         Stage 2: access Linux File System --> read and load
          |                  configuration --> find different OS can boot
          |
          v  select a kernel in GRUB
+-------------------------+  'initrd': is a gzipped cpio archive
| GRUB load kernel to RAM |  known as an initramfs file and contains
| also load initrd        |  a small basic Linux root file system.
| (initial RAM disk)      |  (inclue crucial kernel config and modules)
+-------------------------+  
          |
          v
    +------------+  Which is shell script to detect and mount real root file
    | Run script |  system.
    |   'init'   |
    +------------+
          |
          v
    Run /sbin/init  (PID 1)
```
### Classic System V Init
先读取`/etc/inittab`，根据'runlevels'(0-6)来以不同状态启动系统:

* '0': shutdown
* '1'-'5': 可自定义
* '6': single-user mode

启动脚本:

* `/etc/init.d`: 所有的启动脚本(start or stop for all runlevel)。
* `/etc/rc0.d` - `/etc/rc6.d`: 针对不同的runlevels用的S(start), K (kill), or D (disable)的脚本。
* `/etc/rcS.d`: 在进步某个runlevel之前跑的start-up脚本。
* `/etc/rc.local`: 用户自定义脚本。

### Upstart
event-driven. config in `/etc/init`.  
查看Upstart jobs:
```bash
root@vagrant-ubuntu-trusty:~# initctl list
mountnfs-bootclean.sh start/running
rsyslog start/running, process 621
tty4 start/running, process 724
udev start/running, process 266
upstart-udev-bridge start/running, process 1084
mountall-net stop/waiting
passwd stop/waiting
rc stop/waiting
startpar-bridge stop/waiting
......
```

### Fix GRUB

* 修复GRUB: 如果仍可以进入系统，说明GRUB至少可以成功到stage 2: 
    * 修改`/etc/default/grub`
    * 运行`/usr/sbin/update-grub`。
* Rescue Disk修复: 
    * 用USB image或CD-ROM的rescue disk(rescue mode)。
    * chroot /mnt/sysimage切换并挂载root partition。
    * 在shell中重新安装GRUB到MBR，如`/sbin/grub-install /dev/sda`。其中`/dev/sda`是root device，可以通过`df`查看。

### 显示详细的boot process信息用于debug:

* 在boot时Esc或者Shift进入GRUB menu, 点E来进行修改。
* 修改kernel boot arguments(通常以linux或kernel开头)，删掉'splash'和'quiet'，或者添加'nosplash'。

### 无法挂载root file system
root option: `root=/dev/sda2, root=LABLE=/`或者`root=UUID=xxxxxxxxxx`。用UUID保证root不会重复。  
* `fdisk -l`查看所有partitions。
* `blkid -s UUID /dev/sda2`查看UUID。

------

## Solving Full or Corrupt Disk Issues
### Reserved Blocks
查看系统保留的磁盘空间:
```bash
root@vagrant-ubuntu-trusty:~# tune2fs -l /dev/sda1 | grep -i "block count"
Block count:              10354432
Reserved block count:     517721
```
找到占用磁盘空间的大文件:  
`du -ckx / | sort -nr | head -n 10`，其中，

* `-c`: total
* `-k`: like --block-size=1K
* `x`: one file system

### Out of Inodes
查看inodes的使用量:
```bash
root@vagrant-ubuntu-trusty:~# df -i
Filesystem      Inodes IUsed   IFree IUse% Mounted on
/dev/sda1      2588672 68524 2520148    3% /
```

### Read-only File System
如果发现文件系统是只读，而需要读写操作，可以重挂一次:  
如，重挂`/home`
```bash
sudo mount -o remount,rw /home
```
可以通过`dmesg`查看系统重挂文件系统的信息
```bash
root@vagrant-ubuntu-trusty:~# dmesg | less
#### output
......
[    7.972801] EXT4-fs (sda1): re-mounted. Opts: errors=remount-ro
......
```

### Repair Corrupted File Systems
`fsck`命令检查文件系统:

    * 首先确定unmounted文件系统，否则fsck会破坏文件系统。(mount查看，umount <devicename>卸载文件系统)
    * 用fsck检查，如`/home`之前挂载在`/dev/sda5`
    ```bash
    # fsck -y -C /dev/sda5
    ```
        * `-y`，yes
        * `-C`, progress bar
    * 用backup的superblock修复，例如对ext-based的文件系统:
    ```bash
    # mke2fs -n /dev/sda5
    # fsck -b 8193 -y -C /dev/sda5
    ```
        * `-n` option是list所有的superblocks，如果不加`-n`，文件系统会被格式化。
        * 根据superblocks的值，加`-b`参数修复文件系统

### Repair Software RAID
暂时略过

------

## Tracking Down the Source of Network Problem
### Is it Plugged in?
`ethtool`需要额外安装
```bash
root@vagrant-ubuntu-trusty:~# ethtool eth0
Settings for eth0:
        Supported ports: [ TP ]
        Supported link modes:   10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Supported pause frame use: No
        Supports auto-negotiation: Yes
        Advertised link modes:  10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Advertised pause frame use: No
        Advertised auto-negotiation: Yes
        Speed: 1000Mb/s
        Duplex: Full
        Port: Twisted Pair
        PHYAD: 0
        Transceiver: internal
        Auto-negotiation: on
        MDI-X: off (auto)
        Supports Wake-on: umbg
        Wake-on: d
        Current message level: 0x00000007 (7)
                               drv probe link
        Link detected: yes
```
最后一行"Link detected: yes"说明网络连通。  

如果'Duplex'为half(半双工)，可以用以下命令修改为full(全双工):
```bash
# ethtool -s eth0 autoneg off duplex full
```
### Is the Interface Up?
```bash
root@vagrant-ubuntu-trusty:~# ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 08:00:27:3b:9c:80
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe3b:9c80/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:37220 errors:0 dropped:0 overruns:0 frame:0
          TX packets:27687 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:8736245 (8.7 MB)  TX bytes:2745404 (2.7 MB)
```
如果没有返回interface的config，可以:
```bash
# ifup eth0
# ifconfig eth0
```
或者在config里修改:

    * Debian: `/etc/network/interfaces
    * Red Hat: `/etc/sysconfig/network_scripts/ifcfg-<interface>

### Is it on the Local Network?
查看路由信息
```bash
root@vagrant-ubuntu-trusty:~# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         10.0.2.2        0.0.0.0         UG    0      0        0 eth0
10.0.2.0        *               255.255.255.0   U     0      0        0 eth0
```

    * "10.0.2.0"表示主机所在的网络地址
    * default那行就是表示数据传送目的是访问intenet，由eth0将数据包发送到网关10.0.2.2

Reset Interface:

    * Debian: `service networking restart`
    * Red-Hat: `service network restart`

用`ping`来检查是否可以访问gateway:
```bash
root@vagrant-ubuntu-trusty:~# ping -c 5 10.0.2.2
PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.
64 bytes from 10.0.2.2: icmp_seq=1 ttl=63 time=0.312 ms
64 bytes from 10.0.2.2: icmp_seq=2 ttl=63 time=0.236 ms
64 bytes from 10.0.2.2: icmp_seq=3 ttl=63 time=0.320 ms
64 bytes from 10.0.2.2: icmp_seq=4 ttl=63 time=0.381 ms
64 bytes from 10.0.2.2: icmp_seq=5 ttl=63 time=0.336 ms

--- 10.0.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 3998ms
rtt min/avg/max/mdev = 0.236/0.317/0.381/0.047 ms
```
### Is DNS Working?
检查DNS是否工作:
```bash
root@vagrant-ubuntu-trusty:~# nslookup www.google.com
Server:         10.0.2.3
Address:        10.0.2.3#53

Non-authoritative answer:
Name:   www.google.com
Address: 74.125.239.51
Name:   www.google.com
Address: 74.125.239.48
......
```
**注意:** Unix Base已经不再用nslookup了。  

如果返回如下错误，
```bash
$ nslookup web1
;; connection timed out; no servers could be reached
```
需要去*/etc/resolv.conf*查看相关nameserver的配置是否有误，
```bash
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 10.0.2.3
```
并尝试`ping 10.0.2.3`

