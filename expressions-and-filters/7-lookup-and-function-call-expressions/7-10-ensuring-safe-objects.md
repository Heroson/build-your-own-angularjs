### 在访问对象时启用安全策略
#### Ensuring Safe Objects

Angular 表达式提供的第二个安全措施更多的是为了保护应用开发者免受自己的攻击，具体来说就是不让他们在作用域上添加危险的对象，然后在表达式访问它们。

其中一个危险的对象就是 window。调用 `window` 上的某些函数可能会带来巨大的破坏，所以 Angular 将完全阻止你在表达式中使用它。当然，你不能直接调用 `window` 上的成员，因为表达式只能在作用域上下文范围内进行访问，但需要保证 `window` 不会变成作用域上的一个同名属性。如果你试图这样做，就会抛出一个异常：

_test/parse_spec.js_

```js
it('does not allow accessing window as computed property', function() { var fn = parse('anObject["wnd"]');
  expect(function() { fn({anObject: {wnd: window}}); }).toThrow();
});

it('does not allow accessing window as non-computed property', function() { var fn = parse('anObject.wnd');
  expect(function() { fn({anObject: {wnd: window}}); }).toThrow();
});
```

相应的安全措施就是当我们处理对象时，需要确保它们里面不包含危险的对象。为此，我们先加入一个帮助函数：

_src/parse.js_

```js
function ensureSafeObject(obj) {
  if (obj) {
    if (obj.window === obj) {
    throw 'Referencing window in Angular expressions is disallowed!';
    }
  }
  return obj;
}
```

我们检查对象是否不包含 window 的方式是看这个对象是否有一个 `window` 属性，并且这个属性指向自身——在 JavaScript 中，这种特性只有 `window` 对象有，其他对象没有。

要让测试通过，我们需要先在运行时加入这个新的帮助函数：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = { body: [], nextId: 0, vars: [] };
  // this.recurse(ast);
  // var fnString = 'var fn=function(s,l){' + (this.state.vars.length ?
  //   'var ' + this.state.vars.join(',') + ';' :
  //   ''
  // ) + this.state.body.join('') + '}; return fn;';
  /* jshint -W054 */
  return new Function(
    'ensureSafeMemberName',
    'ensureSafeObject',
    fnString)(
      ensureSafeMemberName,
      ensureSafeObject);
  /* jshint +W054 */
};
```

现在，我们就可以对成员访问的结果调用 `ensureSafeObject` 了：

_src/parse.js_

```js
case AST.MemberExpression:
  // intoId = this.nextId();
  // var left = this.recurse(ast.object, undefined, create);
  // if (context) {
  //   context.context = left;
  // }
  // if (ast.computed) {
  //   var right = this.recurse(ast.property);
  //   this.addEnsureSafeMemberName(right);
  //   if (create) {
  //     this.if_(this.not(this.computedMember(left, right)), this.assign(this.computedMember(left, right), '{}'));
  //   }
  //   this.if_(left,
  //     this.assign(intoId,
        'ensureSafeObject(' + this.computedMember(left, right) + ')'));
  //   if (context) {
  //     context.name = right;
  //     context.computed = true;
  //   }
  // } else {
  //   ensureSafeMemberName(ast.property.name);
  //   if (create) {
  //     this.if_(this.not(this.nonComputedMember(left, ast.property.name)), this.assign(this.nonComputedMember(left, ast.property.name), '{}'));
  //   }
  //   this.if_(left,
  //     this.assign(intoId,
        'ensureSafeObject(' +
        this.nonComputedMember(left, ast.property.name) + ')'));
  //   if (context) {
  //     context.name = ast.property.name;
  //     context.computed = false;
  //   }
  // }
  // return intoId;
```

另外，我们也不允许把存在安全隐患的对象作为函数参数传入：

_test/parse_spec.js_

```js
it('does not allow passing window as function argument', function() {
  var fn = parse('aFunction(wnd)');
  expect(function() {
    fn({aFunction: function() { }, wnd: window});
  }).toThrow();
});
```

我们需要把每一个函数表达式包裹到 `ensureSafeObject` 中：

_src/parse.js_

```js
case AST.CallExpression:
  // var callContext = {};
  // var callee = this.recurse(ast.callee, callContext);
  var args = _.map(ast.arguments, _.bind(function(arg) {
    return 'ensureSafeObject(' + this.recurse(arg) + ')';
  }, this));
  // ...
