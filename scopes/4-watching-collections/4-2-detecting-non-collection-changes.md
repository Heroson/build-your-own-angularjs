### 听侦听非集合数据的变化

#### Detecting Non-Collection Changes

`$watchCollection` 的主要用途就是侦听数组和对象。但是，它也能兼容 watch 函数返回一个非集合类型数据的情况。在这种情况下，它会自动降级为直接调用 `$watch` 。虽然处理这种特殊情况的过程比较乏味，但能让我们看到函数的结构。

下面这个测试用于明确函数的基本行为：

_test/scope\_spec.js_

```js
it('works like a normal watch for non-collections', function() {
  var valueProvided;

  scope.aValue = 42;
  scope.counter = 0;

  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      valueProvided = newValue;
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);
  expect(valueProvided).toBe(scope.aValue);

  scope.aValue = 43;
  scope.$digest();
  expect(scope.counter).toBe(2);

  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

我们使用 `$watchCollection` 侦听作用域上的一个数字类型的属性。我们断言这个 watcher 跟普通的 watcher 一样，会在这种情况下调用 listener 函数。在 listener 函数中，我们会让计数器加一，同时把捕获到的新值保存到一个局部变量中。

> 这里我们暂时忽略 `oldValue` 参数。`$watchCollection` 函数需要对这个参数进行一些特殊的处理，本章后续内容会提到。

`$watchCollection` 内部的 watch 函数执行时，首先会调用原始传入的 watch 函数来获取我们要侦听的值，然后检查这个值是否发生了变化，并把该值作为下一次 digest 时的旧值：

_src/scope.js_

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
  var newValue;
  var oldValue;

  var internalWatchFn = function(scope) {
    newValue = watchFn(scope);

    // Check for changes

    oldValue = newValue;
  };

  var internalListenerFn = function() {
  };

  return this.$watch(internalWatchFn, internalListenerFn);
};
```

我们把 `newValue` 和 `oldValue` 两个变量提取到 `internalListenerFn` 函数体外，这样  `internalWatchFn` 和 `internalListenerFn` 两个函数就都能访问这两个变量了。同时，借助 `$watchCollection` 函数形成的闭包，这两个变量就能在 digest 不同轮次之间持续存在了。这个特性对旧值来说尤其重要，因为我们需要在不同轮次中比较这个值。

内部 listener 函数目前要做的就是直接调用原始传入的 listener 函数，调用时依次传入新值、旧值和作用域即可：

_src/scope.js_

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
  var self = this;
  // var newValue;
  // var oldValue;

  // var internalWatchFn = function(scope) {
  //   newValue = watchFn(scope);

  //   // Check for changes

  //   oldValue = newValue;
  // };

  // var internalListenerFn = function() {
    listenerFn(newValue, oldValue, self);
  // };

  // return this.$watch(internalWatchFn, internalListenerFn);
};
```

让我们回想一下，`$digest` 是通过比较 watch 函数在连续两轮 digest 中的返回值来决定是否调用 listener 函数。但现在内部的 watch 函数什么都没有返回，自然就不会调用 listener 函数了。

那内部的 watch 函数应该返回什么呢？由于在 `$watchCollection` 外部无法访问到这个返回值，那它是什么并没有啥区别。唯一需要关注的是一旦对象发生了改变，连续两轮 digest 的返回值就不同，而这就是 listener 函数的触发条件。

解决这个问题，Angular 采用的方法是引入一个整数计数器，只要检测到对象中有一个元素发生了变化，就让计数器递增。每一个通过 `$watchCollection` 注册的 watcher 都有自己的计数器，它会在 watcher 的整个生命周期中不断递增。只要在 `internalWatchFn` 中返回这个计数器，就能满足 watch 的函数约束了。

对非集合数据来说，我们直接看新旧值是否发生了引用层面的变化就好了：

_src/scope.js_

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
  // var self = this;
  // var newValue;
  // var oldValue;
  var changeCount = 0;

  var internalWatchFn = function(scope) {
    // newValue = watchFn(scope);

    if (newValue !== oldValue) {
      changeCount++;
    }
    // oldValue = newValue;

    return changeCount;
  };

  // var internalListenerFn = function() {
  //   listenerFn(newValue, oldValue, self);
  // };

  // return this.$watch(internalWatchFn, internalListenerFn);
};
```

这样我们就能够处理非集合数据的情况了。但如果这个非集合数据刚好是 `NaN` 又该怎么办呢？

_test/scope\_spec.js_

```js
it('works like a normal watch for NaNs', function() {
  scope.aValue = 0 / 0;
  scope.counter = 0;

  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);

  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

这个单元测试无法通过，原因跟我们在第一章处理 NaN 时是一样的：因为 `NaN` 不等于自身。为了解决这个问题，我们会用现有的帮助函数 `$$areEqual` 取代 `!==` 来进行相等性比较，这个帮助函数已经对值为 `NaN` 的情况进行处理了：

_src/scope.js_

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
  // var self = this;
  // var newValue;
  // var oldValue;
  // var changeCount = 0;

  // var internalWatchFn = function(scope) {
  //   newValue = watchFn(scope);

    if (!self.$$areEqual(newValue, oldValue, false)) {
      changeCount++;
    }
  //   oldValue = newValue;

  //   return changeCount;
  // };

  // var internalListenerFn = function() {
  //   listenerFn(newValue, oldValue, self);
  // };

  // return this.$watch(internalWatchFn, internalListenerFn);
};
```

上面调用 `$$areEqual` 时传入的最后一个参数为 `false`，这表明我们不会在这里使用值比较。在这个情况下，我们使用引用比较就可以了。

现在我们已经有了 `$watchCollection` 的基本结构了，接下来可以把注意力放回到对集合数据的变化侦测上来了。

