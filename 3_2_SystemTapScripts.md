# 3.2. 脚本

在大多数情况下，SystemTap脚本是每个SystemTap会话的基石。SystemTap脚本决定了需要收集的信息类型，也决定了对收集到的信息的处理方式。

在本章的开头曾经提到过，SystemTap脚本由两部分组成：事件和处理程序。一旦SystemTap会话准备就绪，SystemTap会监控操作系统中特定的事件，并在事件发生的时候触发对应的处理程序。

> 一个事件和它对应的处理程序合称探针。一个SystemTap脚本可以有多个探针。
> 一个探针的处理程序部分通常称之为探针主体（probe body）

以应用开发的方式类比，使用事件和处理程序就像在程序的特定位置插入打日志的语句。每当程序运行时，这些日志会帮助你查看程序执行的流程。

SystemTasp脚本允许你在无需重新编译代码，即可插入检测指令，而且处理程序也不限于单纯地打印数据。事件会触发对应的处理程序；对应的处理程序记录下感兴趣的数据，并以你指定的格式输出。

SystemTap脚本的后缀是`.stp`，并以这样的语句表示一个探针：

```
probe   event {statements}
```

译注：如果你写过awk脚本，应该会感觉似曾相识。

SystemTap支持给一个探针指定多个事件；每个事件以逗号隔开。如果给某一个探针指定了多个事件，只要其中一个事件发生，SystemTap就会执行对应的处理程序。

每个探针有自己对应的语句块。语句块由花括号（`{}`）括住，包含事件发生时需要执行的所有语句。SystemTap会顺序执行这些语句；语句间通常不需要特殊的分隔符或终止符。

> SystemTap脚本的语句块使用跟C语言一样的语法。语句块内允许嵌套。

SystemTap允许你编写函数来提取探针间公共的逻辑。所以，与其在多个探针间复制粘贴重复的语句，你不如把它们放入函数中，就像：

```
function function_name(arguments) {statements}

probe event {function_name(arguments)}
```

当探针被触发时，`function_name`中的语句会被执行。`arguments`是传递给函数的可选的入参。

> 本节仅仅是粗略地介绍下SystemTap脚本的结构。要想了解更详细的内容，最好坚持读到第5章，SystemTap脚本集锦；其中的每一节都会详细介绍一个脚本，包含它所监控的事件、它的处理程序和输出内容。

## 事件

SystemTap事件大致分为两类：同步事件和异步事件。

### 同步事件

同步事件会在任意进程执行到内核特定位置时触发。你可以用它来作为其它事件的参照点，毕竟同步事件有着清晰的上下文信息。

同步事件包括：

**syscall.system_call**

进入名为`system_call`的系统调用。如果想要监控的是退出某个系统调用的事件，在后面添加`.return`。举个例子，要想监控进入和退出系统调用`close`的事件，应该使用`syscall.close`和`syscall.close.return`。

**vfs.file_operation**

进入虚拟文件系统（VFS）名为`file_operation`的文件操作。跟系统调用事件一样，在后面添加`.return`可以监控对应的退出事件。
译注：`file_operation`取值的范畴，取决于当前内核中`struct file_operations`的定义的操作（可能位于`include/linux/fs.h`中，版本不同位置会不一样，建议上http://lxr.free-electrons.com/ident 查找`file_operations`）。

**kernel.function("function")**

进入名为`function`的内核函数。举个例子，`kernel.function("sys_open")`即内核函数`sys_open`被调用时所触发的事件。同样，`kernel.function("sys_open").return`会在`sys_open`函数调用返回时被触发。

在定义探测事件时，可以使用像`*`这样的通配符。你也可以用内核源码文件名限定要跟踪的函数。看下面的例子：

>     probe kernel.function("*@net/socket.c") { }
>     probe kernel.function("*@net/socket.c").return { }

