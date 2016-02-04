# 5.2. 磁盘

以下各节的脚本展示了如何监控磁盘和I/O活动。

## 统计磁盘读写状况

本节展示了如何找出磁盘读写最频繁的进程。

disktop.stp
```
#!/usr/bin/env stap
#
# Copyright (C) 2007 Oracle Corp.
#
# Get the status of reading/writing disk every 5 seconds,
# output top ten entries
#
# This is free software,GNU General Public License (GPL);
# either version 2, or (at your option) any later version.
#
# Usage:
#  ./disktop.stp
#

global io_stat,device
global read_bytes,write_bytes

probe vfs.read.return {
  if ($return>0) {
    if (devname!="N/A") {/*skip read from cache*/
      io_stat[pid(),execname(),uid(),ppid(),"R"] += $return
      device[pid(),execname(),uid(),ppid(),"R"] = devname
      read_bytes += $return
    }
  }
}

probe vfs.write.return {
  if ($return>0) {
    if (devname!="N/A") { /*skip update cache*/
      io_stat[pid(),execname(),uid(),ppid(),"W"] += $return
      device[pid(),execname(),uid(),ppid(),"W"] = devname
      write_bytes += $return
    }
  }
}

probe timer.ms(5000) {
  /* skip non-read/write disk */
  if (read_bytes+write_bytes) {

    printf("\n%-25s, %-8s%4dKb/sec, %-7s%6dKb, %-7s%6dKb\n\n",
           ctime(gettimeofday_s()),
           "Average:", ((read_bytes+write_bytes)/1024)/5,
           "Read:",read_bytes/1024,
           "Write:",write_bytes/1024)

    /* print header */
    printf("%8s %8s %8s %25s %8s %4s %12s\n",
           "UID","PID","PPID","CMD","DEVICE","T","BYTES")
  }
  /* print top ten I/O */
  foreach ([process,cmd,userid,parent,action] in io_stat- limit 10)
    printf("%8d %8d %8d %25s %8s %4s %12d\n",
           userid,process,parent,cmd,
           device[process,cmd,userid,parent,action],
           action,io_stat[process,cmd,userid,parent,action])

  /* clear data */
  delete io_stat
  delete device
  read_bytes = 0
  write_bytes = 0
}

probe end{
  delete io_stat
  delete device
  delete read_bytes
  delete write_bytes
}
```

`disktop.stp`输出磁盘读写最频繁的十个进程，包含各个进程的以下数据：
* UID - 进程所有者的UID
* PID - 进程的PID
* PPID - 进程的父进程的PID
* CMD - 进程的名字
* DEVICE - 读/写的设备名
* T - 进程的操作；`W`是写，而`R`是读。
* BYTES - 读/写的数据量

`disktop.stp`使用`ctime()`和`gettimeofday_s()`输出当前时间。`gettimeofday_s`返回当前时间自epoch（1970年1月1日）以来的秒数，`ctime`把它转化成可读的时间戳。
在这个脚本中，`$return`是一个存储着虚拟文件系统读写的字节数的本地变量。`$return`只能在函数返回事件探针中使用（比如这里的`vfs.read.return`和`vfs.write.return`）。

以下是本节脚本的输出：
```
[...]
Mon Sep 29 03:38:28 2008 , Average:  19Kb/sec, Read: 7Kb, Write: 89Kb

UID      PID     PPID                       CMD   DEVICE    T    BYTES
0    26319    26294                   firefox     sda5    W        90229
0     2758     2757           pam_timestamp_c     sda5    R         8064
0     2885        1                     cupsd     sda5    W         1678

Mon Sep 29 03:38:38 2008 , Average:   1Kb/sec, Read: 7Kb, Write: 1Kb

UID      PID     PPID                       CMD   DEVICE    T    BYTES
0     2758     2757           pam_timestamp_c     sda5    R         8064
0     2885        1                     cupsd     sda5    W         1678
```

## 追踪对任意文件的读写

本节展示如何监控各进程读/写任意文件所花费的时间。这可以帮助你发现系统中加载时间过长的文件。

