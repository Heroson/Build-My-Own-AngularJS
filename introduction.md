该书是写给那些正在参加工作的程序员，他们可能是：

- 想要学习 AngularJS 的。
- 已经知道 AngularJS，但想进一步探究它内部是如何运作的。
- 想要知道一个大型的 JavaScript 应用框架是如何一步步地创建出来的。

AngularJS 并不是一个小框架。它里面包含了很多需要掌握的概念。它的代码库也是很庞大的，有 35,000 行 JavaScript 代码。虽然这些新概念和大量的代码能给我们提供构建应用的强大工具，但随之而来的是一个陡峭的学习曲线。

我讨厌使用自己无法掌握的技术进行开发。因为这样做的话，常常会导致一种情况发生：开发的代码能正常运作，但这并不是因为我们真正掌握了这种技术，而是我们已经做了很多次试验、遇到过很多错误之后才发现这种代码是可行的。这种情况下开发出来的代码是很难进行改变和调试的。你无法解释到底是哪里出了问题。你只是在戳代码，直到代码看起来都对齐了。

使用像 AngularJS 这样强大的框架，就很容易出现上面讲到的这种代码。你弄懂了 Angular 是如何实现依赖注入的吗？你知道作用域继承的机制吗？指令进行 transclusion 时到底会发生什么事情？当你没有弄懂这些功能是怎么运作的，就像我刚开始接触 Angular 的时候，你不得不去翻文档或看看 Stack Overflow 上大家是怎么说的。但如果这两个方式都不管用的时候，你就会试着去找其他渠道，直到找到想要的答案。

事实上，虽然 AngularJS 框架很多代码，但这些代码都只是原生 JavaScript 代码而已，跟我们本来在应用中使用的代码并没有什么不同。但里面的代码大多数都是有良好的结构和可读性的而已。你可以对这些代码进行研究，看看 Angular 是怎么做到的。当你研究完了，你会发现自己能更好地处理日常的开发任务。你不仅知道使用 Angular 提供的哪个特性来解决特定问题，也清楚这些特性是如何运作的，如何充分利用这些特性，它们的短板在哪。

本书的目的是帮助你弄清楚 AngularJS 的内部运作。将它进行切分，最后再把各部分整合起来，目的就是为了真正弄懂它是怎么工作的。

一个好的匠人应该对他使用的工具了如指掌。这样他们才能再有需要的时候制作属于自己的工具。这本书将帮助我们在 AngularJS 上达到这个境界。