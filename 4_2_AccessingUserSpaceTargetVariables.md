# 4.2. 访问用户空间目标变量

你可以访问用户空间目标变量，所用的语法与第3.3节第二部分，“目标变量”中访问内核空间的语法相同。在Linux中，用户代码和内核代码使用的地址空间是隔绝的。不过SystemTap可以在使用`->`运算符时找到恰当的地址空间。

对于指向基本类型（如整数和字符串）的指针，可以使用下列的函数访问用户空间的数据。每个函数的第一个参数都是指向数据的指针（`address`）。

**user_char(address)**

从当前用户进程中获取地址对应的字符数据。

**user_short(address)**

从当前用户进程中获取地址对应的short型数据。

**user_int(address)**

从当前用户进程中获取地址对应的int型数据。

**user_long(address)**

从当前用户进程中获取地址对应的long型数据。

**user_string(address)**

从当前用户进程中获取地址对应的字符串数据。

**user_string_n(address, n)**

从当前用户进程中获取地址对应的字符串数据，取前n字节。

译注：这些函数都是在`process(PATH).xxx`事件的处理程序中使用的。当前用户进程指的就是`PATH`。如
```
process(@1).syscall {
    ...
    user_string(field) # field指向@1地址空间中的某个地址
}
```
