# React 基础

## 虚拟 DOM

### 使用 JSX

```jsx
// 创建虚拟DOM
const VDOM = (
  <h1 id="title">
    <span>Hello, React</span>
  </h1>
);
// 渲染虚拟DOM到页面
ReactDOM.render(VDOM, document.getElementById("test"));
```

### 使用 JS

```jsx
// 创建虚拟 DOM
const VDOM = React.createElement(
  "h1",
  { id: "title" },
  React.createElement("span", {}, "Hello, React")
);
// 渲染虚拟DOM到页面
ReactDOM.render(VDOM, document.getElementById("test"));
```

### 虚拟 DOM 和真实 DOM

```jsx
// 创建虚拟 DOM
const VDOM = (
  <h1 id="title">
    <span>Hello, React</span>
  </h1>
);
// 渲染虚拟 DOM 到页面
ReactDOM.render(VDOM, document.getElementById("test"));

const TDOM = document.getElementById("demo");
console.log("虚拟 DOM", VDOM);
console.log("真实 DOM", TDOM);
debugger;
// console.log(typeof VDOM)
// console.log(VDOM instanceof Object)
```

关于虚拟 DOM：

- 本质上是 Object 类型的对象 (一般对象)
- 虚拟 DOM 比较轻，真实 DOM 比较重，因为虚拟 DOM 是 React 内部在用，无需真实 DOM 上那么多属性
- 虚拟 DOM 最终会被 React 转化真实 DOM，呈现在页面上

## jsx

### jsx 语法规则

```jsx
const myId = 'aTgUiGu'
const myData = 'HeLlo,rEaCt'

// 创建虚拟 DOM
const VDOM = (
  <div>
    <h2 className="title" id={myId.toLowerCase()}>
      <span style={{color: 'white', fontSize: '29px'}}>{myData.toLowerCase()}</span>
    </h2>
    <h2 className="title" id={myId.toUpperCase()}>
      <span style={{color: 'white', fontSize: '29px'}}>{{myData.toLowerCase()}}></span>
    </h2>
    <input type='text'/>
  </div>
)

// 渲染虚拟 DOM 到页面
ReactDOM.render(VDOM, document.getElementById('test'))
```

jsx 语法规则

1. 定义虚拟 DOM 时，不要写引号
2. 标签中混入 JS 表达式时要使用 {} 号
3. 样式的类名指定不能使用 class，要使用 className
4. 内联样式，要用 `style={{key: value}}` 的形式书写
5. 只有一个根标签
6. 标签必须闭合
7. 标签首字母
   1. 若小写字母开头，则将该标签转为 `html` 中同名元素，若 `html` 中无该标签对应的同名元素，则报错
   2. 若大写字母开头，React 就渲染对应的组件，若组件没有定义，则报错
8. JS 表达式：一个表达式会产生一个值，可以放在任何一个需要值的地方，如下所示都是 JS 表达式
   1. `a`
   2. `a + b`
   3. `demo(1)`
   4. `arr.map()`
   5. `function test () {}`
9. JS 语句，如下所示都是 JS 语句
   1. `if () {}`
   2. `for () {}`
   3. `switch () {case: xxx}`

> 注意：在书写 jsx 的时候，在 html 标签中嵌套只能写 JS 表达式，而不可以写 JS 语句！！！切记

## react 组件

### 函数式组件

```jsx
// 创建函数式组件
function MyComponent() {
  // 此处的 this 是 undefined，因为 babel 编译后开启了严格模式
  console.log(this);
  return <h2>我是用函数定义的组件(适用于【简单组件】的定义)</h2>;
}

// 渲染组件到页面
ReactDOM.render(<MyComponent />, document.getElementById("test"));
```

执行了 `ReactDOM.render(<MyComponent />, document.getElementById("test"))` 之后，将发生：

- React 解析组件标签，找到了 MyComponent 组件
- 发现组件是使用函数定义的，随后调用该函数，将返回的虚拟 DOM 转为真实 DOM，随后呈现在页面中

