# Webpack产物转译打包逻辑

Webpack 经过了构建阶段 `make`，Webpack 解析出了每个文件对应的 `module` 内容，随即就会进入生成阶段 `seal`，进入到了生成阶段 `seal` 之后，Webpack 就会根据模块的依赖关系、模块特性、`entry` 配置等计算出 `Chunk Graph`，确定最终产物的数量和内容。

经过了 `Chunk Graph` 的生成之后，模块开始来到了转译打包的阶段，大体上的逻辑是这样的：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack17.png)

为了方便理解，将打包过程横向切割成三个阶段：

- **入口**：指代 Webpack 启动到调用 `compilation.codeGeneration` 之前的所有前置操作
- **模块转译**：遍历 `modules` 数组，完成所有模块的转译工作，并将结果存储到 `compilation.codeGenerationResults` 对象。
- **模块合并打包**：在特定上下文框架下，组合业务模块、`runtime` 模块、合并打包成 `bundle`，并调用 `compilation.emitAssets` 输出产物。

上述所说的业务模块指的是开发者所写的项目代码，而 `runtime` 模块指的是 Webpack 分析业务模块之后，动态注入用于支撑项目各项特性的运行时代码。

先来看第一个步骤，模块转译。

## 模块转译原理

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/5RpiTn.png)

上图中的左边就是 Webpack 的产物，右边就是开发者所撰写的项目代码，其中在左边第一个红色方框代表的是 `dependency.js` 文件的构建产物，然后第二个红色方框所代表的是 `main.js` 文件的构建产物，然后两个红色方框中间的代码则是 Webpack 注入用于支撑项目的代码。

上方中的 `main.js` 文件源文件和打包产物之间所产生的变化如下：

- 整个模块被包裹进了 IIFE (立即执行函数) 中。
- 添加 `__webpack_require__.r(__webpack_exports__);` 语句，用于适配 ESM 规范。
- 源码中的 `import` 语句被转译成了 `__webpack_require__` 函数调用。
- 源码 `console` 语句所使用的 `dependency` 变量被转译成了 `_dependency__WEBPACK_IMPORTED_MODULE_0__`。

接下来讲解 Webpack 是如何完成这些转换。

## 核心流程

模块转译操作从 `module.codeGeneration` 调用开始，对应到一开始的流程图的这一部分：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack20.png)

总结关键步骤：

- 调用 `JavascriptGenerator` 的对象的 `generate` 方法，方法内部：
  - 遍历模块的 `dependencies` 和 `presentationalDependencies` 数组。
  - 执行每个数组项 `dependency` 对象对应的 `template.apply` 方法，在 `apply` 方法内修改模块代码，或者更新 `initFragments` 数组。
- 遍历完毕之后，调用 `InitFragment.addToSource` 静态方法，将上一步产生的 `source` 对象和 `initFragments` 数组合并为模块产物。

简单来说就是遍历依赖，在依赖对象中修改 `module` 代码，最后再将所有变更合并成为最终产物。这里边的关键点在于：

- 在 `Template.apply` 函数中，如何更新模块代码。
- 在 `InitFragment.addToSource` 静态方法中，如何合并 `source` 和 `initFragments` 数组生成最终结果。

下边将分开讲解这两个关键点。

## Template.apply 函数

上述流程中，`JavascriptGenerator` 类是无可置疑的 C 位角色，但是它并不会直接修改 `module` 的内容，而是绕了好几层，最终把这个工作委托到了由 `Template` 类型实现。

在 Webpack 5 源码中，`JavascriptGenerator.generate` 函数会遍历模块的 `dependencies` 数组，调用依赖对象对应的 `Template` 子类 `apply` 方法更新模块内容，下面将以伪代码的方式来展示整个流程：

