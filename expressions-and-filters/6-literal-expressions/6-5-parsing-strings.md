### 解析字符串

#### Parsing Strings

解析完数字，我们来看看字符串该怎么解析。字符串处理起来跟数字一样简单，但还有一些特殊情况需要注意。

简单来说，表达式中的字符串就是包裹在单引号或双引号中的字符序列：

_test/parse\_spec.js_

```js
it('can parse a string in single quotes', function() {
  var fn = parse("'abc'");
  expect(fn()).toEqual('abc');
});

it('can parse a string in double quotes', function() {
  var fn = parse('"abc"');
  expect(fn()).toEqual('abc');
});
```

在 `lex` 中，我们可以检测当前字符是否为这两种引号之一，然后执行一个用于读取字符串的函数，稍后我们将实现这个函数：

_src/parse.js_

```js
Lexer.prototype.lex = function(text) {
  // this.text = text;
  // this.index = 0;
  // this.ch = undefined;
  // this.tokens = [];

  // while (this.index < this.text.length) {
  //   this.ch = this.text.charAt(this.index);
  //   if (this.isNumber(this.ch) ||
  //     (this.ch === '.' && this.isNumber(this.peek()))) {
  //     this.readNumber();
    } else if (this.ch === '\'' || this.ch === '"') {
      this.readString();
  //   } else {
  //     throw 'Unexpected next character: ' + this.ch;
  //   }
  // }
  // return this.tokens;
};
```

从顶层结构来看，`readString` 非常类似于 `readNumber` ，它使用 `while` 循环遍历表达式中的字符，然后创建一个局部变量来存放这个字符串，并把它放到一个本地变量中。与 `readNumber` 不同的是，`readString` 在执行 `while` 循环语句之前需要将字符索引往前加 1 位，这样就能跳过字符串开头的引号字符：

_src/parse.js_

```js
Lexer.prototype.readString = function() {
  this.index++;
  var string = '';
  while (this.index < this.text.length) {
    var ch = this.text.charAt(this.index);

    this.index++;
  } 
};
```

那我们到底要在 `while` 循环语句中干什么呢？有两件事：如果当前字符不是引号，我们应该把它拼接到结果字符串中去。如果是引号，说明该字符串已经结束了，我们要把当前拼接好的字符串放到 token 数组中去，然后结束循环。循环语句执行完之后，如果程序还在执行 `readString` 函数，我们要抛出一个异常，因为这意味着表达式都读取完了但字符串还没有被终止：

_src/parse.js_

```js
Lexer.prototype.readString = function() {
  // this.index++;
  // var string = '';
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
    if (ch === '\'' || ch === '"') {
      this.index++;
      this.tokens.push({
        text: string,
        value: string
      });
      return;
    } else
      {
      string += ch;
    }
  //   this.index++;
  // }
  // throw 'Unmatched quote';
};
```

这对于解析字符串来说是一个良好的开始，但是我们还没有完成。单元测试依然还没有通过，因为在 AST 处理结束时，这个 token 会作为一个字面量输出出来，它的值会被原封不动编译为一个 JavaScript 函数中去。表达式 `'abc'` 生成的函数就像下面这样：

```js
function() {
  return abc;
}
```

我们可以发现，字符串两侧的引号没有了，因此函数会去尝试查找名为 `abc` 的变量！

我们的 AST 编译器要对字符串进行_转义_，从而支持在字符串两侧加上应有的引号。我们会新增一个名为 `escape` 的方法来实现转义：

_src/parse.js_

```js
case AST.Literal:
  return this.escape(ast.value);
```

这个方法可以实现在一个变量的两侧加上引号，但这个变量的类型必须是字符串：

_src/parse.js_

```js
ASTCompiler.prototype.escape = function(value) {
  if (_.isString(value)) {
    return '\'' + value + '\'';
  } else {
    return value;
  }
};
```

既然我们用到了 `_.isString`，就得在 `parse.js` 中引入 LoDash 了：

_src/parse.js_

```js
'use strict';

var _ = require('lodash');
```

我们对输入字符串的开始和结束引号也有点太宽松了，因为我们允许字符串使用类型不同的引号结束：

_test/parse\_spec.js_

```js
it('will not parse a string with mismatching quotes', function() {
  expect(function() { parse('"abc\''); }).toThrow();
});
```

