### 本章总结（Summary）

表达式 interpolation 对于 Angular 来说是一个很重要的特性。它是大多数开发者学习 Angular 的第一课，而且没有 Angular 应用能离开它。

但对这个功能的开发并不算十分复杂，因为它是建立在我们已经开发的功能基础上，像是处理表达式解析的`$parse`、Scope 上的 watcher，还有`$compile`提供的指令。对我们来说，这个功能的实现过程是向我们展示这些底层组件是怎么通过组合来成就一个有用的功能的。

我们也学习到了怎么利用 watch 委托和 watch group 减少我们在变化监测时要做的工作。同样的技巧我们也可以用在应用代码中。