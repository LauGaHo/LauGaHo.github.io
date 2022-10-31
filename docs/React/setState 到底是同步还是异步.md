# setState 到底是同步还是异步

## 从一道面试题出发

题目给出一个这样的 App 组件，在他的内部会有如下代码所示的几个不同的 `setState` 操作：

```javascript
import React from "react";
import "./styles.css";
export default class App extends React.Component{
  state = {
    count: 0
  }
  increment = () => {
    console.log('increment setState前的count', this.state.count)
    this.setState({
      count: this.state.count + 1
    });
    console.log('increment setState后的count', this.state.count)
  }
  triple = () => {
    console.log('triple setState前的count', this.state.count)
    this.setState({
      count: this.state.count + 1
    });
    this.setState({
      count: this.state.count + 1
    });
    this.setState({
      count: this.state.count + 1
    });
    console.log('triple setState后的count', this.state.count)
  }
  reduce = () => {
    setTimeout(() => {
      console.log('reduce setState前的count', this.state.count)
      this.setState({
        count: this.state.count - 1
      });
      console.log('reduce setState后的count', this.state.count)
    },0);
  }
  render(){
    return <div>
      <button onClick={this.increment}>点我增加</button>
      <button onClick={this.triple}>点我增加三倍</button>
      <button onClick={this.reduce}>点我减少</button>
    </div>
  }
}
```

接着把组件挂载到 DOM 上：

```javascript
import React from "react";
import ReactDOM from "react-dom";
import App from "./App";

const rootElement = document.getElementById("root");

ReactDOM.render(
    <React.StrictMode>
        <App />
    </React.StrictMode>
)
```

此时浏览器会渲染三个按钮：

![react](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/QVfuZA.png)

此时有一个问题，倘若从左到右依次点击每个按钮，控制台的输出将是怎么样的。实际上的输出结果将如下图所示：

![output](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/gUCJ2Q.png)

如果对 React 比较熟悉的开发者，那么 `increment` 这个方法的输出结果想必没有任何难度，正如大多数的 React 入门教学所声称的那样，“`setState` 是一个异步方法”，这就意味着当执行完 `setState` 之后，`state` 本身不会立刻发生改变。因此紧跟在 `setState` 后面输出的 `state` 值，仍然会维持在它的初始状态 (0)。在同步代码执行完之后的某一个神奇的时刻，`state` 才会恰恰好增加到 1。

但是这个神奇的时刻到底是什么时候发生，所谓的“恰恰好”又是怎么界定，如果对这个问题不清楚，那么 `triple` 方法的输出就将会具有一定的迷惑性——`setState` 一次不好使，`setState` 三次也没用，`state` 到底是在哪一个环节发生了变化。

带着这些疑问，你先看了一下 `reduce` 方法的输出结果，结果更是让人大跌眼镜，`reduce` 方法中的 `setState` 竟然是同步更新的。

要想理解眼前发生的魔幻一切，需要从 `setState` 的工作机制中寻找线索。

## 异步的动机和原理——批量更新的艺术

首先需要认知一个问题，在 `setState` 调用之后，都发生了什么事情？有些同学可能比较倾向站在声明周期的角度去思考这个问题，得出了一个如下图的结论：

![react-lifecycle](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/HLVDNw.png)

从图上可以看到，一个完整的更新流程，涉及包括 `re-render` (重渲染) 在内的多个步骤。`re-render` 本身涉及对 DOM 的操作，它会带来比较大的性能开销。假如一次 `setState` 就触发一个完整的更新流程这个结论成立，那么每一次 `setState` 的调用都会触发一次 `re-render`，那么视图很可能没刷新几次就被卡死了，这个过程如下面代码中的箭头流程图所示：

```javascript
this.setState({
  count: this.state.count + 1    ===>    shouldComponentUpdate->componentWillUpdate->render->componentDidUpdate
});
this.setState({
  count: this.state.count + 1    ===>    shouldComponentUpdate->componentWillUpdate->render->componentDidUpdate
});
this.setState({
  count: this.state.count + 1    ===>    shouldComponentUpdate->componentWillUpdate->render->componentDidUpdate
});
```

事实上，`这正是 setState 异步的一个重要动机：避免频繁的 re-render`。

在实际上的 React 运行时中，`setState` 异步的实现方式有点类似于 Vue 的 `$nextTick` 和浏览器中的 `Event-Loop`：`每来一个 setState，就把它塞进一个队列中“攒起来”`。等到时机成熟，再把“攒起来”的 `state` 结果合并，最后 `只针对最新的 state 值走一次更新流程`。这个过程，叫做“`批量更新`”，批量更新的过程正如下方代码中的箭头流程图所示：

