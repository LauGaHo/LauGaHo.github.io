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

构建的核心流程：

![webpack构建流程](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack2.png)

解释一下：

1. 调用 `handleModuleCreate`，根据文件类型构建 `Module` 子类。
2. 调用 `loader-runner` 仓库的 `runLoaders` 转译 `Module` 对象的内容，通常是从各类资源转移成 JS 文本。
3. 调用 `acorn` 将 JS 文本解析为 AST。
4. 遍历 AST，触发各种钩子。
   1. 在 `HarmonyExportDependencyParserPlugin` 插件监听 `exportImportSpecifier` 钩子，解读 JS 文本对应的资源依赖
   2. 调用 `Module` 对象的 `addDependency` 将依赖对象加入到 `Module` 依赖列表中。
5. AST 遍历完毕后，调用 `module.handleParseResult` 处理模块依赖。
6. 对于 `Module` 对象新增的依赖，调用 `handleModuleCreate`，控制流回到第一步。
7. 所有依赖解析完毕后，构建阶段完成。

> 上方中的 `module` 指代的是 `Module` 对象。

这个过程中的数据流 `module => ast => dependencies => module`，先转 AST，然后再从 AST 中找依赖。这要求 `loaders` 处理完的最后结果必须是可以被 `acorn` 处理的标准 JavaScript 语法，比如图片，需要从二进制转换成类似于 `export default "data:image/png;base64,xxx"` 这类 `base64` 格式，或者 `export default "http://xxx"` 这类 URL 格式。

假设有如下图所示的文件依赖树：

![依赖树](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/gM3DGE.png)

其中 `index.js` 为 `entry` 文件，依赖于 `a.js` 和 `b.js` 文件，而 `a.js` 文件又会依赖于 `c.js` 和 `d.js` 文件。初始化编译环境后，`EntryPlugin` 根据 `entry` 配置找到 `index.js` 文件，调用 `compilation.addEntry` 函数触发构建流程，构建完毕后内部会生成这样的数据结构：

![module](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack3.png)

此时得到的 `module[index.js]` 的内容以及对应的依赖对象 `dependence[a.js]` 和 `dependence[b.js]`。根据上方所讲述的流程，继续调用 `module[index.js]` 的 `handleParseResule` 函数，继续处理 `a.js` 和 `b.js` 文件，递归上述流程，进一步得到 `a.js` 和 `b.js` 文件对应的 `Module` 对象，如下图所示：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack4.png)

从 `a.js` 的 `Module` 对象上又解析到了 `dependence[c.js]` 和 `dependence[d.js]`，于是再继续调用 `module[a.js]` 的 `handleParseResult`，再递归上述流程：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack5.png)

到这里解析完所有模块后，发现没有更多新的依赖，就可以继续推进，进入下一步。

## 生成阶段

构建阶段围绕 `Module` 对象进行展开，生成阶段，则是围绕着 `Chunk` 对象来展开，经过构建阶段之后，webpack 得到了足够的模块内容和模块关系信息，接着就开始生成最终资源了。代码层面上来看，就是开始执行 `compilation.seal` 函数：

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

`seal` 的意思是密封，`seal` 函数主要的工作就是从 `Module` 对象到 `Chunk` 对象的转化，核心流程如下图：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack10.png)

简单梳理：

1. 构建本次编译的 `ChunkGraph` 对象。
2. 遍历 `compilation.modules` 集合，将 `module` 按 `entry/动态引入` 的规则分配给不同的 `Chunk` 对象。
3. `compilation.modules` 集合遍历完毕后，得到完整的 `Chunk` 集合对象，调用 `createXxxAssets` 方法。
4. `createXxxAssets` 遍历 `Module` 或 `Chunk` 对象，调用 `compilation.emitAssets` 方法将 `assets` 信息记录到 `compilation.assets` 对象中。
5. 触发 `seal` 回调，控制流回到 `compiler` 对象。

这一步的关键逻辑是将 `Module` 对象按照规则组织成 `Chunk` 集合，Webpack 内置的 `Chunk` 封装规则比较简单：

