# 5.4. 标识用户空间锁竞争

本节展示如何显示特定时间内用户空间锁竞争的情况。通过展示锁竞争的图景，你可以判断当前的性能问题是否由对`futex`的竞争所造成的。
简单地说，如果在同一时间内多个进程试图获取同一把锁，就会产生对`futex`的竞争。由于仅有一个进程可以持有锁，其他的进程都只能等待锁重新可用，锁竞争会导致性能的下降。
下面的`futexes.stp`脚本通过探测`futex`系统调用来显示锁竞争的情况：

futexes.stp
```
#! /usr/bin/env stap

# This script tries to identify contended user-space locks by hooking
# into the futex system call.

global FUTEX_WAIT = 0 /*, FUTEX_WAKE = 1 */
global FUTEX_PRIVATE_FLAG = 128 /* linux 2.6.22+ */
global FUTEX_CLOCK_REALTIME = 256 /* linux 2.6.29+ */

global lock_waits # long-lived stats on (tid,lock) blockage elapsed time
global process_names # long-lived pid-to-execname mapping

probe syscall.futex.return {  
  if (($op & ~(FUTEX_PRIVATE_FLAG|FUTEX_CLOCK_REALTIME)) != FUTEX_WAIT) next
  process_names[pid()] = execname()
  elapsed = gettimeofday_us() - @entry(gettimeofday_us())
  lock_waits[pid(), $uaddr] <<< elapsed
}

probe end {
  foreach ([pid+, lock] in lock_waits) 
    printf ("%s[%d] lock %p contended %d times, %d avg us\n",
            process_names[pid], pid, lock, @count(lock_waits[pid,lock]),
            @avg(lock_waits[pid,lock]))
}
```

`futexes.stp`需要手动Ctrl+C退出。一旦退出后，它会输出下面信息：
* 参与锁竞争的进程的名字和ID
* 被竞争的锁变量的地址
* 锁被竞争的次数
* 竞争锁的平均耗时

⁠下面是`futexes.stp`在运行约20秒 退出时，大致的输出情况：
```
[...]
automount[2825] lock 0x00bc7784 contended 18 times, 999931 avg us
synergyc[3686] lock 0x0861e96c contended 192 times, 101991 avg us
synergyc[3758] lock 0x08d98744 contended 192 times, 101990 avg us
synergyc[3938] lock 0x0982a8b4 contended 192 times, 101997 avg us
[...]
```
