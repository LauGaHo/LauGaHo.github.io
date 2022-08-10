# Webpack插件架构解析

## 插件的定义

从插件的结构上来看，插件通常是一个带有 `apply` 函数的类，如下：

```javascript
class SomePlugin {

    apply(compiler) {

    }

}
```

Webpack 会在启动后按照注册顺序逐个调用插件对象的 `apply` 函数，同时传入编译器对象 `compiler`，插件开发者可以以此为基点触碰到 Webpack 内部定义的任意钩子，例如：

```javascript
class SomePlugin {

    apply(compiler) {
        compiler.hooks.thisCompilation.tap('SomePlugin', (compilation) => {

        })
    }

}
```

对于以下语句 `compiler.hooks.this.Compilation.tap`，其中 `this.Compilation` 为 Tapable 库中提供的钩子对象；`tap` 为钩子对象的订阅函数，用于注册回调。

Webpack 的插件体系是基于 Tapable 提供的各类钩子展开，所以需要熟悉 Tapable 提供的钩子类型和各自的特点。

## Tapable 解析

### 基本用法

Tapable 使用时需要经历如下步骤：

- 创建钩子实例。
- 调用订阅接口注册回调，调用接口包括：`tap`、`tapAsync`、`tapPromise`。
- 调用发布接口触发回调，发布接口包括：`call`、`callAsync`、`promise`。

举例说明：

```javascript
const { SyncHook } = require("tapable");

// 创建钩子实例
const sleep = new SyncHook();

// 调用订阅接口注册回调
sleep.tap("test", () => {
    console.log("callback A");
});

// 调用发布接口触发回调
sleep.call();

// 运行结果
// callback A
```

示例中使用 `tap` 函数注册回调，使用 `call` 触发回调，在某些钩子还可以使用异步风格的 `tapAsync/callAsync` 或者 `Promise` 风格的 `tapPromise/promise`，具体使用哪一类函数和钩子的类型有关。

### Tapable 钩子类型

|                          | 简介               | 统计                                     |
| ------------------------ | ------------------ | ---------------------------------------- |
| SyncHook                 | 同步钩子           | Compiler.hooks.compilation               |
| SyncBailHook             | 同步熔断钩子       | Compiler.hooks.shouldEmit                |
| SyncWaterfallHook        | 同步瀑布流钩子     | Compilation.hooks.assetPath              |
| SyncLoopHook             | 同步循环钩子       | not use                                  |
| AsyncParallelHook        | 异步并行钩子       | Compiler.hooks.make                      |
| AsyncParallelBailHook    | 异步并行熔断钩子   | not use                                  |
| AsyncSeriesHook          | 异步串行钩子       | Compiler.hooks.done                      |
| AsyncSeriesBailHook      | 异步串行熔断钩子   | Compilation.hooks.optimizeChunkModules   |
| AsyncSeriesLoopHook      | 异步串行循环钩子   | not use                                  |
| AsyncSeriesWaterfallHook | 异步串行瀑布流钩子 | ContextModuleFactory.hooks.beforeResolve |

对于上述的钩子，可以通过两个维度进行区分：
- 按照回调逻辑类型：
  - 基本类型：名称中不携带 `Waterfall`、`Bail`、`Loop` 关键字，和普通的订阅/回调模式相似，按钩子注册顺序，逐次调用。
  - `Waterfall` 类型：前一个回调的返回值会被带入下一个回调。
  - `Bail` 类型：逐次调用回调，若有任何一个回调返回非 `undefined`，则终止后续调用。
  - `Loop` 类型：逐次、循环调用，直到所有回调函数都返回 `undefined`。

- 按照执行回调的并行方式
  - `sync`：同步执行，启动后会按次序逐个执行回调，支持 `call/tap` 调用语句。
  - `async`：异步执行，支持传入 `callback` 或 `promise` 风格的异步回调函数，支持 `callAsync/tapAsync` 和 `promise/tapPromise` 两种调用语句。

