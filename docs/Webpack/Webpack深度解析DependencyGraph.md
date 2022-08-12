# Webpack深度解析DependencyGraph

## 背景

Dependecy Graph 的原文解释如下：

> Any time one file depends on another, webpack treats this as a dependency. This allows webpack to take non-code assets, such as images or web fonts, and also provide them as dependencies for your application.
When webpack processes your application, it starts from a list of modules defined on the command line or in its configuration file. Starting from these entry points, webpack recursively builds a dependency graph that includes every module your application needs, then bundles all of those modules into a small number of bundles - often, just one - to be loaded by the browser.

翻译后的核心意思是：Webpack 处理应用代码时，会从开发者提供的 `entry` 开始递归组建起包含所有模块的 `dependency graph`，之后再将这些 `module` 打包成 `bundles`。

Dependency Graph 贯穿 Webpack 整个运行周期，从 `make` 阶段的模块解析，到 `seal` 阶段的 `chunk` 生成，以及 `tree-shaking` 功能都高度依赖于 Dependency Graph，是 Webpack 资源构建的一个非常核心的数据结构。

接下来将围绕 Webpack 5.0 的 Dependency Graph 来展开讨论：

- Dependency Graph 在 Webpack 实现中以何种数据结构呈现的。
- Webpack 运行过程中如何收集模块间依赖关系，进而构建出 Dependency Graph。
- Dependency Graph 构建完毕后，是如何被消费的。

## Dependency Graph

回顾一下 Webpack 的重要概念：

- **`Module`**：资源在 Webpack 内部的映射对象，包含了资源的路径、上下文、依赖、内容等信息。
- **`Dependency`**：在模块中引用其他模块，例如：`import "a.js"` 语句，Webpack 会先将引用关系表述为 `Dependency` 子类并关联 `module` 内容都解析完毕之后，启动下次循环开始将 `Dependency` 对象转换为适当的 `Module` 子类。
- **`Chunk`**：用于组织输出结构的对象，Webpack 分析完所有模块资源的内容，构建出完整的 `Dependency Graph` 之后，会根据用户配置及 `Dependency Graph` 内容构建出一个或多个 `chunk`，每个 `chunk` 和最终输出的文件大致上是一一对应。

### 数据结构

Webpack 4 的 `Dependency Graph` 实现比较简单，主要由 `Dependency/Module` 内置的系列属性记录引用、被引用关系。

而 Webpack 5 之后则实现了一套相对复杂的类结构记录模块间依赖关系，将模块依赖相关的逻辑从 `Dependency/Module` 解耦为一套独立的类型结构，主要类型有：

- **`ModuleGraph`**：记录 `Dependency Graph` 信息的容器，一方面保存了构建过程中涉及到的所有 `module`、`dependency` 对象，以这些对象互相之间的引用；另一方面提供了各种工具方法，方便使用者迅速读取出 `module` 或 `dependency` 附加的信息。
- **`ModuleGraphConnection`**：记录模块间引用关系的数据结构，内部通过 `originModule` 属性记录引用关系中的父模块，通过 `module` 属性记录子模块。此外还提供了一系列函数工具用于判断对应的引用关系的有效性。
- **`ModuleGraphModule`**：`Module` 对象在 `Dependency Graph` 体系下的补充信息，包含模块对象的 `incomingConnections` — 指向模块本身的 `ModuleGraphConnections` — 该模块对外的依赖，即该模块引用了其他那些模块。

类间关系大致为：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack18.png)

从上方类图就可以得知：

- `ModuleGraph` 对象通过 `_dependencyMap` 属性记录 `Dependency` 对象和 `ModuleGraphConnection` 连接对象之间的映射关系，后续的处理中可以基于这层映射迅速找到 `Dependency` 实例对应的引用和被引用者。
- `ModuleGraph` 对象通过 `_moduleMap` 在 `module` 基础上附加 `ModuleGraphModule` 信息，而 `ModuleGraphModule` 最大的作用就是记录了模块的引用和被引用关系，后续的处理可以基于该属性找到 `module` 实例的所有依赖和被依赖关系。

## 依赖收集过程

`ModuleGraph`、`ModuleGraphConnection`、`ModuleGraphModule` 三者协作，在 Webpack 构建阶段 `make` 中逐步收集模块间的依赖关系。

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack2.png)

根据上图所示，依赖关系收集主要发生在两个节点：

- **`addDependency`**：Webpack 从模块内容中解析出引用关系之后，创建适当的 `Dependency` 子类并调用该方法记录到 `module` 实例中。
- **`handleModuleCreation`**：模块解析完之后，Webpack 遍历父模块的依赖集合，调用该方法创建 `Dependency` 对应的子模块对象，之后调用 `compilation.moduleGraph.setResolvedModule` 方法将父子引用信息记录到 `moduleGraph` 对象上。

