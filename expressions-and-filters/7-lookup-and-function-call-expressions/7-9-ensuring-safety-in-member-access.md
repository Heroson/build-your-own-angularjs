### 在访问对象成员时启用安全策略
#### Ensuring Safety In Member Access

由于表达式一般会被嵌入到 HTML 中，而且嵌入的内容很可能会加入用户生成内容（user-generated content），也就是说，用户有可能通过生成特定表达式的方法来达到在程序中执行任意代码的效果，因此防止注入攻击是我们的重中之重。对此产生的防护措施主要基于一个原理，我们可以把所有表达式的上下文严格限制在 Scope 对象以内：除了字面量，我们只能访问添加在 scope 对象上的数据（或者 locals 上的数据）。存在安全隐患的对象，比如 `window`，我们会直接禁止访问。

在我们当前已经实现的代码中有几种方法可以解决这个问题：尤其是，在本章中，我们已经看到 JavaScript 的 `Function` 构造器是如何接收一个字符串，然后把这个字符串作为一个新函数的源代码。我们会使用这个新函数来生成表达式函数。事实证明，如果我们不采取限制，表达式确实可以利用同一个 Function 构造函数来运行任意的代码。

由于每一个 JavaScript 函数的 `constructor` 属性都指向 Function 构造函数，这让攻击者有机可乘。假如作用域上有一个函数（这很常见），我们可以在表达式中访问到它的构造函数，只要给它传递一段 JavaScript 代码，然后执行生成的函数。到这时，一切就完了。比如，我们可以轻易地拿到全局的 `window` 对象：

```js
aFunction.constructor('return window;')()
```

除了 Function 构造函数，还有一些常见的对象也可能会引发安全问题，允许访问它们可能会产生难以预测的影响：

- `__proto__` 是一个[非标准的、已被弃用的对象原型访问器](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/proto)。它不仅允许读取原型，还可以对原型进行设置，这种特性使得它也存在潜在危险。

- `__defineGetter__, __lookupGetter__, __defineSetter__,`and`__lookupSetter__` 是[根据 getter 和 setter 函数来对对象属性进行定义的非标准函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/__defineGetter__)。由于它们并不是标准的 API，并不被所有的浏览器支持，且它们也有重新定义全局属性的可能性，Angular 会直接在表达式中禁止访问这些函数。

下面我们来加入单元测试，确保在表达式中不能访问以上六个成员：

_test/parse_spec.js_

```js
it('does not allow calling the function constructor', function() {
  expect(function() {
    var fn = parse('aFunction.constructor("return window;")()');
    fn({ aFunction: function() {} });
  }).toThrow();
});

it('does not allow accessing __proto__', function() {
  expect(function() {
    var fn = parse('obj.__proto__');
    fn({ obj: {} });
  }).toThrow();
});

it('does not allow calling __defineGetter__', function() {
  expect(function() {
    var fn = parse('obj.__defineGetter__("evil", fn)');
    fn({ obj: {}, fn: function() {} });
  }).toThrow();
});

it('does not allow calling __defineSetter__', function() {
  expect(function() {
    var fn = parse('obj.__defineSetter__("evil", fn)');
    fn({ obj: {}, fn: function() {} });
  }).toThrow();
});

it('does not allow calling __lookupGetter__', function() {
  expect(function() {
    var fn = parse('obj.__lookupGetter__("evil")');
    fn({ obj: {} });
  }).toThrow();
});

it('does not allow calling __lookupSetter__', function() {
  expect(function() {
    var fn = parse('obj.__lookupSetter__("evil")');
    fn({ obj: {} });
  }).toThrow();
});
```

针对这类攻击，我们将要采取的安全措施是禁止访问对象上任何具有上述成员名称的属性。要实现这个效果，我们要引入一个帮助函数，这个函数可以检查对象成员名称，若有发现名称对应的对象成员已被禁止访问，就会抛出一个异常：

_src/parse.js_