我们需要保证标记字符串开始和结束的引号类型要一致才行。为此，在 `lex` 函数调用 `readString` 方法时，我们需要把字符串开头的引号传递进去：

_src/parse.js_

```js
Lexer.prototype.lex = function(text) {
  // this.text = text;
  // this.index = 0;
  // this.ch = undefined;
  // this.tokens = [];
  // while (this.index < this.text.length) {
  //   this.ch = this.text.charAt(this.index);
  //   if (this.isNumber(this.ch) ||
  //     (this.ch === '.' && this.isNumber(this.peek()))) {
  //     this.readNumber();
  //   } else if (this.ch === '\'' || this.ch === '"') {
      this.readString(this.ch);
  //   } else {
  //     throw 'Unexpected next character: ' + this.ch;
  //   }
  // }
  // return this.tokens;
};
```

在 `readString` 中，我们现在可以用传入的引号字符代替字面量 `'` 或 `"`来检查字符串终止：

_src/parse.js_

```js
Lexer.prototype.readString = function(quote) {
  // this.index++;
  // var string = '';
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
    if (ch === quote) {
      // this.index++;
      // this.tokens.push({
      //   text: string,
      //   value: string
      // });
      // return;
  //   } else {
  //     string += ch;
  //   }
  //   this.index++;
  // }
  // throw 'Unmatched quote';
};
```

就像 JavaScript 字符串，Angular 表达式字符串中也会有转义字符。我们需要支持的转义符有两种类型：

1. 单字符转义符：换行符 `\n`，换页符 `\f`，回车符 `\r`，水平制表符 `\t`，垂直制表符 `\v`，单引号 `\'`，还有双引号 `\"`。
2. Unicode 转义序列：这是以 `\u` 开头，后面跟着 4 个十六进制的字符代码。举个例子，`\u00A0` 表示一个不间断空格字符。

我们先来看看如何处理单字符转义符。首先，我们应该能解析包含引号的字符串：

_test/parse\_spec.js_

```js
it('can parse a string with single quotes inside', function() {
  var fn = parse("'a\\\'b'");
  expect(fn()).toEqual('a\'b');
});

it('can parse a string with double quotes inside', function() {
  var fn = parse('"a\\\"b"');
  expect(fn()).toEqual('a\"b');
});
```

