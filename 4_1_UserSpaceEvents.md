# 4.1. 用户空间事件

所有的用户空间事件都以`process`开头。你可以通过进程ID指定要检测的进程，也可以通过可执行文件名的路径名指定。SystemTap会查看系统的`PATH`环境变量，所以你既可以使用绝对路径，也可以使用在命令行中运行可执行文件时所用的名字。

由于SystemTap静态分析放置探针的位置时离不开调试信息，一些用户空间事件需要给定PID或可执行文件的路径（以下将两者统称为`PATH`）。不过大多数`process`事件中，PID和可执行文件路径名都是可选的。下面列出的事件都需要进程ID或可执行文件的路径。不在其中的`process`事件不需要PID和可执行文件路径名。

**process("PATH").function("function")**

进入可执行文件`PATH`的用户空间函数`function`。该事件相当于内核空间中的`kernel.function("function")`。它允许使用通配符和`.return`后缀。

**process("PATH").statement("statement")**

代码中第一次执行`statement`的地方。该事件相当于内核空间中的`kernel.statement("statement")`。

**process("PATH").mark("marker")**

在`PATH`中定义的静态探测点。你可以使用通配符，在单个探针中指定多个探测点。有些静态探测点中 允许使用编了号（numbered）的参数（`$1`，`$2`等等）。
有些用户空间下的可执行程序提供了这些静态探测点，比如Java。大多数提供了静态探测点的程序也一并给这些探测点提供了易于使用的别名。下面是x86_64 Java hotspot虚拟机中的一个例子：
```
probe hotspot.gc_begin =
  process("/usr/lib/jvm/java-1.6.0-openjdk-1.6.0.0.x86_64/jre/lib/amd64/server/libjvm.so").mark("gc__begin")
```

**process("PATH").begin**

创建了一个用户空间下的进程。你可以限定某个进程ID或可执行文件的路径，如果不限定，任意进程的创建都会触发该事件。

**process("PATH").thread.begin**

创建了一个用户空间下的线程。你可以限定某个进程ID或可执行文件的路径。

**process("PATH").end**

销毁了一个用户空间下的进程。你可以限定某个进程ID或可执行文件的路径。

**process("PATH").thread.end**

销毁了一个用户空间下的线程。你可以限定某个进程ID或可执行文件的路径。

**process("PATH").syscall**

一个用户空间进程调用了系统调用。可以通过上下文变量`$syscall`获取系统调用号。还可以通过`$arg1`到`$arg6`分别获取前六个参数。添加`return`后缀后会捕获退出系统调用的事件。在`syscall.return`中，可以通过上下文变量`$return`获取返回值。
你可以用某个进程ID或可执行文件的路径进行限定。
