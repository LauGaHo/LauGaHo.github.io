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

