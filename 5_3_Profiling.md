# 5.3. 剖析

以下各节的脚本展示了如何通过监控函数调用来剖析（profile）内核活动。

## 统计函数调用次数

本节展示如何统计30秒内某个内核函数调用次数。通过使用通配符，你可以用这个脚本同时统计多个内核函数。

functioncallcount.stp
```
#! /usr/bin/env stap
# The following line command will probe all the functions
# in kernel's memory management code:
#
# stap  functioncallcount.stp "*@mm/*.c"

probe kernel.function(@1).call {  # probe functions listed on commandline
  called[ppfunc()] <<< 1  # add a count efficiently
}

global called

probe end {
  foreach (fn in called-)  # Sort by call count (in decreasing order)
  #       (fn+ in called)  # Sort by function name
    printf("%s %d\n", fn, @count(called[fn]))
  exit()
}
```

`functioncallcount.stp`接受内核函数名作为参数。你可以使用通配符，这样就能同时监控多个内核函数。
它的输出包括调用者的名字和取样时间内调用次数。下面是`stap functioncallcount.stp "*@mm/*.c"`的输出片段：

```
[...]
__vma_link 97
__vma_link_file 66
__vma_link_list 97
__vma_link_rb 97
__xchg 103
add_page_to_active_list 102
add_page_to_inactive_list 19
add_to_page_cache 19
add_to_page_cache_lru 7
all_vm_events 6
alloc_pages_node 4630
alloc_slabmgmt 67
anon_vma_alloc 62
anon_vma_free 62
anon_vma_lock 66
anon_vma_prepare 98
anon_vma_unlink 97
anon_vma_unlock 66
arch_get_unmapped_area_topdown 94
arch_get_unmapped_exec_area 3
arch_unmap_area_topdown 97
atomic_add 2
atomic_add_negative 97
atomic_dec_and_test 5153
atomic_inc 470
atomic_inc_and_test 1
[...]
```

## 追踪函数调用链

本节展示如何追踪函数调用链。

para-callgraph.stp
```
#! /usr/bin/env stap

function trace(entry_p, extra) {
  %( $# > 1 %? if (tid() in trace) %)
  printf("%s%s%s %s\n",
         thread_indent (entry_p),
         (entry_p>0?"->":"<-"),
         ppfunc (),
         extra)
}


%( $# > 1 %?
global trace
probe $2.call {
  trace[tid()] = 1
}
probe $2.return {
  delete trace[tid()]
}
%)

probe $1.call   { trace(1, $$parms) }
probe $1.return { trace(-1, $$return) }
```

