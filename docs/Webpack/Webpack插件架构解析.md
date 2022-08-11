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

- 基本逻辑

    `Waterfall` 钩子的执行逻辑跟 lodash 的 `flow` 函数有点像，大致上就是会将前一个函数的返回值作为参数传入下一个函数，可以简化为如下代码：

    ```javascript
    function waterfallCall(arg) {
        const callbacks = [fn1, fn2, fn3];
        let lastResult = arg;
        for (let i in callbacks) {
            const cb = callbacks[i];
            // 上次执行结果作为参数传入下一个函数
            lastResult = cb(lastResult);
        }
        return lastResult;
    }
    ```

    理解了上边的逻辑，可以知道 `SyncWaterfallHook` 的特点如下：

    1. 上一个函数的结果会被带入下一个函数
    2. 最后一个回调的结果会作为 `call` 调用的结果返回

- 示例

    举例说明：

    ```javascript
    const { SyncWaterfallHook } = require("tapable");

    class Somebody {
        constructor() {
            this.hooks = {
                sleep: new SyncWaterfallHook(["msg"]),
            };
        }
        sleep() {
            return this.hooks.sleep.call("hello");
        }
    }

    const person = new Somebody();

    // 注册回调
    person.hooks.sleep.tap("test", (arg) => {
        console.log(`call 调用传入： ${arg}`);
        return "tecvan";
    });

    person.hooks.sleep.tap("test", (arg) => {
        console.log(`A 回调返回： ${arg}`);
        return "world";
    });

    console.log("最终结果：" + person.sleep());
    // 运行结果：
    // call 调用传入： hello
    // A 回调返回： tecvan
    // 最终结果：world
    ```

    示例中 `sleep` 钩子为 `SyncWaterfallHook` 类型，之后注册了两个回调，从处理结果可以看到第一个回调收到的 `arg = hello`，第二个回调收到的是第一个回调返回的结果 `tevcan`，之后 `call` 调用返回的是第二个回调的结果 `world`。

    使用上，`SyncWaterfallHook` 钩子有一些注意事项：

    1. 初始化是必须提供参数，例如上例中 `new SyncWaterfallHook(["msg"])` 构造函数中必须传入 `["msg"]`，用于动态编译 `call` 的参数依赖，后面会讲到“动态编译”的细节。
    2. 发布调用 `call` 时，需要传入初始化参数。

- Webpack 场景解析

    `SyncWaterfallHook` 在 Webpack 中比较有代表性的例子是 `NormalModuleFactory.hooks.factory`，在 Webpack 内部实现中，会在这个钩子内根据资源类型 `resolve` 出对应的 `module` 对象：

    ```javascript
    class NormalModuleFactory {
        constructor() {
            this.hooks = {
                factory: new SyncWaterfallHook(["filename", "data"]),
            };

            this.hooks.factory.tap("NormalModuleFactory", () => (result, callback) => {
                let resolver = this.hooks.resolver.call(null);

                if (!resolver) return callback();

                resolver(result, (err, data) => {
                    if (err) return callback(err);

                    // direct module
                    if (typeof data.source === "function") return callback(null, data);

                    // ...
                });
            });
        }

        create(data, callback) {
            //   ...
            const factory = this.hooks.factory.call(null);
            // ...
        }
    }   
    ```

    大致上就是在创建模块，通过 `factory` 钩子将 `module` 的创建过程外包出去，在钩子回调队列中依据 `waterfall` 的特性逐步推断出最终的 `module` 对象。

##### SyncLoopHook 钩子

- 基本逻辑

    `loop` 钩子的特点是循环执行知道所有回调都返回 `undefined`，不过这里循环的单位是单个回调函数，例如有回调队列 `[fn1, fn2, fn3]`，`loop` 钩子先执行 `fn1`，如果此时 `fn1` 返回了非 `undefined` 值，则继续执行 `fn1` 直到返回 `undefined` 后才向前推进执行 `fn2`。伪代码如下：

    ```javascript
    function loopCall() {
        const callbacks = [fn1, fn2, fn3];
        for (let i in callbacks) {
            const cb = callbacks[i];
            // 重复执行
            while (cb() !== undefined) {}
        }
    }
    ```

