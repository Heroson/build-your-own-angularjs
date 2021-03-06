### 隔离作用域

#### Isolated Scopes

通过上面的讲解，我们已经知道了在原型继承体系中，父子作用域之间的联系非常密切。无论父作用域上有什么属性，子作用域都可以访问到。如果这个属性是一个对象或者数组，子作用域还可以修改它的内容。

但有时我们并不希望它们之前的联系太过紧密，如果有一种子作用域本身既作为作用域树的一部分，但没有权限访问父作用域数据的话，这种子作用域在特定情况下会更方便。这就是所谓的_隔离作用域_（isolated scope）。

隔离作用域的核心概念是很简单的：我们会像之前一样创建一个作用域，它会成为作用域树的一部分，但我们不会让它在原型链继承它的父作用域。它会从父作用域的原型链结构中切割出来，或者说被隔离起来。

我们可以通过给 `$new` 传入一个布尔值参数来创建一个隔离作用域。当这个参数的值为 `true` 时，创建的作用域就会是被隔离了的。相反，当值为 `false`（指为 `undefined` 或被忽略）时，就会使用原型继承的方式进行创建。当作用域是隔离时，它就无法访问到父作用域上的数据：

_test/scope\_spec.js_

```js
it('does not have access to parent attributes when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  parent.aValue = 'abc';

  expect(child.aValue).toBeUndefined();
});
```

既然无法访问到父作用域上的属性，自然也没办法对这些属性进行侦听了：

```js
it('cannot watch parent attributes when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  parent.aValue = 'abc';
  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.aValueWas = newValue;
    }
  );

  child.$digest();
  expect(child.aValueWas).toBeUndefined();
});
```

隔离作用域是在 `$new` 方法中创建的。我们需要根据传入的布尔值属性来决定是创建跟以往一样的子作用域，还是使用 `Scope` 构造函数来创建一个隔离作用域。这两种方式创建的新作用域都会被添加到当前作用域的子作用域数组（children）中去：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated) {
  var child;
  if (isolated) {
    child = new Scope();
  } else {
    // var ChildScope = function() {};
    // ChildScope.prototype = this;
    child = new ChildScope();
  }
  // this.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  // return child;
};
```

> 如果你在 Angular 指令中用过隔离作用域，就会知道其实隔离作用域一般不会完全与自己的父作用域割裂开来的。相反，我们会根据需要从父作用域中获取的数据，明确定义出一个属性映射。
>
> 但目前我们的作用域还没有实现这个机制。这个机制是指令功能的一部分。后面我们讲到指令作用域的链接（directive scope linking）时再来详细介绍。

由于隔离作用域破坏了原型继承链的规则，我们需要重新回顾一下本章讨论过的 `$digest`、`$apply`、`$evalAsync` 和 `$applyAsync`。

首先是 `$digest` ，我们希望 `$digest` 依旧可以遍历整个作用域继承关系树。由于前面我们已经把隔离作用域放到父作用域下的 `$$children` 属性中，所以这个问题已经解决了，同时下面的这个单元测试也应该通过了：

_test/scope\_spec.js_

```js
it('digests its isolated children', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  child.aValue = 'abc';
  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.aValueWas = newValue;
    }
  );

  parent.$digest();
  expect(child.aValueWas).toBe('abc');
});
```

下面还需要处理 `$apply`、`$evalAsync` 和 `$applyAsync` 三个方法。我们希望调用这些方法时，都是从根作用域开始往下进行 digest，但在树层次结构中的隔离作用域会打破了这个假设，下面两个单元测试足以说明这一点：

```js
it('digests from root on $apply when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);
  var child2 = child.$new();

  parent.aValue = 'abc';
  parent.counter = 0;
  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  child2.$apply(function(scope) {});

  expect(parent.counter).toBe(1);
});