```

假如 `window` 被挂载到 scope 上时，我们也不允许调用 `window` 上的函数：

_test/parse_spec.js_

```js
it('does not allow calling methods on window', function() { var fn = parse('wnd.scrollTo(0)');
  expect(function() {
    fn({wnd: window});
  }).toThrow();
});
```

在这种情况下，我们需要检查方法调用的上下文：

_src/parse.js_

```js
case AST.CallExpression:
  // var callContext = {};
  // var callee = this.recurse(ast.callee, callContext);
  // var args = _.map(ast.arguments, _.bind(function(arg) {
  //   return 'ensureSafeObject(' + this.recurse(arg) + ')';
  // }, this));
  // if (callContext.name) {
    this.addEnsureSafeObject(callContext.context);
  //   if (callContext.computed) {
  //     callee = this.computedMember(callContext.context, callContext.name);
  //   } else {
  //     callee = this.nonComputedMember(callContext.context, callContext.name);
  //   }
  // }
  // return callee + '&&' + callee + '(' + args.join(',') + ')';
```

`addEnsureSafeObject` 是一个新函数，我们得先定义好：

_src/parse.js_

```js
ASTCompiler.prototype.addEnsureSafeObject = function(expr) {
  this.state.body.push('ensureSafeObject(' + expr + ');');
};
```

我们不仅要禁止调用 `window` 上的方法，还要禁止调用会返回 `window` 的函数：

_src/parse.js_

```js
it('does not allow functions to return window', function() {
  var fn = parse('getWnd()');
  expect(function() { fn({ getWnd: _.constant(window) }); }).toThrow();
});
```

这次我们就需要把函数调用的结果值包裹到 `ensureSafeObject` 里去了：

_src/parse.js_

```js
case AST.CallExpression:
  // var callContext = {};
  // var callee = this.recurse(ast.callee, callContext);
  // var args = _.map(ast.arguments, _.bind(function(arg) {
  //   return 'ensureSafeObject(' + this.recurse(arg) + ')';
  // }, this));
  // if (callContext.name) {
  //   this.addEnsureSafeObject(callContext.context);
  //   if (callContext.computed) {
  //     callee = this.computedMember(callContext.context, callContext.name);
  //   } else {
  //     callee = this.nonComputedMember(callContext.context, callContext.name);
  //   }
  // }
  return callee + '&&ensureSafeObject(' + callee + '(' + args.join(',') + '))';
```

我们不允许把一个不安全的对象添加到 scope 上：

_src/parse.js_

```js
it('does not allow assigning window', function() {
  var fn = parse('wnd = anObject');
  expect(function() {
    fn({ anObject: window });
  }).toThrow();
});
```

这意味着，我们也需要对赋值语句的右侧内容进行包裹：

_src/parse.js_

```js
case AST.AssignmentExpression:
  // var leftContext = {};
  // this.recurse(ast.left, leftContext, true);
  // var leftExpr;
  // if (leftContext.computed) {
  //   leftExpr = this.computedMember(leftContext.context, leftContext.name);
  // } else {
  //   leftExpr = this.nonComputedMember(leftContext.context, leftContext.name);
  // }
  // return this.assign(leftExpr,
    'ensureSafeObject(' + this.recurse(ast.right) + ')');
```

最后，我们不允许通过标识符的方式把不安全的对象直接挂载在到 scope 上：

_test/parse_spec.js_

```js
it('does not allow referencing window', function() {
  var fn = parse('wnd');
  expect(function() {
    fn({ wnd: window });
  }).toThrow();
});
```

我们需要为标识符也生成安全检查：

_src/parse.js_

```js
case AST.Identifier:
  // ensureSafeMemberName(ast.name);
  // intoId = this.nextId();
  // this.if_(this.getHasOwnProperty('l', ast.name),
  //   this.assign(intoId, this.nonComputedMember('l', ast.name)));
  // if (create) {
  //   this.if_(this.not(this.getHasOwnProperty('l', ast.name)) + ' && s && ' +
  //     this.not(this.getHasOwnProperty('s', ast.name)), this.assign(this.nonComputedMember('s', ast.name), '{}'));
  // }
  // this.if_(this.not(this.getHasOwnProperty('l', ast.name)) + ' && s',
  //   this.assign(intoId, this.nonComputedMember('s', ast.name)));
  // if (context) {
  //   context.context = this.getHasOwnProperty('l', ast.name) + '?l:s';
  //   context.name = ast.name;
  //   context.computed = false;
  // }
  this.addEnsureSafeObject(intoId);
  // return intoId;
