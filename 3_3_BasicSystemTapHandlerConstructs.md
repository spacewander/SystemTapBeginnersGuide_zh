# 3.3. 处理程序的基本结构

SystemTap支持在处理程序中使用一些基本的结构。它们的语法基本上类似于C或awk。了解最常用的一些结构，有助于你写出更清晰的SystemTap脚本。

## 变量

处理程序里面当然可以使用变量，你所需的不过是给它取个好名字，把函数或表达式的值赋给它，然后就可以使用它了。SystemTap可以自动判定变量的类型。举个例子，如果你用`gettimeofday_s()`给变量`foo`赋值，那么`foo`就是数值类型的，可以在`printf()`中通过`%d`输出。
变量默认只能在其所定义的探针内可用。这意味着变量的生命周期仅仅是处理程序的某次运行。不过你也可以在探针外定义变量，并使用`global`修饰它们，这样就能在探针间共享变量了。
⁠
```
global count_jiffies, count_ms
probe timer.jiffies(100) { count_jiffies ++ }
probe timer.ms(100) { count_ms ++ }
probe timer.ms(12345)
{
  hz=(1000*count_jiffies) / count_ms
  printf ("jiffies:ms ratio %d:%d => CONFIG_HZ=%d\n",
    count_jiffies, count_ms, hz)
  exit ()
}
```

在上面的例子中，`timer-jiffies.stp`通过累加jiffies和milliseconds，来求出内核的`CONFIG_HZ`配置。`global`语句使得`count_jiffies`和`count_ms`在每个探针中可用。

> 在上面的例子中，我们用`++`来将变量的值加一。如下探针中，`count_jiffies`每隔100 jiffies会自增1:
> ```
> probe timer.jiffies(100) { count_jiffies ++ }
> ```
> SystemTap知道`count_jiffies`是一个整数。那是因为`count_jiffies`没有被赋予一个初始值，所以它的值默认为零。


## 目标变量（Target Variables）

跟内核代码相关的事件，如`kernel.function("function")`和`kernel.statement("statement")`，允许使用目标变量获取这部分代码中可访问到的变量的值。你可以使用`-L`选项来列出特定探测点下可用的目标变量。如果已经安装了内核调试信息，你可以通过这个命令获取`vfs_read`中可用的目标变量：
```
stap -L 'kernel.function("vfs_read")'
```

它会有类似如下的输出：
```
kernel.function("vfs_read@fs/read_write.c:277") $file:struct file* $buf:char* $count:size_t $pos:loff_t*
```

每个目标变量前面都以`$`开头，并以`:`加变量类型结尾。上面的输出表示，`vfs_read`函数入口处有三个变量可用：`$file`（指向描述文件的结构体）、`$buf`（指向接收读取的数据的用户空间缓冲区）、`$count`（读取的字节数），和`$pos`（读开始的位置）。
对于那些不属于本地变量的变量，像是全局变量或一个在文件中定义的静态变量，可以用`@var("varname@src/file.c")`获取。
SystemTap会保留目标变量的类型信息，并且允许通过`->`访问其中的成员。跟C语言不同的是，`->`既可以用来访问指针指向的值，也可以用来访问子结构体中的成员。在获取复杂结构体中的信息时，`->`可以链式使用。举个例子，`fs/file_table.c`中的静态目标变量`files_stat`存储着一些当前文件系统中可调节的参数。我们为了获取其中的一个域，可以这么写：

```
stap -e 'probe kernel.function("vfs_read") {
           printf ("current files_stat max_files: %d\n",
                   @var("files_stat@fs/file_table.c")->max_files);
           exit(); }'
```

会有类似如下的输出：
```
current files_stat max_files: 386070
```

有许多函数可以通过指向基本类型的指针获取内核空间对应地址上的数据，在此一一列出。在第4.2节，我们还会谈到获取用户空间数据的类似函数。

**kernel_char(address)**

从内核空间地址中获取char变量

**kernel_short(address)**

从内核空间地址中获取short变量

**kernel_int(address)**

从内核空间地址中获取int变量

**kernel_long(address)**

从内核空间地址中获取long变量