it('schedules a digest from root on $evalAsync when isolated', function(done) {
  var parent = new Scope();
  var child = parent.$new(true);
  var child2 = child.$new();

  parent.aValue = 'abc';
  parent.counter = 0;
  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  child2.$evalAsync(function(scope) {});

  setTimeout(function() {
    expect(parent.counter).toBe(1);
    done();
  }, 50);
});
```

> `$applyAsync` 是在 `$apply` 的基础上实现的，因此也会遇到同样的问题。只要我们修复了 `$apply`，`$applyAsync` 自然也没问题了。

注意，这两个单元测试其实跟之前开发 `$apply` 和 `$evalAsync` 时所写的两个用例大体相同，只是把其中一个作用域换成是隔离作用域而已。

这两个测试都没有通过，因为我们现在依然是依赖 `$root` 属性来访问根作用域。普通的作用域可以通过继承机制访问到根元素上的这个属性，但隔离作用域不能。实际上，由于我们会使用 `Scope` 构造函数来创建隔离作用域，这个构造函数本身会初始化一个 `$root` 属性，结果每一个孤立作用域都会有一个指向自身的 `$root` 属性，这样就没法实现我们想要的效果了。

要修复这个问题也很简单，我们只需要在 `$new` 中把新创建的隔离作用域的 `$root` 属性重新指向真正的根作用域即可：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated) {
  // var child;
  if (isolated) {
  //   child = new Scope();
    child.$root = this.$root;
  } else {
    // var ChildScope = function() {};
    // ChildScope.prototype = this;
    // child = new ChildScope();
  }
  // this.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  // return child;
};
```

在继续了解与继承相关的内容之前，我们还需要修复在隔离作用域存在的一个问题，这个问题与 `$evalAsync`、`$applyAsync` 和 `$$postDigest` 函数中存储的队列有关。回忆一下之前的内容，我们是在 `$digest` 函数中处理 `$$asyncQueue` 和 `$$postDigestQueue` 两个队列，然后在 `$$flushApplyAsync` 中处理 `$$applyAsyncQueue`。这两个函数目前都没有对父子作用域的特性进行额外处理。我们只是简单地假设每个队列都有一个实例，它代表整个层次结构中所有排队的任务。

对于非隔离作用域来说，当我们在任意作用域上访问一个队列时，访问到的队列都是同一个，因为每一个（非隔离）作用域都继承这了个队列。但对隔离作用域就不一样了。与前面提到的 `$root` 类似，隔离作用域创建时，也会初始化 `$asyncQueue`、`$applyAsyncQueue` 和 `$$postDigestQueue` 三个方法，这三个方法会屏蔽掉根作用域上的三个同名队列。这就导致在隔离作用域上使用 `$evalAsync` 或者 `$$postDigest` 设定的延时函数永远不会被执行：

_test/scope\_spec.js_

```js
it('executes $evalAsync functions on isolated scopes', function(done) {
  var parent = new Scope();
  var child = parent.$new(true);

  child.$evalAsync(function(scope) {
    scope.didEvalAsync = true;
  });

  setTimeout(function() {
    expect(child.didEvalAsync).toBe(true);
    done();
  }, 50);
});

it('executes $$postDigest functions on isolated scopes', function() {
  var parent = new Scope();
  var child = parent.$new(true);

  child.$$postDigest(function() {
    child.didPostDigest = true;
  });
  parent.$digest();

  expect(child.didPostDigest).toBe(true);
});
```

