# Hook 原理—状态 Hook

对 `Hook` 原理做一个简单的复习：

1. `FunctionComponent` 类型的 `fiber` 节点，其处理函数是 `updateFunctionComponent`，其中 `renderWithHooks` 中调用对应的 `function`。

2. 在 `function` 中通过 `Hook` API，形如：`useState`， `useEffect` 来创建对象。

   - 状态 `Hook` 实现了状态的持久化 (就好像 `class` 组件维护 `fiber.memoizedState`)。

   - 副作用 `Hook` 则实现了维护 `fiber.flags`，并提供副作用回调 (类似于 `class` 组件的声明周期回调)。

3. 多个 `Hook` 实例对象构成一个链表结构，并将其挂载到 `fiber.memoizedState` 上。

4. `fiber` 树更新阶段，把 `current.memoizedState` 链表上的所有 `Hook` 按照顺序克隆到 `workInProgress.memoizedState` 上，实现数据的持久化。

## 创建 Hook

在 `fiber` 初次构造阶段，`useState` 对应源码为 `mountState`，而 `useReducer` 对应源码为 `mountReducer`。

`mountState` 源码如下所示：

```typescript
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  // 1. 创建hook
  const hook = mountWorkInProgressHook();
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  // 2. 初始化hook的属性
  // 2.1 设置 hook.memoizedState/hook.baseState
  // 2.2 设置 hook.queue
  hook.memoizedState = hook.baseState = initialState;
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    // queue.lastRenderedReducer是内置函数
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  });
  // 2.3 设置 hook.dispatch
  const dispatch: Dispatch<
    BasicStateAction<S>,
  > = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));

  // 3. 返回[当前状态, dispatch函数]
  return [hook.memoizedState, dispatch];
}
```

而 `mountReducer` 源码如下所示：

```typescript
function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 1. 创建hook
  const hook = mountWorkInProgressHook();
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);
  } else {
    initialState = ((initialArg: any): S);
  }
  // 2. 初始化hook的属性
  // 2.1 设置 hook.memoizedState/hook.baseState
  hook.memoizedState = hook.baseState = initialState;
  // 2.2 设置 hook.queue
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    // queue.lastRenderedReducer是由外传入
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  });
  // 2.3 设置 hook.dispatch
  const dispatch: Dispatch<A> = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));

  // 3. 返回[当前状态, dispatch函数]
  return [hook.memoizedState, dispatch];
}
```

`mountState` 和 `mountReducer` 主要逻辑：负责创建 `Hook` 实例对象，初始化 `Hook` 实例对象的属性，最后返回 `[当前状态, dispatch 函数]`

不一样的是 `hook.queue.lastRenderedReducer`。

- `mountState` 使用的是内置的 `basicStateReducer`，代码如下所示：

  ```typescript
  function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
    return typeof action === "function" ? action(state) : action;
  }
  ```

- `mountReducer` 使用的是外部传入的自定义 `reducer`。

从上可以知道 `mountState` 是 `mountReducer` 的一种特殊情况，即 `useState` 是 `useReducer` 的一种特殊情况，也是最简单的一种情况。

尝试用 `useReducer` 来实现一下 `useState`，代码如下所示：

```typescript
const [state, dispatch] = useState({ count: 0 });

// 等价于
const [state, dispatch] = useReducer(
  function basicStateReducer(state, action) {
    return typeof action === "function" ? action(state) : action;
  },
  { count: 0 }
);

// 当需要更新state时, 有2种方式
dispatch({ count: 1 }); // 1.直接设置
dispatch((state) => ({ count: state.count + 1 })); // 2.通过回调函数设置
```

看一个 `useReducer` 的官方例子，如下所示：

```typescript
const [state, dispatch] = useReducer(
  function reducer(state, action) {
    switch (action.type) {
      case "increment":
        return { count: state.count + 1 };
      case "decrement":
        return { count: state.count - 1 };
      default:
        throw new Error();
    }
  },
  { count: 0 }
);

// 当需要更新state时, 只有1种方式
dispatch({ type: "decrement" });
```