在上面的例子中，第一个探针会监控`net/socket.c`中的所有函数的调用。第二个会监控所有这些函数的退出。注意在这个例子里，处理程序是空的；所以，即使事件被触发了，什么也不会发生。
译注：例子中用的是探测内核源码中的函数的语法。完整的语法是`func_name@file_name[:line_num]`，由函数名、文件名、行号三部分组成。其中函数名在例子中为`*`，匹配任意函数。行号是可选的，在上面的例子里就被忽略掉了。如果想指定某个范围内的函数，如从行x到y，使用`:x-y`这样格式作为行号。

**kernel.trace("tracepoint")**

到达名为`tracepoint`的静态内核探测点（tracepoint）。较新的内核（>= 2.6.30）包含了特定事件的检测代码。这些事件一般会被标记成静态内核探测点。一个例子是，`kernel.trace("kfree_skb")`表示内核释放了一个网络缓冲区的事件。（译注：想知道当前内核设置了哪些静态内核探测点吗？你需要运行`sudo perf list`。）

**module("module").function("function")**

进入指定模块`module`的函数`function`。举个例子：

>     probe module("ext3").function("*") { }
>     probe module("ext3").function("*").return { }

上面例子的第一个探针，会在每个ext3模块中的函数被调用时触发。第二个探针会在函数退出时触发。一切就跟`kernel.function()`一样。

系统内的所有内核模块通常都在`/lib/modules/kernel_version`，其中`kernel_version`取当前内核版本号。模块的后缀名为`.ko`。
（译注：在该路径下使用`find -name '*.ko' -printf '%f\n' | sed 's/\.ko$//' `可列出所有的内核模块）

### 异步事件

异步事件跟特定的指令或代码的位置无关。
这部分事件主要包含计数器、定时器和其它类似的东西。

**begin**

SystemTap会话的启动事件，会在脚本开始时触发。

**end**

SystemTap会话的结束事件，会在脚本结束时触发。

**timer events**

用于周期性执行某段处理程序。举个例子：

>    probe timer.s(4)
>    {
>        printf("hello world\n")
>    }

上面的例子中，每隔4秒就会输出`hello world`。还可以使用其它规格的定时器：

```
timer.ms(milliseconds)
timer.us(microseconds)
timer.ns(nanoseconds)
timer.hz(hertz)
timer.jiffies(jiffies)
```

定时事件总是跟其它事件搭配使用。其它事件负责收集信息，而定时事件定期输出当前状况，让你看到数据随时间的变化情况。

> 限于篇幅，还有些SystemTap事件就不再一一介绍了。如果你想了解更多内容，请`man stapprobes`。该man page中的`SEE ALSO`一节，包括了通往其它man page的链接，你还可以随之找到某些特定子系统和组件所支持的事件。

## 处理程序

看一下下面的示例脚本：
```
probe begin
{
  printf ("hello world\n")
  exit ()
}
```

在上面的例子中，每当会话开始时，`begin`事件会触发`{}`内的处理程序，输出`hello world`加一个换行符，然后退出。

> SystemTap脚本会一直运行，直到执行了`exit()`函数。如果你想中途退出一个脚本，可以用`Ctrl+c`中断。 

**printf**

`printf()`是最简单的SystemTap函数之一，可以跟许多函数搭配使用，用来输出数据。通常我们会这样调用`printf()`：

    printf ("format string\n", arguments)
    
`format string`指明`arguments`输出的格式。在前面的例子里，printf语句内没有指定format格式符。在格式字符串（format string）中，你可以用`%s`表示字符串，`%d`表示数字。格式字符串中可以包含多个格式符，每个格式符对应一个参数；每个参数之间用逗号隔开。

> SystemTap的printf语句跟C的printf语句，无论在语法还是在格式字符串上都差不多。

下面让我们再看多一个例子：
```
probe syscall.open
{
  printf ("%s(%d) open\n", execname(), pid())
}
```