```javascript
class JavascriptGenerator {
    generate(module, generateContext) {
        // 先取出 module 的原始代码内容
        const source = new ReplaceSource(module.originalSource());
        const { dependencies, presentationalDependencies } = module;
        const initFragments = [];
        for (const dependency of [...dependencies, ...presentationalDependencies]) {
            // 找到 dependency 对应的 template
            const template = generateContext.dependencyTemplates.get(dependency.constructor);
            // 调用 template.apply，传入 source、initFragments
            // 在 apply 函数可以直接修改 source 内容，或者更改 initFragments 数组，影响后续转译逻辑
            template.apply(dependency, source, {initFragments})
        }
        // 遍历完毕后，调用 InitFragment.addToSource 合并 source 与 initFragments
        return InitFragment.addToSource(source, initFragments, generateContext);
    }
}

// Dependency 子类
class xxxDependency extends Dependency {}

// Dependency 子类对应的 Template 定义
const xxxDependency.Template = class xxxDependencyTemplate extends Template {
    apply(dep, source, {initFragments}) {
        // 1. 直接操作 source，更改模块代码
        source.replace(dep.range[0], dep.range[1] - 1, 'some thing')
        // 2. 通过添加 InitFragment 实例，补充代码
        initFragments.push(new xxxInitFragment())
    }
}
```

从上述伪代码可以看出，`JavascriptGenerator.generate` 函数的逻辑相对比较固定：

- 初始化一系列变量。
- 遍历 `dependencies` 数组，找到每个 `dependency` 对应的 `template` 对象，调用 `template.apply` 函数修改模块内容。
- 调用 `InitFragment.addToSource` 方法，合并 `source` 和 `initFragments` 数组，生成最终结果。

这里的重点是 `JavascriptGenerator.generate` 函数并不操作 `module` 源码，仅仅提供一个执行框架，真正处理模块内容的转译的逻辑都在 `xxxDependencyTemplate` 对象中的 `apply` 函数实现。

每一个 `Dependency` 子类都会映射到唯一的 `Template` 子类，并且通常这两个类都会写在同一个文件中，例如 `ConstDependency` 和 `ConstDependencyTemplate`；`NullDependency` 和 `NullDependencyTemplate`。Webpack 构建阶段 `make`，会通过 `Dependency` 子类记录不同情况下模块之间的依赖关系；到了生成阶段 `seal` 再通过 `Template` 子类修改 `module` 代码。

综上 `Module`、`JavascriptGenerator`、`Dependency`、`Template` 四个类形成如下交互关系：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack21.png)

`Template` 对象可以通过两种方法更新 `module` 的代码：

- 直接操作 `source` 对象，直接修改模块代码，该对象最初的内容等于模块的源码，经过多个 `Template.apply` 函数流转之后逐渐被替换成新的代码形式。
- 操作 `initFragment` 数组，在模块源码之外插入补充代码片段。

这两种操作所产生的副作用，最终都会被传入 `InitFragment.addToSource` 函数，合成最终结果，下面详细说明细节。

### 使用 Source 更改代码

`Source` 是 Webpack 中编辑字符串的一套工具体系，提供了一系列字符串操作方法，包括：

- 字符串合并、替换、插入等。
- 模块代码缓存、SourceMap 映射、hash 计算等。

Webpack 内部以及社区很多插件、loader 都会使用 `Source` 库编辑代码内容，包括上文所说的 `Template.apply` 体系中，逻辑上，在启动模块代码生成流程时，Webpack 会先使用模块原来的内容初始化 `Source` 对象，即：

```javascript
const source = new ReplaceSource(module.originalSource());
```

之后，不同的 `Dependency` 子类按照顺序、按需更改 `source` 内容，例如 `ConstDependencyTemplate` 中的核心代码：

```javascript
ConstDependency.Template = class ConstDependencyTemplate extends (
  NullDependency.Template
) {
  apply(dependency, source, templateContext) {
    // ...
    if (typeof dep.range === "number") {
      source.insert(dep.range, dep.expression);
      return;
    }

    source.replace(dep.range[0], dep.range[1] - 1, dep.expression);
  }
};
```

上述 `ConstDependencyTemplate` 中，`apply` 函数根据参数条件调用 `source.insert` 插入一段代码，或者调用 `source.replace` 替换一段代码。

### 使用 InitFragment 更新代码