iotime.stp
```
#! /usr/bin/env stap

/*
 * Copyright (C) 2006-2007 Red Hat Inc.
 *
 * This copyrighted material is made available to anyone wishing to use,
 * modify, copy, or redistribute it subject to the terms and conditions
 * of the GNU General Public License v.2.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program.  If not, see <http://www.gnu.org/licenses/>.
 *
 * Print out the amount of time spent in the read and write systemcall
 * when each file opened by the process is closed. Note that the systemtap
 * script needs to be running before the open operations occur for
 * the script to record data.
 *
 * This script could be used to to find out which files are slow to load
 * on a machine. e.g.
 *
 * stap iotime.stp -c 'firefox'
 *
 * Output format is:
 * timestamp pid (executabable) info_type path ...
 *
 * 200283135 2573 (cupsd) access /etc/printcap read: 0 write: 7063
 * 200283143 2573 (cupsd) iotime /etc/printcap time: 69
 *
 */

global start
global time_io

function timestamp:long() { return gettimeofday_us() - start }

function proc:string() { return sprintf("%d (%s)", pid(), execname()) }

probe begin { start = gettimeofday_us() }

global filehandles, fileread, filewrite

probe syscall.open.return {
  filename = user_string($filename)
  if ($return != -1) {
    filehandles[pid(), $return] = filename
  } else {
    printf("%d %s access %s fail\n", timestamp(), proc(), filename)
  }
}

probe syscall.read.return {
  p = pid()
  fd = $fd
  bytes = $return
  time = gettimeofday_us() - @entry(gettimeofday_us())
  if (bytes > 0)
    fileread[p, fd] += bytes
  time_io[p, fd] <<< time
}

probe syscall.write.return {
  p = pid()
  fd = $fd
  bytes = $return
  time = gettimeofday_us() - @entry(gettimeofday_us())
  if (bytes > 0)
    filewrite[p, fd] += bytes
  time_io[p, fd] <<< time
}

probe syscall.close {
  if ([pid(), $fd] in filehandles) {
    printf("%d %s access %s read: %d write: %d\n",
           timestamp(), proc(), filehandles[pid(), $fd],
           fileread[pid(), $fd], filewrite[pid(), $fd])
    if (@count(time_io[pid(), $fd]))
      printf("%d %s iotime %s time: %d\n",  timestamp(), proc(),
             filehandles[pid(), $fd], @sum(time_io[pid(), $fd]))
   }
  delete fileread[pid(), $fd]
  delete filewrite[pid(), $fd]
  delete filehandles[pid(), $fd]
  delete time_io[pid(),$fd]
}
```

`iotime.stp`跟踪每次`open`、`close`、`read`和`write`系统调用。对于访问到的每个文件，`iotime.stp`都会计算读写操作花费的时间和读写的数据量（以字节为单位）。
虽然我们可以在读写事件（`syscall.read`和`syscall.write`）中使用本地变量`$count`，但是`$count`存储的是系统调用想要读写的数据量，要获取实际读写到的数据量需要使用`$return`。

```
[...]
825946 3364 (NetworkManager) access /sys/class/net/eth0/carrier read: 8190 write: 0
825955 3364 (NetworkManager) iotime /sys/class/net/eth0/carrier time: 9
[...]
117061 2460 (pcscd) access /dev/bus/usb/003/001 read: 43 write: 0
117065 2460 (pcscd) iotime /dev/bus/usb/003/001 time: 7
[...]
3973737 2886 (sendmail) access /proc/loadavg read: 4096 write: 0
3973744 2886 (sendmail) iotime /proc/loadavg time: 11
[...]
```

本节的脚本会输出以下数据：
* 时间戳，精确到毫秒
* PID和进程名
* access或iotime
* 访问的文件

如果一个进程读写了数据，你会看到`access`和`iotime`成对出现。`access`那一行的时间戳表示进程访问了文件；在结尾处会输出读写的数据（以字节为单位）。`iotime`那一行会输出读写消耗的时间（以毫秒为单位）。如果一行`access`后面没有`iotime`，意味着进程没有读写到数据。


## 追踪I/O的累计总量

本节展示如何累计I/O总量。

