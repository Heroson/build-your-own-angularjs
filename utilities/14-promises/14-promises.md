浏览器中大部分的 JavaScript 程序的执行都是异步的：我们会提前准备好要运行的代码，这段代码会在以后某种条件满足的情况下被触发。触发条件可能为用户的某些交互动作，可能是数据从服务器端返回，也可能是定时器计时结束。这种情况大大影响了 JavaScript 代码的组织方式。

在很长的一段时间中，我们处理异步操作的方式就是回调函数（callback）。我们会在 API 中加入回调函数，然后在 API 调用有结果时调用。基本上 JavaScript 和浏览器中内建的异步 API 都使用回调函数：

```js
element.addEventListener('click', function(evt) {
  console.log('clicked!', evt);
});
```

然而，使用回调函数可能会导致一些麻烦：

* 回调函数的处理方式，可能会在无意中把业务逻辑和实现逻辑混在一起。例如，computeBalance\(form, to, onDone\)，这里的 onDone 就是一个回调函数。实际上，我们在该方法的业务处理只需要 from 和 to 两个参数就可以了，但我们为了下一个流程的顺利进行，不得补加入 onDone 参数。

* 如果一个业务处理中需要多个异步函数的顺序执行，步骤还比较多的话，我们页会面临难以编写和维护该功能代码的问题，会导致诸如“回调地狱“之类的问题出现

* 回调函数的方式并没有错误处理的机制。按照传统的处理，我们会在回调函数中加入一个特殊的错误参数，而且这种方式必须用到每一个出现回调函数的地方去。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter13-high-level-dependency-injection-features)