所有钩子都可以按名称套到这两条规则中，对插件开发者来说不同类型的钩子会直接影响到回调函数的写法，以及插件和其他插件的互通关系，紧接着会细说各种类型钩子的特点。

#### 同步钩子

##### SyncHook 钩子

- 基本逻辑

    `SyncHook` 应该是最简单的钩子，触发后会按照注册的顺序逐个调用回调，并且不关心回调的返回值，逻辑如下：

    ```javascript
    function syncCall() {
        const callbacks = [fn1, fn2, fn3];
        for (let i = 0; i < callbacks.length; i++) {
            const cb = callbacks[i];
            cb();
        }
    }
    ```

- 示例

    ```javascript
    const { SyncHook } = require("tapable");

    class Somebody {
    constructor() {
        this.hooks = {
            sleep: new SyncHook(),
        };
    }
    sleep() {
        //   触发回调
        this.hooks.sleep.call();
    }
    }

    const person = new Somebody();

    // 注册回调
    person.hooks.sleep.tap("test", () => {
        console.log("callback A");
    });
    person.hooks.sleep.tap("test", () => {
        console.log("callback B");
    });
    person.hooks.sleep.tap("test", () => {
        console.log("callback C");
    });

    person.sleep();
    // 输出结果：
    // callback A
    // callback B
    // callback C
    ```

    示例中，`Somebody` 初始化时声明了一个 `sleep` 钩子，并在后续调用 `sleep.tap` 函数连续注册三次回调，在调用 `person.sleep()` 语句触发了 `this.hooks.sleep.call()` 之后，Tapable 会按照注册的先后顺序执行三个回调。

##### SyncBailHook 钩子

- 基本逻辑

    Bail 单词有着熔断的意思，Bail 类型的钩子的特点就是在回调队列中，倘若任意一个回调函数返回了非 `undefined` 的值，则中断后续处理，直接返回该值，例子如下：

    ```javascript
    function bailCall() {
        const callbacks = [fn1, fn2, fn3];
        for (let i in callbacks) {
            const cb = callbacks[i];
            const result = cb(lastResult);
            if (result !== undefined) {
                // 熔断
                return result;
            }
        }
        return undefined;
    }
    ```

- 示例

    `SyncBailHook` 的调用顺序和规则都跟 `SyncHook` 相似，主要区别是 `SyncBailHook` 增加了熔断逻辑，如下：

    ```javascript
    const { SyncBailHook } = require("tapable");

    class Somebody {
    constructor() {
        this.hooks = {
        sleep: new SyncBailHook(),
        };
    }
    sleep() {
        return this.hooks.sleep.call();
    }
    }

    const person = new Somebody();

    // 注册回调
    person.hooks.sleep.tap("test", () => {
        console.log("callback A");
        // 熔断点
        // 返回非 undefined 的任意值都会中断回调队列
        return '返回值：tecvan'
    });
    person.hooks.sleep.tap("test", () => {
        console.log("callback B");
    });

    console.log(person.sleep());

    // 运行结果：
    // callback A
    // 返回值：tecvan
    ```

    相比于 `SyncHook`，`SyncBailHook` 运行结束后，会将熔断值返回给 `call` 函数，例如上例中的 `return '返回值：tecvan'`，`callback A` 返回的返回值 `tecvan` 会成为 `this.hooks.sleep.call` 的调用结果。

- Webpack 场景解析

    `SyncBailHook` 通常在发布者需要关心订阅回调运行结果的场景，Webpack 内部有多个地方用到这种钩子，就如：`compiler.hooks.shouldEmit`，对应的 `call` 语句：

    ```javascript
    class Compiler {
        // ......

        const onCompiled = (err, compilation) => {
            if (this.hooks.shouldEmit.call(compilation) === false) {
                // ......
            }
        };

    }
    ```

    此处 Webpack 会根据 `shouldEmit` 钩子运行的结果确定是否执行后续的操作。

##### SyncWaterfallHook 钩子

