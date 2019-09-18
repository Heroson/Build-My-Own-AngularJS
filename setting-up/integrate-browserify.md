### 集成 Browserify（Integrate Browserify）

目前我们的处理是，Karma 会对所有在 `src` 和 `test` 目录下的 JavaScript 文件都进行加载，并执行里面的全部内容。这意味着文件中的全局变量和函数，比如`sayHello`，能被其他文件直接访问到。

但这种文件组织方式在某种程度上会让代码难以跟踪，因为这样难以定位目标代码的位置。官方的 AngularJS 代码就是使用这种方式的，但这是因为官方项目是在 2009 年创建的，当时并没有真正合适的替代方案。幸运的是，目前已经有替代方案了。

Node.js 遵循的打包标准是 [CommonJS](http://wiki.commonjs.org/wiki/CommonJS)，这个标准规定一个文件就是一个模块。一个模块可以使用 `require` 语法引入其他模块的东西，同时我们可以在模块中使用 `module.exports` 语法定义模块中可对外输出的东西。这套模块系统就非常适合用来做文件组织了。但我们要编写的不是 Node.js 代码，而是客户端的 JavaScript 代码。不过也不是没有办法，我们可以使用一个名为 `Browserify` 的工具，它能使我们在客户端代码中使用模块系统。`Browserify`会对所有文件进行处理，最终输出一个能在浏览器（比如我们测试用到的 PhantomJS 浏览器）中运行的包。

```bash
npm install --save-dev browserify watchify karma-browserify
```

我们来看看在代码中怎么使用这个工具。我们应该在 `hello.js` 中对 `sayHello` 函数进行输出：

_src/hello.js_

```js
module.exports = function sayHello() {
  return 'Hello, world!';
};
```

然后我们就可以在测试用例中使用 `require` 进行加载：

_test/hello_spec.js_

```js
var sayHello = require('../src/hello');

describe('Hello', function() {

  it('says hello', function() {
    expect(sayHello()).toBe('Hello, world!');
  }); 

});
```

在运行代码之前，我们需要先在测试配置中集成对 Browserify 的支持。由于我们已经安装了 `karma-browserify` 插件，现在直接启用它就可以了：

_karma.conf.js_

```js
module.exports = function(config) {
  config.set({
    frameworks: ['browserify', 'jasmine'],
    files: [
      'src/**/*.js',
      'test/**/*_spec.js'
    ],
    preprocessors: {
      'test/**/*.js': ['jshint', 'browserify'],
      'src/**/*.js': ['jshint', 'browserify']
    },
    browsers: ['PhantomJS'],
    browserify: {
      debug: true
    }
  })
}
```

这样配置好之后，以后每次执行代码之前都会先运行 `browserify` 预处理器，此时，Browserify 会处理模块的 import 和 export。

这里要注意的是，我们也启用了 Browserify 的 debug 功能，这意味着 Browserify 会在处理代码的同时输出 代码的[source map](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/)，这样我们能从错误信息中看到发生错误的代码来源于哪个文件的哪一行。如果测试无法通过、要定位错误原因时，这项功能会显得特别重要的。

现在试一下启动 Karma，然后故意加入一些错误代码看看效果。语法错误会出现一个 JSHint 错误信息，而单元测试失败则会出现一个测试错误信息。这两个错误都能在 Karma 的输出中看到。

> 运行单元测试时，偶尔会看到一个“Some of your tests did a full page reload!”的信息，但你实际上并没有在单元测试中做这样的操作。虽然这个问题对我们的开发不会产生实质性的影响，但还是会让人分心。
> 
> 出现这个问题的原因在于侦测到文件变化后测试用例过早地执行了。你可以通过调高 Browserify 的 bundleDelay 配置参数来解决，直接把这个配置参数加入到 `karma.conf.js` 的 `browserify` 部分就可以了。这个配置参数的默认值为 700，而我在电脑上设置 `bundleDelay: 2000` 就可以解决这个问题了。