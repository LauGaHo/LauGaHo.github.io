# React 源码中的高频对象解析

在 React 应用中，有许多出现频率很高的数据结构，它们甚至贯穿了 React 应用的整个生命周期。这些数据结构是打开 React 源码的钥匙。本文主要解析 React 应用从启动到渲染这个过程中出现频次较高的数据结构。

## react 包

### ReactElement 对象

#### 引入

开发者在编写 React 应用的时候，通常都是用 `jsx` 进行编写，当应用进行编译的时候，Babel 会将对应的 `jsx` 编译成对应的 `js` 代码。下边将举一个例子进行说明。

有如下的一段 `jsx`：

```jsx
export function App() {
  return (
    <div>
      <Child />
    </div>
  );
}

export function Child() {
  return (
    <p>
      <span>Hello, this is Child Component</span>
    </p>
  );
}
```

编译过程中，Babel 会对其进行对应的处理，处理完之后，将变成 `js` 文件，如下所示：

```js
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true,
});

exports.App = App;

exports.Child = Child;

var _jsxRuntime = require("react/jsx-runtime");

function App() {
  return /*#__PURE__*/ (0, _jsxRuntime.jsx)("div", {
    children: /*#__PURE__*/ (0, _jsxRuntime.jsx)(Child, {}),
  });
}

function Child() {
  return /*#__PURE__*/ (0, _jsxRuntime.jsx)("p", {
    children: /*#__PURE__*/ (0, _jsxRuntime.jsx)("span", {
      children: "Hello, this is Child Component",
    }),
  });
}
```

编译成 `js` 文件之后，从上边可以看到变成了调用 `_jsxRuntime.jsx` 函数，该函数在 `react` 包中被导出，且该函数调用的结果是生成一个类型为 `ReactElement` 的对象。

所以由上可知，开发者编写的 `jsx` 文件，在经过 Babel 编译之后，就会变成 `js` 文件，如上方代码所示，然后在 `reconcile` 阶段中，就会调用对应 `FunctionComponent` 的函数，最终生成的结果就是 `ReactElement`。

#### 数据结构

`ReactElement` 对象的数据结构如下：

```typescript
export type Type = any;
export type Key = any;
export type Ref = any;
export type props = any;
export type ElementType = any;

export interface ReactElementType {
  $$typeof: symbol | number;
  // 表明 ReactElement 种类，如：HostComponent, ClassComponent, FunctionComponent 等
  // 其中 ClassComponent 的 type 为 Class
  // FunctionComponent 的 type 为 function
  type: ElementType;
  key: Key;
  props: Props;
  ref: Ref;
  __mark: string;
}
```

对于 `ReactElement` 数据结构中的属性，解读如下：

- `key` 属性：该属性在 React 中的 `reconcile` 阶段会用到，基本上所有的 `ReactElement` 对象都有 `key` 属性，并且默认值为 `null`，该属性在 diff 算法时会发挥作用。

- `type` 属性：该属性决定了 `ReactElement` 节点的种类：

  - 它的值可以是多种类型，如下：

    - 字符串：代表了 `HostComponent` 类型，即 `div`, `span` 等 DOM 节点。

    - 函数：代表了 `function`, `class` 类型，或者 React 内部定义的节点类型，如：`portal`, `context`, `fragment`。

  - 在 `reconcile` 阶段，会根据 `type` 执行不同的逻辑

    - 如 `type` 是一个字符串类型，则直接调用。

    - 如 `type` 是一个 `ClassComponent` 类型，则会调用其 `render` 方法获取子节点。

    - 如 `type` 是一个 `FunctionComponent` 类型，则会调用对应的函数获取子节点。

经过上述对 `ReactElement` 数据结构的描述，很容易画出上方例子对应的 `ReactElement` 在内存中的结构：

![ReactElement](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/ReactElementStructure2.png)

注意：

- `ClassComponent` 和 `FunctionComponent` 组件，其具体子节点是在 (`reconcile` 阶段) 才会生成，此处只是单纯表示 `ReactElement` 的数据结构。

- 父级对象和子级对象之间是通过 `props.children` 属性进行关联的 (和 `fiber` 树不一样)，相比 `fiber` 树，`ReactElement` 记录的信息比较有限。

- `ReactElement` 虽然不能算是一个严格的树，也不能算是一个严格的链表，它的生成过程是自顶向下的，是所有组件节点的总和。

- `ReactElement` 树和 `fiber` 树是以 `props.children` 为单元先后交替生成，当 `ReactElement` 树构造完毕，`fiber` 树也随后构造完毕。这个也是一个重点。敲黑板！

