# React 调度器核心原理解析 (Scheduler)

在 React 运行时，调度中心 (`Scheduler`) 位于 `scheduler` 包中，是整个 React 运行 `Concurrent` 模式更新的核心，所以理解了 `Scheduler` 就相当于对 React 的 `Concurrent` 更新有一个整体的把握。

其实在 React 当中分为了 `render` 阶段和 `commit` 阶段，其中跟调度器相关的阶段是 `render` 阶段，React 官方所说的 `Concurrent` 更新主要也是集中在了 `render` 阶段中，只有 `render` 阶段是可以被打断的，`commit` 阶段是不允许被打断。

并且 React 为了避免和 `Scheduler` 之间耦合，`react-reconciler` 包中使用了 `Lane` 来描述任务的优先级，而在 `Scheduler` 中则使用了 `SchedulerPriority` 来描述任务的优先级。

## Scheduler

`Scheduler` 的核心逻辑集中在 `scheduler` 包中 `scheduler/cjs/scheduler.development.js` 文件当中，该文件的主要导出的 `api` 如下所示：

```javascript
/********************* 导出对应的 SchedulerPriority ******************/
exports.unstable_IdlePriority = IdlePriority;
exports.unstable_ImmediatePriority = ImmediatePriority;
exports.unstable_LowPriority = LowPriority;
exports.unstable_NormalPriority = NormalPriority;
exports.unstable_Profiling = unstable_Profiling;
exports.unstable_UserBlockingPriority = UserBlockingPriority;

/********************* 导出对应 Scheduler 的 api ******************/
// 取消回调
exports.unstable_cancelCallback = unstable_cancelCallback;
exports.unstable_continueExecution = unstable_continueExecution;
exports.unstable_forceFrameRate = forceFrameRate;
// 获取当前正在执行的回调对应的 SchedulerPriority
exports.unstable_getCurrentPriorityLevel = unstable_getCurrentPriorityLevel;
// 获取第一个回调
exports.unstable_getFirstCallbackNode = unstable_getFirstCallbackNode;
exports.unstable_next = unstable_next;
// 暂停执行
exports.unstable_pauseExecution = unstable_pauseExecution;
// 请求绘制
exports.unstable_requestPaint = unstable_requestPaint;
// 使用某个优先级进行运行操作
exports.unstable_runWithPriority = unstable_runWithPriority;
// 调度某一个回调
exports.unstable_scheduleCallback = unstable_scheduleCallback;
// 判断是否应该中断执行
exports.unstable_shouldYield = shouldYieldToHost;
exports.unstable_wrapCallback = unstable_wrapCallback;
```

上述 `exports` 的变量是提供给开发者调用或使用，除了上述 `exports` 的变量和函数之外，还有一些没有 `exports` 的变量和函数，但是对于 `Scheduler` 的运行有着举足轻重的作用。

下边将对调度相关的函数进行讲解。

### 调度相关函数

在 `Scheduler` 包中跟调度相关的函数是如下 4 个：