- `entry` 及 `entry` 触达到的模块，组合成一个 `Chunk`。
- 使用动态引入语句引入的模块，各自组合成一个 `Chunk`。

`Chunk` 是输出的基本单位，默认情况下，这些 `Chunk` 集合和最终输出的资源是一一对应的，按照上面的规则大致可以推导出一个 `entry` 会对应打包出一个资源，而通过动态引入语句引入的模块，也会对应打包出相应的资源，看一个示例。

假设有如下配置：

```javascript
const path = require("path");

module.exports = {
  mode: "development",
  context: path.join(__dirname),
  entry: {
    a: "./src/index-a.js",
    b: "./src/index-b.js",
  },
  output: {
    filename: "[name].js",
    path: path.join(__dirname, "./dist"),
  },
  devtool: false,
  target: "web",
  plugins: [],
};
```

实例配置中有两个入口，对应文件结构图片如下：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack11.png)

上图中的 `index-a.js` 依赖于 `c.js`，并且动态引入了 `e.js`，而 `index-b.js` 依赖于 `c.js` 和 `d.js`。根据上边说的规则：

- `entry` 及 `entry` 触达的模块，组合成一个 `Chunk` 对象。
- 使用动态 `import` 的模块，各自组合成一个 `Chunk` 对象。

生成的 `Chunk` 对象集合结构如下：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack12.png)

根据依赖关系，`chunk[a]` 包含了 `index-a.js` 和 `c.js` 这两个模块；而 `chunk[b]` 包含了 `c.js`、`index-b.js` 和 `d.js` 三个模块，`chunk[e-hash]` 则为动态 `import` 的 `e.js` 对应的 `chunk`。

上方的依赖关系中可以知道 `chunk[a]` 和 `chunk[b]` 同时包含了 `c.js` 模块，这个问题具体的应用场景可能是：一个多页面应用，所有页面都依赖于相同的基础库，这些所有的页面对应的 `entry` 都会包含有基础库代码，但是这种情况会导致 `c.js` 被打包多份，为了解决这个问题，webpack 提供了一些插件，如：`CommonChunkPlugin` 和 `SplitChunksPlugin`，使得 webpack 可以在基本规则下进一步优化 `chunks` 结构。

### SplitChunksPlugin 的作用

`SplitChunksPlugin` 是 webpack 架构高扩展的一个绝好的例子，上方说了 webpack 主流程是按照 `entry` 和 `Dynamic Import` 两种情况组织 `Chunk` 对象的，这里必然会导致不必要的重复打包出现，webpack 通过插件的形式解决这个问题。

回顾 `compilation.seal` 函数的代码，大致上可以梳理成 4 个步骤：

- 遍历 `compilation.modules`，记录下模块和 `Chunk` 关系。
- 触发各种模块优化钩子，这一步优化的主要是模块依赖关系。
- 遍历 `module` 构建 `Chunk` 对象集合。
- 触发各种优化钩子。

上边的前三个步骤都是 webpack 的默认规则实现，重点关注第 4 个步骤触发的 `optimizeChunk` 钩子，这个时候已经跑完了主流程的逻辑，得到了 `Chunk` 对象的集合，而 `SplitChunksPlugin` 正是使用这个钩子，分析 `Chunk` 对象集合的内容，按照配置规则增加一些通用的 `Chunk` 对象：

```javascript
module.exports = class SplitChunksPlugin {
  constructor(options = {}) {
    // ...
  }

  _getCacheGroup(cacheGroupSource) {
    // ...
  }

  apply(compiler) {
    // ...
    compiler.hooks.thisCompilation.tap("SplitChunksPlugin", (compilation) => {
      // ...
      compilation.hooks.optimizeChunks.tap(
        {
          name: "SplitChunksPlugin",
          stage: STAGE_ADVANCED,
        },
        (chunks) => {
          // ...
        }
      );
    });
  }
};
```

webpack 插件架构的高扩展性，使得整个编译的主流程是可以固化下来的，分支逻辑和细节需求可以通过“外包”的方式让第三方去实现，这套规则架起了庞大的 webpack 生态。