- `reconcile` 阶段会根据 `ReactElement` 的类型生成对应的 `fiber` 节点，但注意这里并不是一一对应的，比如 `Fragment` 类型的组件在生成 `fiber` 节点的时候会略过。

## react-reconcile 包

`react-reconcile` 包是 React 应用的中枢，连接渲染器 `react-dom` 和调度中心 `scheduler`，同时自身也负责 `fiber` 树的构造。

此处需要了解的是 `fiber` 是核心，React 体系的渲染和更新都要以 `fiber` 作为数据模型，如果不了解 `fiber`，自然也无法理解 `React`。

`react-reconcile` 包中主要包含以下数据结构：

- `Fiber` 对象

- `Update` 对象和 `UpdateQueue` 对象

- `Hook` 对象

### Fiber 对象

`Fiber` 对象的数据结构如下所示：

```typescript
export class FiberNode {
  type: any;
  tag: WorkTag;
  pendingProps: Props;
  key: Key;
  stateNode: any;
  ref: Ref;

  return: FiberNode | null;
  sidling: FiberNode | null;
  child: FiberNode | null;
  index: number;

  memoizedProps: Props | null;
  // 对于 FunctionComponent 来说，存放着 Hook 链表
  memoizedState: any;
  alternate: FiberNode | null;
  flags: Flags;
  subtreeFlags: Flags;
  // 对于 FunctionComponent 来说，存放着 FCUpdateQueue，FCUpdateQueue 中存放着 Effect 环状链表
  updateQueue: unknown;
  // 存放需要被删除的 FiberNode 子节点
  deletions: FiberNode[] | null;

  constructor(tag: WorkTag, pendingProps: Props, key: Key) {
    this.tag = tag;
    this.key = key || null;
    // 对于 HostComponent <div> 该属性就保留了 div DOM
    this.stateNode = null;
    // 对于 FunctionComponent 来说，是一个函数：() => {}
    this.type = null;

    // 构成树状结构
    this.return = null;
    this.sidling = null;
    this.child = null;
    this.index = 0;

    this.ref = null;

    // 作为工作单元
    this.pendingProps = pendingProps;
    this.memoizedProps = null;
    this.memoizedState = null;
    this.updateQueue = null;

    this.alternate = null;
    // 副作用
    this.flags = NoFlags;
    this.subtreeFlags = NoFlags;
    this.deletions = null;
  }
}
```

属性解析：

- `tag`：表示 `fiber` 的种类，根据 `ReactElement` 中的 `type` 生成，其中有：`FunctionComponent`, `HostComponent`, `HostText` 等类别对应的常量。

- `key`：和 `ReactElement` 中的 `key` 属性类似，是每一个组件的唯一性标识。

- `type`：一般来说其值是和 `ReactElement` 中的 `type` 一致，但是对于 `FunctionComponent` 类型来说，对应的 `fiber` 对象中的 `type` 属性应该是是一个函数，即值应当为 `() => {}`，对于 `ClassComponent` 来说，该属性应该为一个 `class`

- `stateNode`：和 `fiber` 对象关联的局部状态节点，不同类型的 `fiber` 对象对应不同类型的局部状态节点，如

  - `HostComponent` 类型 `stateNode` 指向和 `fiber` 节点对应的 `DOM` 节点。

  - 根节点 `stateNode` 指向的是 `FiberRoot`。

  - `ClassComponent` 类型 `stateNode` 指向的是 `class` 实例。

- `return`：指向父节点。

- `child`：指向第一个子节点。

- `sibling`：指向下一个兄弟节点。

- `index`：`fiber` 在兄弟节点中的索引，如果是单节点默认为 0。

- `ref`：指向在 `ReactElement` 组件上设置的 `ref` (`string` 类型的 `ref` 除外，这种类型的 `ref` 已经不推荐使用了，`reconcile` 阶段会将 `string` 类型的 `ref` 转换成一个 `function` 类型)。

- `pendingProps`：输入属性，从 `ReactElement` 对象中传入的 `props`。用于和 `memoizedProps` 之间做比较，得出属性是否发生了变化。

- `memoizedProps`：上一次生成子节点时用到的属性。生成子节点之前叫做 `pendingProps`，生成子节点之后，会将 `pendingProps` 赋值给 `memoizedProps` 属性，用于下一次更新的时候的属性比较。

- `updateQueue`：存储 `update` 实例对象的队列。每一次触发更新，都需要在该属性对应的队列上创建一个 `update` 实例对象。

- `memoizedState`：上一次生成节点之后保存在内存中的局部状态。

- `flags`：标记位，`reconcile` 阶段记录当前 `fiber` 节点有什么操作需要在 `commit` 阶段时进行处理，常见有如下的类型：

  - `Placement`：代表插入或者移动操作。

  - `Update`：代表更新属性。

  - `ChildDeletion`：代表删除子树。