`setResolvedModule` 方法的逻辑大致为：

```javascript
class ModuleGraph {
    constructor() {
        /** @type {Map<Dependency, ModuleGraphConnection>} */
        this._dependencyMap = new Map();
        /** @type {Map<Module, ModuleGraphModule>} */
        this._moduleMap = new Map();
    }

    /**
     * @param {Module} originModule the referencing module
     * @param {Dependency} dependency the referencing dependency
     * @param {Module} module the referenced module
     * @returns {void}
     */
    setResolvedModule(originModule, dependency, module) {
        const connection = new ModuleGraphConnection(
            originModule,
            dependency,
            module,
            undefined,
            dependency.weak,
            dependency.getCondition(this)
        );
        this._dependencyMap.set(dependency, connection);
        const connections = this._getModuleGraphModule(module).incomingConnections;
        connections.add(connection);
        const mgm = this._getModuleGraphModule(originModule);
        if (mgm.outgoingConnections === undefined) {
            mgm.outgoingConnections = new Set();
        }
        mgm.outgoingConnections.add(connection);
    }
}
```

上方代码主要更改了 `_dependencyMap` 及 `moduleGraphModule` 的 `incomingConnections` 和 `outgoingConnections` 属性，以此收集当前模块的上下游依赖关系。

## 实例解析

有一依赖关系，如下图：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack19.png)

Webpack 启动后，在构建阶段递归调用 `compilation.handleModuleCreation` 函数，逐步补齐 `Dependency Graph` 结构，最终可能生成如下数据结构：

```javascript
ModuleGraph: {
    _dependencyMap: Map(3){
        { 
            EntryDependency{request: "./src/index.js"} => ModuleGraphConnection{
                module: NormalModule{request: "./src/index.js"}, 
                // 入口模块没有引用者，故设置为 null
                originModule: null
            } 
        },
        { 
            HarmonyImportSideEffectDependency{request: "./src/a.js"} => ModuleGraphConnection{
                module: NormalModule{request: "./src/a.js"}, 
                originModule: NormalModule{request: "./src/index.js"}
            } 
        },
        { 
            HarmonyImportSideEffectDependency{request: "./src/a.js"} => ModuleGraphConnection{
                module: NormalModule{request: "./src/b.js"}, 
                originModule: NormalModule{request: "./src/index.js"}
            } 
        }
    },

    _moduleMap: Map(3){
        NormalModule{request: "./src/index.js"} => ModuleGraphModule{
            incomingConnections: Set(1) [
                // entry 模块，对应 originModule 为null
                ModuleGraphConnection{ module: NormalModule{request: "./src/index.js"}, originModule:null }
            ],
            outgoingConnections: Set(2) [
                // 从 index 指向 a 模块
                ModuleGraphConnection{ module: NormalModule{request: "./src/a.js"}, originModule: NormalModule{request: "./src/index.js"} },
                // 从 index 指向 b 模块
                ModuleGraphConnection{ module: NormalModule{request: "./src/b.js"}, originModule: NormalModule{request: "./src/index.js"} }
            ]
        },
        NormalModule{request: "./src/a.js"} => ModuleGraphModule{
            incomingConnections: Set(1) [
                ModuleGraphConnection{ module: NormalModule{request: "./src/a.js"}, originModule: NormalModule{request: "./src/index.js"} }
            ],
            // a 模块没有其他依赖，故 outgoingConnections 属性值为 undefined
            outgoingConnections: undefined
        },
        NormalModule{request: "./src/b.js"} => ModuleGraphModule{
            incomingConnections: Set(1) [
                ModuleGraphConnection{ module: NormalModule{request: "./src/b.js"}, originModule: NormalModule{request: "./src/index.js"} }
            ],
            // b 模块没有其他依赖，故 outgoingConnections 属性值为 undefined
            outgoingConnections: undefined
        }
    }
}
```

从上面的 `Dependency Graph` 可以看出，本质上 `ModuleGraph._moduleMap` 已经成为了一个有向无环图结构，其中 `_moduleMap` 中的 key 为图的节点，对应的 value `ModuleGraphModule` 结构中的 `outgoingConnections` 属性则为图的边，则上例中从起点 `index.js` 出发沿着 `outgoingConnections` 向前可遍历出图的所有顶点。

## ModuleGraph Api

`ModuleGraph` 类型提供了许多实现 `module/dependency` 信息查询的工具函数，例如：

- `getModule(dep: Dependency)`：根据 `dep` 查找对应的 `module` 实例。
- `getOutgoingConnections(module: Module)`：查找 `module` 实例的所有依赖。
- `getIssuer(module: Module)`：查找 `module` 在何处被引用。