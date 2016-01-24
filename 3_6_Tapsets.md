## 3.6. Tapsets

tapsets是一些包含常用的探针和函数的内置脚本，你可以在SystemTap脚本中复用它们。当用户运行一个SystemTap脚本时，SystemTap会检测脚本中的事件和处理程序，并在翻译脚本成C代码之前，加载用到的tapset。（可以回顾下本章开头所讲到的，SystemTap会话的启动过程）
就像SystemTap脚本一样，tapset的拓展名也是`.stp`。默认情况下tapset位于`/usr/share/systemtap/tapset/`。跟SystemTap脚本不同的是，tapset不能被直接运行；它只能作为库使用。
tapset库让用户能够在更高的抽象层次上定义事件和函数。tapset提供了一些常用的内核函数的别名，这样用户就不需要记住完整的内核函数名了（尤其是有些函数名可能会因内核版本的不同而不同）。另外tapset也提供了常用的辅助函数，比如之前我们见过的`thread_indent()`。