### 写入文件系统

经过构建阶段 `make` 之后，`compilation` 会获知资源模块的内容和依赖关系，也就知道了“输入是什么”；经过了生成阶段 `seal` 后，`compilation` 则获知资源输出的图谱，也就是知道怎么“输出”，这里所谓的“输出”指的是：哪些模块跟哪些模块是“绑定”在一起输出到哪里。经过 `seal` 后大致的数据结构：

```javascript
compilation = {
  // ...
  modules: [
    /* ... */
  ],
  chunks: [
    {
      id: "entry name",
      files: ["output file name"],
      hash: "xxx",
      runtime: "xxx",
      entryPoint: {xxx}
      // ...
    },
    // ...
  ],
};
```

`seal` 结束之后，紧接着调用 `compiler.emitAssets` 函数，函数内部调用 `compiler.outputFileSystem.writeFile` 方法将 `assets` 集合写入文件系统。

### 资源形态流转

这里开始结合 **资源流转** 的角度重新考察整个过程：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack13.png)

- **`compiler.make` 阶段**
  - `entry` 文件以 `dependence` 对象形式加入 `compilation` 的依赖列表，`dependence` 对象记录有 `entry` 的类型、路径等信息。
  - 根据 `dependence` 调用对应的工厂函数创建 `module` 对象，之后读入 `module` 对应的文件内容，调用 `loader-runner` 对内容做转化，转化结果若有其他依赖则继续读入依赖资源，重复此过程，直到所有依赖均被转化为 `module`。

- **`compilation.seal` 阶段**
  - 遍历 `module` 集合，根据 `entry` 配置及引入资源的方式，将 `module` 分配到不同的 `chunk`。
  - 遍历 `chunk` 集合，调用 `compilation.emitAsset` 方法标记 `chunk` 的输出规则，即转化为 `assets` 集合。

- **`compiler.emitAssets` 阶段**
  - 将 `assets` 写入文件系统。


### Plugin 解析

webpack 中的 Plugin 是通过 webpack 自身数量庞大的钩子函数实现插件功能的拓展的，插件定义的钩子回调中，能够和 webpack 主流程中的上下文数据进行交互，甚至可以修改对应的上下文的数据结构，进而实现拓展原来的 webpack 功能。

#### 插件的定义

从形态上看，插件是一个带有 `apply` 函数的类：

```javascript
class SomePlugin {
  apply(compiler) {

  }
}
```

`apply` 函数运行时会得到参数 `compiler`，以此为起点可以调用 `hook` 对象注册各种钩子回调，例如：`compiler.hooks.make.tapAsync`，这里边 `make` 是钩子名称，`tapAsync` 定义了钩子的调用方式，webpack 的插件架构基于这种模式构建而成，插件开发者可以使用这种模式在钩子回调中，插入特定代码。webpack 各种内置对象都带有 `hooks` 属性，又如 `compilation` 对象：

```javascript
class SomePlugin {
    apply(compiler) {
        compiler.hooks.thisCompilation.tap('SomePlugin', (compilation) => {
            compilation.hooks.optimizeChunkAssets.tapAsync('SomePlugin', () => {

            });
        });
    }
}
```

#### 钩子触发时机

下面举例说明钩子函数的执行时机：

- **`compiler.hooks.compilation`**
  - 时机：启动编译创建出 `compilation` 对象后触发
  - 参数：当前编译的 `compilation` 对象
  - 示例：很多插件基于此事件获取 `compilation` 实例

- **`compiler.hooks.make`**
  - 时机：正式开始编译时触发
  - 参数：同样是当前编译的 `compilation` 对象
  - 示例：webpack 内置的 `EntryPlugin` 基于此钩子实现 `entry` 模块的初始化

- **`compilation.hooks.optimizeChunks`**
  - 时机：`seal` 函数中，`Chunk` 集合构建完毕触发
  - 参数：`Chunk` 对象集合和 `chunkGroups` 集合
  - 示例：`SplitChunksPlugin` 插件基于此钩子实现 `entry` 模块的初始化