```js
function ensureSafeMemberName(name) {
  if (name === 'constructor' || name === '__proto__' ||
    name === '__defineGetter__' || name === '__defineSetter__' ||
    name === '__lookupGetter__' || name === '__lookupSetter__') {
    throw 'Attempting to access a disallowed field in Angular expressions!';
  }
}
```

现在我们需要在 AST 编译器几处地方使用这个函数。在针对标识符（identifier）的处理分支中，我们会把标识符的名称作为这个函数的参数进行调用：

_src/parse.js_

```js
case AST.Identifier:
  ensureSafeMemberName(ast.name);
// ...
```

在非计算属性访问中，我们会对属性名称进行检查：

_src/parse.js_

```js
case AST.Identifier:
  // ensureSafeMemberName(ast.name);
  // // ...
  // intoId = this.nextId();
  // var left = this.recurse(ast.object, undefined, create);
  // if (context) {
  //   context.context = left;
  // }
  // if (ast.computed) {
  //   var right = this.recurse(ast.property);
  //   if (create) {
  //     this.if_(this.not(this.computedMember(left, right)), this.assign(this.computedMember(left, right), '{}'));
  //   }
  //   this.if_(left,
  //     this.assign(intoId, this.computedMember(left, right)));
  //   if (context) {
  //     context.name = right;
  //     context.computed = true;
  //   }
  // } else {
    ensureSafeMemberName(ast.property.name);
  //   if (create) {
  //     this.if_(this.not(this.nonComputedMember(left, ast.property.name)),
  //       this.assign(this.nonComputedMember(left, ast.property.name), '{}'));
  //   }
  //   this.if_(left,
  //     this.assign(intoId, this.nonComputedMember(left, ast.property.name)));
  //   if (context) {
  //     context.name = ast.property.name;
  //     context.computed = false;
  //   }
  // }
  // return intoId;
```

在对计算属性成员进行访问时，我们需要做一些额外的工作，因为我们在解析时还不知道属性名称。当需要对表达式进行求值时，我们要在运行时调用 `ensureSafeMemberName`。

首先，我们需要确保在运行时表达式可以访问到 `ensureSafeMemberName`。首先，我们需要生成的函数代码进行重构，让它不再是表达式函数本身，而是一个会返回表达式函数的函数：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = { body: [], nextId: 0, vars: [] };
  // this.recurse(ast);
  var fnString = 'var fn=function(s,l){' + (this.state.vars.length ?
    'var ' + this.state.vars.join(',') + ';' : ''
  ) + this.state.body.join('') + '}; return fn;';
  /* jshint -W054 */
  return new Function(fnString)();
  /* jshint +W054 */
};
```

有了这个高阶函数以后，我们可以传递一些参数给它，这样生成的代码可以通过闭包访问到这些参数。这时，我们就可以传递 `ensureSafeMemberName` 作为它的参数，这样表达式函数就可以在运行时用上这个函数：

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
  return new Function('ensureSafeMemberName', fnString)(ensureSafeMemberName);
  /* jshint +W054 */
```

下一步，我们会对计算属性访问表达式的右侧内容生成一个对这个函数的调用：

```js
case AST.MemberExpression:
  // intoId = this.nextId();
  // var left = this.recurse(ast.object, undefined, create);
  // if (context) {
  //   context.context = left;
  // }
  // if (ast.computed) {
  //   var right = this.recurse(ast.property);
    this.addEnsureSafeMemberName(right);
  //   if (create) {
  //     this.if_(this.not(this.computedMember(left, right)),
  //       this.assign(this.computedMember(left, right), '{}'));
  //   }
  //   this.if_(left,
  //     this.assign(intoId, this.computedMember(left, right)));
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
  //     this.assign(intoId, this.nonComputedMember(left, ast.property.name)));
  //   if (context) {
  //     context.name = ast.property.name;
  //     context.computed = false;
  //   }
  // }
  // return intoId;
```

`addEnsureSafeMemberName` 函数还未定义好。它会生成一个对 `ensureSafeMemberName` 的调用：

_src/parse.js_

```js
ASTCompiler.prototype.addEnsureSafeMemberName = function(expr) {
  this.state.body.push('ensureSafeMemberName(' + expr + ');');
};
```