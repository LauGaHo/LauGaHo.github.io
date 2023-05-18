# Hook 原理

对于 `FunctionComponent` 来讲，状态的更改是通过内部的 `Hook` 来实现的。

`Hook` 的引入是为了更好的在组件之间进行逻辑复用。`Hook` 是一个特殊的函数，他能够在指定的时刻钩入 `React` 的特性。

## Hook 和 Fiber

其实 `Hook` 的使用最终是为了控制 `fiber` 节点的**状态**和**副作用**。

## Hook 数据结构

`Hook` 的数据结构如下：

```typescript
type Update<S, A> = {
  lane: Lane,
  action: A,
  eagerReducer: ((S, A) => S) | null,
  eagerState: S | null,
  next: Update<S, A>,
  priority?: ReactPriorityLevel,
};

type UpdateQueue<S, A> = {
  pending: Update<S, A> | null,
  dispatch: (A => mixed) | null,
  lastRenderedReducer: ((S, A) => S) | null,
  lastRenderedState: S | null,
};

export type Hook = {
  memoizedState: any, // 当前状态
  baseState: any, // 基状态
  baseQueue: Update<any, any> | null, // 基队列
  queue: UpdateQueue<any, any> | null, // 更新队列
  next: Hook | null, // next指针
};
```

从定义来看，`Hook` 对象共有 5 个属性，如下所示：

1. `hook.memoizedState`: `Hook` 保持在内存中的局部状态。

2. `hook.baseState`: `hook.baseQueue` 中所有 `update` 对象合并之后的状态。

3. `hook.baseQueue`: 存储 `update` 对象的环形链表，只包括高于本次渲染优先级的 `update` 对象。

4. `hook.queue`: 存储 `update` 对象的环形链表，包括所有优先级的 `update` 对象。

5. `hook.next`: `next` 指针，指向链表中的下一个 `hook`。

所以 `Hook` 是一个链表，单个 `Hook` 拥有自己的状态 `hook.memoizedState` 和自己的更新队列 `hook.queue`。

![HookQueue](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/HookQueue.png)

注意：其中 `hook.queue` 和 `fiber.updateQueue` 虽然都是 `update` 环形链表，尽管 `update` 对象的数据结构和处理方式都是高度相似的，但是这两个队列中的 `update` 对象是完全独立的。`hook.queue` 只作用于 `hook` 对象的状态维护，切忌和 `fiber.updateQueue` 混淆。

## Hook 分类

在 React 的官网中定义了 14 种 Hook，如下所示：

```typescript
export type HookType =
  | "useState"
  | "useReducer"
  | "useContext"
  | "useRef"
  | "useEffect"
  | "useLayoutEffect"
  | "useCallback"
  | "useMemo"
  | "useImperativeHandle"
  | "useDebugValue"
  | "useDeferredValue"
  | "useTransition"
  | "useMutableSource"
  | "useOpaqueIdentifier";
```

React 官方将 `Hook` 分为了两类，一类是状态 `Hook` (`State Hook`)，另一类是副作用 `Hook` (`Effect Hook`)。

### 状态 Hook

客观来讲，`useState` 和 `useReducer` 可以在 `FunctionComponent` 中添加内部的 `state`，并且 `useState` 实际上是 `useReducer` 的简单封装，是一个最特殊的 `useReducer`，所以将 `useState` 和 `useReducer` 称为状态 `Hook`。

通俗来讲，只要能实现数据持久化，且没有副作用的 `Hook`，均可以看作为状态 `Hook`，所以状态 `Hook` 还包括 `useContext`, `useRef`, `useCallback`, `useMemo` 等。这类 `Hook` 内部没有使用 `useState` 或 `useReducer`，但是他们也能够实现多次 `render` 时，保持其初始值不变 (即数据持久化) 且没有任何副作用。

### 副作用 Hook

状态 `Hook` 实现了状态持久化，等同于 `class` 组件维护 `fiber.memoizedState`。而副作用 `Hook` 则会修改 `fiber.flags`。副作用 `Hook` 还提供了副作用回调，类似于 `class` 组件的生命周期回调，比如：

```typescript
// 在 fiber 树构造完成之后，commitRoot 阶段会处理这些副作用回调
useEffect(() => {
  console.log("这是一个副作用回调函数");
}, []);
```

在 React 内部，`useEffect` 就是最标准的副作用 `Hook`。其他比如 `useLayoutEffect` 以及自定义 `Hook`，如果要实现副作用，必须直接或间接的调用 `useEffect`。

### 组合 Hook

大多数的 `Hook` 都是由上面所说的两种 `Hook` 组合而成的，同时拥有这两种 `Hook` 的特性。

- 在 React 内部有 `useDebugValue`, `useTransition`, `useMutableSource`, `useOpaqueIdentifier`。

- 平时开发中，自定义 `Hook` 大部分都是组合 `Hook`。