- **`compiler.hooks.done`**
  - 时机：编译完成后触发
  - 参数：`stats` 对象，包含编译过程中的各类统计信息
  - 示例：`webpack-bundle-analyzer` 插件基于此钩子实现打包分析

webpack 从启动到结束，`compiler` 对象逐次触发如下钩子：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack14.png)

`compilation` 对象逐次触发：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack15.png)

所以理解清楚了 webpack 工作的主流程，基本上就清除了钩子函数的触发时机。

#### 参数

传递参数和具体的钩子强相关，这里阅读的方法是直接在源码里面搜索调用语句，例如对于 `compilation.hooks.optimizeTree`，可以在 webpack 源码中搜索 `hooks.optimizeTree.call` 关键字，就可以找到调用代码：

```javascript
// lib/compilation.js #2297
this.hooks.optiomizeTree.callAsync(this.chunks, this.modules, err => {

})
```

结合代码所在的上下文，可以判断出此时传递的是经过优化的 `chunks` 及 `modules` 集合。

#### 插件示例

webpack 的钩子复杂程度不一，通过例子可窥一二。在 `compilation.seal` 函数内部有 `optimizeModules` 和 `afterOptimizeModules` 这一对看起来对偶的钩子函数，`optimizeModules` 从字面上可以理解为用于优化已经编译出的 `modules`，而 `afterOptimizeModules` 钩子在 webpack 源码中唯一用途是 `ProgressPlugin`，大体逻辑如下：

```javascript
compilation.hooks.afterOptimizeModules.intercept({
  name: "ProgressPlugin",
  call() {
    handler(percentage, "sealing", title);
  },
  done() {
    progressReporters.set(compiler, undefined);
    handler(percentage, "sealing", title);
  },
  result() {
    handler(percentage, "sealing", title);
  },
  error() {
    handler(percentage, "sealing", title);
  },
  tap(tap) {
    // p is percentage from 0 to 1
    // args is any number of messages in a hierarchical matter
    progressReporters.set(compilation.compiler, (p, ...args) => {
      handler(percentage, "sealing", title, tap.name, ...args);
    });
    handler(percentage, "sealing", title, tap.name);
  }
});
```

基本上可以猜测出，`afterOptimizeModules` 的设计初衷就是用于通知优化行为的结束。

`apply` 虽然是一个函数，但是从设计上就只用输入，webpack 其实对输出时无感的，所以在插件中只能通过调用类型实体的各种方法或者更改实体的配置信息，从而达到变更编译行为的目的。例如：

- `compilation.addModule`：添加模块，可以在原有的 `module` 构建规则之外，添加自定义模块。
- `compilation.emitAssets`：将内容写入到特定的路径处。

#### 如何影响编译状态

webpack 的插件体系和平常所见的订阅/发布模式的差别是很大的，是一种强耦合的设计，`hooks` 回调由 webpack 决定什么时候，以什么方式执行；而在 `hooks` 回调内部可以通过修改状态、调用上下文 `API` 等方式变更编译行为。

比如，`EntryPlugin` 插件：

```javascript
class EntryPlugin {
  apply(compiler) {
    compiler.hooks.compilation.tap(
      "EntryPlugin",
      (compilation, { normalModuleFactory }) => {
        compilation.dependencyFactories.set(
          EntryDependency,
          normalModuleFactory
        );
      }
    );

    compiler.hooks.make.tapAsync("EntryPlugin", (compilation, callback) => {
      const { entry, options, context } = this;

      const dep = EntryPlugin.createDependency(entry, options);
      compilation.addEntry(context, dep, options, (err) => {
        callback(err);
      });
    });
  }
}
```

上述代码片段调用了两个影响 `compilation` 对象状态的接口：

- `compilation.dependencyFactories.set`
- `compilation.addEntry`

这里的重点是，webpack 会将上下文信息以参数或 `this` (`compiler` 对象) 形式传递给钩子回调，在回调中可以调用上下文对象的方法或者直接修改上下文对象属性的方式，对原定的流程产生影响。所以要想编写自己的插件，除了需要理解调用时机，还需要了解可以调用哪些 API。

