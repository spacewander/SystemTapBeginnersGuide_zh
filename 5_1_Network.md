# 5.1. 网络

以下各节的脚本展示了如何跟踪网络相关的函数和剖析（profile）网络活动。

## 剖析网络活动

本节展示SystemTap中剖析网络活动的方式。下面的`nettop.stp`允许我们一窥每个进程的网络流量使用情况。

nettop.stp
```
#! /usr/bin/env stap

global ifxmit, ifrecv
global ifmerged

probe netdev.transmit
{
  ifxmit[pid(), dev_name, execname(), uid()] <<< length
}

probe netdev.receive
{
  ifrecv[pid(), dev_name, execname(), uid()] <<< length
}

function print_activity()
{
  printf("%5s %5s %-7s %7s %7s %7s %7s %-15s\n",
         "PID", "UID", "DEV", "XMIT_PK", "RECV_PK",
         "XMIT_KB", "RECV_KB", "COMMAND")

  foreach ([pid, dev, exec, uid] in ifrecv) {
      ifmerged[pid, dev, exec, uid] += @count(ifrecv[pid,dev,exec,uid]);
  }
  foreach ([pid, dev, exec, uid] in ifxmit) {
      ifmerged[pid, dev, exec, uid] += @count(ifxmit[pid,dev,exec,uid]);
  }
  foreach ([pid, dev, exec, uid] in ifmerged-) {
    n_xmit = @count(ifxmit[pid, dev, exec, uid])
    n_recv = @count(ifrecv[pid, dev, exec, uid])
    printf("%5d %5d %-7s %7d %7d %7d %7d %-15s\n",
           pid, uid, dev, n_xmit, n_recv,
           n_xmit ? @sum(ifxmit[pid, dev, exec, uid])/1024 : 0,
           n_recv ? @sum(ifrecv[pid, dev, exec, uid])/1024 : 0,
           exec)
  }

  print("\n")

  delete ifxmit
  delete ifrecv
  delete ifmerged
}

probe timer.ms(5000), end, error
{
  print_activity()
}
```

注意看`print_activity()`的这几个表达式：
```
n_xmit ? @sum(ifxmit[pid, dev, exec, uid])/1024 : 0
n_recv ? @sum(ifrecv[pid, dev, exec, uid])/1024 : 0
```

它们也是`if/else`语句，等价于如下的伪代码：
```
if n_recv != 0 then
  @sum(ifrecv[pid, dev, exec, uid])/1024
else
  0
```

`nettop.stp`跟踪用了网络流量的进程，并逐个进程输出如下的信息：
* PID — 进程的PID.
* UID — 进程所有者的UID。
* DEV — 进程使用的端口，如`eth0`、`eth1`。
* XMIT_PK — 发送的包的数量
* RECV_PK — 接收的包的数量
* XMIT_KB — 发送的KB数
* RECV_KB — 接收的KB数

`nettop.stp`每隔5秒就会取样一次。你可以修改`probe timer.ms(5000)`来调整取样间隔。`nettop.stp`在20秒内的输出如下：
```
[...]
  PID   UID DEV     XMIT_PK RECV_PK XMIT_KB RECV_KB COMMAND
    0     0 eth0          0       5       0       0 swapper
11178     0 eth0          2       0       0       0 synergyc

  PID   UID DEV     XMIT_PK RECV_PK XMIT_KB RECV_KB COMMAND
 2886     4 eth0         79       0       5       0 cups-polld
11362     0 eth0          0      61       0       5 firefox
    0     0 eth0          3      32       0       3 swapper
 2886     4 lo            4       4       0       0 cups-polld
11178     0 eth0          3       0       0       0 synergyc

  PID   UID DEV     XMIT_PK RECV_PK XMIT_KB RECV_KB COMMAND
    0     0 eth0          0       6       0       0 swapper
 2886     4 lo            2       2       0       0 cups-polld
11178     0 eth0          3       0       0       0 synergyc
 3611     0 eth0          0       1       0       0 Xorg

  PID   UID DEV     XMIT_PK RECV_PK XMIT_KB RECV_KB COMMAND
    0     0 eth0          3      42       0       2 swapper
11178     0 eth0         43       1       3       0 synergyc
11362     0 eth0          0       7       0       0 firefox
 3897     0 eth0          0       1       0       0 multiload-apple
[...]
```