`para-callgraph.stp`接受两个命令行参数：
1. 想要跟踪的函数（`$1`）
2. 可选的触发函数。该函数可以在线程范围内启动/停止追踪。只要触发函数不退出，追踪就不会结束。
`para-callgraph.stp`使用了`thread_indent()`；此外它的输出包括了时间戳、进程名，和`$1`所在的线程ID。关于`thread_indent()`的更多信息，请参考[。。。](3.2节)。
（译注：这个脚本的编码风格小朋友们可不要学。前两个探针里的`trace`是数组，后两个探针里的`trace`是函数。另外`$#`表示参数的个数，写过shell的都明白。`%( $# > 1 %? if (tid() in trace) %)`是一个预处理三元表达式，见[langref](https://sourceware.org/systemtap/langref/Language_elements.html)中的“5.8.1 Conditions”）

下面是`stap para-callgraph.stp 'kernel.function("*@fs/*.c")' 'kernel.function("sys_read")'`的输出片段：
```
[...]
   267 gnome-terminal(2921): <-do_sync_read return=0xfffffffffffffff5
   269 gnome-terminal(2921):<-vfs_read return=0xfffffffffffffff5
     0 gnome-terminal(2921):->fput file=0xffff880111eebbc0
     2 gnome-terminal(2921):<-fput
     0 gnome-terminal(2921):->fget_light fd=0x3 fput_needed=0xffff88010544df54
     3 gnome-terminal(2921):<-fget_light return=0xffff8801116ce980
     0 gnome-terminal(2921):->vfs_read file=0xffff8801116ce980 buf=0xc86504 count=0x1000 pos=0xffff88010544df48
     4 gnome-terminal(2921): ->rw_verify_area read_write=0x0 file=0xffff8801116ce980 ppos=0xffff88010544df48 count=0x1000
     7 gnome-terminal(2921): <-rw_verify_area return=0x1000
    12 gnome-terminal(2921): ->do_sync_read filp=0xffff8801116ce980 buf=0xc86504 len=0x1000 ppos=0xffff88010544df48
    15 gnome-terminal(2921): <-do_sync_read return=0xfffffffffffffff5
    18 gnome-terminal(2921):<-vfs_read return=0xfffffffffffffff5
     0 gnome-terminal(2921):->fput file=0xffff8801116ce980
```


## 统计给定线程在内核空间和用户空间上的耗时

本节展示如何统计给定线程花费在内核空间或用户空间上的运行时间。

thread-times.stp
```
#! /usr/bin/env stap

probe perf.sw.cpu_clock!, timer.profile {
  // NB: To avoid contention on SMP machines, no global scalars/arrays used,
  // only contention-free statistics aggregates.
  tid=tid(); e=execname()
  if (!user_mode())
    kticks[e,tid] <<< 1
  else
    uticks[e,tid] <<< 1
  ticks <<< 1
  tids[e,tid] <<< 1
}

global uticks%, kticks%, ticks

global tids%

probe timer.s(5), end {
  allticks = @count(ticks)
  printf ("%16s %5s %7s %7s (of %d ticks)\n",
          "comm", "tid", "%user", "%kernel", allticks)
  foreach ([e,tid] in tids- limit 20) {
    uscaled = @count(uticks[e,tid])*10000/allticks
    kscaled = @count(kticks[e,tid])*10000/allticks
    printf ("%16s %5d %3d.%02d%% %3d.%02d%%\n",
      e, tid, uscaled/100, uscaled%100, kscaled/100, kscaled%100)
  }
  printf("\n")

  delete uticks
  delete kticks
  delete ticks
  delete tids
}
```

`thread-time.stp`列出5秒内花费CPU时间最多的20个进程，和这段时间CPU滴答（ticks）的总数。脚本的输出还包括每个进程CPU占用百分比，分别按内核空间和用户空间列出。
下面就是它的输出:

```
  tid   %user %kernel (of 20002 ticks)
    0   0.00%  87.88%
32169   5.24%   0.03%
 9815   3.33%   0.36%
 9859   0.95%   0.00%
 3611   0.56%   0.12%
 9861   0.62%   0.01%
11106   0.37%   0.02%
32167   0.08%   0.08%
 3897   0.01%   0.08%
 3800   0.03%   0.00%
 2886   0.02%   0.00%
 3243   0.00%   0.01%
 3862   0.01%   0.00%
 3782   0.00%   0.00%
21767   0.00%   0.00%
 2522   0.00%   0.00%
 3883   0.00%   0.00%
 3775   0.00%   0.00%
 3943   0.00%   0.00%
 3873   0.00%   0.00%
 ```
 
## 监控应用轮询情况

本节展示如何监控应用的轮询（polling）情况。这将允许你跟踪多余的或过度的轮询，帮助锁定CPU使用或能源消耗需要改善的地方。

timeout.stp
```
#! /usr/bin/env stap
# Copyright (C) 2009 Red Hat, Inc.
# Written by Ulrich Drepper <drepper@redhat.com>
# Modified by William Cohen <wcohen@redhat.com>

global process, timeout_count, to
global poll_timeout, epoll_timeout, select_timeout, itimer_timeout
global nanosleep_timeout, futex_timeout, signal_timeout

probe syscall.poll, syscall.epoll_wait {
  if (timeout) to[pid()]=timeout
}

probe syscall.poll.return {
  if ($return == 0 && to[pid()] > 0 ) {
    poll_timeout[pid()]++
    timeout_count[pid()]++
    process[pid()] = execname()
    delete to[pid()]
  }
}

probe syscall.epoll_wait.return {
  if ($return == 0 && to[pid()] > 0 ) {
    epoll_timeout[pid()]++
    timeout_count[pid()]++
    process[p] = execname()
    delete to[pid()]
  }
}

probe syscall.select.return {
  if ($return == 0) {
    select_timeout[pid()]++
    timeout_count[pid()]++
    process[pid()] = execname()
  }
}

probe syscall.futex.return {
  if (errno_str($return) == "ETIMEDOUT") {
    futex_timeout[pid()]++
    timeout_count[pid()]++
    process[pid()] = execname()
  }
}

probe syscall.nanosleep.return {
  if ($return == 0) {
    nanosleep_timeout[pid()]++
    timeout_count[pid()]++
    process[pid()] = execname()
  }
}

probe kernel.function("it_real_fn") {
  itimer_timeout[pid()]++
  timeout_count[pid()]++
  process[pid()] = execname()
}

probe syscall.rt_sigtimedwait.return {
  if (errno_str($return) == "EAGAIN") {
    signal_timeout[pid()]++
    timeout_count[pid()]++
    process[pid()] = execname()
  }
}

probe syscall.exit {
  if (pid() in process) {
    delete process[pid()]
    delete timeout_count[pid()]
    delete poll_timeout[pid()]
    delete epoll_timeout[pid()]
    delete select_timeout[pid()]
    delete itimer_timeout[pid()]
    delete futex_timeout[pid()]
    delete nanosleep_timeout[pid()]
    delete signal_timeout[pid()]
  }
}

probe timer.s(1) {
  ansi_clear_screen()
  printf ("  pid |   poll  select   epoll  itimer   futex nanosle  signal| process\n")
  foreach (p in timeout_count- limit 20) {
     printf ("%5d |%7d %7d %7d %7d %7d %7d %7d| %-.38s\n", p,
              poll_timeout[p], select_timeout[p],
              epoll_timeout[p], itimer_timeout[p],
              futex_timeout[p], nanosleep_timeout[p],
              signal_timeout[p], process[p])
  }
}
```

`timeout.stp`跟踪下列系统调用，并仅当因为超时而退出该调用时记录次数：
* poll
* select
* epoll
* itimer
* futex
* nanosleep
* signal

```
  uid |   poll  select   epoll  itimer   futex nanosle  signal| process
28937 | 148793       0       0    4727   37288       0       0| firefox
22945 |      0   56949       0       1       0       0       0| scim-bridge
    0 |      0       0       0   36414       0       0       0| swapper
 4275 |  23140       0       0       1       0       0       0| mixer_applet2
 4191 |      0   14405       0       0       0       0       0| scim-launcher
22941 |   7908       1       0      62       0       0       0| gnome-terminal
 4261 |      0       0       0       2       0    7622       0| escd
 3695 |      0       0       0       0       0    7622       0| gdm-binary
 3483 |      0    7206       0       0       0       0       0| dhcdbd
 4189 |   6916       0       0       2       0       0       0| scim-panel-gtk
 1863 |   5767       0       0       0       0       0       0| iscsid
 2562 |      0    2881       0       1       0    1438       0| pcscd
 4257 |   4255       0       0       1       0       0       0| gnome-power-man
 4278 |   3876       0       0      60       0       0       0| multiload-apple
 4083 |      0    1331       0    1728       0       0       0| Xorg
 3921 |   1603       0       0       0       0       0       0| gam_server
 4248 |   1591       0       0       0       0       0       0| nm-applet
 3165 |      0    1441       0       0       0       0       0| xterm
29548 |      0    1440       0       0       0       0       0| httpd
 1862 |      0       0       0       0       0    1438       0| iscsid
```

你可以通过修改最后一个探针（`timer.s(1)`）来增大取样时间。`timeout.stp`的输出包括前20个轮询应用的名字和UID，连带每个应用调用每种轮询系统调用的累计次数。在上面的输出片段中，由于某个插件模块，firefox进行了过度的轮询。

## 监控最常调用的系统调用

上一节的`timeout.stp`通过监控系统调用的某个子集，帮助你找到过度轮询的应用。同样，如果你怀疑某些系统调用被过度地调用了，通过类似的监控，你也能把它们找出来。下面就通过`topsys.stp`实现这一点：

topsys.stp
```
#! /usr/bin/env stap
#
# This script continuously lists the top 20 systemcalls in the interval 
# 5 seconds
#

global syscalls_count

probe syscall.* {
  syscalls_count[name] <<< 1
}

function print_systop () {
  printf ("%25s %10s\n", "SYSCALL", "COUNT")
  foreach (syscall in syscalls_count- limit 20) {
    printf("%25s %10d\n", syscall, @count(syscalls_count[syscall]))
  }
  delete syscalls_count
}

probe timer.s(5) {
  print_systop ()
  printf("--------------------------------------------------------------\n")
}
```

`topsys.stp`每5秒列出调用最多的20个系统调用。它也列出了这段时间内每个系统调用被调用的次数。下面是它的一个输出：

```
--------------------------------------------------------------
                  SYSCALL      COUNT
             gettimeofday       1857
                     read       1821
                    ioctl       1568
                     poll       1033
                    close        638
                     open        503
                   select        455
                    write        391
                   writev        335
                    futex        303
                  recvmsg        251
                   socket        137
            clock_gettime        124
           rt_sigprocmask        121
                   sendto        120
                setitimer        106
                     stat         90
                     time         81
                sigreturn         72
                    fstat         66
--------------------------------------------------------------
```

## 找出每个进程的系统调用量

本节展示如何找出调用系统调用最多的进程。在上一节，我们谈到了如何找出调用最多的系统调用。而在上上节，我们也谈到了如何找出轮询最多的进程。通过监控每个进程的调用系统调用的次数，可以在调查轮询进程和其他滥用资源者时提供更多的数据。

`syscalls_by_proc.stp`
```
#! /usr/bin/env stap

# Copyright (C) 2006 IBM Corp.
#
# This file is part of systemtap, and is free software.  You can
# redistribute it and/or modify it under the terms of the GNU General
# Public License (GPL); either version 2, or (at your option) any
# later version.

#
# Print the system call count by process name in descending order.
#

global syscalls

probe begin {
  print ("Collecting data... Type Ctrl-C to exit and display results\n")
}

probe nd_syscall.* {
  syscalls[execname()]++
}

probe end {
  printf ("%-10s %-s\n", "#SysCalls", "Process Name")
  foreach (proc in syscalls-)
    printf("%-10d %-s\n", syscalls[proc], proc)
}
```
`syscalls_by_proc.stp`列出调用系统调用最多的20个进程。它也列出了这段时间内每个进程调用系统调用的数量。下面是它的输出：

```
Collecting data... Type Ctrl-C to exit and display results
#SysCalls  Process Name
1577       multiload-apple
692        synergyc
408        pcscd
376        mixer_applet2
299        gnome-terminal
293        Xorg
206        scim-panel-gtk
95         gnome-power-man
90         artsd
85         dhcdbd
84         scim-bridge
78         gnome-screensav
66         scim-launcher
[...]
```

要想输出进程PID而非进程名，改用下面的脚本：

syscalls_by_pid.stp
```
#! /usr/bin/env stap

# Copyright (C) 2006 IBM Corp.
#
# This file is part of systemtap, and is free software.  You can
# redistribute it and/or modify it under the terms of the GNU General
# Public License (GPL); either version 2, or (at your option) any
# later version.

#
# Print the system call count by process ID in descending order.
#

global syscalls

probe begin {
  print ("Collecting data... Type Ctrl-C to exit and display results\n")
}

probe nd_syscall.* {
  syscalls[pid()]++
}

probe end {
  printf ("%-10s %-s\n", "#SysCalls", "PID")
  foreach (pid in syscalls-)
    printf("%-10d %-d\n", syscalls[pid], pid)
}
```

正如在输出中提到的，你需要手动Ctrl-C退出程序来显示结果。你可以简单地添加一个`timer.s()`探针，让脚本在给定时间后自动退出。举个例子，要想让脚本5秒后退出，往里面添加下面的探针：
```
probe timer.s(5)
{
    exit()
}
```