### 类式组件

```jsx
class MyComponent extends React.Component {
  render() {
    // render 放在了 MyComponent 的原型对象上，供实例使用
    // render 中的 this 指向 MyComponent 的实例对象，即 MyComponent 组件实例对象
    console.log("render 中的 this：", this);
    return <h2>我是用类定义的组件(适用于【复杂组件】的定义)</h2>;
  }
}

// 渲染组件到页面
ReactDOM.render(<MyComponent />, document.getElementById("test"));
```

执行了 `ReactDOM.render(<MyComponent/>......)` 后将发生：

1. React 解析组件标签，找到 MyComponent 组件
2. 发现组件使用类定义，随后 new 出该类的实例，并通过该实例调用到原型上的 render 方法
3. 将 render 返回的虚拟 DOM 转为真实 DOM，随后呈现在页面中

## 组件三大属性之—state

### state

```jsx
class Weather extends React.Component {
  // 构造器调用 1 次
  constructor(props) {
    console.log("constructor");
    super(props);
    // 初始化状态
    this.state = { isHot: false, wind: "微风" };
    // 解决 changeWeather 中的 this 指向问题
    this.changeWeather = this.changeWeather.bind(this);
  }

  // render 调用 1 + n 次，1 是初始化的那次，n 是状态更新的次数
  render() {
    console.log("render");
    // 读取状态
    const { isHot, wind } = this.state;
    return (
      <h1 onClick={this.changeWeather}>
        今天天气很{isHot ? "炎热" : "凉爽"}，{wind}
      </h1>
    );
  }

  // changeWeather 点击几次就调用几次
  changeWeather() {
    // changeWeather 放在 Weather 的原型对象上，供实例使用
    // 由于 changeWeather 作为 onClick 的回调，所以不是通过实例调用的，是直接调用
    // 类中的方法默认开启了局部的严格模式，所以 changeWeather 中的 this 为 undefined
    console.log("changeWeather");
    // 获取原来的 isHot 值
    const isHot = this.state.isHot;
    // 严重注意：状态必须通过 setState 进行更新，且更新是一种合并，不是替换
    this.setState({ isHot: !isHot });
    console.log(this);

    // 严重注意：状态 (state) 不可直接更改，下面这行就是直接更改，是错误的做法！！！
    // this.state.isHot = !isHot // 错误的写法
  }
}

// 渲染组件到页面
ReactDOM.render(<Weather />, document.getElementById("test"));
```

### state 的简写方式

```jsx
class Weather extends React.Component {
  //初始化状态
  state = { isHot: false, wind: "微风" };

  render() {
    const { isHot, wind } = this.state;
    return (
      <h1 onClick={this.changeWeather}>
        今天天气很{isHot ? "炎热" : "凉爽"}，{wind}
      </h1>
    );
  }

  // 自定义方法—要用赋值语句的形式 + 箭头函数
  changeWeather = () => {
    const isHot = this.state.isHot;
    this.setState({ isHot: !isHot });
  };
}
// 渲染组件到页面
ReactDOM.render(<Weather />, document.getElementById("test"));
```

## 组件三大属性之—props

### props 基本使用

```jsx
class Person extends React.Component {
  render() {
    const { name, age, sex } = this.props;
    return (
      <ul>
        <li>姓名：{name}</li>
        <li>性别：{sex}</li>
        <li>年龄：{age + 1}</li>
      </ul>
    );
  }
}

// 渲染组件到页面
ReactDOM.render(
  <Person name="jerry" age={19} sex="男" />,
  document.getElementById("test1")
);
ReactDOM.render(
  <Person name="tom" age={18} sex="女" />,
  document.getElementById("test2")
);

const p = { name: "老刘", age: 18, sex: "女" };
// ReactDOM.render(<Person name={p.name} age={p.age} sex={p.sex}/>,document.getElementById('test3'))
ReactDOM.render(<Person {...p} />, document.getElementById("test3"));
```
