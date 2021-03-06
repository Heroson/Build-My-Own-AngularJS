### 双向数据绑定（Two-Way Data Binding）

双向数据绑定跟单向数据绑定很类似。之后我们会介绍两者之间几个重要的差异点，但我们先把双向数据绑定中跟单向数据绑定相同的特性先实现出来。

对于双向数据绑定，最简单的配置方式是在作用域定义对象中用`'='`符号来进行指定：

_test/compile_spec.js_

```js
it('allows binding two-way expression to isolate scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        anAttr: '='
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive an-attr="42"></div>');
    $compile(el)($rootScope);

    expect(givenScope.anAttr).toBe(42);
  });
})
```

我们也可以对双向绑定的属性取一个别名：

```js
it('allows aliasing two-way expression attribute on isolate scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '=theAttr'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive the-attr="42"></div>');
    $compile(el)($rootScope);
    
    expect(givenScope.myAttr).toBe(42);
  });
});
```

像单向数据绑定一样，双向数据绑定的属性也是会被 watch 的：

```js
it('watches two-way expressions', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '='
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-attr="parentAttr + 1"></div>');
    $compile(el)($rootScope);
    
    $rootScope.parentAttr = 41;
    $rootScope.$digest();
    expect(givenScope.myAttr).toBe(42);
  });
});
```

属性值也可以是可选的，我们可以使用`=?`的语法：

```js
it('does not watch optional missing two-way expressions', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '=?'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect($rootScope.$$watchers.length).toBe(0);
  });
});
```

上面这几个测试用例，我们只需要在之前使用的属性绑定正则表达式中加入对`=`符号的支持即可。

_src/compile.js_

```js
function parseIsolateBindings(scope) {
  var bindings = {};
  _.forEach(scope, function(defnition, scopeName) {
    var match = defnition.match(/\s*([@<=])(\??)\s*(\w*)\s*/);
    // ...
```

然后，我们把之前实现单向数据绑定的代码复制到双向数据绑定的处理逻辑中：

```js
_.forEach(
  // newIsolateScopeDirective.$$isolateBindings,
  // function(defnition, scopeName) {
  //   var attrName = defnition.attrName;
    var parentGet, unwatch;
    // switch (defnition.mode) {
    //   case '@':
    //     attrs.$observe(attrName, function(newAttrValue) {
    //       isolateScope[scopeName] = newAttrValue;
    //     });
    //     if (attrs[attrName]) {
    //       isolateScope[scopeName] = attrs[attrName];
    //     }
    //     break;
    //   case '<':
    //     if (defnition.optional && !attrs[attrName]) {
    //       break;
    //     }
        parentGet = $parse(attrs[attrName]);
        // isolateScope[scopeName] = parentGet(scope);
        unwatch = scope.$watch(parentGet, function(newValue) {
        //   isolateScope[scopeName] = newValue;
        // });
        // isolateScope.$on('$destroy', unwatch);
        // break;
      case '=':
        if (defnition.optional && !attrs[attrName]) {
          break;
        }
        parentGet = $parse(attrs[attrName]);
        isolateScope[scopeName] = parentGet(scope);
        unwatch = scope.$watch(parentGet, function(newValue) {
          isolateScope[scopeName] = newValue;
        });
        isolateScope.$on('$destroy', unwatch);
        break;
    // }
  });
```

但如果这就是双向数据绑定的全部内容，那双向数据绑定也没有什么新意，是吧？现在，我们开始介绍双向数据绑定的“双向”特性：当我们在独立作用域上声明一个双向数据绑定属性，它也会对父作用域的对应绑定产生影响。这是我们之前并未接触过的。下面就是一个例子：

_test/compile_spec.js_

```js
it('allows assigning to two-way scope expressions', function() {
  var isolateScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '='
      },
      link: function(scope) {
        isolateScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-attr="parentAttr"></div>');
    $compile(el)($rootScope);
    
    isolateScope.myAttr = 42;
    $rootScope.$digest();
    expect($rootScope.parentAttr).toBe(42);
  });
});
```

在这个测试中我们会把一个独立作用域上的属性`myAttr`和父作用域上的`parentAttr`进行绑定。我们对子作用域的属性赋值并运行一个 digest 循环，测试对应在父作用域上的属性是否也会同步更新。

一旦数据能够实现双向联系，优先级就会成为一个问题：如果父作用域和子作用域属性都在同一次 digest 循环中都改变了该怎么办？哪个属性的值变更会成为两者最后的计算结果？在 Angular，父元素的属性值变更会被优先使用：

_test/compile_spec.js_

