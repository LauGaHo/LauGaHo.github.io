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