所以 `useState` 就是对 `useReducer` 的基本封装，内置了一个 `reducer`。创建 `Hook` 实例对象之后返回值 `[hook.memoizedState, dispatch]` 中的 `dispatch` 实际上是会调用 `reducer` 函数。

## 状态初始化

在 `useState(initialState)` 函数的内部，设置了 `hook.memoizedState = hook.baseState = initialState`，初始状态被同时保存到 `hook.baseState`, `hook.memoizedState` 中。

1. `hook.memoizedState`: 当前状态。

2. `hook.baseState`: 基础状态，作为合并 `hook.baseQueue` 的初始值。

最后返回 `[hook.memoizedState, dispatch]`，所以在 `function` 中使用的是 `hook.memoizedState`。

## 状态更新

有如下代码：

```typescript
import { useState } from "react";

export default function App() {
  const [count, setCount] = useState(0);
  return (
    <button
      onClick={() => {
        setCount(1);
        setCount(2);
        setCount(3);
      }}
    >
      {count}
    </button>
  );
}
```

初次渲染的时候，`count = 0`，此时 `hook` 对象的内存状态如下图所示：

![useStateMount](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/useStateMount.png)

点击 `button`，通过 `setCount` 函数进行更新，`setCount` 函数实际上就是 `dispatchAction` 的源码，如下所示：

```typescript
function dispatchAction<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  // 1. 创建update对象
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(fiber); // Legacy模式返回SyncLane
  const update: Update<S, A> = {
    lane,
    action,
    eagerReducer: null,
    eagerState: null,
    next: (null: any),
  };

  // 2. 将update对象添加到hook.queue.pending队列
  const pending = queue.pending;
  if (pending === null) {
    // 首个update, 创建一个环形链表
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;

  const alternate = fiber.alternate;
  if (
    fiber === currentlyRenderingFiber ||
    (alternate !== null && alternate === currentlyRenderingFiber)
  ) {
    // 渲染时更新, 做好全局标记
    didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
  } else {
    // ...省略性能优化部分, 下文介绍

    // 3. 发起调度更新, 进入`reconciler 运作流程`中的输入阶段.
    scheduleUpdateOnFiber(fiber, lane, eventTime);
  }
}
```

上方源码的逻辑如下解释：

1. 创建 `update` 实例对象，其中 `update.lane` 代表着优先级。

2. 将 `update` 实例对象添加到 `hook.queue.pending` 环形链表中。

   - 环形链表：为了方便添加新元素和快速拿到队首元素 (都是 `O(1)`)，所以 `pending` 指针指向了链表中最后一个元素。

3. 发起调度更新：调用 `scheduleUpdateOnFiber` 函数，进入 `reconciler` 运行流程中的输入阶段。

从调用 `scheduleUpdateOnFiber` 开始，就进入了 `react-reconciler` 包。

> 注意：上方示例中虽然同时执行了 3 次 `setCount`，会请求 3 次调度，但是由于调度中心的节流优化，最后只会执行一次渲染。

在 `fiber` 树的对比更新过程中，再次调用 `function`，这时 `useState` 对应的函数是 `updateState`。

```typescript
function updateState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
```

在执行 `updateReducer` 函数之前，`hook` 相关的内存结构如下图所示：

![HookBeforeUpdateState](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/HookBeforeUpdateState.png)

而 `updateReducer` 的源码如下所示：

