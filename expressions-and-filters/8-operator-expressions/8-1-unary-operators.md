### 一元运算符
#### Unary Operators

我们将从优先级最高的的运算符开始，然后按照优先级递减的顺序向下进行。首先要介绍的是我们已经实现的 primary 表达式：每当有函数被调用或者有计算型或非计算型的属性访问时，它们都会被最先进行计算。而在 primary 表达式之后的是一元运算符。

一元运算符就是只有一个操作数的运算符：

- `-` 会把它的操作数取反，比如 `-42` 或者 `-a`
- `+` 实际上没做什么，但它可以用于强调或让代码更清晰，比如 `+42` 或者 `+a`
- 非运算符会对操作数的布尔值进行取反操作，比如 `!true` 或 `!a`

由于 `+` 运算符最简单，因为下面我们先从它开始讲起，它会原封不动地返回操作数：

_test/parse_spec.js_

```js
it('parses a unary +', function() {
  expect(parse('+42')()).toBe(42);
  expect(parse('+a')({ a: 42 })).toBe(42);
});
```

在 AST 构建器中，我们会加入一个名为 `unary` 的新方法来处理一元元算符，如果当前处理的字符不是一元运算符，它就会回退到使用 `primary` 进行处理：

_src/parse.js_

```js
AST.prototype.unary = function() {
  if (this.expect('+')) {

  } else {
  return this.primary();
  }
};
```

`unary` 方法实际上会构建一个 `UnaryExpression` 的 token，它唯一的参数将会是一个 primary 表达式：

_src/parse.js_

```js
AST.prototype.unary = function() {
  // if (this.expect('+')) {
    return {
      type: AST.UnaryExpression,
      operator: '+',
      argument: this.primary()
    };
  // } else {
  //   return this.primary();
  // }
};
```

`UnaryExpression` 是一个新的 AST 节点类型：

_src/parse.js_

```js
// AST.Program = 'Program';
// AST.Literal = 'Literal';
// AST.ArrayExpression = 'ArrayExpression';
// AST.ObjectExpression = 'ObjectExpression';
// AST.Property = 'Property';
// AST.Identifier = 'Identifier';
// AST.ThisExpression = 'ThisExpression';
// AST.LocalsExpression = 'LocalsExpression';
// AST.MemberExpression = 'MemberExpression';
// AST.CallExpression = 'CallExpression';
// AST.AssignmentExpression = 'AssignmentExpression';
AST.UnaryExpression = 'UnaryExpression';
```

要让一元元算符表达式可以被真正解析，我们还需要在某处地方调用 `unary` 方法。我们会在 `assignment` 中调用 `unary`，也就是我们之前调用 `primary` 的地方。赋值表达式的左右两边不一定是 primary 表达式，也有可能是一元表达式：

_src/parse.js_

```js
AST.prototype.assignment = function() {
  var left = this.unary();
  // if (this.expect('=')) {
    var right = this.unary();
  //   return { type: AST.AssignmentExpression, left: left, right: right };
  // }
  // return left;
};
```

由于 `unary` 可以回退到 `primary`，`assignment` 现在就可以同时兼容它们二者了。

然后我们需要看看 Lexer 需要做些什么事情了。我们的 AST 构建器已经做好处理一元运算符 `+` 的准备了，但 Lexer 还没有发出（emit）这个符号。

在前面的章节中，我们已经处理过几个要发出纯文本字符 token 的情况了，比如 `[, ]` 和 `.`。而对于运算符来说 ，我们需要进行不同的处理：我们需要增加一个“常量”对象 `OPERATORS`，里面会包含所有我们考虑到的运算符：

_src/parse.js_

```js
var OPERATORS = {
  '+': true
};
```

这个对象中的所有属性的值都会是 `true`。我们使用对象而不是数组的原因是，对于常量，对象能让我们更有效滴检查属性是否存在。

Lexer 仍然需要发出 `+`。我们需要在 `lex` 方法中加入最后一个 `else` 分支，这可以让我们在 `OPERATORS` 对象中尝试找到当前字符是否是其中一个运算符：

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
  //     (this.is('.') && this.isNumber(this.peek()))) {
  //     this.readNumber();
  //   } else if (this.is('\'"')) {
  //     this.readString(this.ch);
  //   } else if (this.is('[],{}:.()=')) {
  //     this.tokens.push({
  //       text: this.ch
  //     });
  //     this.index++;
  //   } else if (this.isIdent(this.ch)) {
  //     this.readIdent();
  //   } else if (this.isWhitespace(this.ch)) {
  //     this.index++;
  //   } else {
      var op = OPERATORS[this.ch];
      if (op) {
        this.tokens.push({ text: this.ch });
        this.index++;
      } else {
        throw 'Unexpected next character: ' + this.ch;
      }
  //   }
  // }
  
  // return this.tokens;
};
```

基本上，`lex` 现在尝试将该字符与它知道的所有信息进行匹配，如果其他 else 分支都失效的话，就会查看 `OPERATORS` 对象中到底有没有包含这个字符。

最后，我们现在要知道 AST 编译器是如何处理一元运算符的。我们可以返回一个 JavaScript 片段，它由表达式的操作符和对参数的递归求值组成：

_src/parse.js_

```js
case AST.UnaryExpression:
  return ast.operator + '(' + this.recurse(ast.argument) + ')';