```js
it('gives parent change precedence when both parent and child change', function() {
  var isolateScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '='
      },
      link: function(scope) {
        isolateScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-attr="parentAttr"></div>');
    $compile(el)($rootScope);

    $rootScope.parentAttr = 42;
    isolateScope.myAttr = 43;
    $rootScope.$digest();
    expect($rootScope.parentAttr).toBe(42);
    expect(isolateScope.myAttr).toBe(42);
  });
});
```

这就是双向数据绑定运行的基本原则。下面我们就来实现它，它并不复杂，但有几个小细节我们需要先解决。

首先在值变更时，我们需要在单纯的观察者机制的基础上加入更多控制。为了让事情变得更简单清晰，我们只注册一个 watch 函数而忽略对应的监听处理函数。我们仅依赖 watch 函数在每次 digest 函数中会被调用的特性，并在 watch 函数中加入自己的处理逻辑：

_src/compile.js_

```js
case '=':
  // if (defnition.optional && !attrs[attrName]) {
  //   break;
  // }
  // parentGet = $parse(attrs[attrName]);
  // isolateScope[scopeName] = parentGet(scope);
  var parentValueWatch = function() {
    var parentValue = parentGet(scope);
    if (isolateScope[scopeName] !== parentValue) {
      isolateScope[scopeName] = parentValue;
    }
    return parentValue;
  };
  unwatch = scope.$watch(parentValueWatch);
  // isolateScope.$on(‘$destroy’, unwatch);
  // break;
```

当前实现也只是让旧的单元测试通过而已，而最新的单元测试仍没通过，但是现在的代码更适合我们实现双向数据绑定的“双向”特性。

现在我们只检查了当前的属性值是否不同于独立作用域的属性值，但没有检查到这个变化实际上是发生自哪里的。为了能检查获取这个信息，我们需要引进一个新的变量`lastValue`，它会一直保存最近一次 digest 循环后在父作用域上的属性值：

_src/compile.js_

```js
case '=':
  // if (defnition.optional && !attrs[attrName]) {
  //   break;
  // }
  // parentGet = $parse(attrs[attrName]);
  var lastValue = isolateScope[scopeName] = parentGet(scope);
  // var parentValueWatch = function() {
  //   var parentValue = parentGet(scope);
  //   if (isolateScope[scopeName] !== parentValue) {
      if (parentValue !== lastValue) {
        // isolateScope[scopeName] = parentValue;
      }
    // }
    lastValue = parentValue;
    return lastValue;
  // };
  // unwatch = scope.$watch(parentValueWatch);
  // isolateScope.$on('$destroy', unwatch);
  // break;
```

这个变量的作用就是让我们能分辨以下情况：独立作用域属性的当前值不同于父作用域上的属性值，但父作用域属性值等于`lastValue`，这就说明属性值是在独立作用域上更改的，我们需要将变更同步给父作用域属性：

_src/compile.js_

```js
case '=':
  // if (defnition.optional && !attrs[attrName]) {
  //   break;
  // }
  // parentGet = $parse(attrs[attrName]);
  // var lastValue = isolateScope[scopeName] = parentGet(scope);
  // var parentValueWatch = function() {
  //   var parentValue = parentGet(scope);
  //   if (isolateScope[scopeName] !== parentValue) {
  //     if (parentValue !== lastValue) {
  //       isolateScope[scopeName] = parentValue;
      // } 
      else {

      }
  //   }
  //   lastValue = parentValue;
  //   return lastValue;
  // };
  // unwatch = scope.$watch(parentValueWatch);
  // break;
```

我们该怎么更新父作用域上的属性呢？我们之前开发表达式的时候，曾经讲到有一些表达式是可以从外部进行赋值的，这意味着它们不仅可以通过计算获取结果值，还可以利用表达式的`assign`函数把一个新值赋值给表达式。这就是我们更新父作用域属性时要用到的工具：

_src/compile.js_

```js
case '=':
  // if (defnition.optional && !attrs[attrName]) {
  //   break;
  // }
  // parentGet = $parse(attrs[attrName]);
  // var lastValue = isolateScope[scopeName] = parentGet(scope);
  // var parentValueWatch = function() {
  //   var parentValue = parentGet(scope);
  //   if (isolateScope[scopeName] !== parentValue) {
  //     if (parentValue !== lastValue) {
  //       isolateScope[scopeName] = parentValue;
  //     } else {
        parentValue = isolateScope[scopeName];
        parentGet.assign(scope, parentValue);
  //     }
  //   }
  //   lastValue = parentValue;
  //   return lastValue;
  // };
  // unwatch = scope.$watch(parentValueWatch);
  // isolateScope.$on(‘$destroy’, unwatch);
  // break;
```

