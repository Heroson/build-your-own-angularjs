### 异常处理

#### Handling Exceptions

我们之前开发的 `$evalAsync`、`$applyAsync` 和 `$$postDigest` 都存在一个问题，就是会在发生异常时终止函数的执行，同时 digest 循环也会被迫提前结束。官方 Angular 的实现方式则要健壮得多，无论是在 digest 运行之前、之中或之后抛出的异常都能被 Angular 捕捉到，并进行日志记录。

对于 `$evalAsync` 来说，我们可以（在 `describe('$evalAsync'` 中\) 定义一个测试用例 ，在这个测试中，当有 `$evalAsync` 延时函数抛出异常时，我们验证 watcher 是否依然会被执行：

_test/scope\_spec.js_

```js
it('catches exceptions in $evalAsync', function(done) {
  scope.aValue = 'abc';
  scope.counter = 0;

  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$evalAsync(function(scope) {
    throw 'Error';
  });

  setTimeout(function() {
    expect(scope.counter).toBe(1);
    done();
  }, 50);
});
```

对于 `$applyAsync`，我们会使用 `$applyAsync` 设定几个延时执行函数，当前面的函数抛出异常时，我们验证最后的那个函数会不会被正常调用。这个单元测试会被放到 `describe('$applyAsync')` 的测试板块中。

_test/scope\_spec.js_

```js
it('catches exceptions in $applyAsync', function(done) {
  scope.$applyAsync(function(scope) {
    throw 'Error';
  });
  scope.$applyAsync(function(scope) {
    throw 'Error';
  });
  scope.$applyAsync(function(scope) {
    scope.applied = true;
  });

  setTimeout(function() {
    expect(scope.applied).toBe(true);
    done();
  }, 50);
});
```

> 注意，这里我们连续用了两个会抛出异常的函数，如果我们只用一个的话，第二个函数本来就一定会被执行。`$apply` 函数中的 `finally` 代码块中会触发 `$digest`，在这个 `$digest` 中，`$applyAsync` 创建的异步任务队列都会被执行完毕。
>
> （译者注：上面的解释比较简略，这里详细说下流程。如果我们只用一个会抛出异常的函数，那就是 `$applyAsync` 一共会设定两个函数，一个会抛异常，另一个不会。第一个 `$applyAsync` 函数会设定一个定时器，定时器到时执行 `$apply`，`$apply` 会调用 `$eval` ，`$eval` 调用 `$$flushApplyAsync`，`$$flushApplyAsync` shift 出第一个异步任务并调用，抛出异常，导致 `$$flushApplyAsync` 调用过程发生中断，返回到上层也就是 `$apply`，由于 `$apply` 使用了 `try...finally` 代码块对 `$eval` 进行包裹，因此异常会被忽略，并进入到 `finally` 代码块，`finally` 代码块中调用了 `$digest`，这样最后一个异步任务就会在 `$digest` 函数处理 `$applyAsync` 设定的异步任务时被调用了）

由于 `$$postDigest` 延时函数会在 digest 结束之后执行，我们就不能再利用 watcher 进行测试了。但我们可以使用第二个 `$$postDigest` 延时函数来测试，我们要确保在第一个函数发生异常时，第二个函数照样会被执行。我们把这个测试放到 `describe('$$postDigest')` 测试集合中去。

_test/scope\_spec.js_

```js
it('catches exceptions in $$postDigest', function() {
  var didRun = false;

  scope.$$postDigest(function() {
    throw 'Error';
  });
  scope.$$postDigest(function() {
    didRun = true;
  });

  scope.$digest();
  expect(didRun).toBe(true);
});
```

要解决 `$evalAsync` 和 `$$postDigest` 的异常处理问题，关键在于 `$digest` 函数。我们只需要在 `$digest` 执行延时函数的位置外都包裹上 `try...catch` 代码块就可以了：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  // this.$$lastDirtyWatch = null;
  // this.$beginPhase('$digest');

  // if (this.$$applyAsyncId) {
  //   clearTimeout(this.$$applyAsyncId);
  //   this.$$flushApplyAsync();
  // }

  do {
    while (this.$$asyncQueue.length) {
      try {
        // var asyncTask = this.$$asyncQueue.shift();
        // asyncTask.scope.$eval(asyncTask.expression);
      } catch (e) {
        console.error(e);
      }
    }
    // dirty = this.$$digestOnce();
    // if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
    //   throw '10 digest iterations reached';
    // }
  } while (dirty || this.$$asyncQueue.length);
  // this.$clearPhase();

  while (this.$$postDigestQueue.length) {
    try {
      // this.$$postDigestQueue.shift()();
    } catch (e) {
      console.error(e);
    }
  }
};
```

而对于 `$applyAsync`，我们就需要在 `$$flushApplyAsync` 遍历异步任务时进行处理了：

_src/scope.js_

```js
Scope.prototype.$$flushApplyAsync = function() {
  while (this.$$applyAsyncQueue.length) {
    try {
      // this.$$applyAsyncQueue.shift()();
    } catch (e) {
      console.error(e);
    }
  }
  // this.$$applyAsyncId = null;
};
```

加入异常处理以后，现在的 digest 周期就比之前健壮多了。