```javascript
this.setState({
  count: this.state.count + 1    ===>    入队，[count+1的任务]
});
this.setState({
  count: this.state.count + 1    ===>    入队，[count+1的任务，count+1的任务]
});
this.setState({
  count: this.state.count + 1    ===>    入队, [count+1的任务，count+1的任务, count+1的任务]
});
                                          ↓
                                         合并 state，[count+1的任务]
                                          ↓
                                         执行 count+1的任务
```

> 值得注意的是，只要我们的同步代码还在执行，“攒起来”这个动作就不会停止。(注：这里之所以多次 `+1` 最终只有一次生效，是因为在同一个方法中多次 `setState` 的合并动作不是单纯地更新累加。比如这里对于相同属性的设置，React 只会为其保留最后一次的更新)。因此就算在 React 中写了这样一个 100 次的 `setState` 循环：

```javascript
this.setState({
  count: this.state.count + 1    ===>    入队，[count+1的任务]
});
this.setState({
  count: this.state.count + 1    ===>    入队，[count+1的任务，count+1的任务]
});
this.setState({
  count: this.state.count + 1    ===>    入队, [count+1的任务，count+1的任务, count+1的任务]
});
                                          ↓
                                         合并 state，[count+1的任务]
                                          ↓
                                         执行 count+1的任务
```

> 也只是会增加 `state` 任务入队的次数，并不会带来频繁的 `re-render`。当 100 次调用结束后，仅仅是 `state` 的任务队列发生了变化，`state` 本身并不会立即发生改变。

![console](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/BPsQt2.png)

## “同步现象”背后的股市：从源码角度看 setState 工作流

接下来就要重点理解刚刚代码中最诡异的一部分—— `setState` 的同步现象：

```javascript
reduce = () => {
  setTimeout(() => {
    console.log('reduce setState前的count', this.state.count)
    this.setState({
      count: this.state.count - 1
    });
    console.log('reduce setState后的count', this.state.count)
  },0);
}
```

从题目上看，`setState` 似乎是在 `setTimeout` 函数的“保护”之下，才有了同步的这一个“特异功能”。事实也的确如此，假如将 `setTimeout` 摘掉，`setState` 前后的 `console.log` 表现将会和 `increment` 方法中无异：

```javascript
reduce = () => {
  // setTimeout(() => {
  console.log('reduce setState前的count', this.state.count)
  this.setState({
    count: this.state.count - 1
  });
  console.log('reduce setState后的count', this.state.count)
  // },0);
}
```

点击后的输出结果如下图所示：

![console](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/W4ltHk.png)

现在问题就变得清晰多了：`为什么 setTimeout 可以将 setState 的执行顺序从异步变为同步`？

> 这里先给出一个结论：`并不是 setTimeout 改变了 setState，而是 setTimeout 帮助 setState “逃脱”了 React 对它的管控`。`只要是在 React 管控下的 setState，一定是异步的`。

接下来将从 React 源码中，寻找佐证这个结论的线索。

> 时下虽然市场中的 React 16、React 17 十分火热，但就 `setState` 这块只是来说，React 15 仍然是最佳的学习素材。因此上下文所有涉及到源码的分析，都会围绕 React 15 展开。关于 React 16 之后 Fiber 机制给 `setState` 带来的改变，不在本讲的讨论范围中。

## 解读 setState 工作流

React 中对于功能的拆分是比较细致的，`setState` 这部分涉及了多个方法。为了方便理解，这里把主流程提取为一张大图：

![process](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/rDvX3H.png)

接着沿着这个流程，逐个在源码中对号入座。首先是 `setState` 入口函数：

```javascript
ReactComponent.prototype.setState = function (partialState, callback) {
  this.updater.enqueueSetState(this, partialState);
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState');
  }
};
```

入口函数在这里充当了一个分发器的角色，根据入参的不同，将其分发到不同的功能函数中去。这里以对象形式的入参为例，可以看到直接调用了 `this.updater.enqueueSetState` 这个方法：

```javascript
enqueueSetState: function (publicInstance, partialState) {
  // 根据 this 拿到对应的组件实例
  var internalInstance = getInternalInstanceReadyForUpdate(publicInstance, 'setState');
  // 这个 queue 对应的就是一个组件实例的 state 数组
  var queue = internalInstance._pendingStateQueue || (internalInstance._pendingStateQueue = []);
  queue.push(partialState);
  //  enqueueUpdate 用来处理当前的组件实例
  enqueueUpdate(internalInstance);
}
```

这里总结一下，`enqueueSetState` 做了两件事：