在上面的例子中，SystemTap会在每次`open`被调用时，输出调用程序的名字和PID，外加`open`这个词。该探针输出的结果看上去会是这样：
```
vmware-guestd(2206) open
hald(2360) open
hald(2360) open
hald(2360) open
df(3433) open
df(3433) open
df(3433) open
hald(2360) open
```

你可以在`printf()`里使用其他的SystemTap函数。比如上面的例子中就用到`execname()`（获取触发事件的进程名）和`pid()`（当前进程ID）。

下面列出常用的SystemTap函数：

**tid()**

当前的tid（thread id）。

**uid()**

当前的uid。

**cpu()**

当前的CPU号

**gettimeofday_s()**

自epoch以来的秒数

**ctime()**

将上一个函数返回的秒数转化成时间字符串

**pp()**

返回描述当前处理的探测点的字符串

**thread_indent()**

你可以用这个函数来组织你的输出结果。这个函数接受一个表示缩进差额的参数，用来更新当前线程的“缩进计数器”（其实就是用于缩进的空格数）。它返回的是加了足够缩进的标识字符串。
这个标识字符串包括一个时间戳（表示自从该线程首次调用`thread_indent()`以来所经过的毫秒数），一个进程名，一个tid。由此可以清晰地看出函数的调用次序和调用层级，和每次调用时的间隔。
如果一个函数调用后随即退出，很容易就能看出被触发的两个事件是相关的。然而，在大多数情况下，一个函数调用和退出之间，往往会有调用其他别的函数。通过缩进，可以相对更清晰地看出某个函数调用和退出的时机。

看一下下面使用`thread_indent()`的例子：
```
probe kernel.function("*@net/socket.c").call
{
  printf ("%s -> %s\n", thread_indent(1), probefunc())
}
probe kernel.function("*@net/socket.c").return
{
  printf ("%s <- %s\n", thread_indent(-1), probefunc())
}
```

它输出的结果大概是这个样子的，注意箭头前面的空格数:

```
0 ftp(7223): -> sys_socketcall
1159 ftp(7223):  -> sys_socket
2173 ftp(7223):   -> __sock_create
2286 ftp(7223):    -> sock_alloc_inode
2737 ftp(7223):    <- sock_alloc_inode
3349 ftp(7223):    -> sock_alloc
3389 ftp(7223):    <- sock_alloc
3417 ftp(7223):   <- __sock_create
4117 ftp(7223):   -> sock_create
4160 ftp(7223):   <- sock_create
4301 ftp(7223):   -> sock_map_fd
4644 ftp(7223):    -> sock_map_file
4699 ftp(7223):    <- sock_map_file
4715 ftp(7223):   <- sock_map_fd
4732 ftp(7223):  <- sys_socket
4775 ftp(7223): <- sys_socketcall
```

上面的输出包含如下信息：
* 自从该线程首次调用`thread_indent()`以来所经过的毫秒数。
* 进程名和PID。
* 用于缩进的若干个空格。以上三项均为`thread_indent()`的输出。
* `->`表示函数调用，`<-`表示函数退出。
* 触发事件的函数名。

**name**

返回系统调用的名字。这个变量只能在`syscall.system_call`触发的处理程序中使用。

**target()**

当你通过`stap script -x PID`或`stap script -c command`来执行某个脚本`script`时，`target()`会返回你指定的PID或命令名。举个例子：

```
probe syscall.* {
  if (pid() == target())
    printf("%s\n", name)
}
```

当上面的例子中的脚本带命令行参数`-x PID`运行时，它会监控所有的系统调用（`syscall.*`），并输出其中由指定进程所触发的系统调用。
你当然可以把上面例子中的`target()`替换成你想要指定的PID。不过使用`target()`让你的脚本可以重用。现在你只需在运行时指定PID，而无需每次都修改掉硬编码的PID值。

要想了解更多关于SystemTap函数的信息，请`man stapfuncs`。
