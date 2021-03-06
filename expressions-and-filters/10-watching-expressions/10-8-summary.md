### 本章小结
#### Summary

将作用域系统和表达式系统结合起来，就实现了一个强大的、具有表达能力的脏值检测系统，它已经可以作为一个独立的工具，同时也可以与本书后面要实现的指令系统组合起来使用。

作用域和表达式的结合也产生了一些强大的功能：常量优化、单次绑定和输入项跟踪。对于开发人员来说，它们为应用程序提供了潜在的、非常重要的性能改进。

在本章，我们学习到了：

- `Scope` 是如何利用表达式解析器实现把表达式用作一个侦听器的。
- `Scope` 是如何利用表达式解析器实现对表达式进行求值的。
- 表达式是如何被标记为 `constant` 或者 `literal` 的。
- 表达式解析器有时会提供绕过正常侦听行为的侦听委托。
- 常量侦听委托是如何在常量表达式的第一次调用后解除对它的侦听的。
- 单词绑定是如何运作的，以及它是如何通过一个单次的侦听委托实现的，以及该委托在怎么在 watch 稳定在某个已定义的值之后被解除的。
- 数组和对象是如何通过等待它们包含的元素值稳定下来实现单次绑定的。
- 表达式解析器是如何使用 _输入项跟踪_ 来最小化对符合表达式的运算，方法就是只在输入表达式发生改变时才执行运算
- 过滤器是如何通过被标记为有状态的，从而禁用对它们的常量优化和输入项优化。
- 表达式函数上的 `assign` 是如何支持在一个给定的作用域范围内对表达式的值进行重新赋值的。