**kernel_string(address)**

从内核空间地址中获取字符串

**kernel_string_n(address, n)**

从内核空间地址中获取长为n的字符串

### 整齐打印目标变量（Pretty Printing Target Variables）

某些场景中，我们可能需要输出当前可访问的各种变量，以便于记录底层的变化。SystemTap提供了一些操作，可以生成描述特定目标变量的字符串：

**$$vars**

输出作用域内每个变量的值。等价于`sprintf("parm1=%x ... parmN=%x var1=%x ... varN=%x", parm1, ..., parmN, var1, ..., varN)`。如果变量的值在运行时找不到，输出`=?`。

**$$locals**

同`$$vars`，只输出本地变量。

**$$parms**

同`$$vars`，只输出函数入参。

**$$return**

仅在带`return`的探针中可用。如果被监控的函数有返回值，它等价于`sprintf("return=%x", $return)`，否则为空字符串。

下面的例子中，我们会输出`vfs_read`的入参：
```
stap -e 'probe kernel.function("vfs_read") {printf("%s\n", $$parms); exit(); }'
```

`vfs_read`的入参有四个：`file`，`buf`，`count`，和`pos`。`$$params`会给这些入参生成描述字符串。在这个例子里，四个变量都是指针。下面是之前的命令的输出：
```
file=0xffff8800b40d4c80 buf=0x7fff634403e0 count=0x2004 pos=0xffff8800af96df48
```

关输出个地址值没什么用啊。要想输出指针指向的值，我们可以加上`$`后缀。下面的命令使用`$`后缀来输出`vfs_read`入参的实际值：
```
stap -e 'probe kernel.function("vfs_read") {printf("%s\n", $$parms$); exit(); }'
```

输出的结果：
```
file={.f_u={...}, .f_path={...}, .f_op=0xffffffffa06e1d80, .f_lock={...}, .f_count={...}, .f_flags=34818, .f_mode=31, .f_pos=0, .f_owner={...}, .f_cred=0xffff88013148fc80, .f_ra={...}, .f_version=0, .f_security=0xffff8800b8dce560, .private_data=0x0, .f_ep_links={...}, .f_mapping=0xffff880037f8fdf8} buf="" count=8196 pos=-131938753921208
```

只使用`$`后缀的话，是不会展开结构体里面嵌套的结构体的。要想展开嵌套的结构体，你需要使用`$$`后缀。下面是一个使用`$$`的例子：
```
stap -e 'probe kernel.function("vfs_read") {printf("%s\n", $$parms$$); exit(); }'
```

注意`$$`的输出，会受到字符串最长长度的限制。来自上面命令的输出，就因此被截断了：
```
file={.f_u={.fu_list={.next=0xffff8801336ca0e8, .prev=0xffff88012ded0840}, .fu_rcuhead={.next=0xffff8801336ca0e8, .func=0xffff88012ded0840}}, .f_path={.mnt=0xffff880132fc97c0, .dentry=0xffff88001a889cc0}, .f_op=0xffffffffa06f64c0, .f_lock={.raw_lock={.slock=196611}}, .f_count={.counter=2}, .f_flags=34818, .f_mode=31, .f_pos=0, .f_owner={.lock={.raw_lock={.lock=16777216}}, .pid=0x0, .pid_type=0, .uid=0, .euid=0, .signum=0}, .f_cred=0xffff880130129a80, .f_ra={.start=0, .size=0, .async_size=0, .ra_pages=32, .
```

## 条件语句

有些时候，你写的SystemTap脚本较为复杂，可能需要用上条件语句。SystemTap支持C风格的条件语句，另外还支持`foreach (VAR in ARRAY) {}`形式的遍历。

## 命令行参数

通过`$`或`@`加个数字的形式可以访问对应位置的命令行参数。用`$`会把用户输入当作整数，用`@`会把用户输入当作字符串。

```
probe kernel.function(@1) { }
probe kernel.function(@1).return { }
```

上面的脚本期望用户把要监控的函数作为命令行参数传递进来。你可以让脚本接受多个命令行参数，分别命名为`@1`，`@2`等等，按用户输入的次序逐个对应。
