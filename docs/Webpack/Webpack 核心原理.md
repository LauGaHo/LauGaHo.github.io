# Webpack 核心原理

## 核心流程解析

Webpack 最核心的功能：

> As its core, webpack is a static module bundler for modern JavaScript application.

也就是将各类型的资源，包括图片、CSS、JS 等，转译、组合、拼接、生成 JS 格式的 bundler 文件。

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/azM7YD.png)

这个过程核心完成了 **内容转换** 和 **资源合并** 两种功能，实现包含三个阶段：

1. 初始化阶段：
   1. **初始化参数**：从配置文件、配置对象、Shell 参数中读取，和默认配置结合得出最终的参数。
   2. **创建编译器对象**：用上一步得到的参数创建 `Compiler` 对象。
   3. **初始化编译环境**：包括注入内置插件、注册各种模块工厂、初始化 `RuleSet` 集合、加载配置的插件等。
   4. **开始编译**：执行 `compiler` 对象的 `run` 方法。
   5. **确定入口**：执行配置中的 `entry` 找出所有的入口文件，调用 `compilition.addEntry` 将入口转换为 `dependence` 对象。
2. 构建阶段
   1. **编译模块(make)**：根据 `entry` 对应的 `dependence` 创建 `module` 对象，调用 `loader` 将模块转译成标准的 JS 内容，调用 JS 解释器将内容转换成 AST 对象，从中找出该模块依赖的模块，再递归本步骤直到所有入口文件依赖的文件都经过了本步骤的处理。
   2. **完成模块编译**：上一步递归处理所有能触达的模块后，得到了每个模块被翻译后的内容以及它们之间的 **依赖关系图**。
3. 生成阶段
   1. **输出资源(seal)**：根据入口和模块之间的依赖关系，组装成一个个包含多个模块的 `Chunk`，再把每个 `Chunk` 转换成一个单独的文件加入到输出列表，这步是可以修改输出内容的最后机会。
   2. **写入文件系统(emitAssets)**：在确定好输出内容之后，根据配置确定输出的路径和文件名，把文件内容写入到文件系统中。

单次构建过程自上而下按顺序执行，下面展开细节来讲，先统一一下名词：

- `Entry`：编译入口，webpack 编译的起点。
- `Compiler`：编译管理器，webpack 启动后会创建 `compiler` 对象，该对象一直存活直到结束退出。
- `Compilation`：单次编译过程的管理器，比如 `watch = true` 时，运行过程中只有一个 `compiler` 但每次文件变更触发重新编译时，都会创建一个新的 `compilation` 对象。即一次编译，一个 `compilation` 对象。
- `Dependence`：依赖对象，webpack 基于该类型记录模块之间的依赖关系。
- `Module`：webpack 内部所有资源都会以 `Module` 对象形式存在，所有关于资源的操作、转译、合并都是以 `Module` 为基本单位进行的。
- `Chunk`：编译完成准备输出时，webpack 会将 `Module` 对象按照特定的规则组织成一个一个的 `Chunk`，这些 `Chunk` 某种程度上跟最终的输出一一对应的。
- `Loader`：资源内容转换器，其实就是实现从内容 A 转换 B 的转换器。
- `Plugin`：webpack 构建过程中，会在特定的时机广播对应的事件，插件监听这些事件，在特定的时间点介入编译过程。

webpack 编译过程都是围绕着这些关键对象进行展开的。

## 初始化阶段

webpack 的初始化过程：

![webpack初始化](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack1.png)

过程解析：

1. 将 `process.args` 和 `webpack.conf.js` 合并成用户配置。
2. 调用 `validateSchema` 校验配置。
3. 调用 `getNormalizedWebpackOptions` 和 `applyWebpackOptionsBaseDefaults` 合并出最终配置。
4. 创建 `compiler` 对象。
5. 遍历用户定义的 `plugins` 集合，执行插件的 `apply` 方法。
6. 调用 `new WebpackOptionsApply().process` 方法，加载各种内置插件。

主要逻辑集中在 `WebpackOptionsApply` 类，webpack 内置了数百个插件，这些插件并不需要手动去配置，`WebpackOptionsApply` 会在初始化阶段根据配置内容动态注入对应的插件，包括：

- 注入 `EntryOptionPlugin` 插件，处理 `entry` 配置。
- 根据 `devtool` 值判断后续用哪个插件处理 `sourceMap`，可选值有：
  - `EvalSourceMapDevToolPlugin`
  - `SourceMapDevToolPlugin`
  - `EvalDevToolModulePlugin`
- 注入 `RuntimePlugin`，用于根据代码内容动态注入 webpack 运行时插件。
  
到了此时，`compiler` 实例也就完成了创建工作，然后就是调用 `compiler.compile` 函数进行编译工作，`compiler.compile` 代码如下：

```javascript
// webpack/lib/compiler.js 
compile(callback) {
    const params = this.newCompilationParams();
    this.hooks.beforeCompile.callAsync(params, err => {
      // ...
      const compilation = this.newCompilation(params);
      this.hooks.make.callAsync(compilation, err => {
        // ...
        this.hooks.finishMake.callAsync(compilation, err => {
          // ...
          process.nextTick(() => {
            compilation.finish(err => {
              compilation.seal(err => {...});
            });
          });
        });
      });
    });
  }
```

Webpack 架构灵活，但是代价牺牲了源码的直观性，比如上方所说的初始化流程，从创建 `compiler` 实例到调用 `make` 钩子，逻辑链路很长：

- 启动 Webpack，触发 `lib/webpack.js` 文件中的 `createCompiler` 方法。
- `createCompiler` 方法内部调用 `WebpackOptionsApply`，`WebpackOptionsApply` 定义在 `lib/WebpackOptionsApply.js` 文件，内部根据 `entry` 配置决定注入相关的 `entry` 插件，其中包括：
  - `DllEntryPlugin`
  - `DynamicEntryPlugin`
  - `EntryPlugin`
  - `PrefetchPlugin`
  - `ProgressPlugin`
  - `ContainerPlugin`
- 注入后的插件会监听相关的钩子函数，就如 `lib/EntryPlugin.js` 的 `EntryPlugin` 监听 `compiler.make` 钩子。
- `lib/compiler.js` 中的 `compile` 函数内调用 `this.hooks.make.callAsync` 这里就是调用对应的钩子的回调函数。
- 触发 `EntryPlugin` 的 `make` 回调，在回调中执行 `compilation.addEntry` 函数。
- `compilation.addEntry` 函数内部经过一部分和主流程无关的 `hook` 之后，再调用 `handleModuleCreate` 函数，正式开始构建内容。

## 构建阶段