## 跟踪网络连接中的内核函数调用

本节展示如何跟踪内核的`net/socket.c`中的函数的调用情况。这将帮助你从细节上看清各进程是怎么跟内核的网络功能打交道的。

socket-trace.stp
```
#! /usr/bin/env stap

probe kernel.function("*@net/socket.c").call {
  printf ("%s -> %s\n", thread_indent(1), ppfunc())
}

probe kernel.function("*@net/socket.c").return {
  printf ("%s <- %s\n", thread_indent(-1), ppfunc())
}
```

`socket-trace.stp`这个脚本其实在我们之前在第3章介绍`thread_indent()`的时候已经见过了。下面是它在3秒内的输出：
```
[...]
0 Xorg(3611): -> sock_poll
3 Xorg(3611): <- sock_poll
0 Xorg(3611): -> sock_poll
3 Xorg(3611): <- sock_poll
0 gnome-terminal(11106): -> sock_poll
5 gnome-terminal(11106): <- sock_poll
0 scim-bridge(3883): -> sock_poll
3 scim-bridge(3883): <- sock_poll
0 scim-bridge(3883): -> sys_socketcall
4 scim-bridge(3883):  -> sys_recv
8 scim-bridge(3883):   -> sys_recvfrom
12 scim-bridge(3883):-> sock_from_file
16 scim-bridge(3883):<- sock_from_file
20 scim-bridge(3883):-> sock_recvmsg
24 scim-bridge(3883):<- sock_recvmsg
28 scim-bridge(3883):   <- sys_recvfrom
31 scim-bridge(3883):  <- sys_recv
35 scim-bridge(3883): <- sys_socketcall
[...]
```

## 监控TCP连接的创建

本节展示如何监控TCP连接的创建。这可以帮助你第一时间识别出任何未授权的、可疑的或其它不请自来的网络连接。

tcp_connections.stp
```
#! /usr/bin/env stap

probe begin {
  printf("%6s %16s %6s %6s %16s\n",
         "UID", "CMD", "PID", "PORT", "IP_SOURCE")
}

probe kernel.function("tcp_accept").return?,
      kernel.function("inet_csk_accept").return? {
  sock = $return
  if (sock != 0)
    printf("%6d %16s %6d %6d %16s\n", uid(), execname(), pid(),
           inet_get_local_port(sock), inet_get_ip_source(sock))
}
```

当`tcp_connections.stp`运行时，它会实时输出新创建的TCP连接的如下信息：
* 当前UID
* 接受连接的程序名
* 接受连接的进程PID
* 创建连接的远程IP地址

```
UID            CMD    PID   PORT        IP_SOURCE
0             sshd   3165     22      10.64.0.227
0             sshd   3165     22      10.64.0.227
```

## 监控TCP包

本节展示如何监控收到的TCP包。这可以帮助你分析应用的流量使用情况。

tcpdumplike.stp
```
#! /usr/bin/env stap

// A TCP dump like example

probe begin, timer.s(1) {
  printf("-----------------------------------------------------------------\n")
  printf("       Source IP         Dest IP  SPort  DPort  U  A  P  R  S  F \n")
  printf("-----------------------------------------------------------------\n")
}

probe udp.recvmsg /* ,udp.sendmsg */ {
  printf(" %15s %15s  %5d  %5d  UDP\n",
         saddr, daddr, sport, dport)
}

probe tcp.receive {
  printf(" %15s %15s  %5d  %5d  %d  %d  %d  %d  %d  %d\n",
         saddr, daddr, sport, dport, urg, ack, psh, rst, syn, fin)
}
```

当`tcpdumplike.stp`运行时，它会实时输出收到的TCP包的如下信息：
* 源IP地址和目标IP地址（saddr和daddr）
* 源端口和目标端口（sport和dport）
* 包标识

`tcpdumplike.stp`使用了以下函数来获取包的标识信息：
* urg - urgent
* ack - acknowledgement
* psh - push
* rst - reset
* syn - synchronize
* fin - finished

