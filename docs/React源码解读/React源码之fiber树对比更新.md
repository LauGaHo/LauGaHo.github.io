# React 源码之 fiber 树对比更新

## fiber 树更新入口

在 React 的协调算法执行过程中，只要涉及到了 `fiber` 树的变更 (首次渲染 `mount` 阶段或者对比更新 `update` 阶段)，都会调用 `scheduleUpdateOnFiber` 函数，所以基本上可以认为 `scheduleUpdateOnFiber` 函数是 `fiber` 树变更的入口函数。

### 更新方式

一般会有两种常见的方式会触发 `fiebr` 树的更新，如下所示：

- `Class` 组件中调用 `this.setState` 方法。

- `Function` 组件中调用 `Hook` 对象暴露的 `dispatchAction` (这里一般指代 `useState` 函数返回的 `Array` 对象中的第二个元素对应的函数)。

#### Class 组件调用 this.setState

React 中的 `Class` 组件对象的原型上拥有着 `setState` 方法，源码如下：

```typescript
Component.prototype.setState = function (partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, "setState");
};
```

在 `fiber` 树的初次构建过程中，`Class` 组件初始化完成之后，`this.updater` 对象对应的结构如下代码所示：

```typescript
const classComponentUpdater = {
  isMounted,
  enqueueSetState(inst, payload, callback) {
    // 1. 获取class实例对应的fiber节点
    const fiber = getInstance(inst);
    // 2. 创建update对象
    const eventTime = requestEventTime();
    const lane = requestUpdateLane(fiber); // 确定当前update对象的优先级
    const update = createUpdate(eventTime, lane);
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      update.callback = callback;
    }
    // 3. 将update对象添加到当前Fiber节点的updateQueue队列当中
    enqueueUpdate(fiber, update);
    // 4. 进入reconciler运作流程中的`输入`环节
    scheduleUpdateOnFiber(fiber, lane, eventTime); // 传入的lane是update优先级
  },
};
```

从上述代码中可以知道，所做的事情是：

1. `Class` 组件调用 `setState` 函数，其本质是创建一个 `Update` 实例对象，然后对其进行赋值操作.

2. 随后就会将当前的这个 `Update` 实例对象放到当前 `Class` 组件对应 `fiber` 节点中的 `updateQueue` 队列当中。

#### Function 组件调用 dispatchAction 方法

在 `Function` 组件中，调用 `useState` 函数会返回一个数组，其中数组中的第二项是一个名为 `dispatchAction` 的一个函数，这个是 React 暴露给开发者用于改变对应的 `state` 从而达到触发更新的一个目的。

`dispatchAction` 函数对应的源码如下所示：

```typescript
function dispatchAction<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  // 1. 创建update对象
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(fiber); // 确定当前update对象的优先级
  const update: Update<S, A> = {
    lane,
    action,
    eagerReducer: null,
    eagerState: null,
    next: (null: any),
  };
  // 2. 将update对象添加到当前Hook对象的updateQueue队列当中
  const pending = queue.pending;
  if (pending === null) {
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;
  // 3. 请求调度, 进入reconciler运作流程中的`输入`环节.
  scheduleUpdateOnFiber(fiber, lane, eventTime); // 传入的lane是update优先级
}
```

`dispatchAction` 函数所做的事情如下：

1. 创建一个 `Update` 实例对象，并对其进行赋值操作。

2. 随后将刚创建的 `Update` 实例对象放进 `Hook` 实例对象中的 `updateQueue` (注意：这里的 `updateQueue` 和 `fiber` 对象上的 `updateQueue` 是不一样的)。

## fiber 树的构造阶段

既然 `fiber` 树的对比更新最终入口还是 `scheduleUpdateOnFiber` 函数，那么将视线回到 `scheduleUpdateOnFiber` 函数上，如下所示：

