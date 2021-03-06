### 本章小结
#### Summary

本章我们补全了 Angular 表达式语言的最后一个功能——过滤器。这个功能可以使用预定义的筛选器函数来修改表达式的结果值，它非常有用，也可以在整个应用程序中重用这些筛选器函数。

在本章，你学到了：

- 在表达式中通过管道符号 `|` 来启用一个过滤器。
- Angular 表达式不支持逐位计算运算符，否则就会跟应用过滤器的符号产生冲突。
- 使用 `filter` 服务来注册和获取过滤器。
- 如何通过给过滤器服务传入对象的方式来一次性注册多个过滤器。
- 过滤器表达式是如何被 AST 构建器和编译器当作函数调用表达式进行处理的。
- 过滤器表达式是所有类型的表达式中优先级最低的。
- AST 编译器是如何生成 JavaScript 代码，以便在运行时从筛选器服务中查找表达式用到的所有过滤器。
- 如何链式调用多个过滤器。
- 如果给过滤器传递额外的参数，以及如何把它们作为第二个、第三个……参数传递给过滤器函数。
- 内置的 `filter` 过滤器是如何运作的：使用判定函数、原始类型或者对象、嵌套的对象和数组、通配符 `$`，还自定义的比较函数作为过滤器表达式。