- 示例

    由于 `loop` 钩子循环执行的特性，使用时需要注意避免陷入死循环。示例：

    ```javascript
    const { SyncLoopHook } = require("tapable");

    class Somebody {
        constructor() {
            this.hooks = {
                sleep: new SyncLoopHook(),
            };
        }
        sleep() {
            return this.hooks.sleep.call();
        }
    }

    const person = new Somebody();
    let times = 0;

    // 注册回调
    person.hooks.sleep.tap("test", (arg) => {
        ++times;
        console.log(`第 ${times} 次执行回调A`);
        if (times < 4) {
            return times;
        }
    });

    person.hooks.sleep.tap("test", (arg) => {
        console.log(`执行回调B`);
    });

    person.sleep();
    // 运行结果
    // 第 1 次执行回调A
    // 第 2 次执行回调A
    // 第 3 次执行回调A
    // 第 4 次执行回调A
    // 执行回调B
    ```

    可以看到示例中一直在执行回调 A，直到满足判定条件 `times >= 4`，A 返回 `undefined` 后才开始执行回调 B。

    虽然 Tapable 提供了 `SyncLoopHook` 钩子，但是 Webpack 中并没有使用到。

#### 异步钩子

除了同步钩子外，Tapable 还提供了一系列 `Async` 的异步钩子，支持在回调函数中执行异步操作，逻辑相对比较复杂。

##### AsyncSeriesHook 钩子

- 基本逻辑

    1. 支持异步回调，可以在回调函数中写 `callback` 或 `promise` 风格的异步操作。
    2. 回调队列一次执行，前一个执行结束后，才会开始执行下一个。
    3. 和 `SyncHook` 一样，不需要关心回调的执行结果。

    用一段伪代码来表示：

    ```javascript
    function asyncSeriesCall(callback) {
        const callbacks = [fn1, fn2, fn3];
        //   执行回调 1
        fn1((err1) => {
            if (err1) {
                callback(err1);
            } else {
                //   执行回调 2
                fn2((err2) => {
                    if (err2) {
                        callback(err2);
                    } else {
                        //   执行回调 3
                        fn3((err3) => {
                            if (err3) {
                                callback(err2);
                            }
                        });
                    }
                });
            }
        });
    }
    ```

- 示例

    先看 `callback` 风格的示例：

    ```javascript
    const { AsyncSeriesHook } = require("tapable");

    const hook = new AsyncSeriesHook();

    // 注册回调
    hook.tapAsync("test", (cb) => {
        console.log("callback A");
        setTimeout(() => {
            console.log("callback A 异步操作结束");
            // 回调结束时，调用 cb 通知 tapable 当前回调结束
            cb();
        }, 100);
    });

    hook.tapAsync("test", () => {
        console.log("callback B");
    });

    hook.callAsync();
    // 运行结果：
    // callback A
    // callback A 异步操作结束
    // callback B
    ```

    从代码输出结果可以看出，A 回调内部的 `setTimeout` 执行完毕调用 `cb` 函数，Tapable 才认为当前回调执行完毕，开始执行 B 回调。

    除了 `callback` 风格外，也可以使用 `promise` 风格调用 `tap/call` 函数，改造上例如下：

    ```javascript
    const { AsyncSeriesHook } = require("tapable");

    const hook = new AsyncSeriesHook();

    // 注册回调
    hook.tapPromise("test", () => {
        console.log("callback A");
        return new Promise((resolve) => {
            setTimeout(() => {
                console.log("callback A 异步操作结束");
                resolve();
            }, 100);
        });
    });

    hook.tapPromise("test", () => {
        console.log("callback B");
        return Promise.resolve();
    });

    hook.promise();
    // 运行结果
    // callback A
    // callback A 异步操作
    // callback B
    ```

    有三个改动点：

    1. 将 `tapAsync` 更改为 `tapPromise`。
    2. 回调函数需要返回 `Promise` 对象。
    3. `callAsync` 调用需要更改为 `promise`。