- `subtreeFlags`：记录当前 `fiber` 节点下的子节点有什么 `flags`。其实相当于是从子节点将 `flags` 一直往上冒泡。

- `deletions`：记录当前节点下有哪些子节点需要被删除。

- `alternate`：指向内存中的另一个 `fiber`，每个被更新过的 `fiber` 节点在内存中都是成对出现的 (`current` 和 `workInProgress`。两者通过 `alternate` 进行连接)。

根据上面的例子，可以画出对应的 `fiber` 树的图，如下所示：

![fiber](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/FiberNodeStruct.png)

### Update 和 UpdateQueue 对象

在 `fiber` 对象中有一个属性 `updateQueue`，是一个链式队列 (使用链表实现的队列存储结构)。

先解析 `Update` 对象的数据结构：

```typescript
export interface Update<State> {
  action: Action<State>;
  lane: Lane;
  next: Update<any> | null;
}
export type Action<State> = State | ((prevState: State) => State);
```

属性解析：

- `action`：当前 `update` 对象需要执行的动作，其类型一般为一个具体的数值，或者是一个函数。

- `lane`：当前 `update` 对象对应的优先级。

- `next`：因为 `updateQueue` 是一个环形链表，所以每一个 `update` 都会指向下一个 `update` 对象，最后一个 `update` 对象会指向第一个 `update` 对象。

看完了 `Update` 对象的数据结构，再看 `UpdateQueue` 的数据结构，如下：

```typescript
export interface UpdateQueue<State> {
  shared: {
    pending: Update<State> | null;
  };
  dispatch: Dispatch<State> | null;
}
```

属性解析：

- `shared.pending`：指向当前 `update` 环形链表中最新添加的 `update` 实例对象。

- `dispatch`：针对 Hook 使用的一个属性。

`updateQueue` 是 `fiber` 对象中的一个属性，也可以是 `hook` 对象中的一个属性。

对于 `fiber` 对象来说，其结构如下图所示：

![fiber.updateQueue](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/FiberNode.updateQueue.png)

### Hook 对象

`Hook` 用于 `FunctionComponent` 当中，能够保持 `FunctionComponent` 的状态 (和 `ClassComponent` 中的 `state` 在性质上是相同的，都是为了保持组件的状态)。在 `react@16.8` 之后，官方推荐使用 `Hook` 语法，常用的 `api` 为：`useState`, `useEffect`, `useCallback` 等。

这些 `api` 背后都会创建一个 `Hook` 对象，`Hook` 数据结构如下代码所示：

```typescript
interface Hook {
  memoizedState: any;
  updateQueue: unknown;
  next: Hook | null;
}
```

属性解析：

- `memoizedState`：此处的 `memoizedState` 和 `fiber.memoizedState` 名字一样，但是含义不一样，`fiber.memoizedState` 中所承载的数据是 `fiber` 自身的数据，但是 `hook.memoizedState` 所承载的数据是 `hook` 自身的数据。

  > 此处需要注意，一个 `useState` 对应一个 `hook` 实例对象，一个 `useEffect` 也是对应一个 `hook` 对象。

- `updateQueue`：此处的 `updateQueue` 和 `fiber.updateQueue` 也是同名，同样的，跟上边一样，一个是针对 `fiber` 层面的，这个是针对 `hook` 层面的。也是用于承载 `update` 对象。

`Hook` 和 `fiber` 的关系：

在 `fiber` 对象中有一个属性 `memoizedState` 指向 `fiber` 对象的内存状态，在 `FunctionComponent` 中 `fiber.memoizedState` 就指向 `Hook` 队列 (`Hook` 队列保存了 `FunctionComponent` 的组件状态)。如下图所示：

![fiber.memoizedState](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/fiberNode.memoizedState.png)

## scheduler 包

### Task 对象

`scheduler` 包中对于 `Task` 的定义为：

```typescript
var newTask = {
  id: taskIdCounter++,
  callback,
  priorityLevel,
  startTime,
  expirationTime,
  sortIndex: -1,
};
```

属性解析：

- `id`：`task` 的唯一标识。

- `callback`：`task` 中最核心的字段，指向 `react-reconcile` 包提供的回调函数。

- `priorityLevel`：优先级。

- `startTime`：任务的开始时间 (任务创建时间 + 延时时间)。

- `expirationTime`：任务的过期时间。

- `sortIndex`：控制任务在队列中的次序，值越小越靠前。

> `Task` 中没有 `next` 属性，其顺序是通过一个小顶堆数组来实现的，始终保证数组中的第一个 `task` 对象的优先级最高。