注意这里我们不仅调用了`assign`，还更新了本地的`parentValue`变量，也就是`lastValue`也会被赋上这个值。这些都会在下一次 digest 循环中同步。

同时还要注意这里的优先级是怎么处理的：我们是利用了一个 if-else 条件判断，先考虑父作用域属性是否改变了，若没有改变再考虑是否是子作用域属性改变了。当父作用域和子作用域都发生变化，子作用域的变更将会被忽略并重写。

你可能注意到数据绑定使用的是基于引用的监视机制来对值变更进行监测。虽然我们在数据绑定中没法改用基于值的监视机制，但我们还是可以对集合进行“浅监测”（shallow-watch）。我们可以在作用域定义对象中使用一种特殊的语法来告诉框架我们应该用`$watchCollection`代替`$watch`来进行双向数据绑定。这个特性的用处可以在这样一个例子中显现出来：我们需要往属性中绑定一个函数，每次调用函数都会返回一个新数组：

```js
it('throws when two-way expression returns new arrays', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '='
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  
  injector.invoke(function($compile, $rootScope) {
    $rootScope.parentFunction = function() {
      return [1, 2, 3];
    };
    var el = $('<div my-directive my-attr="parentFunction()"></div>');
    $compile(el)($rootScope);
    expect(function() {
      $rootScope.$digest();
    }).toThrow();
  });
});
```

正如我们所见，普通的基于引用的监测无法处理这种情况：这个监测每次看到返回的新数组，都会认为是出现了一个新值。这样 digest 会运行超出重试的次数上限，并抛出异常，这也是上面这个单元测试希望发生的事情。

要修复这个问题，我们可以在作用域定义中通过`=*`语法引入基于集合的监测机制：

_test/compile_spec.js_

```js
it('can watch two-way bindings as collections', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '=*'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    $rootScope.parentFunction = function() {
      return [1, 2, 3];
    };
    var el = $('<div my-directive my-attr="parentFunction()"></div>');
    $compile(el)($rootScope);
    $rootScope.$digest();
    expect(givenScope.myAttr).toEqual([1, 2, 3]);
  });
});
```

我们需要再次对解析函数进行扩展，要允许在`=`符号后加入一个可选的星号：

```js
/\s*([@<]|=(\*?))(\??)\s*(\w*)\s*/
```

现在这个正则表达式就可以匹配以`@`或`<`或`=(*)`开头的字符串。

使用这个正则表达式后，`parseIsolateBindings`就可以获取一个`collection`标识，这个标识的值取决于我们是否在定义时加入了星号。注意，这里需要再次更改捕获组的索引值：

_src/compile.js_

```js
function parseIsolateBindings(scope) {
  // var bindings = {};
  // _.forEach(scope, function(defnition, scopeName) {
    var match = defnition.match(/\s*([@<]|=(\*?))(\??)\s*(\w*)\s*/);
    // bindings[scopeName] = {
      mode: match[1][0],
      collection: match[2] === '*',
      optional: match[3]
      attrName: match[4] || scopeName
  //   };
  // });
  // return bindings;
}
```

现在我就可以根据`collection`这个标识值决定是使用`watch`还是`watchCollection`来进行监测。其余的代码可以保持不变：

```js
case '=':
  // if (defnition.optional && !attrs[attrName]) {
  //   break;
  // }
  // parentGet = $parse(attrs[attrName]);
  // var lastValue = isolateScope[scopeName] = parentGet(scope);
  // var parentValueWatch = function() {
  //   var parentValue = parentGet(scope);
  //   if (isolateScope[scopeName] !== parentValue) {
  //     if (parentValue !== lastValue) {
  //       isolateScope[scopeName] = parentValue;
  //     } else {
  //       parentValue = isolateScope[scopeName];
  //       parentGet.assign(scope, parentValue);
  //     }
  //   }
  //   lastValue = parentValue;
  //   return lastValue;
  // };
  if (defnition.collection) {
    unwatch = scope.$watchCollection(attrs[attrName], parentValueWatch);
  } else {
    unwatch = scope.$watch(parentValueWatch);
  }
  // break;
```

注意在`$watchCollection`分支中，我们会把监听函数作为 listen 函数而不是 watch 函数。这主要是因为`$watchCollection`并不像`$watch`一样可以忽略 listen 函数。但这不会有问题，因为只要父作用域属性始终都指向相同的数组或对象，我们就不需要做任何工作：这又是因为我们已经对同一个数组或者对象进行了引用拷贝，它里面发生的变化都会被自动“同步”。仅当父作用域属性指向了一个我们真正需要作出响应的新数组或新对象时，listen 函数才会被调用。

这样，我们就把双向数据绑定也完成了！