- Webpack 场景解析

    AsyncSeriesHook 钩子在 Webpack 中的应用要数构建完毕后触发 `compiler.hooks.done` 钩子，用于通知单次构建已经结束：

    ```javascript
    class Compiler {
        run(callback) {
            if (err) return finalCallback(err);

            this.emitAssets(compilation, (err) => {
                if (err) return finalCallback(err);

                if (compilation.hooks.needAdditionalPass.call()) {
                    // ...
                    this.hooks.done.callAsync(stats, (err) => {
                        if (err) return finalCallback(err);

                        this.hooks.additionalPass.callAsync((err) => {
                            if (err) return finalCallback(err);
                            this.compile(onCompiled);
                        });
                    });
                    return;
                }

                this.emitRecords((err) => {
                    if (err) return finalCallback(err);

                    // ...
                    this.hooks.done.callAsync(stats, (err) => {
                        if (err) return finalCallback(err);
                        return finalCallback(null, stats);
                    });
                });
            });
        }
    }
    ```

##### AsyncParallelHook 钩子

- 基本逻辑

    和 `AsyncSeriesHook` 类似，`AsyncParallelHook` 也支持异步风格的回调，不过 `AsyncParallelHook` 是以并行方式，其特点：
    
    1. 支持异步风格。
    2. 并行执行回调队列，不需要做任何等待。
    3. 和 `SyncHook` 一样，不关心回调执行结果。

    逻辑上近似于：

    ```javascript
    function asyncParallelCall(callback) {
        const callbacks = [fn1, fn2];
        // 内部维护了一个计数器
        var _counter = 2;

        var _done = function() {
            _callback();
        };
        if (_counter <= 0) return;
        // 按序执行回调
        var _fn0 = callbacks[0];
        _fn0(function(_err0) {
            if (_err0) {
            if (_counter > 0) {
                // 出错时，忽略后续回调，直接退出
                _callback(_err0);
                _counter = 0;
            }
            } else {
                if (--_counter === 0) _done();
            }
        });
        if (_counter <= 0) return;
        // 不需要等待前面回调结束，直接开始执行下一个回调
        var _fn1 = callbacks[1];
        _fn1(function(_err1) {
            if (_err1) {
            if (_counter > 0) {
                _callback(_err1);
                _counter = 0;
            }
            } else {
                if (--_counter === 0) _done();
            }
        });
    }
    ```

### Tapable 动态编译

Tapable 中有一套大胆的设计：动态编译，所谓的同步、异步、Bail、Waterfall、Loop 等回调规则都是基于动态编译能力实现的，所有深入学习 Tapable 必须细究动态编译的特性。

当用户执行钩子发布函数 `call/callAsync/promise` 时，Tapable 会根据钩子类型、参数、回调队列等信息动态生成执行函数，例如对于下面的例子：

```javascript
const { SyncHook } = require("tapable");

const sleep = new SyncHook();

sleep.tap("test", () => {
    console.log("callback A");
});

sleep.call();
```

调用 `sleep.call()` 时，Tapable 内部处理流程大致为：

![webpack](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/webpack16.png)

编译过程主要设计三个实体：

- `tapable/lib/SyncHook.js`：定义 `SyncHook` 的入口文件
- `tapable/lib/Hook.js`：`SyncHook` 只是一个简单接口，内部实际上调用了 `Hook` 类，由 `Hook` 实现钩子的逻辑，其他钩子也是类似的道理。
- `tapable/lib/HookCodeFactory.js`：动态编译出 `call`、`callAsync`、`promise` 函数内容的工厂类，其他钩子也是会用到 `HookCodeFactory` 工厂函数。

`SyncHook` 调用了 `call` 函数之后，`Hook` 基类收集上下文信息，并调用 `createCall` 及子类传入的 `compile` 函数，`compile` 调用 `HookCodeFactory` 进而使用 `new Function` 方法动态拼接出回调执行函数。上方例子生成的对应函数如下：

```javascript
(function anonymous() {
    "use strict";
    var _context;
    var _x = this._x;
    var _fn0 = _x[0];
    _fn0();
})
```

动态编译在一般场景下会存在性能或安全的问题，因而社区甚少见到类似的设计，看一个稍微复杂一点的例子：