- `requestHostCallback`：请求一个回调函数

  ```javascript
  function requestHostCallback(callback) {
    // 将需要调度的 callback 赋值给全局变量 scheduledHostCallback
    scheduledHostCallback = callback;

    if (!isMessageLoopRunning) {
      isMessageLoopRunning = true;
      // 将 performWorkUntilDeadline 函数放进宏任务队列，在下一个任务周期执行
      schedulePerformWorkUntilDeadline();
    }
  }
  ```

  补充说明：上方函数中调用了 `schedulePerformWorkUntilDeadline` 函数将 performWorkUntilDeadline 函数放进了宏任务队列中，具体方法如下：

  ```javascript
  var schedulePerformWorkUntilDeadline;

  // 优先使用 setImmediate 方法把 performWorkUntilDeadline 函数放进宏任务队列
  if (typeof localSetImmediate === "function") {
    schedulePerformWorkUntilDeadline = function () {
      localSetImmediate(performWorkUntilDeadline);
    };
  }
  // 其次使用 MessageChannel 将 performWorkUntilDeadline 函数放进宏任务队列
  else if (typeof MessageChannel !== "undefined") {
    var channel = new MessageChannel();
    var port = channel.port2;
    channel.port1.onmessage = performWorkUntilDeadline;

    schedulePerformWorkUntilDeadline = function () {
      port.postMessage(null);
    };
  }
  // 最后使用 setTimeout 来进行兜底
  else {
    schedulePerformWorkUntilDeadline = function () {
      localSetTimeout(performWorkUntilDeadline, 0);
    };
  }
  ```

  这里的 `performWorkUntilDeadline` 函数其实就是执行全局变量 `scheduledHostCallback` 对应的回调函数，如下所示：

  ```javascript
  var performWorkUntilDeadline = function () {
    if (scheduledHostCallback !== null) {
      // 获取当前时间
      var currentTime = exports.unstable_now();

      // 将当前时间赋值给开始时间
      startTime = currentTime;
      // 在当前函数执行上下文定义变量 hasTimeRemaining，标识当前时间切片是否还是剩余时间
      var hasTimeRemaining = true;

      // 在当前函数执行上下文定义变量 hasMoreWork，标识当前任务是否执行完，是否还有任务剩余
      var hasMoreWork = true;

      try {
        // 执行全局变量 scheduledHostCallback 变量对应的回调函数，并将返回结果赋值给 hasMoreWork 变量
        hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
      } finally {
        // 如果上一个任务还没有执行完，则将 performWorkUntilDeadline 函数重新再调度一次
        if (hasMoreWork) {
          schedulePerformWorkUntilDeadline();
        } else {
          isMessageLoopRunning = false;
          scheduledHostCallback = null;
        }
      }
    } else {
      isMessageLoopRunning = false;
    }
  };
  ```

- `cancelHostCallback`：取消请求某个回调函数

  ```javascript
  cancelHostCallback = function () {
    // 将 scheduledHostCallback 全局变量置空
    scheduledHostCallback = null;
  };
  ```

- `requestHostTimeout`：请求一个延时任务

  ```javascript
  function requestHostTimeout(callback, ms) {
    taskTimeoutID = localSetTimeout(function () {
      callback(exports.unstable_now());
    }, ms);
  }
  ```

- `cancelHostTimeout`：取消某一个延时任务

  ```javascript
  function cancelHostTimeout() {
    // 清除计时器
    localClearTimeout(taskTimeoutID);
    // 将全局变量 taskTimeoutID 置为 -1
    taskTimeoutID = -1;
  }
  ```

### 时间切片相关函数

执行时间切片分割，让出主线程 (把控制权归还浏览器，浏览器可以处理用户输入，UI 绘制等紧急任务)，而跟事件切片相关的函数为如下所示：

- `getCurrentTime`：获取当前事件。

  ```javascript
  const localPerformance = performance;
  getCurrentTime = () => localPerformance.now();
  ```

- `shouldYieldToHost`：判断是否让出主线程。计算方法如下方代码注释所示：

  ```javascript
  function shouldYieldToHost() {
    // 用当前事件减去当前任务的开始时间得出当前任务执行了多长时间
    var timeElapsed = exports.unstable_now() - startTime;

    // 如果当前任务执行的时间 > 浏览器一帧对应的时间 (一般为 5ms)
    if (timeElapsed < frameInterval) {
      // 返回 false，即不需要让出主线程
      return false;
    }

    // 反之则返回 true，即需要让出主线程
    return true;
  }
  ```

- `requestPaint`：请求绘制。

  ```javascript
  requestPaint = function () {
    // 将 needsPaint 全局变量赋值为 true
    needsPaint = true;
  };
  ```

- `forceFrameRate`：根据给定的帧率 fps 设置时间切片的周期。

  ```javascript
  function forceFrameRate(fps) {
    if (fps < 0 || fps > 125) {
      console["error"](
        "forceFrameRate takes a positive int between 0 and 125, " +
          "forcing frame rates higher than 125 fps is not supported"
      );
      return;
    }

    if (fps > 0) {
      frameInterval = Math.floor(1000 / fps);
    } else {
      // reset the framerate
      frameInterval = frameYieldMs;
    }
  }
  ```