```

Angular 里面的一元算术运算符跟 JavaScript 原有的一元运算符是不一样的，Angular 的一元运算符会把 undefined 的值当作 0，而 JavaScript 会把它当作 `NaN`：

_test/parse_spec.js_

```js
it('replaces undefined with zero for unary +', function() { expect(parse('+a')({})).toBe(0);
});
```

我们会使用一个名为 `ifDefined` 的新方法来处理一元运算符表达式，这个方法会接收两个参数：一个表达式，另一个就是在值为 undefined 时要替换的值。在这种情况下，这个替换值就是 `0`：

_src/parse.js_

```js
case AST.UnaryExpression:
  return ast.operator +
    '(' + this.ifDefined(this.recurse(ast.argument), 0) + ')';
```

`ifDefined`方法会根据传入的两个参数，生成一个在运行时调用 `ifDefined` 函数的 JavaScript 表达式：

_src/parse.js_

```js
ASTCompiler.prototype.ifDefined = function(value, defaultValue) {
  return 'ifDefined(' + value + ',' + this.escape(defaultValue) + ')';
};
```

这个函数会被传递到要生成的 JavaScript 函数中去：

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
  // /* jshint -W054 */
  // return new Function(
  //   'ensureSafeMemberName',
  //   'ensureSafeObject',
  //   'ensureSafeFunction',
    'ifDefined',
    // fnString)(
    // ensureSafeMemberName,
    // ensureSafeObject,
    // ensureSafeFunction,
    ifDefined);
  /* jshint +W054 */
};
```

最后，如果 `ifDefined` 返回的实际值已经定义了，那就会直接返回这个值本身，否则就返回默认值：

_src/parse.js_

```js
function ifDefined(value, defaultValue) {
  return typeof value === 'undefined' ? defaultValue : value;
}
```

> 这里我们不用 LoDash 的原因是，LoDash 在运行时可能还未能被访问。

这样我们就能顺利处理 `+` 了。我们来介绍下一个一元运算符，它更有趣，因为它确实（对运算结果）产生了影响：

_test/parse_spec.js_

```js
it('parses a unary !', function() {
  expect(parse('!true')()).toBe(false);
  expect(parse('!42')()).toBe(false);
  expect(parse('!a')({a: false})).toBe(true);
  expect(parse('!!a')({a: false})).toBe(false);
});
```

Angular 表达式中非运算符与 JavaScript 保持一致。上面测试中的最后一个验证（expection）就展示了我们可以连续使用多个非运算符。

我们需要把这个运算符也加入到 `OPERATORS` 中：

_src/parse.js_

```js
var OPERATORS = {
  '+': true,
  '!': true
};
```

在 AST 构建器的 `unary` 方法中，我们要可以处理 `+` 或 `!`。同时，我们要用实际传入的运算符代替之前硬编码进去的 `+`：

_src/parse.js_

```js
AST.prototype.unary = function() {
  var token;
  if ((token = this.expect('+', '!'))) {
    return {
      type: AST.UnaryExpression,
      operator: token.text,
      argument: this.primary()
    };
  } else {
    return this.primary();
  }
};
```

这可以部分修复我们的测试用例，我们不需要对 AST 编译器做任何修改就可以实现这个功能。但这个测试依然无法通过，因为我们还不支持在一行里连续使用 `!`。解决办法是把另一个一元表达式作为 `unary` 的参数：

_src/parse.js_

```js
AST.prototype.unary = function() {
  // var token;
  // if ((token = this.expect('+', '!'))) {
  //   return {
  //     type: AST.UnaryExpression,
  //     operator: token.text,
      argument: this.unary()
  //   };
  // } else {
  //   return this.primary();
  // }
};
```

第三个，也是最后一个我们要支持的一元运算符是 `-`，用于对数字进行取反操作：

_test/parse_spec.js_

```js
it('parses a unary -', function() {
  expect(parse('-42')()).toBe(-42);
  expect(parse('-a')({ a: -42 })).toBe(42);
  expect(parse('--a')({ a: -42 })).toBe(-42);
  expect(parse('-a')({})).toBe(0);
});
```

首先我们还是得把这个运算符加入到 `OPERATORS` 对象中去：

_src/parse.js_

```js
var OPERATORS = {
  '+': true,
  '!': true,
  '-': true
};
```

然后我们要在 AST 构建器的 unary 方法允许接收这个符号：

_src/parse.js_

```js
AST.prototype.unary = function() {
  // var token;
  if ((token = this.expect('+', '!', '-'))) {
  //   return {
  //     type: AST.UnaryExpression,
  //     operator: token.text,
  //     argument: this.unary()
  //   };
  // } else {
  //   return this.primary();
  // }
};
```

在我们往操作符对象中增加匹配字符时，其实无意中给字符串表达式引入了一个 bug。如果一个字符串也出现了一个惊叹号——这个惊叹在字符串以外是作为一个操作符存在的：

_test/parse_spec.js_

```js
it('parses a ! in a string', function() {
  expect(parse('"!"')()).toBe('!');
});
```

这时，字符串 token 将包含一个值为 `!` 的 `text` 属性，但 AST 构建器会把它当作是一个操作符！我们不应该允许这种事件发生。

我们需要修改字符串 token，使其文本属性不仅仅包含字符串中的字符，而是包含整个原始字符串（包括周围的引号）。这样就不会跟操作符混淆了。我们将在 Lexer 中的 `readString` 方法中把原始字符串赋值到 `rawString` 变量中：

_src/parse.js_

```js
Lexer.prototype.readString = function(quote) {
  // this.index++;
  // var string = '';
  var rawString = quote;
  // var escape = false;
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
    rawString += ch;
  //   if (escape) {
  //     if (ch === 'u') {
  //       var hex = this.text.substring(this.index + 1, this.index + 5);
  //       if (!hex.match(/[\da-f]{4}/i)) {
  //         throw 'Invalid unicode escape';
  //       }
  //       this.index += 4;
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
        text: rawString,
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
};
```

这样我们就完成了对一元运算符的处理了！