我们要做的就是在解析过程中，一旦发现反斜杠 `\`，就进入“转义模式”，这样我们就可以对接下来的字符做特殊处理：

_src/parse.js_

```js
Lexer.prototype.readString = function(quote) {
  // this.index++;
  // var string = '';
  var escape = false;
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
    if (escape) {

    } else if (ch === quote) {
      // this.index++;
      // this.tokens.push({
      //   text: string,
      //   value: string
      // });
      // return;
    } else if (ch === '\\') {
      escape = true;
  //   } else {
  //     string += ch;
  //   }
  //   this.index++;
  // }
  // throw 'Unmatched quote';
};
```

进入转义模式以后，如果我们发现有单字符转义符，就要看一下它属于哪种单字符转义符，然后替换成对应的转义字符。我们会把要支持的转义字符都放到 `parse.js` 顶层作用域的一个对象常量中进行保存。它会包含我们上面单元测试中出现的引号字符：

_src/parse.js_

```js
var ESCAPES = {'n':'\n', 'f':'\f', 'r':'\r', 't':'\t',
               'v':'\v', '\'':'\'', '"':'"'};
```

然后在 `readString` 里，我们会从这个对象中查找是否有这个转义字符。如果有，我们就会替换成对应的转义字符，再把替换后的字符加入到结果字符穿中。如果没有找到，我们就把这个字符原封不动地加入到结果字符串中，直接忽略掉用于转义的反斜杠就好：

_src/parse.js_

```js
Lexer.prototype.readString = function(quote) {
  // this.index++;
  // var string = '';
  // var escape = false;
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
  //   if (escape) {
      var replacement = ESCAPES[ch];
      if (replacement) {
        string += replacement;
      } else {
        string += ch;
      }
      escape = false;
  //   } else if (ch === quote) {
  //     this.index++;
  //     this.tokens.push({
  //       text: string,
  //       value: string
  //     });
  //     return;
  //   } else if (ch === '\\') {
  //     escape = true;
  //   } else {
  //     string += ch;
  //   }
  //   this.index++;
  // }
  // throw 'Unmatched quote';
};
```

在进入 AST 编译阶段之前，我们还需要解决几个问题。当 AST 编译器遇到像 `'` 和 `"` 这样的字面量时，它只会直接把它放到结果中，这样会产出一些非访的 JavaScript 代码。编译器的 `escape` 方法需要能够处理这些字符。我们可以在转义过程中加入一个正则进行转义：

_src/parse.js_

```js
ASTCompiler.prototype.escape = function(value) {
  // if (_.isString(value)) {
    return '\'' +
      value.replace(this.stringEscapeRegex, this.stringEscapeFn) +
      '\'';
  // } else {
  //   return value;
  // }
};
```

需要进行转义，是除了空格、字母和数字以外的字符：

_src/parse.js_

```js
ASTCompiler.prototype.stringEscapeRegex = /[^ a-zA-Z0-9]/g;
```

而在用于替换的函数中，我们会获取要转义字符的 Unicode 数字代码（利用 [charCodeAt](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/String/charCodeAt)），然后把它转成相对应的十六进制值（以 16 为基数）的 Unicode 转义序列，然后我们就可以安全地把这哥序列加入到要生成的 JavaScript 代码中：

_src/parse.js_

```js
ASTCompiler.prototype.stringEscapeFn = function(c) {
  return '\\u' + ('0000' + c.charCodeAt(0).toString(16)).slice(-4);
};
```

最后，我们要考虑一下输入表达式本身含有转义序列的情况：

_test/parse\_test.js_

```js
it('will parse a string with unicode escapes', function() {
  var fn = parse('"\\u00A0"');
  expect(fn()).toEqual('\u00A0');
});
`
```

我们需要做的就是检查一下紧接着反斜杠的字符是不是 `u`，是的话，就把后面的 4 个字符都抓取出来，把它们转换为 16 进制的数字，然后查找对应数字代码的字符。根据数字代码查找字符，我们可以利用 JavaScript 内建的 `String.fromCharCode` 函数：

_src/parse.js_

```js
Lexer.prototype.readString = function(quote) {
  // this.index++;
  // var string = '';
  // var escape = false;
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
  //   if (escape) {
      if (ch === 'u') {
        var hex = this.text.substring(this.index + 1, this.index + 5);
        this.index += 4;
        string += String.fromCharCode(parseInt(hex, 16));
      } else {
        // var replacement = ESCAPES[ch];
        // if (replacement) {
        //   string += replacement;
        // } else {
        //   string += ch;
        // }
      }
  //     escape = false;
  //   } else if (ch === quote) {
  //     this.index++;
  //     this.tokens.push({
  //       text: string,
  //       value: string
  //     });
  //     return;
  //   } else if (ch === '\\') {
  //     escape = true;
  //   } else {
  //     string += ch;
  //   }
  //   this.index++;
  // }
  // throw 'Unmatched quote';
};
```

最后，我们还要考虑紧跟着 `\u` 的字符编码是非法的情况。如果出现这种情况，我们应该抛出一个异常：

_test/parse_spec.js_

```js
it('will not parse a string with invalid unicode escapes', function() {
  expect(function() { parse('"\\u00T0"'); }).toThrow();
});
```

我们会使用一个正则表达式来检验这 4 个字符是否都是由数字和字母 a-f 组成，也就是验证它们是否合法的十六进制数字。由于 unicode 转义序列都是不区分大小写的，所以我们也要同时支持传入大小写字母：

_src/parse.js_

```js
Lexer.prototype.readString = function(quote) {
  // this.index++;
  // var string = '';
  // var escape = false;
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
  //   if (escape) {
  //     if (ch === 'u') {
        var hex = this.text.substring(this.index + 1, this.index + 5);
          if (!hex.match(/[\da-f]{4}/i)) {
            throw 'Invalid unicode escape';
          }
  //         this.index += 4;
  //       string += String.fromCharCode(parseInt(hex, 16));
  //     } else {
  //       var replacement = ESCAPES[ch];
  //       if (replacement) {
  //         string += replacement;
  //       } else {
  //         string += ch;
  //       }
  //     }
  //     escape = false;
  //   } else if (ch === quote) {
  //     this.index++;
  //     this.tokens.push({
  //       text: string,
  //       value: string
  //     });
  //     return;
  //   } else if (ch === '\\') {
  //     escape = true;
  //   } else {
  //     string += ch;
  //   }
  //   this.index++;
  // }
  // throw 'Unmatched quote';
};
```

这样，我们就能解析字符串了！
