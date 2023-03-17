# React 源码之 fiber 树初次创建

`fiber` 树构造的 2 种情况，如下：

- 初次创建：在 React 应用首次启动的时候，界面还没有开始渲染，此时不会进入到对比过程，相当于直接构造一棵全新的树。

- 对比更新：React 应用启动后，界面已经渲染，如果再次发生更新，创建新 `fiber` 之前需要和旧 `fiber` 之间进行对比，最后构造的 `fiber` 树可能是全新的，也可能是部分更新的。

本章节只讨论 **初次创建** 这种情况，为了节省篇幅，本文将在 `Legacy` 模式下 (同步模式下) 进行分析。

本文所用的示例代码如下所示：

```typescript
export function App() {
  return (
    <div className="app">
      <header>header</header>
      <Content />
    </div>
  );
}

export function Content() {
  return (
    <>
      <p>1</p>
      <p>2</p>
    </>
  );
}
```

## 启动阶段

在 React 应用进入 `react-reconcile` 包之前 (调用 `updateContainer` 之前)，内存状态如下图所示：

![react-state1](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/react-state1.png)

此时已经初始化了 `FiberRoot` 和 `HostRootFiber`。

然后就会进入 `react-reconciler` 包调用 `updateContainer` 函数：

```typescript
export function updateContainer(
  element: ReactElementType | null,
  root: FiberRootNode
) {
  // 获取当前 current 树的根节点
  const hostRootFiber = root.current;
  // 请求获取一个优先级
  const lane = requestUpdateLane();
  // 根据对应的优先级和 ReactElement 创建一个 update 对象，此处的 element 应该是 <App /> 这个 ReactElement
  const update = createUpdate<ReactElementType | null>(element, lane);
  // 将对应的 update 丢进 hostRootFiber 中的 updateQueue 中
  enqueueUpdate(
    hostRootFiber.updateQueue as UpdateQueue<ReactElementType | null>,
    update
  );
  // 进入调度环节
  scheduleUpdateOnFiber(hostRootFiber, lane);
  return element;
}
```

由于 `update` 对象的创建，此时的内存结构如下所示：

![react-state2](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/react-state2.png)

此时，最初的 `ReactElement` 类型的 `<App />` 对象被挂载到 `hostRootFiber.updateQueue.shared.pending.payload.element` 中，后续就会从这里开始 `fiber` 树的构造。这里可以说是梦的开始。

## 构造阶段

为了突出构造过程，排除其他无关对象的干扰，这里先把内存状态图做一个简化操作，先提取后续会使用到对象，如下图所示：

![react-create1](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/react-create1.png)

在 `scheduleUpdateOnFiber` 函数中：

```typescript
export function scheduleUpdateOnFiber(fiber: FiberNode, lane: Lane) {
  // 调度功能
  // ...
  // 从当前 fiber 节点开始向上寻找，直到找到根节点并返回根节点
  const root = markUpdateFromFiberToRoot(fiber);
  // 将当前新增任务的优先级合并到 root 节点上的 pendingLanes 上
  markRootUpdated(root, lane);
  // 进入 schedule 阶段
  ensureRootIsScheduled(root);
}
```

`scheduleUpdateOnFiber` 函数最后会调用 `ensureRootIsScheduled` 函数，该函数中会进入 `schedule` 阶段，`ensureRootIsScheduled` 函数中会触发调度器进行调度，最终执行的函数是 `performSyncWorkOnRoot`，而 `performSyncWorkOnRoot` 函数源码如下所示：

```typescript
function performSyncWorkOnRoot(root: FiberRootNode, lane: Lane) {
  const nextLane = getHighestPriorityLane(root.pendingLanes);

  if (nextLane !== SyncLane) {
    // 其他比 SyncLane 低的优先级
    // NoLane
    // 重新调一下 ensureRootIsScheduled 函数，如果是 NoLane 则直接返回，如果是比 SyncLane 低的优先级，就重新在 ensureRootIsScheduled 进行调度
    ensureRootIsScheduled(root);
    return;
  }

  if (__DEV__) {
    console.warn("render 阶段开始");
  }

  // 初始化
  prepareFreshStack(root, lane);

  // render 阶段
  do {
    try {
      workLoop();
      break;
    } catch (e) {
      if (__DEV__) {
        console.log(e);
      }
      workInProgress = null;
    }
  } while (true);

  // 获取 render 阶段形成的一棵完整的 fiberNode 树，并赋值给 root.finishedWork 属性中
  const finishedWork = root.current.alternate;
  root.finishedWork = finishedWork;
  // 记录本次消费的 Lane
  root.finishedLane = lane;
  // 更新结束之后，重新将 wipRootRenderLane 赋值为 NoLane
  wipRootRenderLane = NoLane;

  // commit 阶段
  // 根据 wip fiberNode 树中的 flags 提交给 render
  commitRoot(root);
}
```