举例如下：

```typescript
import { useState, useEffect } from "react";

function useFriendStatus(friendId: string) {
  // 调用 useState，创建一个状态 Hook
  const [isOnline, setIsOnline] = useState(null);

  // 调用 useEffect，创建一个副作用 Hook
  useEffect(() => {
    function handleStatusChange(status) {
      setIsOnline(status.isOnline);
    }
    ChatAPI.subscribeToFriendStatus(friendId, handleStatusChange);
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(friendId, handleStatusChange);
    };
  });
  return isOnline;
}
```

## FunctionComponent 执行之前

调用 `FunctionComponent` 之前，React 还需要进行一些处理。

### 处理函数

在 `fiber` 树构造的过程当中，不同的 `fiber` 类型，只需要调用不同的 `updateXXX` 函数进行相应的处理，并返回 `fiber` 子节点，形如 `FunctionComponent` 在这里就会调用 `updateFunctionComponent` 函数。

`updateFunctionComponent` 函数源码如下所示：

```typescript
// 只保留FunctionComponent相关:
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  const updateLanes = workInProgress.lanes;
  switch (workInProgress.tag) {
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes
      );
    }
  }
}

function updateFunctionComponent(
  current,
  workInProgress,
  Component,
  nextProps: any,
  renderLanes
) {
  // ...省略无关代码
  let context;
  let nextChildren;
  prepareToReadContext(workInProgress, renderLanes);

  // 进入Hooks相关逻辑, 最后返回下级ReactElement对象
  nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,
    nextProps,
    context,
    renderLanes
  );
  // 进入reconcile函数, 生成下级fiber节点
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  // 返回下级fiber节点
  return workInProgress.child;
}
```

在 `updateFunctionComponent` 函数中调用了 `renderWithHooks`，在 `renderWithHooks` 函数中执行 `FunctionComponent` 对应的函数，生成了 `FunctionComponent` 对应子节点的 `ReactElement` 并赋值为 `nextChildren`。然后将 `nextChildren` 传入 `reconcileChildren` 函数中，根据 `ReactElement` 类型的子节点生成 `FiberNode` 类型的子节点。至此 `updateFunctionComponent` 完成。

### 全局变量

在分析 `renderWithHooks` 函数前，需要对定义的全局变量有一个理解：

```typescript
// 渲染优先级
let renderLanes: Lanes = NoLanes

// 当前正在构造的 fiberNode，类似 workInProgress
let currentlyRenderingFiber: Fiber = (null: any)

// Hook 链表存储在 fiber.memoizedState，这里的 currentHook 表示当前正在处理的 Hook 实例对象
let currentHook: Hook | null = null // currentHook = fiber(current).memoizedState

let workInProgressHook: Hook | null = null  // workInProgressHook = fiber(workInProgress).memoizedState

// 在 function 的执行过程中，是否再次发起了更新，只有 function 被完全执行完之后才会重置
// 当 render 异常时，通过该变量可以决定是否清除 render 过程中的更新
let didScheduleRenderPhaseUpdate: boolean = false

// 在本次 function 执行过程中，是否再次发起了更新，每一次调用 function 都会被重置
let didScheduleRenderPhaseUpdateDuringThisPass: boolean = false

// 在本次 function 执行过程中，重新发起更新的最大次数
const RE_RENDER_LIMIT = 25
```

### renderWithHooks 函数

`renderWithHooks` 函数简化版源码如下所示：

```typescript
// ...省略无关代码
export function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes,
): any {
  // --------------- 1. 设置全局变量 -------------------
  renderLanes = nextRenderLanes; // 当前渲染优先级
  currentlyRenderingFiber = workInProgress; // 当前fiber节点, 也就是function组件对应的fiber节点

  // 清除当前fiber的遗留状态
  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  workInProgress.lanes = NoLanes;

  // --------------- 2. 调用function,生成子级ReactElement对象 -------------------
  // 指定dispatcher, 区分mount和update
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount
      : HooksDispatcherOnUpdate;
  // 执行function函数, 其中进行分析Hooks的使用
  let children = Component(props, secondArg);

  // --------------- 3. 重置全局变量,并返回 -------------------
  // 执行function之后, 还原被修改的全局变量, 不影响下一次调用
  renderLanes = NoLanes;
  currentlyRenderingFiber = (null: any);

  currentHook = null;
  workInProgressHook = null;
  didScheduleRenderPhaseUpdate = false;

  return children;
}
```

- 调用 `function` 前：设置全局变量，标记渲染优先级和当前 `fiber`，清除当前 `fiber` 的遗留状态。

- 调用 `function`：构造出 `Hooks` 链表，最后生成子级 `ReactElement` 对象 (`children`)。

- 调用 `function` 后：重置全局变量，返回 `children`。

  - 为了保证不同的 `function` 节点在调用时 `renderWithHooks` 互不影响，所以退出时重置全局变量。

