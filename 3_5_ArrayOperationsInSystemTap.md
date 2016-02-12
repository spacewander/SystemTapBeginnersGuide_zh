# 3.5. 数组操作 

本节将列举SystemTap中若干常用的数组操作。

## 设置给定键的值

使用`=`来设置给定键所对应的值，正如：
```
foo[tid()] = gettimeofday_s()
```

SystemTap会把`tid()`的结果作为一个键，并把`gettimeofday_s()`的结果赋给这个键。如果这个键已经存在`foo`中，原先关联的值会被覆盖掉。

## 获取给定键的值

使用`array_name[index_expression]`可以获取对应键上的值。比如：
```
delta = gettimeofday_s() - foo[tid()]
```

> 如果数组中没有`index_expression`对应的键，默认情况下它会返回0（在数值计算中）或者空字符串（在字符串操作中）。

## 自增给定键的值

使用`++`来增加对应键上的值，比如：`array_name[index_expression] ++`。在下面的例子里，每次`vfs.read`都会把当前进程名所关联的值加一：
```
probe vfs.read
{
  reads[execname()] ++
}
```

你可以用`if (index_expression in array_name)`来判断数组是否有指定的键。

## 遍历数组中的多个元素

一旦已经收集了足够的信息到数组里，你往往需要去遍历它。正如上面的例子中，在收集了各个进程的读次数后，你可能需要遍历它，输出每个进程的结果。那该怎么做呢？

最好的方法就是使用`foreach`语句。看下这个例子：
```
global reads
probe vfs.read
{
  reads[execname()] ++
}

probe timer.s(3)
{
  foreach (count in reads)
    printf("%s : %d \n", count, reads[count])
}
```

在第二个探针中的`foreach`语句里，`count`引用了`reads`的键，所以可以通过`reads[count]`读取对应键所关联的值。

在这个`foreach`语句里面，我们依次遍历`reads`的每个值。假如我们不想遍历整个数组，或者想指定遍历的顺序，该怎么做呢？你可以给数组名加个后缀`+`来表示按升序遍历，或`-`按降序遍历。另外，你可以用`limit`加一个数字来限制迭代的次数。
看下这个类似于上一个探针的例子：
```
probe timer.s(3)
{
  foreach (count in reads- limit 10)
    printf("%s : %d \n", count, reads[count])
}
```

上面的`foreach`语句会按关联的值降序遍历数组。`limit 10`表示`foreach`语句只会迭代10次（也即输出最高的10个值）。

## 清除数组或数组中某个元素

有时，你需要清除数组值某个值，或者清空整个数组以便于在另一个探针值重用。在之前统计`vfs.read`的例子里，每三秒统计一次各个进程的调用读操作的次数。如果要想统计三秒内各个进程的数据，需要每三秒清空一次数组。你可以使用`delete`运算符来删除数组中的某个元素，或整个数组。看下下面的例子：
```
global reads
probe vfs.read
{
  reads[execname()] ++
}

probe timer.s(3)
{
  foreach (count in reads)
    printf("%s : %d \n", count, reads[count])
  delete reads
}
```

在上面的例子中，第二个探针仅输出三秒内每个进程的读次数。这里的`delete`语句清空了整个`reads`数组。

## 使用聚集变量（use aggregates）

有时候你需要快速处理新的数值，并且数据量较大，这时候可以考虑使用聚集变量（aggregates），因为它实现了对数据的流式处理。聚集变量可以用作全局变量，也可以用作数组中的值。使用`<<<`运算符可以往聚集变量中添加新数据。

```
global reads
probe vfs.read
{
  reads[execname()] <<< $count
}
```

假设在上面的例子中，`$count`的值是一段时间内当前进程的读次数。`<<<`会把`$count`的值存储到`reads`数组`execname()`关联的聚集变量中。请注意，我们是把值存储在聚集变量里面；它们既没有加到原来的值上，也没有覆盖掉原来的值。可以这么说，就像是`reads`数组值每个键都有多个关联的值，并且探针的每次触发都会添加新的值。

要想从聚集变量中获取汇总的结果，使用这样的语法`@extractor(variable/array index expression)`。`extractor`可以取以下的函数：

**count**

返回`variable/array index expression`中存储的数值的数目。以上面为例，`@count(reads[execname()])`返回对应进程的聚集变量所存储的数据数。

**sum**

返回`variable/array index expression`中存储的数值的和。以上面为例，`@count(reads[execname()])`返回对应进程的读总数。

**min**

返回`variable/array index expression`中存储的数值的最小值。

**max**

返回`variable/array index expression`中存储的数值的最大值。

**avg**

返回`variable/array index expression`中存储的数值的数目。

你可以使用多重索引表达式在数组里关联一个聚集变量（最多使用9个索引）。这么做的好处在于，你可以在数组中附加更多的上下文信息。举个例子：
```
global reads
probe vfs.read
{
  reads[execname(),pid()] <<< 1
}

probe timer.s(3)
{
  foreach([var1,var2] in reads)
    printf("%s (%d) : %d \n", var1, var2, @count(reads[var1,var2]))
}
```

在上面的例子中，第一个探针记录每个进程的`vfs.read`次数。跟之前的例子不同的是，这里的数组同时使用进程名和PID作为索引。

在第二个探针里，我们使用`foreach`语句遍历并输出每个进程的数据。注意这里我们分别使用`var1`和`var2`来引用进程名和PID。