除直接操作 `source` 外，`Template.apply` 中还可以通过直接操作 `initFragments` 数组达成修改模块产物的效果。`initFragments` 数组项通常为 `InitFragment` 子类实例，通常带有两个函数：`getContent`、`getEndContent`，分别用于获取代码片段的头尾部分。

例如 `HarmonyImportDependencyTemplate` 的 `apply` 函数中：

```javascript
HarmonyImportDependency.Template = class HarmonyImportDependencyTemplate extends (
  ModuleDependency.Template
) {
  apply(dependency, source, templateContext) {
    // ...
    templateContext.initFragments.push(
        new ConditionalInitFragment(
          importStatement[0] + importStatement[1],
          InitFragment.STAGE_HARMONY_IMPORTS,
          dep.sourceOrder,
          key,
          runtimeCondition
        )
      );
    //...
  }
}
```

## 代码合并

上述 `Template.apply` 处理完毕后，产生转译后的 `source` 对象和代码片段 `inintFragments` 数组，接着就需要调用 `InitFragment.addToSource` 函数将两者合并为模块产物。

`addToSource` 的核心代码如下：

```javascript
class InitFragment {
  static addToSource(source, initFragments, generateContext) {
    // 先排好顺序
    const sortedFragments = initFragments
      .map(extractFragmentIndex)
      .sort(sortFragmentWithIndex);
    // ...

    const concatSource = new ConcatSource();
    const endContents = [];
    for (const fragment of sortedFragments) {
        // 合并 fragment.getContent 取出的片段内容
      concatSource.add(fragment.getContent(generateContext));
      const endContent = fragment.getEndContent(generateContext);
      if (endContent) {
        endContents.push(endContent);
      }
    }

    // 合并 source
    concatSource.add(source);
    // 合并 fragment.getEndContent 取出的片段内容
    for (const content of endContents.reverse()) {
      concatSource.add(content);
    }
    return concatSource;
  }
}
```

可以看到，`addToSource` 函数的逻辑：

- 遍历 `initFragments` 数组，按顺序合并 `fragment.getContent()` 的产物。
- 合并 `source` 对象。
- 遍历 `initFragments` 数组，按顺序合并 `fragment.getEndContent()` 的产物。

所以，模块代码合并操作主要就是用 `initFragments` 数组一层层包裹住模块代码 `source`，而两者都在 `Template.apply` 层面维护。

## 示例：自定义 Banner 插件

经过 `Template.apply` 转译和 `InitFragment.addToSource` 合并之后，模块就完成了从用户代码形态的转变，为加深对上述模块转译流程的理解，紧接着尝试着开发一个 Banner 插件，实现在每个模块前自动插入一段字符串。

实现上，插件主要涉及 `Dependency`、`Template`、`hooks` 对象，代码：

```javascript
const { Dependency, Template } = require("webpack");

class DemoDependency extends Dependency {
  constructor() {
    super();
  }
}

DemoDependency.Template = class DemoDependencyTemplate extends Template {
  apply(dependency, source) {
    const today = new Date().toLocaleDateString();
    source.insert(0, `/* Author: Tecvan */
/* Date: ${today} */
`);
  }
};

module.exports = class DemoPlugin {
  apply(compiler) {
    compiler.hooks.thisCompilation.tap("DemoPlugin", (compilation) => {
      // 调用 dependencyTemplates ，注册 Dependency 到 Template 的映射
      compilation.dependencyTemplates.set(
        DemoDependency,
        new DemoDependency.Template()
      );
      compilation.hooks.succeedModule.tap("DemoPlugin", (module) => {
        // 模块构建完毕后，插入 DemoDependency 对象
        module.addDependency(new DemoDependency());
      });
    });
  }
};
```

示例插件的关键步骤：

- 编写 `DemoDependency` 和 `DemoDependencyTemplate` 类，其中 `DemoDependency` 仅做示例用，没有实际功能；`DemoDependencyTemplate` 则在其 `apply` 中调用 `source.insert` 插入字符串。
- 使用 `compilation.dependencyTemplates` 注册 `DemoDependency` 和 `DemoDependencyTemplate` 的映射关系。
- 使用 `thisCompilation` 钩子取得 `compilation` 对象。
- 使用 `successModule` 钩子订阅 `module` 构建完毕事件，并调用 `module.addDependency` 方法添加 `DemoDependency` 依赖。

