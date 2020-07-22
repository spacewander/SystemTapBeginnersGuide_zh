# 2.1. 安装和配置

要想使用`SystemTap`，需要安装跟目标内核版本匹配的`-devel`、`-debuginfo`和`-debuginfo-common-arch`包。如果要在不止一个内核上运行`SystemTap`，需要根据每个内核的版本安装对应的`-devel`和`-debuginfo`包。

接下来的几个小节里，我们会详细讲解这一过程。（译注：SystemTap的[wiki](https://sourceware.org/systemtap/wiki/)里面有针对Linux各发行版的安装步骤。本节内容仅适用于RHEL，且不能保证及时更新，建议跳过本节，直接参考官方文档。如果你正好用的是ubuntu，可以参考ubuntu的[wiki](https://wiki.ubuntu.com/Kernel/Systemtap)）

> 很多用户把`-debuginfo`记成了`-debug`。要想使用SystemTap，切记安装对应内核的`-debuginfo`包，不是`-debug`包。

## 安装SystemTap

要想使用SystemTap，安装下面的RPM包：

* systemtap
* systemtap-runtime

以root权限运行下面的命令：

```
yum install systemtap systemtap-runtime
```

注意要想用上SystemTap，还得安装依赖的内核调试信息包。在较新的系统上，仅需以root权限运行下面的命令：

```
stap-prep
```

如果这个命令没起作用，你需要按照下面的步骤手动安装。

## 手动安装依赖的内核调试信息包

SystemTap需要内核信息，这样才能注入指令。此外，这些信息还能帮助SystemTap生成合适的检测代码。

这些必要的内核信息分别包括在特定内核版本所对应的`-devel`，`-debuginfo`和`-debuginfo-common`包中。对于“标准版”内核（指按照常规配置编译的内核），所需的`-devel`和`-debuginfo`等包命名为：

```
kernel-debuginfo
kernel-debuginfo-common
kernel-devel
```

同样的，启用了PAE的内核所需的包分别为`kernel-PAE-debuginfo`，`kernel-PAE-debuginfo-common`，和`kernel-PAE-devel`。（译注：PAE即Physical Address Extension（物理地址拓展），32位Linux可以用它拓展内存访问空间）

要想确定当前系统的内核版本，敲入：
```
uname -r
```

举个例子，如果你想在i686环境下的2.6.18-53.el5内核上使用SystenTap，需要下载安装如下的RPM包：

```
kernel-debuginfo-2.6.18-53.1.13.el5.i686.rpm
kernel-debuginfo-common-2.6.18-53.1.13.el5.i686.rpm
kernel-devel-2.6.18-53.1.13.el5.i686.rpm
```

> 你安装的`-devel`、`-debuginfo`和`-debuginfo-common`包的版本一定要匹配目标内核的版本/特性/架构。

安装依赖的内核信息包最简单的方法，就是用`yum install`和`debuginfo-install`命令。`debuginfo-install`命令包含在版本1.1.10以上的`yum-utils`包里，还需要一个能够下载安装`-debuginfo`和`-debuginfo-common`包的yum源。
确保系统包管理的源满足要求，运行下面的命令就能安装特定内核对应的包：

```
yum install kernelname-devel-version
debuginfo-install kernelname-version
```

把命令中的`kernelname`替换成对应的内核名（比如，`kernel-PAE`），`version`换成目标内核的版本。举个例子，安装`kernel-PAE-2.6.18-53.1.13.el5`内核所对应的内核信息包，需要运行：

```
yum install kernel-PAE-devel-2.6.18-53.1.13.el5
debuginfo-install kernel-PAE-2.6.18-53.1.13.el5
```

一旦手动下载了所依赖的包之后，以root权限运行下面的命令来安装它们：

```
rpm --force -ivh package_names
```

## 检查安装是否成功

如果你正在用的内核就是你的目标内核，你现在就能检查下安装是否成功。如果不是，重启下系统并载入目标内核。

运行下面命令开始检查：

```
stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}'
```

这个命令让SystemTap在虚拟文件系统的读事件发生之后，输出`read performed`接着退出。如果SystemTap安装成功了，应该会输出类似下面的内容：

```
Pass 1: parsed user script and 45 library script(s) in 340usr/0sys/358real ms.
Pass 2: analyzed script: 1 probe(s), 1 function(s), 0 embed(s), 0 global(s) in 290usr/260sys/568real ms.
Pass 3: translated to C into "/tmp/stapiArgLX/stap_e5886fa50499994e6a87aacdc43cd392_399.c" in 490usr/430sys/938real ms.
Pass 4: compiled C into "stap_e5886fa50499994e6a87aacdc43cd392_399.ko" in 3310usr/430sys/3714real ms.
Pass 5: starting run.
read performed
Pass 5: run completed in 10usr/40sys/73real ms.
```

从`Pass 5`开始的最后三行说明SystemTap已经成功地注入并运行了内核探测指令，捕获了要探测的事件（在这个例子里，指虚拟文件系统的读事件），并执行了有效的处理程序（输出“read performed”并正常退出）。
