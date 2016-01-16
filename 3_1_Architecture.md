# 3.1. 结构

SystemTap脚本运行时，会启动一个对应的SystemTap会话。整个会话大致流程如下：

首先，SystemTap会检查脚本中用到的`tapset`，确保它们都存在于tapset库中（通常是`/usr/share/systemtap/tapset/`）。然后SystemTap会把找到的`tapset`替换成在tapset库中对应的定义。（译注：tapset是tap（听诊器）的集合，指一些预定义的SystemTap事件或函数。完整的tapset列表见 https://sourceware.org/systemtap/tapsets/ ）

SystemTap接着会把脚本转化成C代码，运行系统的C编译器编译出一个内核模块。完成这一步的工具包含在systemtap包中（详见第2.1节，“安装和配置”）

SystemTap随即加载该模块，并启用脚本中所有的探针（包括事件和对应的处理程序）。这一步由system-runtime包的`staprun`完成。（详见第2.1节，“安装和配置”）

每当被监控的事件发生，对应的处理程序就会被执行。

一旦SystemTap会话终止，探针会被禁用，内核模块也会被卸载。

这一串流程皆始于一个简单的命令行程序：`stap`。这个程序包揽了SystemTap主要的功能。要想了解关于`stap`的更多信息，请`man stap`（前提是你的机器上已经安装了SystemTap）