```typescript
// ...省略部分代码
export function scheduleUpdateOnFiber(
  fiber: Fiber, // fiber表示被更新的节点
  lane: Lane, // lane表示update优先级
  eventTime: number
) {
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (lane === SyncLane) {
    if (
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      // 初次渲染
      performSyncWorkOnRoot(root);
    } else {
      // 对比更新
      ensureRootIsScheduled(root, eventTime);
    }
  }
  mostRecentlyUpdatedRoot = root;
}
```

上方代码中调用了 `markUpdateLaneFromFiberToRoot` 函数，这个函数只在对比更新阶段，即 `update` 阶段才会发挥其作用。其找出了 `fiber` 树中受到本次 `update` 影响的所有节点，并且设置这些节点的 `fiber.lanes` 或 `fiber.childLanes` 供 `fiber` 树构造阶段使用。

```typescript
function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber, // sourceFiber表示被更新的节点
  lane: Lane // lane表示update优先级
): FiberRoot | null {
  // 1. 将update优先级设置到sourceFiber.lanes
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    // 同时设置sourceFiber.alternate的优先级
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  // 2. 从sourceFiber开始, 向上遍历所有节点, 直到HostRoot. 设置沿途所有节点(包括alternate)的childLanes
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    }
    node = parent;
    parent = parent.return;
  }
  if (node.tag === HostRoot) {
    const root: FiberRoot = node.stateNode;
    return root;
  } else {
    return null;
  }
}
```

在上方 `scheduleUpdateOnFiber` 函数中，对比更新 `update` 阶段并没有直接调用 `performSyncWorkOnRoot`，而是通过调用 `ensureRootIsScheduled` 函数使用调度器的方式来实现更新，但是本次讲解是在同步模式下进行讲解，所以最终也还是会调用 `performSyncWorkOnRoot` 函数的。并且调用链如下所示：

`performSyncWorkOnRoot -> renderRootSync -> workLoopSync` 其实是跟初次构造一致。

其中 `renderRootSync` 函数如下所示：

```typescript
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
  // 如果fiberRoot变动, 或者update.lane变动, 都会刷新栈帧, 丢弃上一次渲染进度
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    // 刷新栈帧, legacy模式下都会进入
    prepareFreshStack(root, lanes);
  }
  do {
    try {
      workLoopSync();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);
  executionContext = prevExecutionContext;
  // 重置全局变量, 表明render结束
  workInProgressRoot = null;
  workInProgressRootRenderLanes = NoLanes;
  return workInProgressRootExitStatus;
}
```

进入循环构造 `workLoopSync` 前，会刷新栈帧 (调用 `prepareFreshStack`)。

注意：

- `fiberRoot.current` 指向和当前页面对应的 `fiber` 树，`workInProgress` 指向正在构造的 `fiber` 树。

- 刷新栈帧会调用 `createWorkInProgress()` 函数，使得 `workInProgress.flags` 和 `workInProgress.effects` 都被重置，且 `workInProgress.child = current.child`。所以在进入循环构造之前，`HostRootFiber` 和 `HostRootFiber.alternate` 共用一个 `child`。共用一个 `child` 的原因在于：在调用 `createWorkInProgress()` 函数调用的时候，会把 `current` 上的属性赋值给 `workInProgress`，其中 `child` 属性也不例外，所以这里就会有 `workInProgress.child = current.child`，所以 `workInProgress.child` 和 `current.child` 此时是指向同一个对象。

### 循环构造

回顾 `fiber` 树构造中的介绍，整个 `fiber` 树的构造过程是一个深度优先遍历的，其中有 2 个重要的变量，分别是 `workInProgress` 和 `current` 指针。

- `workInProgress` 和 `current` 都视为指针。

- `workInProgress` 指向当前正在构造的 `fiber` 节点。

- `current = workInProgress.alternate`，指向当前页面正在使用的 `fiber` 节点。

在深度优先遍历中，每个 `fiber` 节点都会经历 2 个阶段：

1. 探寻阶段 `beginWork`。

2. 回溯阶段 `completeWork`。