完成上述操作后，`module` 对象的产物在生成过程就会调用到 `DemoDependencyTemplate.apply` 函数，插入定义好的字符串。

# 模块合并打包原理

## 简介

讲解完单个模块的转译过程之后，回到这个流程图：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack17.png)

流程图中，`compilation.codeGeneration` 函数执行完毕 — 也就是模块转译阶段完成后，模块的转译结果会一一保存到 `compilation.codeGenerationResults` 对象中，之后会启动一个新的执行流程 — 模块合并打包。

模块合并打包过程会将 `chunk` 对应的 `module` 及 `runtimeModule` 按照规则塞进模板框架中，最终合并输出成完整的 bundle 文件，如上例：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/5RpiTn.png)

上图中的左边的 bundle 文件中，红色方框中框出来的部分是用户代码文件，红色方框中间的为运行时模块生成产物，其余部分撑起了一个 IIFE 形式的运行框架即为模板框架，也就是：

```javascript
(() => { // webpackBootstrap
    "use strict";
    var __webpack_modules__ = ({
        "module-a": ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => {
            // ! module 代码，
        }),
        "module-b": ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => {
            // ! module 代码，
        })
    });
    // The module cache
    var __webpack_module_cache__ = {};
    // The require function
    function __webpack_require__(moduleId) {
        // ! webpack CMD 实现
    }
    /************************************************************************/
    // ! 各种 runtime
    /************************************************************************/
    var __webpack_exports__ = {};
    // This entry need to be wrapped in an IIFE because it need to be isolated against other modules in the chunk.
    (() => {
        // ! entry 模块
    })();
})();
```

这里的关键在于：

- 最外层由一个 IIFE 包裹
- 一个记录除了 `entry` 外的其他模块代码的 `__webpack_modules__` 对象，对象的 `key` 为模块标志符；值为模块转译后的代码。
- 一个极度简化的 CMD 实现：`__webpack_require__` 函数。
- 最后，一个包裹了 `entry` 代码的 IIFE 函数。

模块转译是将 `module` 转译为可以在宿主环境如浏览器上运行的代码形式；而模块合并操作则是串联这些 `modules`，使之整体符合开发预期，能够正常运行整个应用逻辑。接下来将对这部分代码的生成原理进行解密。

## 核心流程

在 `compilation.codeGeneration` 执行完毕，即所有用户代码模块和运行时模块都执行完转译操作后，`seal` 函数调用 `compilation.createChunkAssets` 函数，触发 `renderManifest` 钩子，`JavascriptModulesPlugin` 插件监听到这个钩子消息后开始组装 bundle，伪代码如下：

```javascript
// Webpack 5
// lib/Compilation.js
class Compilation {
  seal() {
    // 先把所有模块的代码都转译，准备好
    this.codeGenerationResults = this.codeGeneration(this.modules);
    // 1. 调用 createChunkAssets
    this.createChunkAssets();
  }

  createChunkAssets() {
    // 遍历 chunks ，为每个 chunk 执行 render 操作
    for (const chunk of this.chunks) {
      // 2. 触发 renderManifest 钩子
      const res = this.hooks.renderManifest.call([], {
        chunk,
        codeGenerationResults: this.codeGenerationResults,
        ...others,
      });
      // 提交组装结果
      this.emitAsset(res.render(), ...others);
    }
  }
}

// lib/javascript/JavascriptModulesPlugin.js
class JavascriptModulesPlugin {
  apply() {
    compiler.hooks.compilation.tap("JavascriptModulesPlugin", (compilation) => {
      compilation.hooks.renderManifest.tap("JavascriptModulesPlugin", (result, options) => {
          // JavascriptModulesPlugin 插件中通过 renderManifest 钩子返回组装函数 render
          const render = () =>
            // render 内部根据 chunk 内容，选择使用模板 `renderMain` 或 `renderChunk`
            // 3. 监听钩子，返回打包函数
            this.renderMain(options);

          result.push({ render /* arguments */ });
          return result;
        }
      );
    });
  }

  renderMain() {/*  */}

  renderChunk() {/*  */}
}
```