```typescript
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 1. 获取workInProgressHook对象
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  queue.lastRenderedReducer = reducer;
  const current: Hook = (currentHook: any);
  let baseQueue = current.baseQueue;

  // 2. 链表拼接: 将 hook.queue.pending 拼接到 current.baseQueue
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    if (baseQueue !== null) {
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }
  // 3. 状态计算
  if (baseQueue !== null) {
    const first = baseQueue.next;
    let newState = current.baseState;

    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;

    do {
      const updateLane = update.lane;
      // 3.1 优先级提取update
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 优先级不够: 加入到baseQueue中, 等待下一次render
        const clone: Update<S, A> = {
          lane: updateLane,
          action: update.action,
          eagerReducer: update.eagerReducer,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane,
        );
        markSkippedUpdateLanes(updateLane);
      } else {
        // 优先级足够: 状态合并
        if (newBaseQueueLast !== null) {
          // 更新baseQueue
          const clone: Update<S, A> = {
            lane: NoLane,
            action: update.action,
            eagerReducer: update.eagerReducer,
            eagerState: update.eagerState,
            next: (null: any),
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        if (update.eagerReducer === reducer) {
          // 性能优化: 如果存在 update.eagerReducer, 直接使用update.eagerState.避免重复调用reducer
          newState = ((update.eagerState: any): S);
        } else {
          const action = update.action;
          // 调用reducer获取最新状态
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    // 3.2. 更新属性
    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }
    // 把计算之后的结果更新到workInProgressHook上
    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;
    queue.lastRenderedState = newState;
  }

  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```

`updateReducer` 函数逻辑如下所示：

1. 调用 `updateWorkInProgressHook` 获取 `workInProgressHook` 对象。

2. 链表拼接：将 `hook.queue.pending` 拼接到 `current.baseQueue` 中。

   ![beforeComputeState](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/beforeComputeState.png)

3. 状态计算

   1. `update` 对象优先级不够：加入到 `baseQueue` 中，等待下一次 `render`。

   2. `update` 对象优先级足够：状态合并。

   3. 更新属性。

      ![afterComputeState](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/afterComputeState.png)

### 性能优化

`dispatchAction` 函数中，在调用 `scheduleUpdateOnFiber` 之前，针对 `update` 对象做了性能优化。

1. `queue.pending` 中只包含当前 `update` 实例对象时，即当前 `update` 是 `queue.pending` 中的第一个 `update` 对象。

2. 直接调用 `queue.lastRenderedReducer`，计算 `update` 之后的 `state`，记为 `eagerState`。

3. 如果 `eagerState` 和 `currentState` 相同，则直接退出，不用发起调度更新。

4. 已经挂载到 `queue.pending` 上的 `update` 会在下一次 `render` 时再次合并。

```typescript
function dispatchAction<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  // ...省略无关代码 ...只保留性能优化部分代码:

  // 下面这个if判断, 能保证当前创建的update, 是`queue.pending`中第一个`update`. 为什么? 发起更新之后fiber.lanes会被改动(可以回顾`fiber 树构造(对比更新)`章节), 如果`fiber.lanes && alternate.lanes`没有被改动, 自然就是首个update
  if (
    fiber.lanes === NoLanes &&
    (alternate === null || alternate.lanes === NoLanes)
  ) {
    const lastRenderedReducer = queue.lastRenderedReducer;
    if (lastRenderedReducer !== null) {
      let prevDispatcher;
      const currentState: S = (queue.lastRenderedState: any);
      const eagerState = lastRenderedReducer(currentState, action);
      // 暂存`eagerReducer`和`eagerState`, 如果在render阶段reducer==update.eagerReducer, 则可以直接使用无需再次计算
      update.eagerReducer = lastRenderedReducer;
      update.eagerState = eagerState;
      if (is(eagerState, currentState)) {
        // 快速通道, eagerState与currentState相同, 无需调度更新
        // 注: update已经被添加到了queue.pending, 并没有丢弃. 之后需要更新的时候, 此update还是会起作用
        return;
      }
    }
  }
  // 发起调度更新, 进入`reconciler 运作流程`中的输入阶段.
  scheduleUpdateOnFiber(fiber, lane, eventTime);
}
```

### 异步更新

上述的示例都是在 `Legacy` 模式下，均为同步更新。所以 `update` 实例对象会被全量合并，`hook.baseState` 和 `hook.baseQueue` 没有起到实质性的作用。

然而在最新版的 React 18 中，将全面进入异步时代，也就是传说中的 `Concurrent` 模式，所以本节提前梳理 `update` 实例对象合并的逻辑，同时帮助同学理解 `hook.baseState` 和 `hook.baseQueue` 的作用。

