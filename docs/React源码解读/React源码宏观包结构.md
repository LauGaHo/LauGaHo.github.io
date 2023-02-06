# React 源码宏观包结构

## 基础包结构

### react

`react` 基础包，只提供定义 `react` 组件 (`ReactElement`) 的必要函数，一般来说是需要和渲染器 (`react-dom`, `react-native`) 一同配合使用，在编写 `react` 应用的代码时，大部分都是调用此包的 api。

> 注意：上边所说的渲染器 (`react-dom`, `react-native`) 其实本质上就是宿主环境，因为 React 这个框架的使用是需要配合不同的宿主环境。`react-dom` 表示宿主环境为浏览器；`react-native` 表示宿主环境为应用端。

### react-dom

React 的渲染器之一，是 React 和 Web 平台连接的桥梁，将 `react-reconciler` 中的运行结果输出到 Web 页面。在编写 React 应用的代码时，大多数场景下，可以用到此包的就是一个入口函数 `ReactDOM.render(<App />, document.getElementById('root'))`，其余使用的 api，基本上是 react 包提供的。

### react-reconciler

React 得以运行的核心包 (综合协调 `react-dom`, `react`, `scheduler` 各个包之间的调用和配合)。管理 React 应用状态的输入和结果的输出。将输入信号最终转化成信号输出给渲染器。

- `scheduleUpdateOnFiber` 函数接收输入。
- 将 `fiber` 树生成逻辑封装到一个回调函数 `performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`。这其中涉及 `fiber` 树形结构，`fiber.updateQueue` 队列，调和算法等。
- 把 `performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot` 送入 `scheduler` 进行调度。
- 调度器 `scheduler` 会控制回调函数执行的时机，回调函数执行完成后得到全新的 `fiber` 树。
- 再调用渲染器，形如 (`react-dom`, `react-native` 等)，将 `fiber` 树形结构最终反映到界面上。

### scheduler

调度机制的核心实现，控制由 `react-reconciler` 送入的回调函数 (这里指代 `performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`) 的执行时机，在 `concurrent` 模式下可以实现任务分片。在编写 React 应用的代码时，几乎不会直接使用到此包提供的 api。

## 宏观总览

### 架构分层

为了便于理解，可以将 React 应用整体架构分为接口层 (api) 和内核层 (core) 2 个部分。

1. **接口层 (api)**：`react` 包，平时在开发过程中使用的绝大部分的 api 都是来自于此包。在 React 启动之后，正常可以改变渲染的基本操作有三个：

   - `ClassComponent` 中使用 `setState()`
   - `FunctionComponent` 中使用 Hook，并发起 `dispatchAction()` 改变 Hook 实例对象。
   - 改变 `context`，实际上也是需要 `setState()` 或者 `dispatchAction()` 的辅助才能改变。

   以上的 `setState` 和 `dispatchAction` 都是由 `react` 包直接暴露。所以一般开发者都是直接调用 `react` 包的 api 和其他 core 包进行交互。

2. **内核层 (core)**：内核层由 3 个部分构成

   1. 调度器 (`scheduler` 包)：核心职责只有 1 个，就是执行回调。

      - 把 `react-reconciler` 提供的回调函数 (这里指代 `performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot` 函数) 包装成一个任务对象中，并丢进调度器 `scheduler` 其内部的任务队列中。
      - 调度器 `scheduler` 在内部维护一个任务队列，优先级高的排在最前面。
      - 循环消费任务队列，直到任务队列清空。

   2. 协调器 (`react-reconciler` 包)：核心职责有 3 个。

      - 装填渲染器，渲染器必须实现 `HostConfig` 协议，每个渲染器 (如：`react-dom` 和 `react-native`) 都拥有不同的 `HostConfig` 协议。保证在需要的时候，能够正确调用渲染器的 api，生成实际节点 (如：DOM 节点)。
      - 接收 `react-dom` 包 (初次 `render`) 和 `react` 包 (后续更新 `setState` 或 `dispatchAction`) 发起的更新请求。
      - 将 `fiber` 树的构造过程包装在一个回调函数 (这里指代 `performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot` 函数)，并将此回调函数传入到 `scheduler` 包等待调度。

   3. 渲染器 (`react-dom` 包)：核心职责有 2 个。

      - 引导 React 应用的启动 (通过 `ReactDOM.render()` 函数)。
      - 实现 `HostConfig` 协议，能够将 `react-reconciler` 包构造出来的 `fiber` 树表现出来，生成 DOM 节点 (宿主环境为浏览器)，或者生成字符串 (服务端渲染 `SSR`)。