这里的核心逻辑是，`compilation` 以 `renderManifest` 钩子方式对外发布 bundle 打包需求；`JavascriptModulesPlugin` 监听这个钩子，按照 `chunk` 的内容特性，返回不同的打包函数。

`JavascriptModulesPlugin` 内置的打包函数有：

- `renderMain`：打包主 `chunk` 时使用。
- `renderChunk`：打包子 `chunk`，如异步模块 `chunk` 时使用。

两个打包函数实现的逻辑接近，都是按照顺序拼接个个模块，下面简单介绍 `renderMain` 的实现。

## `renderMain` 函数

`renderMain` 函数涉及比较多场景判断，原始代码很长，摘几个重点步骤：

```javascript
class JavascriptModulesPlugin {
  renderMain(renderContext, hooks, compilation) {
    const { chunk, chunkGraph, runtimeTemplate } = renderContext;

    const source = new ConcatSource();
    // ...
    // 1. 先计算出 bundle CMD 核心代码，包含：
    //      - "var __webpack_module_cache__ = {};" 语句
    //      - "__webpack_require__" 函数
    const bootstrap = this.renderBootstrap(renderContext, hooks);

    // 2. 计算出当前 chunk 下，除 entry 外其它模块的代码
    const chunkModules = Template.renderChunkModules(
      renderContext,
      inlinedModules
        ? allModules.filter((m) => !inlinedModules.has(m))
        : allModules,
      (module) =>
        this.renderModule(
          module,
          renderContext,
          hooks,
          allStrict ? "strict" : true
        ),
      prefix
    );

    // 3. 计算出运行时模块代码
    const runtimeModules =
      renderContext.chunkGraph.getChunkRuntimeModulesInOrder(chunk);

    // 4. 重点来了，开始拼接 bundle
    // 4.1 首先，合并核心 CMD 实现，即上述 bootstrap 代码
    const beforeStartup = Template.asString(bootstrap.beforeStartup) + "\n";
    source.add(
      new PrefixSource(
        prefix,
        useSourceMap
          ? new OriginalSource(beforeStartup, "webpack/before-startup")
          : new RawSource(beforeStartup)
      )
    );

    // 4.2 合并 runtime 模块代码
    if (runtimeModules.length > 0) {
      for (const module of runtimeModules) {
        compilation.codeGeneratedModules.add(module);
      }
    }
    // 4.3 合并除 entry 外其它模块代码
    for (const m of chunkModules) {
      const renderedModule = this.renderModule(m, renderContext, hooks, false);
      source.add(renderedModule)
    }

    // 4.4 合并 entry 模块代码
    if (
      hasEntryModules &&
      runtimeRequirements.has(RuntimeGlobals.returnExportsFromRuntime)
    ) {
      source.add(`${prefix}return __webpack_exports__;\n`);
    }

    return source;
  }
}
```

核心逻辑为：

- 先计算出 bundle CMD 代码，即 `__webpack_require__` 函数。
- 计算出当前 `chunk` 下，除 `entry` 外其他模块代码 `chunkModules`。
- 计算出运行时模块代码。
- 开始执行合并操作，子步骤有：
  - 合并 CMD 代码
  - 合并 `runtime` 模块代码
  - 遍历 `chunkModules` 变量，合并除了 `entry` 外其他模块的代码。
  - 合并 `entry` 模块代码
- 返回结果

总结：先计算出不同组成部分的产物形态，之后按照顺序拼接打包，输出合并后的版本。

至此，Webpack 完成 bundle 的转译、打包流程，后续调用 `compilation.emitAssets`，按照上下文环境将产物输出到 `fs` 即可，Webpack 单次编译打包过程就结束了。

# 总结

打包流程的后半截：

- 首先遍历 `chunk` 中的所有模块，为每个模块执行转译操作，产出模块级别的产物。
- 根据 `chunk` 的类型，选择不同的结构框架，按序逐次组装模块产物，打包成最终的 bundle。
