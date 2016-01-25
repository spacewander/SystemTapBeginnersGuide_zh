# 4. 用户空间探测

SystemTap诞生的最初使命，是探测内核空间。由于许多情况下用户空间探测有助于诊断问题，SystemTap从0.6版本开始也支持探测用户空间的进程。SystemTap可以探测用户空间进程内函数的调用和退出，可以探测用户代码中预定义的标记，可以探测用户进程的事件。

SystemTap进行用户空间探测需要uprobes模块。如果你的Linux内核版本大于等于3.5, 它已经内置了`uprobes`。要想验证当前内核是否原生支持uprobes，运行下面命令：
```
grep CONFIG_UPROBES /boot/config-`uname -r`
```

如果当前内核集成了uprobes，就会输出以下内容：
```
CONFIG_UPROBES=y
```

如果你的内核版本小于3.5, SystemTap会自动构建uprobes模块。不过，SystemTap的用户空间事件跟踪功能依然需要你的内核支持utrace拓展。可以从这个链接获取更多关于utrace的细节：http://sourceware.org/systemtap/wiki/utrace 。要想验证当前内核是否提供了必要的utrace支持，在终端中输入下面的命令：
```
grep CONFIG_UTRACE /boot/config-`uname -r`
```

如果当前内核支持用户空间探测，就会输出以下内容：
```
CONFIG_UTRACE=y
```