traceio.stp
```
#! /usr/bin/env stap
# traceio.stp
# Copyright (C) 2007 Red Hat, Inc., Eugene Teo <eteo@redhat.com>
# Copyright (C) 2009 Kai Meyer <kai@unixlords.com>
#   Fixed a bug that allows this to run longer
#   And added the humanreadable function
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License version 2 as
# published by the Free Software Foundation.
#

global reads, writes, total_io

probe vfs.read.return {
  if ($return > 0) {
    reads[pid(),execname()] += $return
    total_io[pid(),execname()] += $return
  }
}

probe vfs.write.return {
  if ($return > 0) {
    writes[pid(),execname()] += $return
    total_io[pid(),execname()] += $return
  }
}

function humanreadable(bytes) {
  if (bytes > 1024*1024*1024) {
    return sprintf("%d GiB", bytes/1024/1024/1024)
  } else if (bytes > 1024*1024) {
    return sprintf("%d MiB", bytes/1024/1024)
  } else if (bytes > 1024) {
    return sprintf("%d KiB", bytes/1024)
  } else {
    return sprintf("%d   B", bytes)
  }
}

probe timer.s(1) {
  foreach([p,e] in total_io- limit 10)
    printf("%8d %15s r: %12s w: %12s\n",
           p, e, humanreadable(reads[p,e]),
           humanreadable(writes[p,e]))
  printf("\n")
  # Note we don't zero out reads, writes and total_io,
  # so the values are cumulative since the script started.
}
```

`traceio.stp`逐秒输出累计I/O最频繁的前十个进程。此外，它还会累计每个进程的I/O情况。注意该脚本跟开头找出磁盘读写最频繁的进程的脚本一样，也通过本地变量`$return`获取实际的读写数据量

```
[...]
           Xorg r:   583401 KiB w:        0 KiB
       floaters r:       96 KiB w:     7130 KiB
multiload-apple r:      538 KiB w:      537 KiB
           sshd r:       71 KiB w:       72 KiB
pam_timestamp_c r:      138 KiB w:        0 KiB
        staprun r:       51 KiB w:       51 KiB
          snmpd r:       46 KiB w:        0 KiB
          pcscd r:       28 KiB w:        0 KiB
     irqbalance r:       27 KiB w:        4 KiB
          cupsd r:        4 KiB w:       18 KiB

           Xorg r:   588140 KiB w:        0 KiB
       floaters r:       97 KiB w:     7143 KiB
multiload-apple r:      543 KiB w:      542 KiB
           sshd r:       72 KiB w:       72 KiB
pam_timestamp_c r:      138 KiB w:        0 KiB
        staprun r:       51 KiB w:       51 KiB
          snmpd r:       46 KiB w:        0 KiB
          pcscd r:       28 KiB w:        0 KiB
     irqbalance r:       27 KiB w:        4 KiB
          cupsd r:        4 KiB w:       18 KiB
```

## 监控指定设备的I/O

本节展示如何监控指定设备的I/O活动。

traceio2.stp
```
#! /usr/bin/env stap

global device_of_interest

probe begin {
  /* The following is not the most efficient way to do this.
      One could directly put the result of usrdev2kerndev()
      into device_of_interest.  However, want to test out
      the other device functions */
  dev = usrdev2kerndev($1)
  device_of_interest = MKDEV(MAJOR(dev), MINOR(dev))
}

probe vfs.write, vfs.read
{
  if (dev == device_of_interest)
    printf ("%s(%d) %s 0x%x\n",
            execname(), pid(), ppfunc(), dev)
}
```

`traceio2.stp`接受一个参数：设备号，要想获取名为`directory`的文件夹所在设备的设备号，使用`stat -c "0x%D" directory`。
`usrdev2kerndev()`把设备号转化成内核理解的格式。`usrdev2kerndev()`的输出经过`MAJOR()`和`MINOR()`处理，分别得到主设备号和次设备号，再经过`MKDEV()`处理，得到内核里对应的设备号。
`traceio2.stp`的输出包括了读/写进程的名字和PID，所调用的函数（`vfs_read`或`vfs_write`）和内核里对应的设备号。

下面是`stap traceio2.stp 0x805`的输出，其中`0x805`是`/home`的设备号。`/home`位于`/dev/sda5`，正是我们想要监控的设备。
```
[...]
synergyc(3722) vfs_read 0x800005
synergyc(3722) vfs_read 0x800005
cupsd(2889) vfs_write 0x800005
cupsd(2889) vfs_write 0x800005
cupsd(2889) vfs_write 0x800005
[...]
```

## 监控对指定文件的读写

本节展示如何实时监控对指定文件的读写。

inodewatch.stp
```
#! /usr/bin/env stap

probe vfs.write, vfs.read
{
  # dev and ino are defined by vfs.write and vfs.read
  if (dev == MKDEV($1,$2) # major/minor device
      && ino == $3)
    printf ("%s(%d) %s 0x%x/%u\n",
      execname(), pid(), ppfunc(), dev, ino)
}
```

`inodewatch.stp`从命令行中依次获取如下参数：
1. 文件的主设备号
2. 文件的次设备号
3. 文件的inode号

