### 在调用函数时启用安全策略
#### Ensuring Safe Functions

我们刚刚看到，如果一个表达式里调用的函数恰好是函数构造函数，就不能调用。Angular 还会阻止你对函数做另一件事，那就是把函数的接受者（`this`）绑定到其他对象上：

_test/parse_spec.js_

```js
it('does not allow calling call', function() {
  var fn = parse('fun.call(obj)');
  expect(function() { fn({ fun: function() {}, obj: {} }); }).toThrow();
});

it('does not allow calling apply', function() {
  var fn = parse('fun.apply(obj)');
  expect(function() { fn({ fun: function() {}, obj: {} }); }).toThrow();
});
```

`call` 和 `apply`（还有 `bind`）方法都是调用函数的其他途径，因此其接收方与原始方不同。在上面两个测试用例中，`fun`函数的 `this` 会被强制绑定到 `obj` 上去。由于重新绑定 `this` 可能会让函数的行为变得跟函数作者最初的意图不一致，Angular 为了避免攻击者利用它们来进行注入攻击，会直接禁用这些方法。

在之前的章节中，我们使用 `ensureSafeObject` 来确保函数调用的安全性。现在我们需要换一个新函数 `ensureSafeFunction` 来对函数调用进行检测。这个函数要做第一件事是检查这个函数是否位函数构造器，跟 `ensureSafeObject` 一样：

_src/parse.js_

```js
function ensureSafeFunction(obj) {
  if (obj) {
    if (obj.constructor === obj) {
      throw 'Referencing Function in Angular expressions is disallowed!';
    }
  }
  return obj;
}
```

`ensureSafeFunction` 的第二个职责是看函数是否不是 `call`，`apply` 或 `bind`。我们会在 `parse.js` 中引入它们作为常量，这样我们就可以与传入的函数进行比较：

_src/parse.js_

```js
var CALL = Function.prototype.call;
var APPLY = Function.prototype.apply;
var BIND = Function.prototype.bind;
```

现在我们可以在 `ensureSafeFunction` 中引用这些常量：

_src/parse.js_

```js
function ensureSafeFunction(obj) {
  // if (obj) {
  //   if (obj.constructor === obj) {
  //     throw 'Referencing Function in Angular expressions is disallowed!';
    } else if (obj === CALL || obj === APPLY || obj === BIND) {
      throw 'Referencing call, apply, or bind in Angular expressions ' +
        'is disallowed!';
    }
  // }
  // return obj;
}
```

接下来，我们需要在运行时访问到 `ensureSafeFunction`：

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
    // 'ensureSafeMemberName',
    // 'ensureSafeObject',
    'ensureSafeFunction',
    // fnString)(
    // ensureSafeMemberName,
    // ensureSafeObject,
    ensureSafeFunction);
  /* jshint +W054 */
};
```

然后，我们就可以生成对函数调用表达式的调用：

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
  this.addEnsureSafeFunction(callee);
  // return callee + '&&ensureSafeObject(' + callee + '(' + args.join(',') + '))';
```

当然，我们还要先加入 `addEnsureSafeFunction` 辅助函数：

_src/parse.js_

```js
ASTCompiler.prototype.addEnsureSafeFunction = function(expr) {
  this.state.body.push('ensureSafeFunction(' + expr + ');');
};
```

这就是 Angular 规避注入攻击的方式。虽然这些安全措施谈不上十全十美，而且几乎可以肯定还有几种方法可以在作用域上添加危险的属性。但是，在攻击者可以利用这些危险属性的前提是，应用程序开发人员必须首先将这些危险属性放到作用域中，这大大降低了与此相关的风险，而他们也不应该这样做。