## 调用 function

### Hooks 构造

在 `function` 中，若使用了 `Hook` API，就会创建一个与之对应的 `Hook` 对象。

有一个 `App` 的 `FunctionComponent` 源码如下所示：

```typescript
import { useState, useEffect } from "react";

export default function App() {
  // 1.
  const [a, setA] = useState(1);
  // 2.
  useEffect(() => {
    console.log(`effect 1 created`);
  });
  // 3.
  const [b] = useState(2);
  // 4.
  useEffect(() => {
    console.log(`effect 2 created`);
  });

  return (
    <>
      <button onClick={() => setA(a + 1)}>{a}</button>
      <button>{b}</button>
    </>
  );
}
```

初次渲染时，逻辑执行的链路如下：`performUnitOfWork -> beginWork -> updateFunctionComponent -> renderWithHooks`。

当执行 `renderWithHooks` 时，开始调用 `function`。在 `function` 内部中，一共使用了 4 次 `Hook` API，按照顺序分别是 `useState`, `useEffect`, `useState`, `useEffect`。

而 `useState` 和 `useEffect` 在 `fiber` 初次构造时分别对应源码执行的方法是 `mountState` 和 `mountEffect` 还有 `mountEffectImpl` 方法。源码如下所示：

```typescript
function mountState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountWorkInProgressHook();
  // ......
  return [hook.memoizedState, dispatch];
}

function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = mountWorkInProgressHook();
  // ......
}
```

无论是 `useState` 还是 `useEffect`，其内部的是实现都是通过新建一个 `Hook` 实例对象。

### 链表存储

`mountWorkInProgressHook` 函数的逻辑如下：

```typescript
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // 链表中首个hook
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 将hook添加到链表末尾
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

上方函数的主要逻辑是：创建 `Hook` 实例对象，一个 `FunctionComponent` 中的多个 `Hook` 实例对象之间通过链表的形式连在一起保存。

在上方事例中，就会创建 4 个 `Hook` 实例对象。结构如下所示：

![HookDataStruct](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/HookDataStruct.png)

### 顺序克隆

`fiber` 树构造 (对比更新) 阶段，执行 `updateFunctionComponent -> renderWithHooks` 时再次调用 `function`。

> 注意：在 `renderWithHooks` 函数中已经设置了 `workInProgress.memoizedState = null`，等待调用 `function` 时重新设置。

紧接着会调用 `function`，同样依次调用 `useState`, `useEffect`, `useState`, `useEffect`，而 `useState` 和 `useEffect` 在 `fiber` 对比更新时分别对应着 `updateState -> updateReducer` 和 `updateEffect -> updateEffectImpl`。

```typescript
// ----- 状态Hook --------
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  // ...省略部分本节不讨论
}

// ----- 副作用Hook --------
function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = updateWorkInProgressHook();
  // ...省略部分本节不讨论
}
```

无论是 `updateReducer` 还是 `updateEffectImpl`，内部都是通过调用 `updateWorkInProgressHook` 函数来获取一个 `hook` 实例对象。

```typescript
function updateWorkInProgressHook(): Hook {
  // 1. 移动currentHook指针
  let nextCurrentHook: null | Hook;
  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;
    if (current !== null) {
      nextCurrentHook = current.memoizedState;
    } else {
      nextCurrentHook = null;
    }
  } else {
    nextCurrentHook = currentHook.next;
  }

  // 2. 移动workInProgressHook指针
  let nextWorkInProgressHook: null | Hook;
  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // 渲染时更新: 本节不讨论
  } else {
    currentHook = nextCurrentHook;
    // 3. 克隆currentHook作为新的workInProgressHook.
    // 随后逻辑与mountWorkInProgressHook一致
    const newHook: Hook = {
      memoizedState: currentHook.memoizedState,

      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,

      next: null, // 注意next指针是null
    };
    if (workInProgressHook === null) {
      currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
    } else {
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }
  return workInProgressHook;
}
```

`updateWorkInProgressHook` 函数逻辑很简单：就是为了让 `currentHook` 和 `workInProgressHook` 两个指针同时向后移动。

1. 由于 `renderWithHooks` 函数设置了 `workInProgress.memoizedState = null`，所以 `workInProgressHook` 初始值必然为 `null`，所以此时就只能从 `currentHook` 克隆得来。

2. 而从 `currentHook` 克隆得来的 `newHook.next = null`，所以导致 `workInProgressHook` 链表需要完全重建。

根据上方的代码可以知道：

1. 因为双缓存技术，将 `current.memoizedState` 按照顺序克隆到了 `workInProgress.memoizedState` 中。

2. `Hook` 经过了一次克隆，内部的属性 (`hook.memoizedState` 等) 都没发生变动，所以其状态并不会丢失。
