# Webpack—Chunk分包规则详解

## 背景

Webpack 实现中，原始的资源模块以 `Module` 对象形式存在、流转、解析处理。

而 `Chunk` 则是输出产物的基本组织单位，在生成阶段 Webpack 按规则将 `entry` 及其它 `Module` 插入 `Chunk` 中，之后再由 `SplitChunksPlugin` 插件根据优化规则与 `ChunkGraph` 对 `Chunk` 做一系列的变化、拆解、合并操作，重新组织成一批性能 (可能) 更高的 `Chunks`。运行完毕之后 Webpack 继续将 `chunk` 一一写入物理文件中，完成编译工作。

`Module` 主要作用是在 Webpack 编译过程的前半段，解决原始资源“如何读”的问题；而 `Chunk` 对象则主要作用在编译的后半段，解决编译产物“如何写”的问题，两者结合搭建起 Webpack 搭建主流程。

`Chunk` 的编排规则非常复杂，涉及 `entry`、`optimization` 等诸多配置项，本文将讲解 `entry`、`runtime`、异步模块三条规则的细节和原理。

## 默认分包规则

Webpack 4 之后编译过程大致分成了四个阶段：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/未命名绘图.png)

在构建阶段，Webpack 从 `entry` 出发根据模块的引用关系逐步构建出模块依赖图 `ModuleDependencyGraph`，依赖关系图表达了模块和模块之间互相引用的先后次序，基于这种次序 Webpack 就可以推断出模块运行之前需要先执行哪些依赖模块，也就可以推断出哪些模块应该打包在一起，哪些模块可以延后加载 (异步加载)，关于模块依赖图的更多信息将在后面的文章讲解。

到了生成阶段，Webpack 会根据模块依赖图的内容组织分包—`Chunk` 对象，默认的分包规则有：

- 同一个 `entry` 下触达到的模块组织成一个 `chunk`
- 异步模块单独组织为一个 `chunk`
- `entry.runtime` 单独组织成一个 `chunk`

默认规则集中在 `compilation.seal` 函数实现，`seal` 核心逻辑运行结束后会生成一系列的 `Chunk`、`ChunkGroup`、`ChunkGraph` 对象，后续如 `SplitChunksPlugin` 插件会在 `Chunk` 系列对象上进行进一步的拆解、优化，最终反映到输出上才会表现出复杂的分包结果。

接下来解释一下，默认生成规则。

## `Entry` 分包处理

> 生成阶段会遍历 `entry` 对象，为每一个 `entry` 单独生成 `chunk`，之后再根据模块依赖图将 `entry` 触达的所有模块打包进 `chunk` 中。

在生成阶段，Webpack 首先根据遍历用户提供的 `entry` 属性值，为每一个 `entry` 创建 `Chunk` 对象，比如对于如下配置：

```javascript
module.exports = {
  entry: {
    main: "./src/main",
    home: "./src/home",
  }
};
```

Webpack 遍历 `entry` 对象属性并创建出 `chunk[main]`、`chunk[home]` 两个对象，此时两个 `chunk` 分别包含 `main`、`home` 模块：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/pic2.png)

初始化完毕后，Webpack 会读取 `ModuleDependencyGraph` 的内容，将 `entry` 所对应的内容塞入对应的 `chunk` (发生在 `webpack/lib/buildChunkGraph.js` 文件)。比如对于如下文件依赖：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/pic4.png)

`main.js` 以同步方式直接或间接引用了 `a/b/c/d` 四个文件，分析 `ModuleDependencyGraph` 过程会逐步将 `a/b/c/d` 模块逐步添加到 `chunk[main]` 中，最终形成：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/LAaLiG.png)

> 基于 `entry` 生成的 `chunk` 在 Webpack 官方文档中，通常称之为 `intial chunk`。
