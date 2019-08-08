### 本章总结（Summary）

指令模板并不是一个复杂的特性，不难理解，也不难开发。实际上，本章开始时只用了几页纸就实现了一个有基本功能的指令模板。

然而，当我们开始要支持模板的异步加载时，事情就变得有点复杂了。我们需要构建一个完整的、可以暂停也可以继续的编译和链接功能，而且在这个过程中，我们还需要在函数之间传递一些状态变量。有时我们会惊讶于实现一个简单的功能需要如此复杂的过程！