### 任务队列管理

在 `Scheduler` 中，维护了一个 `taskQueue` 和一个 `timerQueue`，其数据结构是一个数组，原理是一个堆。

```javascript
var taskQueue = [];
var timerQueue = [];
```

注意：

- `taskQueue` 是一个小顶堆数组，关于堆排序的原理，可以自行查阅。

- `timerQueue` 是给延时任务使用的。

#### 创建任务

任务创建的逻辑是在 `unstable_scheduleCallback` 函数中，这个函数其实也是 `exports` 给开发者使用的 `api`：

```javascript
function unstable_scheduleCallback(priorityLevel, callback, options) {
  // 获取当前事件
  var currentTime = exports.unstable_now();
  // 定义 startTime 变量描述任务开始时间
  var startTime;

  // 延时任务：startTime = currentTime + delay
  // 非延时任务：startTime = currentTime
  if (typeof options === "object" && options !== null) {
    var delay = options.delay;

    if (typeof delay === "number" && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }

  // 定义 timeout 变量描述当前任务 timeout 变量
  var timeout;

  // 根据不同的优先级给 timeout 变量赋不同的值
  // 就像 ImmediatePriority 优先级，timeout 就为 -1
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;

    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;

    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;

    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;

    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }

  // 定义变量 expirationTime 描述过期时间，expirationTime = startTime + timeout
  var expirationTime = startTime + timeout;
  // 创建任务实例对象 newTask
  var newTask = {
    id: taskIdCounter++,
    callback: callback,
    priorityLevel: priorityLevel,
    startTime: startTime,
    expirationTime: expirationTime,
    sortIndex: -1,
  };

  // startTime > currentTime 说明当前任务为延时任务，丢进 timerQueue
  if (startTime > currentTime) {
    // This is a delayed task.
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);

    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      } // Schedule a timeout.

      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  }
  // 反之丢进 taskQueue
  else {
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    // wait until the next time we yield.

    // 请求调度 flushWork，在 flushWork 中会执行 taskQueue 中的任务
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

#### 消费任务

创建任务之后，最后会调用 `requestHostCallback` 函数请求调度，其中 `flushWork` 函数作为参数被传入调度中心内核等待回调。其中请求调度之后，就会在下一个事件回调过程中执行 `performWorkUntilDeadline`，而在 `performWorkUntilDeadline` 函数中会最终执行 `flushWork` 函数。

消费任务相关的函数如下：

- `flushWork`：刷新任务队列。

  ```javascript
  function flushWork(hasTimeRemaining, initialTime) {
    // 全局标记，标识进入回调执行阶段
    isHostCallbackScheduled = false;

    // 假设调度了一个延时任务，此时将 isHostTimeoutScheduled 变量赋值为 false，并且取消被调度的延时任务
    if (isHostTimeoutScheduled) {
      // We scheduled a timeout but it's no longer needed. Cancel it.
      isHostTimeoutScheduled = false;
      cancelHostTimeout();
    }

    isPerformingWork = true;
    // 使用 previousPriorityLevel 变量记录上一个任务的优先级
    var previousPriorityLevel = currentPriorityLevel;

    try {
      if (enableProfiling) {
        try {
          // 循环消费队列
          return workLoop(hasTimeRemaining, initialTime);
        } catch (error) {
          if (currentTask !== null) {
            var currentTime = exports.unstable_now();
            markTaskErrored(currentTask, currentTime);
            currentTask.isQueued = false;
          }

          throw error;
        }
      } else {
        // No catch in prod code path.
        // 循环消费队列
        return workLoop(hasTimeRemaining, initialTime);
      }
    } finally {
      currentTask = null;
      // 还原 currentPriorityLevel 变量
      currentPriorityLevel = previousPriorityLevel;
      isPerformingWork = false;
    }
  }
  ```

- `workLoop`：

  ```javascript
  function workLoop(hasTimeRemaining, initialTime) {
    // 将 currentTime 赋值为传入的 initialTime 变量
    var currentTime = initialTime;
    // 用 currentTime 传入 advanceTimers 函数看 timerQueue 中是否有 task 到期需要丢进 taskQueue 中
    advanceTimers(currentTime);
    // 从 taskQueue 中取出优先级最高的任务，也就是 expirationTime 最小的任务
    currentTask = peek(taskQueue);

    while (currentTask !== null && !enableSchedulerDebugging) {
      // 当前任务没有过期 && (当前事件循环没有时间剩下 || 应该退出主线程)
      // 符合这种 if 条件的情况是：currentTask 没有过期，但是执行时间超过了限制
      if (
        currentTask.expirationTime > currentTime &&
        (!hasTimeRemaining || shouldYieldToHost())
      ) {
        // This currentTask hasn't expired, and we've reached the deadline.
        break;
      }

      // 将当前 task 中的 callback 属性赋值为 callback 变量
      var callback = currentTask.callback;

      if (typeof callback === "function") {
        // 将 task 中的 callback 属性置空
        currentTask.callback = null;
        // 将 task 的优先级 priorityLevel 赋值给 currentPriorityLevel
        currentPriorityLevel = currentTask.priorityLevel;
        // 定义变量 didUserCallbackTimeout 表示当前 task 是否过期
        var didUserCallbackTimeout = currentTask.expirationTime <= currentTime;

        // 执行 callback 回调
        var continuationCallback = callback(didUserCallbackTimeout);
        // 更新 currentTime 变量
        currentTime = exports.unstable_now();

        if (typeof continuationCallback === "function") {
          // 如果 callback 执行结果返回的结果是一个函数类型的对象，则将其返回结果赋值到 currentTask 中的 callback 属性
          currentTask.callback = continuationCallback;
        } else {
          // 反之说明执行完毕，直接在 taskQueue 中直接弹出
          if (currentTask === peek(taskQueue)) {
            pop(taskQueue);
          }
        }

        // 重新使用 currentTime 刷新一边 timerQueue
        advanceTimers(currentTime);
      } else {
        // 如果 callback 不为函数类型，直接在 taskQueue 直接弹出
        pop(taskQueue);
      }

      // 重新在 taskQueue 中弹出一个优先级高的 task
      currentTask = peek(taskQueue);
    } // Return whether there's additional work

    if (currentTask !== null) {
      // currentTask 未执行完，则交给调度器执行下一次调度
      return true;
    } else {
      var firstTimer = peek(timerQueue);

      if (firstTimer !== null) {
        requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
      }

      return false;
    }
  }
  ```

  每次 `while` 循环的退出就是一个时间切片，`while` 循环的退出条件：

  1. 队列被完全清空。

  2. 执行超时，消费 `taskQueue` 时，在执行 `task.callback` 之前，都会检测是否超时，所以超时检测是以 `task` 作为单位的。

     - 如果某个 `task.callback` 执行时间太长 (如 `fiber` 树很大) 也会造成超时。

     - 所以在执行 `task.callback` 过程中，也需要一种机制检测是否超时，如果超时就暂停 `task.callback` 的执行。

#### 时间切片原理

消费任务队列过程中，可以消费 `1-n` 个 `task`，甚至清空整个 `taskQueue`。但是在每一次具体执行 `task.callback` 之前都要进行超时检测，如果超时了可以立即退出循环并等待下一次调用。

#### 可中断原理

在时间切片的基础上，如果单个 `task.callback` 的执行时间过长，例如执行时间为 `200ms`，这个时候就需要 `task.callback` 自己能够检测是否超时，所以在 `fiber` 树构造的过程中，每构造完一个单元，都会检测一次是否超时，如果遇到超时就退出 `fiber` 树的构造，并返回一个新的回调函数，就是此处的 `continuationCallback`，并等待下次回调继续未完成的构造 `fiber` 树。
