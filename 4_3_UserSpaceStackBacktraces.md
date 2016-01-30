# 4.3. 用户空间栈回溯

`pp`（probe point）函数可以返回触发当前处理程序的事件名（包含展开了的通配符和别名）。如果该事件与特定的函数相关，`pp`的输出会包括触发了该事件的函数名。然而，许多情况下触发同一个事件的函数可能来自于程序中不同的模块；特别是在该函数位于某个共享库的情况下。还好SystemTap提供了用户空间栈的回溯（backtrace）功能，便于查看事件是怎么被触发的。

编译器优化代码时会消除栈帧指针（stack frame pointers），这将混淆用户空间栈回溯的结果。所以要想查看栈回溯，需要有编译器生成的调试信息。SystemTap用户空间栈回溯机制可以利用这些调试信息来重建栈回溯的现场，不过该功能当前只实现在32位和64位x86处理器上，还不支持其他架构的处理器。要想使用这些调试信息来重建栈回溯，给可执行文件加上`-d executable`选项，并给共享库加上`-ldd`选项。

举个例子，你可以使用`ubacktrace`（user-space backtrace）函数来输出`ls`命令中`xmalloc`函数的调用情况。如果你已经安装了`ls`命令的debuginfo，下面的SystemTap命令会在`xmalloc`函数调用时输出栈回溯的结果：
```
stap -d /bin/ls --ldd \
-e 'probe process("ls").function("xmalloc") {print_usyms(ubacktrace())}' \
-c "ls /"
```

译注：要想成功运行上面的命令，你需要安装coreutils的debuginfo。具体安装方式请根据自己用的发行版搜索一下。如果你跟我一样用的也是Ubuntu，可以看下[askubuntu上这个回答](http://askubuntu.com/questions/427318/how-can-i-install-a-debug-build-for-coreutils)，运行`sudo apt-get install coreutils-dbgsym`。

运行后，上面的命令会有类似下面的输出：
```
bin dev   lib     media  net         proc   sbin     sys  var
boot    etc   lib64   misc   op_session  profilerc  selinux  tmp
cgroup  home  lost+found  mnt    opt         root   srv  usr
 0x4116c0 : xmalloc+0x0/0x20 [/bin/ls]
 0x4116fc : xmemdup+0x1c/0x40 [/bin/ls]
 0x40e68b : clone_quoting_options+0x3b/0x50 [/bin/ls]
 0x4087e4 : main+0x3b4/0x1900 [/bin/ls]
 0x3fa441ec5d : __libc_start_main+0xfd/0x1d0 [/lib64/libc-2.12.so]
 0x402799 : _start+0x29/0x2c [/bin/ls]
 0x4116c0 : xmalloc+0x0/0x20 [/bin/ls]
 0x4116fc : xmemdup+0x1c/0x40 [/bin/ls]
 0x40e68b : clone_quoting_options+0x3b/0x50 [/bin/ls]
 0x40884a : main+0x41a/0x1900 [/bin/ls]
 0x3fa441ec5d : __libc_start_main+0xfd/0x1d0 [/lib64/libc-2.12.so]
 ...
 ```

关于在用户空间栈回溯中可用的函数的更多内容，请查看`ucontext-symbols.stp`和`ucontext-unwind.stp`两个tapset。上述tapset中的函数的描述信息也可以在[SystemTap Tapset Reference Manual](https://sourceware.org/systemtap/tapsets/)找到。