要获取上述信息，使用`stat -c '%D %i' filename`，注意`filename`取绝对路径。
比如：要监控`/etc/crontab`，先运行`stat -c '%D %i' /etc/crontab`。应该会有如下输出：
```
805 1078319
```
805是十六进制的设备号。最小的两位是次设备号，其余是主设备号。1078319是inode号。要监控`/etc/crontab`，运行`stap inodewatch.stp 0x8 0x05 1078319`.（加`0x`以表示这是十六进制的数）

该命令的输出包括进程名和进程PID，以及调用的函数（`vfs_read`或`vfs_write`），设备号（以十六进制的格式输出）和inode号。下面就是`stap inodewatch.stp 0x8 0x05 1078319`的输出（当脚本运行时，`/etc/crontab`也正在执行中）：
```
cat(16437) vfs_read 0x800005/1078319
cat(16437) vfs_read 0x800005/1078319
```

## 监控对指定文件的属性的修改

本节展示如何实时监控对指定文件的属性的修改。

inodewatch2.stp
```
#! /usr/bin/env stap

global ATTR_MODE = 1

probe kernel.function("setattr_copy")!,
      kernel.function("generic_setattr")!,
      kernel.function("inode_setattr") {
  dev_nr = $inode->i_sb->s_dev
  inode_nr = $inode->i_ino

  if (dev_nr == MKDEV($1,$2) # major/minor device
      && inode_nr == $3
      && $attr->ia_valid & ATTR_MODE)
    printf ("%s(%d) %s 0x%x/%u %o %d\n",
      execname(), pid(), ppfunc(), dev_nr, inode_nr, $attr->ia_mode, uid())
}
```
跟上一节的脚本类似，`inodewatch2.stp`也需要提供目标文件的设备号和inode号作为参数。用上一节的方法可以获取这些数据。
`inodewatch.stp`的输出也类似于上一节的脚本，不过`inodewatch.stp`还包括文件属性的变化，和对应用户的UID。下面就是监控`/home/joe/bigfile`时，用户job执行`chmod 777 /home/joe/bigfile`和`chmod 666 /home/joe/bigfile`后的输出：

```
chmod(17448) inode_setattr 0x800005/6011835 100777 500
chmod(17449) inode_setattr 0x800005/6011835 100666 500
```

## 定期输出块I/O等待时间

本节展示如何跟踪每个块I/O的等待时间。这可以帮助你发现给定时间内块I/O操作是否排起了长队。

ioblktime.stp
```
#! /usr/bin/env stap

global req_time%[25000], etimes

probe ioblock.request
{
  req_time[$bio] = gettimeofday_us()
}

probe ioblock.end
{
  t = gettimeofday_us()
  s =  req_time[$bio]
  delete req_time[$bio]
  if (s) {
    etimes[devname, bio_rw_str(rw)] <<< t - s
  }
}

/* for time being delete things that get merged with others */
probe kernel.trace("block_bio_frontmerge"),
      kernel.trace("block_bio_backmerge")
{
  delete req_time[$bio]
}

probe timer.s(10), end {
  ansi_clear_screen()
  printf("%10s %3s %10s %10s %10s\n",
         "device", "rw", "total (us)", "count", "avg (us)")
  foreach ([dev,rw] in etimes - limit 20) {
    printf("%10s %3s %10d %10d %10d\n", dev, rw,
           @sum(etimes[dev,rw]), @count(etimes[dev,rw]), @avg(etimes[dev,rw]))
  }
  delete etimes
}
```
`ioblktime.stp`计算每个设备上块I/O平均等待时间，每10秒更新一次。你可以修改`probe timer.s(10), end {`来更改刷新频率。
有时候，在设备上的块I/O操作实在太多，以致于超过了默认的`MAXMAPENTRIES`值。如果你在定义数组时没有指定大小，SystemTap会以`MAXMAPENTRIES`作为数组的最大长度。它的默认值是2048,不过你可以使用stap命令的选项`-DMAXMAPENTRIES=10000`来指定该变量的值。

```
    device  rw total (us)      count   avg (us)
       sda   W       9659          6       1609
      dm-0   W      20278          6       3379
      dm-0   R      20524          5       4104
       sda   R      19277          5       3855
```

上面的输出展示了设备名，操作类型（`rw`），总等待时间（`total(us)`），操作数（`count`），和平均等待时间（`avg(us)`）。这里面的时间都是以毫秒为单位。