```

这样我们就囊括所有的情况了。但事实上，`window` 并不是唯一一个我们需要注意的“危险”对象。另一个类似的对象是 DOM 元素。如果攻击着可以访问到 DOM 元素，就可能利用它来遍历并修改网页内容，因此我们也需要禁止对 DOM 元素的访问：

_test/parse_spec.js_

```js
it('does not allow calling functions on DOM elements', function() {
  var fn = parse('el.setAttribute("evil", "true")');
  expect(function() { fn({el: document.documentElement}); }).toThrow();
});
```

AngularJS 会使用下面这个检查条件来检查表达式是否是“无 DOM 元素的”：

_src/parse.js_

```js
function ensureSafeObject(obj) {
  // if (obj) {
  //   if (obj.window === obj) {
  //     throw 'Referencing window in Angular expressions is disallowed!';
    } else if (obj.children &&
      (obj.nodeName || (obj.prop && obj.attr && obj.find))) {
      throw 'Referencing DOM nodes in Angular expressions is disallowed!';
  //   }
  // }
  // return obj;
}
```

第三种危险对象是我们的老朋友，函数构造器。虽然我们已经能保证使用者无法通过函数 `constructor` 属性来访问到构造器，但他们依然可以为函数构造器定义一个别名：

_src/parse.js_

```js
it('does not allow calling the aliased function constructor', function() {
  var fn = parse('fnConstructor("return window;")');
  expect(function() {
    fn({ fnConstructor: (function() {}).constructor });
  }).toThrow();
});
```

对这种的情况的处理要比对 `window` 和 DOM 元素的处理简单得多：函数的 `constructor` 属性也是一个函数，所以它也会有一个 `contructor` 属性指向自身。

_src/parse.js_

```js
function ensureSafeObject(obj) {
  // if (obj) {
  //   if (obj.window === obj) {
  //     throw 'Referencing window in Angular expressions is disallowed!';
  //   } else if (obj.children &&
  //     (obj.nodeName || (obj.prop && obj.attr && obj.find))) {
  //     throw 'Referencing DOM nodes in Angular expressions is disallowed!';
    } else if (obj.constructor === obj) {
      throw 'Referencing Function in Angular expressions is disallowed!';
  //   }
  // }
  // return obj;
}
```

第四个，也是最后一个我们要关注的危险对象就是 `Object` 这个对象。除了作为原始对象的构造函数以外，它还包含许多辅助函数，比如 `Object.defineProperty(), Object.freeze(), Object.getOwnPropertyDescriptor() 和 Object.setPrototypeOf()`。我们要关心的正是 `Object` 的后一种角色。如果攻击者可以访问到这些函数的其中之一，就可能会成为危险之源。因此，我们会完全禁止对 _Object_ 的访问：

_test/parse_spec.js_

```js
it('does not allow calling functions on Object', function() {
  var fn = parse('obj.create({})');
  expect(function() {
    fn({ obj: Object });
  }).toThrow();
});
```

我们只需要检查这个对象是否 `Object` 就可以了：

_src/parse.js_

```js
function ensureSafeObject(obj) {
  // if (obj) {
  //   if (obj.window === obj) {
  //     throw 'Referencing window in Angular expressions is disallowed!';
  //   } else if (obj.children &&
  //     (obj.nodeName || (obj.prop && obj.attr && obj.find))) {
  //     throw 'Referencing DOM nodes in Angular expressions is disallowed!';
  //   } else if (obj.constructor === obj) {
  //     throw 'Referencing Function in Angular expressions is disallowed!';
    } else if (obj === Object) {
      throw 'Referencing Object in Angular expressions is disallowed!';
  //   }
  // }
  // return obj;
}
```