# 3.4. 关联数组

SystemTap支持关联数组。关联数组就像其它编程语言中的map/dict/hash，你可以把它看作由互不相同的键所组成的数组，每个键都有一个关联的值。

关联数组需要定义为全局变量。访问关联数组的值的语法跟awk类似，就是`array_name[index_expression]`。
这里的`array_name`指关联数组的名字，`index_expression`指数组中某个唯一的键。比如在下面的例子中，我们需要在数组`foo`中存tom、dick、harry三个人的年龄，可以这么写：
```
foo["tom"] = 23
foo["dick"] = 24
foo["harry"] = 25
```

在一个数组语句中你最多可以指定**九个**表达式，每个表达式间以`,`隔开。这样做可以给单个键附加多个信息。下面一行代码中，数组`device`的键包含五个表达式：进程PID，可执行程序名，用户UID，父进程PID，和字符串“W”。`devname`值关联到这个键上面。
```
device[pid(),execname(),uid(),ppid(),"W"] = devname
```

> 所有的关联数组都必须是全局变量，不管它们是否使用在多个探针内。
