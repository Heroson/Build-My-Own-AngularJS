### 本章小结

在本章中，我们已经让作用域从只有一个作用域对象，变成一个能够通过从父作用域上继承属性进而实现信息联通的树结构，也就实现了内置在每个 AngularJS 应用的作用域树结构。

在本章，你学习了：

* 子作用域是如何被创建的。
* 作用域继承和 JavaScript 原生的原型继承机制之间的联系。
* 属性屏蔽及其含义。
* 从父作用域到子作用域的递归 digest 过程。
* `$digest` 和 `$apply` 在 digest 起始点上的区别。
* 隔离作用域以及它与普通子作用域的区别。
* 如何销毁子作用域。

下一章我们会介绍与 watcher 相关的另一个知识点：Angular 内置的 `$watchCollection` 机制，可以对对象和数组进行高效的侦听。

