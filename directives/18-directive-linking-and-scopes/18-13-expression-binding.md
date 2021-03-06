### 表达式式绑定（Expression Binding）

第四种也是最后一种往独立作用域绑定的方式是绑定一个表达式，这是通过在作用域定义对象中使用`'&'`符号来实现的。这种方式跟其它两种绑定方式的主要区别是它是主要用来绑定行为而不是绑定数据的：当你使用这个指令，就是提供了一个可供指令在某一个时刻调用的表达式。这对于事件驱动型的指令如`ngClick`来说是尤为有用，同时也有广泛的适用性。

绑定的表达式将会以一个函数的形式出现在一个独立作用域上，而这个函数我们是可以通过调用返回的，例如，一个 link 函数：

_test/compile\_spec.js_

```js
it('allows binding an invokable expression on the parent scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myExpr: '&'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    $rootScope.parentFunction = function() {
      return 42;
    };
    var el = $('<div my-directive my-expr="parentFunction() + 1"></div>');
    $compile(el)($rootScope);
    expect(givenScope.myExpr()).toBe(43);
  });
});
```

独立作用域上的`myExpr`函数实际上是`parentFunction() + 1`这个表达式的表达式函数，它会调用父作用域上的`parentFunction`并对返回函数调用结果加1。

要达成这个效果，我们需要再次回顾我们在独立作用域定义使用的解析函数。  
这里我们需要把`&`符号变成`mode`属性中一个合法项：

_src/compile.js_

```js
function parseIsolateBindings(scope) {
  // var bindings = {};
  // _.forEach(scope, function(defnition, scopeName) {
    var match = defnition.match(/\s*([@<&]|=(\*?))(\??)\s*(\w*)\s*/);
  //   bindings[scopeName] = {
  //     mode: match[1][0],
  //     collection: match[2] === '*',
  //     optional: match[3],
  //     attrName: match[4] || scopeName
  //   };
  // });
  // return bindings;
}
```

剩余的工作实际上会变得很简单。当我们遇到一个`&`模式的绑定，我们会将对应的属性值转变成一个表达式函数。然后我们就需要在独立作用域上为其添加一个包裹函数。所有的表达式函数都需要接收一个作用域对象作为它的第一个参数，而这个包裹函数就是提供者。最关键的是，我们需要使用父作用域作用表达式调用的上下文，而不是独立作用域。这是因为表达式是指令的使用者定义的，而不是指令自身调用的：

```js
_.forEach(
  newIsolateScopeDirective.$$isolateBindings,
  function(defnition, scopeName) {
    var attrName = defnition.attrName;
    switch (defnition.mode) {
      case '@':
        // ...
        break;
      case '<':
        // ...
        break;
      case '=':
        // ...
        break;
      case '&':
        var parentExpr = $parse(attrs[attrName]);
        isolateScope[scopeName] = function() {
          return parentExpr(scope);
        };
        break;
    }
  })
`
```

现在的代码实现是允许在父作用域上调用函数的，但还不允许传递任何参数，这就显得比较局限了。我们可以通过一些改动来修复这个问题。但这种方法能行得通与直接的函数调用的原理还是有点区别的。我们来看看下面这种在父作用域上表达式：

```xml
<div my-expr="parentFunction(a, b)"></div>
```

有人可能以为可以用独立作用域通过下面这种方式来进行调用：

```js
scope.myExpr(1, 2);
```

但事实上不是这样的。如果你从指令使用者的角度来思考，`a`和`b`并不一定是从指令内部中来的，它们也可能来自父作用域自身。如果你不能在这些作用域上使用父作用域上的属性就会变得比较局限了。

那么，我们该怎么支持函数参数不是来自独立作用域内部的情况呢？好吧，其实我们可以使用的就是以对象形式表示的_命名参数_：

```js
scope.myExpr({a: 1, b: 2});
```

这样的话，如果你的系统需要设计成`b`来自独立作用域，而`a`来自于父作用域，这也是可以满足的。

> Angular 框架自身会使用`$`前缀来区分来自独立作用域的参数。举例来说，像 ngClick 会使用 `$event` 参数来传递 DOM 事件对象。

如果将我们的想法转变成单元测试，就会是像下面这样的：

_test/compile\_spec.js_

```js
it('allows passing arguments to parent scope expression', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myExpr: '&'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var gotArg;
    $rootScope.parentFunction = function(arg) {
      gotArg = arg;
    };
    var el = $('<div my-directive my-expr="parentFunction(argFromChild)"></div>');

    $compile(el)($rootScope);
    givenScope.myExpr({
      argFromChild: 42
    });
    expect(gotArg).toBe(42);
  });
});
```

这里我们设置了一个定义在父作用域上的函数，这个函数会被绑定到独立作用域的`myExpr`表达式调用。在表达式中我们会引用`argFromChild`参数，而这就是我们会从独立作用域传递过去的命名函数。

就让我们来继续实现这个特性。实际上这实现起来是十分简单的，这是因为我们已经有传递命名参数给表达式的解决方案：它就是表达式函数允许接收的第二个参数`locals`。我们的包裹函数会接收一个参数，也就是本地变量，然后把它传递给表达式函数作为第二个参数：

_src/compile.js_

```js
case '&':
  // var parentExpr = $parse(attrs[attrName]);
  isolateScope[scopeName] = function(locals) {
    return parentExpr(scope, locals);
  };
  // break;
```

所以，当对表达式`parentFunction(argFromChild)`进行 evaluate 时，对`argFromChild`的查找也是 evaluate 的过程之一。如果本地变量对象（对应的是来自独立作用域的“命名参数”对象）中有匹配项，它就会变成`argFromChild`的值。如果在本地变量对象中找不到`argFromChild`这个属性，我们就会从（父）作用域上查找。

最后，我们也允许独立作用域上的表达式式绑定被标记为可选的。如果真的进行标记并且指令使用者没有写上表达式，那就没有函数会在作用域上被调用：

_test/compile\_spec.js_

```js
it('sets missing optional parent scope expression to undefned', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myExpr: '&?'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var gotArg;
    $rootScope.parentFunction = function(arg) {
      gotArg = arg;
    };
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(givenScope.myExpr).toBeUndefned();
  });
});
```

回顾之前讲表达式的章节，如果传递一些`$parse`无法进行解析的值（比如`undefined`或`null`），那$parse的结果值就会是一个 LoDash 的空函数。我们可以利用这个事实来处理表达式可选的情况，从而跳过创建绑定的环节：

_src/compile.js_

```js
case '&':
  // var parentExpr = $parse(attrs[attrName]);
  if (parentExpr === _.noop && defnition.optional) {
    break;
  }
  // isolateScope[scopeName] = function(locals) {
  //   return parentExpr(scope, locals);
  // };
  // break;
```