- 将新的 `state` 放进组件的状态队列中
- 用 `enqueueUpdate` 来处理将要更新的实例对象

继续往下走，看看 `enqueueUpdate` 函数的内容：

```javascript
function enqueueUpdate(component) {
  ensureInjected();
  // 注意这一句是问题的关键，isBatchingUpdates标识着当前是否处于批量创建/更新组件的阶段
  if (!batchingStrategy.isBatchingUpdates) {
    // 若当前没有处于批量创建/更新组件的阶段，则立即更新组件
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }
  // 否则，先把组件塞入 dirtyComponents 队列里，让它“再等等”
  dirtyComponents.push(component);
  if (component._updateBatchNumber == null) {
    component._updateBatchNumber = updateBatchNumber + 1;
  }
}
```

这个 `enqueueUpdate` 的内容十分有意思，它引出了一个关键的对象 `batchingStrategy`，该对象具备的 `isBatchingUpdates` 属性直接决定了当下是走批量更新流程，还是排队等待。其中的 `batchedUpdates` 方法更是能够直接发起批量更新流程。由此可以推测，`batchingStrategy` 正是 React 内部专门用于管控批量更新的对象。

接着将细看这个 `batchingStrategy`：

```javascript
/**
 * batchingStrategy源码
**/
 
var ReactDefaultBatchingStrategy = {
  // 全局唯一的锁标识
  isBatchingUpdates: false,
 
  // 发起更新动作的方法
  batchedUpdates: function(callback, a, b, c, d, e) {
    // 缓存锁变量
    var alreadyBatchingStrategy = ReactDefaultBatchingStrategy.isBatchingUpdates
    // 把锁“锁上”
    ReactDefaultBatchingStrategy.isBatchingUpdates = true

    if (alreadyBatchingStrategy) {
      callback(a, b, c, d, e)
    } else {
      // 启动事务，将 callback 放进事务里执行
      transaction.perform(callback, null, a, b, c, d, e)
    }
  }
}
```

`batchingStrategy` 对象并不复杂，可以理解它为一个“锁管理器”。

这里的“锁”，是指 React 全局唯一的 `isBatchingUpdates` 变量，`isBatchingUpdates` 的初始值是 `false`，意味着“当前并未进行任何批量更新操作”。每当 React 调用 `batchedUpdates` 方法执行更新动作的时候，会先把这个锁给“上锁”，也就是置为 `true`，表明“现在正处于批量更新过程中”。当锁被“锁上”的时候，任何需要更新的组件都只能暂时进入 `dirtyComponents` 里排队等候下一次的批量更新，而不能随意“插队”。此处体现的是“任务锁”的思想，是 React 面对大量状态仍然能实现有序分批处理的基础。

理解了批量更新整体的管理机制，还需要注意的是 `batchedUpdates` 中，有一个引人注意的调用：

```javascript
transaction.perform(callback, null, a, b, c, d, e)
```

这行代码引出了一个更加硬核的概念：React 中的 `Transaction` (事务) 机制。

## 理解 React 中的 Transaction (事务) 机制

`Transaction` 在 React 源码中的分布可以说是非常广泛。如果在 Debug React 项目的过程中，发现函数调用栈中出现了 `initialize`、`perform`、`close`、`closeAll` 或者 `notifyAll` 这样的方法名，那么很可能当前处于一个 `Transaction` 中。

`Transaction` 在 React 源码中表现为一个核心类，React 官方曾经这样描述它：`Transaction` 是创建一个黑盒，该黑盒可以封装任何的方法。因此那些需要在函数运行前，后运行的方法可以通过此方法进行封装 (即使函数运行中有异常抛出，这些固定的方法仍可运行)，实例化 `Transaction` 时只需要提供相关的方法即可。

这段话有点拗口，结合 React 源码中一阵针对 `Transaction` 的注释来进行理解：

```javascript
* <pre>
 *                       wrappers (injected at creation time)
 *                                      +        +
 *                                      |        |
 *                    +-----------------|--------|--------------+
 *                    |                 v        |              |
 *                    |      +---------------+   |              |
 *                    |   +--|    wrapper1   |---|----+         |
 *                    |   |  +---------------+   v    |         |
 *                    |   |          +-------------+  |         |
 *                    |   |     +----|   wrapper2  |--------+   |
 *                    |   |     |    +-------------+  |     |   |
 *                    |   |     |                     |     |   |
 *                    |   v     v                     v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | +---+ +---+   +---------+   +---+ +---+ |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
 * </pre>
 * 
```

