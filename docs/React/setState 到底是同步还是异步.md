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