跟 `$root` 一样，无论作用域是不是隔离作用域，我们都希望所有作用域共享 `$$asyncQueue` 和 `$$postDigestQueue` 的同一副本。如果作用域不是隔离作用域，它能自动得到一个副本。但如果是隔离作用域，我们就需要显式地进行赋值了：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated) {
  // var child;
  if (isolated) {
    // child = new Scope();
    // child.$root = this.$root;
    child.$$asyncQueue = this.$$asyncQueue;
    child.$$postDigestQueue = this.$$postDigestQueue;
  } else {
    // var ChildScope = function() {};
    // ChildScope.prototype = this;
    // child = new ChildScope();
  // }
  // this.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  // return child;
};
```

`$$applyAsyncQueue` 出现的问题就不太一样了：`$$applyAsyncQueue` 是否执行任务队列是由 `$$applyAsyncId` 控制的，而现在由于出现了隔离作用域，树结构的作用域可能会有自己的 `$$applyAsyncId`，所以实际上会出现多个 `$applyAsync` 进程，每个隔离作用域一个。这就违背了 `$applyAsync` 合并 `$apply` 调用的初衷了。

我们可以利用 `$applyAsyncQueue` 会在 `$digest` 函数中全部执行完毕的这个特性进行测试。如果我们在子作用域调用 `$digest`，父作用域设定的 `$applyAsync` 任务应该会被执行，但目前并不是这样的：

_test/scope\_spec.js_

```js
it("executes $applyAsync functions on isolated scopes", function() {
  var parent = new Scope();
  var child = parent.$new(true);
  var applied = false;

  parent.$applyAsync(function() {
    applied = true;
  });
  child.$digest();

  expect(applied).toBe(true);
});
```

要解决这个问题，我们要像处理 `$evalAsync` 和 `$$postDigest` 任务队列一样，让每个作用域都能访问到（根作用域上的）这个队列：

_src/scope.js_

```js
Scope.prototype.$new = function(isolated) {
  // var child;
  if (isolated) {
    // child = new Scope();
    // child.$root = this.$root;
    // child.$$asyncQueue = this.$$asyncQueue;
    // child.$$postDigestQueue = this.$$postDigestQueue;
    child.$$applyAsyncQueue = this.$$applyAsyncQueue;
  } else {
    // var ChildScope = function() {};
    // ChildScope.prototype = this;
    // child = new ChildScope();
  }
  // this.$$children.push(child);
  // child.$$watchers = [];
  // child.$$children = [];
  // return child;
};
```

其次，我们需要共享 `$$applyAsyncId` 属性。但我们不能直接在 `$new` 函数中拷贝这个属性，因为隔离作用域还要对这个属性进行赋值。我们可以直接通过 `$root` 获取到这个属性值：

> 译者注: 因为 $$applyAsyncId 是一个值类型，而不是引用类型，所以如果在隔离作用域拷贝了这个属性，之后对这个属性进行赋值时，只会影响隔离作用域上的这个属性，不会影响根作用域上的同名属性。

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  // this.$root.$$lastDirtyWatch = null;
  // this.$beginPhase('$digest');

  if (this.$root.$$applyAsyncId) {
    clearTimeout(this.$root.$$applyAsyncId);
    // this.$$flushApplyAsync();
  }

  // do {
  //   while (this.$$asyncQueue.length) {
  //     try {
  //       var asyncTask = this.$$asyncQueue.shift();
  //       asyncTask.scope.$eval(asyncTask.expression);
  //     } catch (e) {
  //       console.error(e);

  //     }
  //     dirty = this.$$digestOnce();
  //     if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
  //       throw '10 digest iterations reached';
  //     }
  //   }
  // } while (dirty || this.$$asyncQueue.length);
  // this.$clearPhase();

  // while (this.$$postDigestQueue.length) {
  //   try {
  //     this.$$postDigestQueue.shift()();
  //   } catch (e) {
  //     console.error(e);
  //   }
  // }
};

Scope.prototype.$applyAsync = function(expr) {
  // var self = this;
  // self.$$applyAsyncQueue.push(function() {
  //   self.$eval(expr);
  // });
  if (self.$root.$$applyAsyncId === null) {
    self.$root.$$applyAsyncId = setTimeout(function() {
      // self.$apply(_.bind(self.$$flushApplyAsync, self));
    }, 0);
  }
};

Scope.prototype.$$flushApplyAsync = function() {
  // while (this.$$applyAsyncQueue.length) {
  //   try {
  //     this.$$applyAsyncQueue.shift()();
  //   } catch (e) {
  //     console.error(e);
  //   }
  // }
  this.$root.$$applyAsyncId = null;
};
```

这样隔离作用域的相关内容就都能正常运行了！

