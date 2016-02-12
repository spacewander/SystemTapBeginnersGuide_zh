# 6.2. 运行时错误和警告

运行时错误和警告发生在SystemTap安装了检测代码并开始收集数据的时候。

**⁠WARNING: Number of errors: N, skipped probes: M**

在运行时出错并/或跳过某些探针。由于诸如给定时间不足以执行完处理程序的原因，某些探针没有得到执行，`N`和`M`就是这些探针的数目。

**⁠division by 0**

代码里出现了除零错误。

**⁠aggregate element not found**

在一个空的聚集变量上调用除`@count`以外的提取函数。这就跟除零差不多。关于聚集变量的更多信息，参见[3.5 数组操作](3_5_ArrayOperationsInSystemTap.md)中的“使用聚集变量”部分。

**⁠aggregation overflow**

聚集变量数组中包含太多的键。（译注：前文有提及，数组的索引中最多只能使用九个键）

**⁠MAXNESTING exceeded**

过多的嵌套函数调用。默认函数调用层级是10。可以使用`-DMAXNESTING=NN`重编译脚本来修改这个限制。

**⁠MAXACTION exceeded**

处理程序太长了。默认一个探针的处理程序里面最多只能执行1000个语句。可以使用`--DMAXACTION=NN`或`-DMAXACTION_INTERRUPTIBLE=NN`重编译脚本来修改这个限制。

**⁠kernel/user string copy fault at ADDR**

处理程序试图把一个来自内核或用户空间的字符串拷贝到无效地址`ADDR`。

**pointer dereference fault**

指针解引用时发生了一个错误，可能发生在诸如计算目标变量的时候。