在 `performSyncWorkOnRoot` 函数中，分了 3 步：

- `prepareFreshStack`：构建 `fiber` 树之前的准备工作，创建并返回 `workInProgress` 指针。

- `workLoop`：循环构造 `fiber` 树。

- `commitRoot`：`commit` 阶段，根据 `fiber` 树上的 `flags` 提交给 `render`。让 `render` 进行 DOM 的操作。

在经过了 `prepareFreshStack` 函数之后，就会创建了 `hostRootFiber.alternate`，并且为全局变量 `workInProgress` 赋值，如下图所示：

![react-create2](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/react-create2.png)

### 循环构造

逻辑来到了 `workLoop` 阶段，代码如下所示：

```typescript
function workLoop() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

`workLoop` 函数中在一个 `while` 循环中循环调用 `performUnitOfWork` 函数。`performUnitOfWork` 代码如下所示：

```typescript
function performUnitOfWork(fiber: FiberNode) {
  const next = beginWork(fiber, wipRootRenderLane);
  fiber.memoizedProps = fiber.pendingProps;

  if (next === null) {
    completeUnitOfWork(fiber);
  } else {
    workInProgress = next;
  }
}
```

`performUnitOfWork` 函数中可以看出整个 `fiber` 树的构造是一个深度优先遍历的过程，其中最重要的变量是 `workInProgress` 和 `current`。

- `workInProgress` 和 `current` 都视为指针。

- `workInProgress` 指向当前正在构造的 `fiber` 节点。

- `current = workInProgress.alternate` (即 `fiber.alternate`)，指向当前页面正在使用的 `fiber` 节点。

在深度优先遍历中，每个 `fiber` 节点都会经历 2 个阶段：

1. 探寻阶段 `beginWork`。

2. 回溯阶段 `completeWork`。

这两个阶段共同完成每一个 `fiber` 节点的创建，所有 `fiber` 节点则构成了 `fiber` 树。

### 探寻阶段 beginWork

`beginWork` 针对所有的 `fiber` 类型，其中的每一个 `case` 处理一种 `fiber` 类型 (也就是对应一个 `updateXXX` 函数，如：`updateHostRoot`, `updateFunctionComponent` 等)。`beginWork` 的逻辑如下代码所示：

```typescript
export const beginWork = (wip: FiberNode, renderLane: Lane) => {
  // 比较，并返回子 fiberNode
  switch (wip.tag) {
    case HostRoot:
      return updateHostRoot(wip, renderLane);

    case HostComponent:
      return updateHostComponent(wip);

    case HostText:
      return null;

    case FunctionComponent:
      return updateFunctionComponent(wip, renderLane);

    case Fragment:
      return updateFragment(wip);

    default:
      if (__DEV__) {
        console.warn("beginWork为实现的类型");
      }
      break;
  }
  return null;
};
```

`updateXXX` 函数 (如：`updateHostRoot`, `updateHostComponent`, `updateFunctionComponent` 等) 其主要逻辑可以概括为以下步骤：

1. 根据 `fiber.pendingProps`, `fiber.updateQueue` 等输入数据状态，计算 `fiber.memoizedState` 作为输出状态。

2. 获取下级的 `ReactElement` 对象：

   1. `ClassComponent` 类型：

      - 构建 `React.Component` 实例。

      - 把新实例挂载到 `fiber.stateNode` 上。

      - 执行 `render` 之前的生命周期函数。

      - 执行 `render` 方法，获取下级 `ReactElement` 实例对象。

      - 根据实际情况，设置 `fiber.flags`

   2. `FunctionComponent` 类型：

      - 执行对应的 `function`，获取下级的 `ReactElement` 实例对象。

      - 根据实际情况，设置 `fiber.flags`，因为有时候会执行 `useEffect` 这个 `Hook`，它需要标记 `flags`

   3. `HostComponent` 类型 (如：`div`, `span`, `button` 等)：

      - 将 `pendingProps.children` 作为下级的 `ReactElement` 实例对象。

      - 如果下级节点是文本节点，则设置下级节点为 `null`。准备进入 `completeUnitOfWork` 阶段。

      - 根据实际情况，设置 `fiber.flags`。

3. 根据 `ReactElement` 对象，调用 `reconcileChildren` 生成 `fiber` 子节点 (只生成次级子节点)。

   > 注意：这里所说的只生成次级子节点是重点，哪怕下边有多级子节点，也只生成次级子节点！

   - 根据实际情况，设置 `fiber.flags`。

不同的 `updateXXX` 函数处理的 `fiber` 节点类型不同，总体的目的都是为了向下生成子节点。在这个过程中把一些需要持久化的数据挂载到 `fiber` 节点上 (如 `fiber.stateNode`, `fiber.memoizedState` 等)；把 `fiber` 节点的特殊操作设置到 `fiber.flags` 中。

下边将列出 `updateHostRoot`, `updateHostComponent`, `updateFunctionComponent` 的代码。

`updateHostRoot` 代码：

```typescript
function updateHostRoot(wip: FiberNode, renderLane: Lane) {
  // 获取 wip 原本的 state
  const baseState = wip.memoizedState;
  // 获取 wip 当前的 updateQueue (里边装着最新的 Update 实例对象)
  const updateQueue = wip.updateQueue as UpdateQueue<Element>;
  // 获取 updateQueue 中最新的 Update 实例对象
  const pending = updateQueue.shared.pending;
  // 获取最新的 Update 对象后置空
  updateQueue.shared.pending = null;
  // 将原本的 state 和当前最新的 Update 对象进行比较，得到的结果是 ReactElementType 类型对象
  // 这里的 memoizedState 相当于 <App/> 的 ReactElementType 对象
  const { memoizedState } = processUpdateQueue(baseState, pending, renderLane);
  // 将最新的 memoizedState 赋值给 wip 的 memoizedState 属性中
  wip.memoizedState = memoizedState;

  const nextChildren = wip.memoizedState;
  // 将 wip 和 nextChildren 传给 reconcileChildren 函数用于生成子节点的 fiberNode
  reconcileChildren(wip, nextChildren);
  return wip.child;
}
```

`updateHostComponent` 代码如下：

```typescript
function updateHostComponent(wip: FiberNode) {
  // 获取 HostComponent 节点的 children 属性
  const nextProps = wip.pendingProps;
  const nextChildren = nextProps.children;
  // 传值给 reconcileChildren 用于生成子节点的 fiberNode
  reconcileChildren(wip, nextChildren);
  return wip.child;
}
```

`updateFunctionComponent` 代码如下：

```typescript
function updateFunctionComponent(wip: FiberNode, renderLane: Lane) {
  const nextChildren = renderWithHooks(wip, renderLane);
  reconcileChildren(wip, nextChildren);
  return wip.child;
}
```

上方所说的 3 个函数最终都是调用 `reconcileChildren` 这个函数，其中传入的参数类型为 `ReactElement` 或数组。

`reconcileChildren` 函数的代码如下所示：

```typescript
function reconcileChildFibers(
  returnFiber: FiberNode,
  currentFiber: FiberNode | null,
  newChild?: any
) {
  // 判断 Fragment
  // 这里的重点是 newChild.key 为空，则直接将 newChild.props.children 赋值给 newChild
  const isUnkeyedTopLevelFragment =
    typeof newChild === "object" &&
    newChild !== null &&
    newChild.type === REACT_FRAGMENT_TYPE &&
    newChild.key === null;

  // 如果根节点是 Fragment，那么直接将根节点中的 props.children 赋值给 newChild 变量，这样就可以直接走 reconcileChildrenArray
  if (isUnkeyedTopLevelFragment) {
    newChild = newChild.props.children;
  }
  // 判断当前 fiber 的类型
  if (typeof newChild === "object" && newChild !== null) {
    // 多节点的情况 ul>li*3
    if (Array.isArray(newChild)) {
      return reconcileChildrenArray(returnFiber, currentFiber, newChild);
    }

    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(returnFiber, currentFiber, newChild)
        );

      default:
        if (__DEV__) {
          console.warn("未实现的reconcile类型: ", newChild);
        }
        break;
    }
  }

  // HostText
  if (typeof newChild === "string" || typeof newChild === "number") {
    return placeSingleChild(
      reconcileSingleTextNode(returnFiber, currentFiber, newChild)
    );
  }

  if (currentFiber !== null) {
    // 兜底删除
    deleteRemainingChildren(returnFiber, currentFiber);
  }

  if (__DEV__) {
    console.warn("未实现的reconcile类型: ", newChild);
  }

  return null;
}
```

上方 `reconcileChildFibers` 函数中形参对应的解释如下所示：

- `returnFiber`：当前正在处理的 `ReactElement` 对应 `wip` 树上的父节点 `fiberNode` 实例对象。

- `currentFiber`：当前正在处理的 `ReactElement` 在 `current` 树上对应的 `fiberNode` 实例对象。

- `newChild`：当前正在被处理的 `ReactElement` 实例对象。其类型可以是 `ReactElement` 也可以是 `FiberNode` 类型，也可以是 `string` 或者 `number` 类型。

`reconcileChildFibers` 函数实际上就是根据 `newChild` 的类型来判断使用哪个函数来为其来生成或复用 `FiberNode` 实例对象。例如：

- `newChild` 为 `Array` 类型，使用 `reconcileChildrenArray` 方法。

- `newChild` 为 `ReactElement` 类型，使用 `reconcileSingleElement` 方法。

- `newChild` 为 `string` 或者 `number` 类型，使用 `reconcileSingleTextNode` 方法。

其目的都是根据对应的 `ReactElement` 类型，来生成或者复用原有的 `FiberNode` 实例对象。

### 回溯阶段 completeWork

`completeWork` 函数是在 `completeUnitOfWork` 函数中被调用的，而 `completeUnitOfWork` 函数的源码如下：

```typescript
function completeUnitOfWork(fiber: FiberNode) {
  let node: FiberNode | null = fiber;

  do {
    completeWork(node);
    const sibling = node.sidling;

    // 当前 sibling 对应的 fiber 节点不为空，说明 fiber 形参的旁边仍有 fiber 节点
    if (sibling !== null) {
      // 将全局变量 workInProgress 赋值为 sibling，用于进入下一个 beginWork 循环构造
      workInProgress = sibling;
      return;
    }
    // 反之则一层一层往上赋值，对其父节点及其祖先节点都进行 completeWork 函数的调用操作
    node = node.return;
    // 更新全局变量 workInProgress
    workInProgress = node;
  } while (node !== null);
}
```

`completeUnitOfWork` 函数在 `mount` 阶段和 `update` 阶段逻辑基本一致，都是用来处理 `beginWork` 阶段创建的 `fiber` 节点，并且，如果当前是 `mount` 阶段的话，会通过调用 `react-dom` 相关的 `api` 创建一棵离屏的 DOM 树，属于是归的阶段。

而 `completeWork` 函数当属是重中之重，其中 `completeWork` 源码如下所示：

```typescript
export const completeWork = (wip: FiberNode) => {
  // 递归中的归

  const newProps = wip.pendingProps;
  const current = wip.alternate;

  switch (wip.tag) {
    case HostComponent:
      if (current !== null && wip.stateNode) {
        // TODO update
        // 1. props 是否变化
        // 2. 变化了，需要打上一个 Update 的 Flags
        // 判断 className, style 是否变化
        markUpdate(wip);
      } else {
        // mount
        // 1. 构建 DOM
        // 2. 将 DOM 插入到 DOM 树
        // const instance = createInstance(wip.type, newProps);
        const instance = createInstance(wip.type, newProps);
        appendAllChildren(instance, wip);
        wip.stateNode = instance;
      }
      bubbleProperties(wip);
      return null;

    case HostText:
      if (current !== null && wip.stateNode) {
        // update
        // 获取旧的 text
        const oldText = current.memoizedProps?.content;
        // 获取新的 text
        const newText = newProps.content;
        // 比对 oldText 和 newText 是否相同，不同则标记 Update 的 flags
        if (oldText !== newText) {
          markUpdate(wip);
        }
      } else {
        // mount
        // 1. 构建 DOM
        // 2. 将 DOM 插入到 DOM 树
        const instance = createTextInstance(newProps.content);
        wip.stateNode = instance;
      }
      bubbleProperties(wip);
      return null;

    case HostRoot:
    case FunctionComponent:
    case Fragment:
      bubbleProperties(wip);
      return null;

    default:
      if (__DEV__) {
        console.warn("未处理的completeWork情况: ", wip);
      }
      break;
  }
};
```

可以看到，`completeWork` 代码中针对 `mount` 和 `update` 阶段，有不同的处理逻辑：

- `mount` 阶段：

  1. 根据 `wip` 来构建对应的 DOM 结构。

  2. 将当前 `wip` 中的子元素的 `stateNode` 挂载到当前 `wip` 生成的 DOM 之下。

  3. 将 `wip` 子节点的 `flags` 和 `subtreeFlags` 合并到当前 `wip` 中的 `subtreeFlags` 属性上 (注意这里指代的子节点是严格的下一层的子节点，并不会触及到孙节点，这里需要尤为重要)。

- `update` 阶段：

  1. 根据 `wip` 的 `pendingProps` 和 `wip` 对应的 `current` 节点的 `memoizedProps` 进行比对，得出变更后的属性。

  2. 如果上一步中的 `pendingProps` 和 `memoizedProps` 比对过程中发生了变化，则在当前 `wip` 中的 `flags` 上标记上 `Update` 记号。

  3. 将 `wip` 子节点的 `flags` 和 `subtreeFlags` 合并到当前 `wip` 中的 `subtreeFlags` 属性上 (注意这里指代的子节点是严格的下一层的子节点，并不会触及到孙节点，这里需要尤为重要)。