说白了，`Transaction` 就像是一个“壳子”，它首先会将目标函数 `wrapper` (一组 `initialize` 及 `close` 方法称为一个 `wrapper`) 封装起来，同时需要使用 `Transaction` 类暴露的 `perform` 方法去执行它。如上方的注释所示，在 `anyMethod` 执行之前，`perform` 会先执行所有 `wrapper` 的 `initialize` 方法，执行完后，再执行所有 `wrapper` 的 `close` 方法。这就是 React 中的事务机制。

## “同步现象” 的本质

下面结合对事务机制的理解，继续看 `ReactDefaultBatchingStrategy` 这个对象。`ReactDefaultBatchingStrategy` 其实就是一个批量更新策略事务，他的 `wrapper` 有两个，分别是：`FLUSH_BATCHED_UPDATES` 和 `RESET_BATCHED_UPDATES`，如下代码所示：

```javascript
var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function () {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  }
};
var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates)
};
var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];
```

把这两个 `wrapper` 套进 `Transaction` 的执行机制中，不难得出一个这样的流程：

> 在 `callback` 执行完之后，`RESET_BATCHED_UPDATES` 将 `isBatchingUpdates` 设置为 `false`，`FLUSH_BATCHED_UPDATES` 执行 `flushBatchedUpdates`，然后里边会循环所有的 `dirtyComponent`，调用 `updateComponent` 来执行所有的生命周期方法 (`componentWillReceiveProps` -> `shouldComponentUpdate` -> `componentWillUpdate` -> `render` -> `componentDidUpdate`)，最后实现组件的更新。

讲解到了这里，对于 `isBatchingUpdates` 管控下的批量更新机制已经了然于胸了。但是 `setState` 为何会表现出同步的问题，似乎还是没有从当前展示的源码中得到根本的解答。这时因为 `batchedUpdates` 这个方法，不仅仅会在 `setState` 之后才被调用。若在 React 源码中全局搜索 `batchedUpdates`，会发现调用它的地方会有很多，但是和更新流有关的只有这两个地方：

```javascript
// ReactMount.js
_renderNewRootComponent: function( nextElement, container, shouldReuseMarkup, context ) {
  // 实例化组件
  var componentInstance = instantiateReactComponent(nextElement);
  // 初始渲染直接调用 batchedUpdates 进行批量渲染
  ReactUpdates.batchedUpdates(
    batchedMountComponentIntoNode,
    componentInstance,
    container,
    shouldReuseMarkup,
    context
  );
  ...
}
```

这段代码是在首次渲染组件时会执行的一个方法，可以看到其内部调用了一次 `batchedUpdates`，这是因为在组件渲染过程中，会按照顺序调用各个生命周期。开发者很有可能在声明周期函数中调用 `setState`。因此，需要通过开启 `batch` 来确保所有的更新都能够进入 `dirtyComponents` 中，进而确保初始渲染流程中所有的 `setState` 都是有效的。

下方的代码是 React 事件系统中的一部分。当在组件上绑定了事件之后，事件中也有可能会触发 `setState`。为了确保每一次 `setState` 都有效，React 同样会在此处手动开启批量更新

```javascript
// ReactEventListener.js
dispatchEvent: function (topLevelType, nativeEvent) {
  ...
  try {
    // 处理事件
    ReactUpdates.batchedUpdates(handleTopLevelImpl, bookKeeping);
  } finally {
    TopLevelCallbackBookKeeping.release(bookKeeping);
  }
}
```

说到这里，一切变得明朗了起来：`isBatchingUpdates` 这个变量，在 React 的声明周期函数以及合成事件执行之前，已经被 React 悄悄修改为了 `true`，这时所做的 `setState` 操作自然不会立即生效。当函数执行完毕之后，事务的 `close` 方法会再把 `isBatchingUpdates` 变量改为 `false`。

以开头示例中的 `increment` 方法为例，整个流程像是这样：

```javascript
// ReactEventListener.js
increment = () => {
  // 进来先锁上
  isBatchingUpdates = true
  console.log('increment setState前的count', this.state.count)
  this.setState({
    count: this.state.count + 1
  });
  console.log('increment setState后的count', this.state.count)
  // 执行完函数再放开
  isBatchingUpdates = false
}
```

很明显，在 `isBatchingUpdates` 的约束下，`setState` 只能是异步的。而当 `setTimeout` 从中作怪的时候，事情就会发生一点点变化。

```javascript
reduce = () => {
  // 进来先锁上
  isBatchingUpdates = true
  setTimeout(() => {
    console.log('reduce setState前的count', this.state.count)
    this.setState({
      count: this.state.count - 1
    });
    console.log('reduce setState后的count', this.state.count)
  },0);
  // 执行完函数再放开
  isBatchingUpdates = false
}
```