假设有一个 `queue.pending` 链表，其中 `update` 实例对象的优先级不同，`绿色` 表示高优先级，`灰色` 表示低优先级，`红色` 表示最高优先级。

在执行 `updateReducer` 之前，`hook.memoizedState` 有如下结构，如下图所示：

![hook-baseQueue-before](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/hook-baseQueue-before.png)

`baseQueue` 链表拼接：

和同步更新时一致，将 `queue.pending` 拼接到 `current.baseQueue` 中，如下图显示：

![hook-baseQueue-after](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/hook-baseQueue-after.png)

链表完成了拼接之后，就到了计算结果的步骤了，在上图中，存在着两种优先级的 `update` 实例对象，分别是：

- 高优先级：`update1` 和 `update3`。

- 低优先级：`update2` 和 `update4`。

所以在第一次执行计算 `state` 的过程中，只会执行 `update1` 和 `update3` 这两个对象，其中详细流程如下所述：

1. 首先本次更新的优先级为高优先级，所以就会拿到 `current.baseQueue` 中的链表进行遍历。

2. 遍历 `update` 链表的顺序是从第一个开始，这里也就是对应从 `update1` 开始。

   1. 由于 `update1` 属于高优先级，所以会直接执行对应的计算逻辑。

   2. 然后到 `update2` 对象，但是由于 `update2` 为低优先级，在这里是低于高优先级，所以是会跳过 `update2` 实例对象，但是在跳过的同时，也会克隆一份 `update2` 对象出来，用于给下一轮进行计算，由于出现了第一个被跳过的 `update` 实例对象，并且会将优先级设置为 `NoLane`。所以这里会记录一下当前的 `baseState` 用于给下一轮更新时的计算。这里克隆的 `update` 对象会存放在 `baseQueue` 中。

   3. 遍历到了 `update3` 对象，由于 `update3` 也是一个高优先级的对象，符合当前的更新优先级，所以也会进行对应的计算操作，但是这里也需要注意，由于 `update2` 是第一个被跳过的 `update` 实例对象，这里有一个规则：第一个被跳过的 `update` 实例对象后面的 `update` 实例对象，就算优先级跟当前更新的优先级匹配，也要将其克隆一份用于下一轮计算，这里的目的是为了能够保证结果在 `Concurrent` 模式下是正确的。同样的，克隆的 `update` 对象也会存放在 `baseQueue` 中。开启第二轮更新前的内存状态如下图所示：

   ![hook-baseQueue-second-before](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/hook-baseQueue-second-before1.png)

3. 开启下一次的更新，此次的更新的优先级为低优先级，此时重复上面的拼接和遍历 `update` 链表操作，会从 `update2` 开始执行。拼接完，遍历前的内存状态如下图所示：

   ![hook-baseQueue-second-before](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/hook-baseQueue-second-before2.png)

   1. `update2` 优先级符合当前更新优先级，执行计算操作。

   2. `update3` 优先级为 `NoLane`，由于此时判断 `update` 是否符合当前优先级的函数源码如下所示：

      ```typescript
      export function isSubsetOfLanes(set: Lanes, subset: Lane) {
        return (set & subset) === subset;
      }
      ```

      从上可以知道，如果 `subset === NoLane` 的话，改函数的返回值恒为 `true`，也就是说，无论当前执行的任务是什么优先级，他都能符合当前的优先级，这里也是为什么给优先级赋值为 `NoLane`，就是让他在下一个更新，无论优先级是什么，都能够被执行。

   3. `update3` 执行完了之后，又到了 `update4` 执行，又因为 `update4` 的优先级符合要求，所以也是能够被执行。

4. 最终经过了两次更新，`baseQueue` 成功被清空执行完了，并且得出最终结果为 `memoizedState = 4`。内存结构如下图所示：

![hook-baseQueue-second-after](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/hook-baseQueue-second-after.png)

> 尽管 `update` 链表的优先级有所不一样，中间的 `render` 次数可能会有若干次，但是最终的更新结果等于 `update` 链表按顺序合并。