```javascript
const { AsyncSeriesWaterfallHook } = require("tapable");

const sleep = new AsyncSeriesWaterfallHook(["name"]);

sleep.tapAsync("test1", (name, cb) => {
  console.log(`执行 A 回调： 参数 name=${name}`);
  setTimeout(() => {
    cb(undefined, "tecvan2");
  }, 100);
});

sleep.tapAsync("test", (name, cb) => {
  console.log(`执行 B 回调： 参数 name=${name}`);
  setTimeout(() => {
    cb(undefined, "tecvan3");
  }, 100);
});

sleep.tapAsync("test", (name, cb) => {
  console.log(`执行 C 回调： 参数 name=${name}`);
  setTimeout(() => {
    cb(undefined, "tecvan4");
  }, 100);
});

sleep.callAsync("tecvan", (err, name) => {
  console.log(`回调结束， name=${name}`);
});

// 运行结果：
// 执行 A 回调： 参数 name=tecvan
// 执行 B 回调： 参数 name=tecvan2
// 执行 C 回调： 参数 name=tecvan3
// 回调结束， name=tecvan4
```

示例用到了 `AsyncSeriesWaterfallHook`，这个钩子的特点是：异步、串行、前一个回调的返回值会传入下一个回调中，对应生成的函数如下：

```javascript
(function anonymous(name, _callback) {
  "use strict";
  var _context;
  var _x = this._x;
  function _next1() {
    var _fn2 = _x[2];
    _fn2(name, function(_err2, _result2) {
      if (_err2) {
        _callback(_err2);
      } else {
        if (_result2 !== undefined) {
          name = _result2;
        }
        _callback(null, name);
      }
    });
  }
  function _next0() {
    var _fn1 = _x[1];
    _fn1(name, function(_err1, _result1) {
      if (_err1) {
        _callback(_err1);
      } else {
        if (_result1 !== undefined) {
          name = _result1;
        }
        _next1();
      }
    });
  }
  var _fn0 = _x[0];
  _fn0(name, function(_err0, _result0) {
    if (_err0) {
      _callback(_err0);
    } else {
      if (_result0 !== undefined) {
        name = _result0;
      }
      _next0();
    }
  });
});
```

这段生成函数的特点：

- 生成函数将回调队列中各个回调函数封装成对应的函数，如上的对应的回调函数分别为：`_fn0`、`_next0`、`_next1`，这些函数中的逻辑存在一定的相似度。
- 按回调定义的顺序，逐次执行，上一个回调结束后，才调用下一个回调。

相对于使用递归、循环之类的实现，这段生成函数的逻辑会更加清晰，更容易理解。

Tapable 提供的大多数特性都是基于 `Hook` 和 `HookCodeFactory` 来实现的，源码位置在 `tapable/lib/Hook.js` 中。

### 高级特性 Intercept

除了通常使用的 `tap/call` 之外，Tapable 还提供了简易的中间件机制— `intercept` 接口，如下所示：

```javascript
const sleep = new SyncHook();

sleep.intercept({
  name: "test",
  context: true,
  call() {
    console.log("before call");
  },
  loop(){
    console.log("before loop");
  },
  tap() {
    console.log("before each callback");
  },
  register() {
    console.log("every time call tap");
  },
});
```

`intercept` 支持注册如下类型的中间件：

<table>
    <th>
        <td>签名</td>
        <td>解释</td>
    </th>
    <tr>
        <td>call</td>
        <td>(...args) => void</td>
        <td>调用 call/callAsync/promise 时触发</td>
    </tr>
    <tr>
        <td>tap</td>
        <td>(tap: Tap) => void</td>
        <td>调用 call 类函数后，每次调用回调之前触发</td>
    </tr>
    <tr>
        <td>loop</td>
        <td>(...args) => void</td>
        <td>仅 loop 型钩子函数之前触发</td>
    </tr>
    <tr>
        <td>register</td>
        <td>(tap: Tap) => Tap | undefined</td>
        <td>调用 tap/tapAsync/tapPromise 时触发</td>
    </tr>
</table>

