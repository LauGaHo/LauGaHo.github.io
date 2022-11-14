## Fiber架构之首次创建Fiber树

本文将突出解析 `fiber 树` 构造的过程，并且是在 `Legacy` 模式下进行分析，因为针对 `fiber 树` 的构造原理，`Concurrent` 模式和 `Legacy` 模式是没有区别的。

本文讲解将使用如下的示例代码如下：

```javascript
class App extends React.Component {
  componentDidMount() {
    console.log(`App Mount`);
    console.log(`App 组件对应的fiber节点: `, this._reactInternals);
  }
  render() {
    return (
      <div className="app">
        <header>header</header>
        <Content />
      </div>
    );
  }
}

class Content extends React.Component {
  componentDidMount() {
    console.log(`Content Mount`);
    console.log(`Content 组件对应的fiber节点: `, this._reactInternals);
  }
  render() {
    return (
      <React.Fragment>
        <p>1</p>
        <p>2</p>
      </React.Fragment>
    );
  }
}
export default App;
```

## 构造阶段

为了突出构造过程，排除干扰，先把内存状态图中的 `FiberRoot` 和 `HostRootFiber` 单独提取出来 (后文在此基础上添加)：

![fiber构造](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/fiber构造1.png)

在 `scheduleUpdateOnFiber` 函数中：

```javascript
// ...省略部分代码
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
) {
  // 标记优先级
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (lane === SyncLane) {
    if (
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // 首次渲染, 直接进行`fiber构造`
      performSyncWorkOnRoot(root);
    }
    // ...
  }
}
```

可以得知，在 `Legacy` 模式下且首次渲染时，有两个函数 `markUpdateLaneFromFiberToRoot` 和 `performSyncWorkRoot`。

其中 `markUpdateLaneFromFiberToRoot(fiber, lane)` 函数在 `fiber树对比更新` 中才会发挥作用，因为初次创建时并没有和当前页面所对应的 `fiber树`，所以核心代码并没有执行，最后直接返回了 `FiberRoot` 对象。