上述函数返回1或0来表示包中是否存在对应的标识。
⁠
```
-----------------------------------------------------------------
       Source IP         Dest IP  SPort  DPort  U  A  P  R  S  F
-----------------------------------------------------------------
  209.85.229.147       10.0.2.15     80  20373  0  1  1  0  0  0
  92.122.126.240       10.0.2.15     80  53214  0  1  0  0  1  0
  92.122.126.240       10.0.2.15     80  53214  0  1  0  0  0  0
  209.85.229.118       10.0.2.15     80  63433  0  1  0  0  1  0
  209.85.229.118       10.0.2.15     80  63433  0  1  0  0  0  0
  209.85.229.147       10.0.2.15     80  21141  0  1  1  0  0  0
  209.85.229.147       10.0.2.15     80  21141  0  1  1  0  0  0
  209.85.229.147       10.0.2.15     80  21141  0  1  1  0  0  0
  209.85.229.147       10.0.2.15     80  21141  0  1  1  0  0  0
  209.85.229.147       10.0.2.15     80  21141  0  1  1  0  0  0
  209.85.229.118       10.0.2.15     80  63433  0  1  1  0  0  0
[...]
```

## 监控内核中的网络丢包情况

某些情况下Linux网络栈会丢包。有些版本的Linux内核包含静态内核探测点`kernel.trace("kfree_skb")`，它可以帮助你跟踪包丢掉的原因。`dropwatch.stp`就使用了它来跟踪丢包；这个脚本每五秒统计一次丢包的位置。

dropwatch.stp
```
#! /usr/bin/env stap

############################################################
# Dropwatch.stp
# Author: Neil Horman <nhorman@redhat.com>
# An example script to mimic the behavior of the dropwatch utility
# http://fedorahosted.org/dropwatch
############################################################

# Array to hold the list of drop points we find
global locations

# Note when we turn the monitor on and off
probe begin { printf("Monitoring for dropped packets\n") }
probe end { printf("Stopping dropped packet monitor\n") }

# increment a drop counter for every location we drop at
probe kernel.trace("kfree_skb") { locations[$location] <<< 1 }

# Every 5 seconds report our drop locations
probe timer.sec(5)
{
  printf("\n")
  foreach (l in locations-) {
    printf("%d packets dropped at %s\n",
           @count(locations[l]), symname(l))
  }
  delete locations
}
```

`kernel.trace("kfree_skb")`跟踪内核中网络包被丢弃的位置。它有两个参数：一个指向将被释放的缓冲区的指针`$skb`，和释放缓冲区时的内核位置`$location`。如果可以获取`$location`所存储的内核地址上对应的函数名，`dropwatch.stp`脚本可以把它的值映射成对应的函数。这个映射默认不会启用。对于1.4及以上的SystemTap，你可以指定`--all-modules`选项来启用该映射：
```
stap --all-modules dropwatch.stp
```

在低版本的SystemTap，你可以使用下面的命令模拟`--all-modules`选项：
```
stap -dkernel \
`cat /proc/modules | awk 'BEGIN { ORS = " " } {print "-d"$1}'` \
dropwatch.stp
```

运行`dropwatch.stp`15秒会输出类似下面的结果。输出的结果会按函数名或地址聚合丢包的次数。
```
Monitoring for dropped packets

1762 packets dropped at unix_stream_recvmsg
4 packets dropped at tun_do_read
2 packets dropped at nf_hook_slow

467 packets dropped at unix_stream_recvmsg
20 packets dropped at nf_hook_slow
6 packets dropped at tun_do_read

446 packets dropped at unix_stream_recvmsg
4 packets dropped at tun_do_read
4 packets dropped at nf_hook_slow
Stopping dropped packet monitor
```

当运行脚本的机器不支持`--all-modules`和`/proc/modules`时，`symname`只会输出原始的地址。你可以通过`/boot/System.map-$(uname -r)`按地址找出对应的函数。下面的`/boot/System.map-$(uname -r)`片段中，地址`0xffffffff8149a8ed`映射到函数`unix_stream_recvmsg`：
```
[...]
ffffffff8149a420 t unix_dgram_poll
ffffffff8149a5e0 t unix_stream_recvmsg
ffffffff8149ad00 t unix_find_other
[...]
```