这两个阶段共同完成了每一个 `fiber` 节点的创建或更新，所有 `fiber` 节点则构成了 `fiber` 树。

```typescript
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// ... 省略部分无关代码
function performUnitOfWork(unitOfWork: Fiber): void {
  // unitOfWork就是被传入的workInProgress
  const current = unitOfWork.alternate;
  let next;
  next = beginWork(current, unitOfWork, subtreeRenderLanes);
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // 如果没有派生出新的节点, 则进入completeWork阶段, 传入的是当前unitOfWork
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

> 注意：在对比更新过程中 `current = unitOfWork.alternate`，这里主要是获取页面正在渲染的 `fiber` 树上的节点，后面的调用逻辑中会大量使用此处传入的 `current`。

### 探寻阶段 beginWork

```typescript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  const updateLanes = workInProgress.lanes;
  if (current !== null) {
    // 进入对比
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      // 当前渲染优先级renderLanes不包括fiber.lanes, 表明当前fiber节点无需更新
      didReceiveUpdate = false;
      switch (
        workInProgress.tag
        // switch 语句中包括 context相关逻辑, 本节暂不讨论(不影响分析fiber树构造)
      ) {
      }
      // 当前fiber节点无需更新, 调用bailoutOnAlreadyFinishedWork循环检测子节点是否需要更新
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }
  // 余下逻辑与初次创建共用
  // 1. 设置workInProgress优先级为NoLanes(最高优先级)
  workInProgress.lanes = NoLanes;
  // 2. 根据workInProgress节点的类型, 用不同的方法派生出子节点
  switch (
    workInProgress.tag // 只列出部分case
  ) {
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes
      );
    }
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderLanes);
    case HostText:
      return updateHostText(current, workInProgress);
    case Fragment:
      return updateFragment(current, workInProgress, renderLanes);
  }
}
```

#### `bailout` 逻辑

> 在源码中，`bailout` 用于判断子树节点是否完全复用，如果可以复用，则会略过 `fiber` 树构造。

和初次创建不一样，在对比更新过程中，如果是老节点，即 `current !== null`，需要进行对比，然后决定是否复用老节点及其子树，即走 `bailout` 逻辑。

1. `!includesSomeLane(renderLanes, updateLanes)` 这个判断分支，包含了 `渲染优先级` 和 `update 优先级` 的比较，如果当前节点无需更新，则会进入到 `bailout` 逻辑。

2. 如果当前节点无需要更新，即 `!includesSomeLane(renderLanes, updateLanes)`，则会调用 `bailoutOnAlreadyFinishedWork` 函数：

   - 如果同时满足 `!includesSomeLane(renderLanes, workInProgress.childLanes)`，表明该 `fiber` 节点及其子树都无需更新，可以直接进入回溯阶段 `completeUnitOfWork`。

   - 如果不满足 `!includesSomeLane(renderLanes, workInProgress.childLanes)`，意味着子节点需要进行更新，`clone` 并返回子节点。

```typescript
// 省略部分无关代码
function bailoutOnAlreadyFinishedWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
    // 渲染优先级不包括 workInProgress.childLanes, 表明子节点也无需更新. 返回null, 直接进入回溯阶段.
    return null;
  } else {
    // 本fiber虽然不用更新, 但是子节点需要更新. clone并返回子节点
    cloneChildFibers(current, workInProgress);
    return workInProgress.child;
  }
}
```

> 注意：`cloneChildFibers` 内部调用 `createWorkInProgress` 函数，在构造 `fiber` 节点时会优先复用 `workInProgress.alternate`，否则才会创建新的 `fiber` 对象。

#### `updateXXX` 函数

`updateXXX` 函数 (如：`updateHostRoot`, `updateClassComponent` 等) 的主干逻辑和初次构造过程完全一致，总的目的就是为了向下生成子节点，并在这个过程中调用 `reconcileChildren` 函数，只要 `fiber` 节点有副作用，就会把特殊操作设置到 `fiber.flags` (如：`节点 ref`, `class 组件的生命周期`, `function 组件的 hook`, `节点删除` 等)。

对比更新过程中的不同之处：

1. `bailoutOnAlreadyFinishedWork`

   - 对比更新时如果遇到当前节点无需更新 (如：`class` 类型的节点且 `shouldComponentUpdate` 返回 `false`)，会再次进入 `bailoutOnAlreadyFinishedWork` 逻辑。

2. `reconcileChildren`

   - 调和函数是 `updateXXX` 函数中的一项重要逻辑，其作用是向下生成子节点，并设置 `fiber.flags`。

   - 初次创建时 `fiber` 节点没有比较对象，所以在乡下生成子节点的时候没有任何多余的逻辑，直观创建就行。

   - 对比更新时需要把 `ReactElement` 对象和旧 `fiber` 对象进行比较，判断是否需要复用旧 `fiber` 对象。

这里需要注意 `recocileChildren` 函数的目的：

- 给新增，移动，删除节点设置 `fiber.flags`，用于后续的 `commit` 阶段。其中 (新增，移动：`Placement`，删除：`Deletion`)。

### 回溯阶段 completeWork

`completeUnitOfWork` 函数在初次创建和对比更新逻辑一致，都是处理 `beginWork` 阶段已经创建出来的 `fiber` 节点，最后创建 (更新) DOM 对象，并上移副作用队列。

这里需要注意 `completeWork` 函数中，`current !== null` 的情况：

```typescript
// ...省略无关代码
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  const newProps = workInProgress.pendingProps;
  switch (workInProgress.tag) {
    case HostComponent: {
      // 非文本节点
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
        // 处理改动
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          rootContainerInstance
        );
        if (current.ref !== workInProgress.ref) {
          markRef(workInProgress);
        }
      } else {
        // ...省略无关代码
      }
      return null;
    }
    case HostText: {
      // 文本节点
      const newText = newProps;
      if (current && workInProgress.stateNode != null) {
        const oldText = current.memoizedProps;
        // 处理改动
        updateHostText(current, workInProgress, oldText, newText);
      } else {
        // ...省略无关代码
      }
      return null;
    }
  }
}
```

对应 `updateHostComponent` 函数的逻辑

```typescript
updateHostComponent = function(
  current: Fiber,
  workInProgress: Fiber,
  type: Type,
  newProps: Props,
  rootContainerInstance: Container,
) {
  const oldProps = current.memoizedProps;
  if (oldProps === newProps) {
    return;
  }
  const instance: Instance = workInProgress.stateNode;
  const currentHostContext = getHostContext();
  const updatePayload = prepareUpdate(
    instance,
    type,
    oldProps,
    newProps,
    rootContainerInstance,
    currentHostContext,
  );
  workInProgress.updateQueue = (updatePayload: any);
  // 如果有属性变动, 设置fiber.flags |= Update, 等待`commit`阶段的处理
  if (updatePayload) {
    markUpdate(workInProgress);
  }
};
updateHostText = function(
  current: Fiber,
  workInProgress: Fiber,
  oldText: string,
  newText: string,
) {
  // 如果有属性变动, 设置fiber.flags |= Update, 等待`commit`阶段的处理
  if (oldText !== newText) {
    markUpdate(workInProgress);
  }
};
```

> 注意：在 `beginWork` 函数中有 `updateHostComponent` 一系列函数，在 `completeWork` 中也有一系列的 `updateHostComponent` 函数，这里不一样的是，`beginWork` 中的 `updateHostComponent` 是比较 `ReactElement` 和 `current` 树对应的 `fiber` 节点，并由此生成对应的 `wip` 的 `fiber` 节点，而 `completeWork` 中的 `updateHostComponent` 一系列函数是通过判断 `wip` 和 `current` 节点中的属性值是否相同来决定是否打上 `Update` 的 `Flag`，就形如：`className` 和 `style` 属性那样。
