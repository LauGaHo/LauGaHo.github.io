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

### 对 props 进行限制

```jsx
class Person extends React.Component {
  render() {
    const { name, age, sex } = this.props;
    // props 是只读的
    // this.props.name = 'jack' // 此行代码会报错，因为 props 是只读的
    return (
      <ul>
        <li>姓名：{name}</li>
        <li>性别：{sex}</li>
        <li>年龄：{age + 1}</li>
      </ul>
    );
  }
}

// 对标签属性进行类型、必要性的限制
Person.propTypes = {
  name: PropTypes.string.isRequired, // 限制 name 必传，且为字符串
  sex: PropTypes.string, // 限制 sex 为字符串
  age: PropTypes.number, // 限制 age 为数值
  speak: PropTypes.func, // 限制 speak 为函数
};

// 指定默认 props 属性值
Person.defaultProps = {
  sex: "男", // sex 默认值为男
  age: 18, // age 默认值为 18
};

// 渲染组件到页面
ReactDOM.render(<Person name={100} speak={speak} />, document.getElementById('test1'))
ReactDOM.render(<Person name"tom" age={18} sex="女" />, document.getElementById('test2'))

const p = { name: '老刘', age: 18, sex: '女'}
ReactDOM.render(<Person {...p} />, document.getElementById('test3'))

function speak() {
    console.log('我说话了')
}
```

> 上方使用了 PropTypes 包，所以务必在 `package.json` 中安装对应的依赖！

### props 的简写方式

```jsx
class Person extends React.Component {
  constructor(props) {
    // 构造器是否接收 props，是否传递给 super，取决于：是否希望在构造器中通过 this 访问 props
    // console.log(props)
    super(props);
    console.log("constructor", this.props);
  }

  // 对标签属性进行类型、必要性限制
  static propTypes = {
    name: PropTypes.string.isRequired,
    sex: PropTypes.string,
    age: PropTypes.number,
  };

  // 对 props 指定默认值
  static defaultProps = {
    sex: "男",
    age: 18,
  };

  render() {
    const { name, age, sex } = this.props;
    // props 是只读的
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
ReactDOM.render(<Person name="jerry" />, document.getElementById("test1"));
```

### 函数式组件使用 props

```jsx
function Person(props) {
  const { name, age, sex } = props;
  return (
    <ul>
      <li>姓名：{name}</li>
      <li>性别：{sex}</li>
      <li>年龄：{age}</li>
    </ul>
  );
}

Person.propTypes = {
  name: PropTypes.string.isRequired,
  sex: PropTypes.string,
  age: PropTypes.number,
};

// 指定 props 默认值
Person.defaultProps = {
  sex: "男",
  age: 18,
};

// 渲染组件到页面
ReactDOM.render(<Person name="jerry" />, document.getElementById("test1"));
```

## 组件三大属性之—refs

### 字符串形式的 refs

```jsx
class Demo extends React.Component {
  // 展示左侧输入框的数据
  showData = () => {
    const { input1 } = this.refs;
  };

  // 展示右侧输入框的数据
  showData2 = () => {
    const { input2 } = this.refs;
    alert(input2.value);
  };

  render() {
    return (
      <div>
        <input ref="input1" type="text" placeholder="点击按钮提示数据" />
        &nbsp;
        <button onClick={this.showData}>点我提示左侧的数据</button>&nbsp;
        <input
          ref="input2"
          onBlur={this.showData2}
          type="text"
          placeholder="失去焦点提示数据"
        />
      </div>
    );
  }
}

// 渲染组件到页面
ReactDOM.render(<Demo a="1" b="2" />, document.getElementById("test"));
```

### 回调函数形式的 refs

```jsx
class Demo extends React.Component {
  // 展示左侧输入框的数据
  showData = () => {
    const { input1 } = this;
    alert(input1.value);
  };

  // 展示右侧输入框的数据
  showData2 = () => {
    const { input2 } = this;
    alert(input2.value);
  };

  render() {
    return (
      <div>
        <input
          ref={(c) => (this.input1 = c)}
          type="text"
          placeholder="点击按钮提示数据"
        />
        &nbsp;
        <button onClick={this.showData}>点我提示左侧的数据</button>&nbsp;
        <input
          onBlur={this.showData2}
          ref={(c) => (this.input2 = c)}
          type="text"
          placeholder="失去焦点提示数据"
        />
        &nbsp;
      </div>
    );
  }
}

// 渲染组件到页面
ReactDOM.render(<Demo a="1" b="2" />, document.getElementById("test"));
```

### 回调 ref 中的回调执行次数的问题

```jsx
class Demo extends React.Component {
  state = { isHot: false };

  showInfo = () => {
    const { input1 } = this;
    alert(input1.value);
  };

  changeWeather = () => {
    // 获取原来的状态
    const { isHot } = this.state;
    // 更新状态
    this.setState({ isHot: !isHot });
  };

  saveInput = (c) => {
    this.input1 = c;
    // 只会打印一次，即该回调函数只会执行一次
    console.log("@", c);
  };

  render() {
    const { isHot } = this.state;
    return (
      <div>
        <h2>今天天气很{isHot ? "炎热" : "凉爽"}</h2>
        <input ref={this.saveInput} type="text" />
        <br />
        <br />
        <button onClick={this.showInfo}>点我提示输入的数据</button>
        <button onClick={this.changeWeather}>点我切换天气</button>
      </div>
    );
  }
}
```

### createRef 的使用

```jsx
class Demo extends React.Component {
  // React.createRef 调用后可以返回一个容器，该容器可以存储被 ref 所标识的节点，该容器是“专人专用”
  myRef = React.createRef();
  myRef2 = React.createRef();

  // 展示左侧输入框的数据
  showData = () => {
    alert(this.myRef.current.value);
  };

  // 展示右侧输入框的数据
  showData2 = () => {
    alert(this.myRef2.current.value);
  };

  render() {
    return (
      <div>
        <input ref={this.myRef} type="text" placeholder="点击按钮提示数据" />
        &nbsp;
        <button onClick={this.showData}>点我提示左侧的数据</button>&nbsp;
        <input
          onBlur={this.showData2}
          ref={this.myRef2}
          type="text"
          placeholder="失去焦点提示数据"
        />
        &nbsp;
      </div>
    );
  }
}

// 渲染组件到页面
ReactDOM.render(<Demo a="1" b="2" />, document.getElementById("test"));
```

## React 中事件处理

### 事件处理

```jsx
class Demo extends React.Component {
  // 创建 ref 容器
  myRef = React.createRef();
  myRef2 = React.createRef();

  // 展示左侧输入框的数据
  showData = (event) => {
    console.log(event.target);
    alert(this.myRef.current.value);
  };

  // 展示右侧输入框的数据
  showData2 = (event) => {
    alert(event.target.value);
  };

  render() {
    return (
      <div>
        <input ref={this.myRef} type="text" placeholder="点击按钮提示数据" />
        &nbsp;
        <button onClick={this.showData}>点我提示左侧的数据</button>&nbsp;
        <input
          onBlur={this.showData2}
          type="text"
          placeholder="失去焦点提示数据"
        />
        &nbsp;
      </div>
    );
  }
}
```

1. 通过 `onXxx` 属性指定事件处理函数 (注意大小写)
   1. React 使用的是自定义 (合成) 事件，而不是使用的原生 DOM 事件——为了更好的兼容性
   2. React 中的事件是通过事件委托方式处理的 (委托给组件最外层的元素)——为了高效
2. 通过 `event.target` 得到发生事件的 DOM 元素对象——不要过度使用 ref

## React 表单数据管理

针对对于表单数据管理方式，可以分为两种组件，如下：

- 非受控组件：表单数据将交给 DOM 节点来处理
- 受控组件：表单数据将交给 React 组件进行管理

> 通常情况下，推荐使用受控组件

### 受控组件

```jsx
class Login extends React.Component {
  // 初始化状态
  state = {
    username: "", // 用户名
    password: "", // 密码
  };

  // 保存用户名到状态中
  saveUsername = (event) => {
    this.setState({ username: event.target.value });
  };

  // 保存密码到状态中
  savePassword = (event) => {
    this.setState({ password: event.target.value });
  };

  // 表单提交的回调
  handleSubmit = (event) => {
    event.preventDefault(); // 阻止表单提交
    const { username, password } = this.state;
    alert(`你输入的用户名：${username}，你输入的密码是：${password}`);
  };

  render() {
    return (
      <form onSubmit={this.handleSubmit}>
        用户名：
        <input onChange={this.saveUsername} type="text" name="username" />
        密码：
        <input onChange={this.savePassword} type="password" name="password" />
        <button>登录</button>
      </form>
    );
  }
}

// 渲染组件
ReactDOM.render(<Login />, document.getElementById("test"));
```

### 非受控组件

```jsx
class Login extends React.Component {
  handleSubmit = (event) => {
    event.preventDefault();
    const { username, password } = this;
    alert(
      `你输入的用户名是：${username.value},你输入的密码是：${password.value}`
    );
  };

  render() {
    return (
      <form onSubmit={this.handleSubmit}>
        用户名：
        <input ref={(c) => (this.username = c)} type="text" name="username" />
        密码：
        <input
          ref={(c) => (this.password = c)}
          type="password"
          name="password"
        />
        <button>登录</button>
      </form>
    );
  }
}

// 渲染组件
ReactDOM.render(<Login />, document.getElementById("test"));
```

## 高阶函数

### 柯里化

- 高阶函数：如果一个函数符合下面 2 个规范中的任何一个，那么该函数就是高阶函数：

  1. 若 A 函数，接收的参数是一个函数，那么 A 就可以称之为高阶函数
  2. 若 A 函数，调用的返回值依然是一个函数，那么 A 就可以称之为高阶函数

> 常见的高阶函数有：`Promise`、`setTimeout`、`arr.map()` 等等

- 函数柯里化：通过函数调用继续返回函数的方式，实现多次接收参数最后统一处理的函数编码方式

  ```js
  function sum(a) {
    return (b) => {
      return (c) => {
        return a + b + c;
      };
    };
  }
  ```

```jsx
class Login extends React.Component {
  // 初始化状态
  state = {
    username: "", // 用户名
    password: "", // 密码
  };

  // 保存表单数据到状态中
  saveFormData = (dataType) => {
    return (event) => {
      this.setState({ [dataType]: event.target.value });
    };
  };

  // 表单提交的回调
  handleSubmit = (event) => {
    event.preventDefault(); // 阻止表单提交
    const { username, password } = this.state;
    alert(`你输入的用户名是：${username},你输入的密码是：${password}`);
  };

  render() {
    return (
      <form onSubmit={this.handleSubmit}>
        用户名：
        <input
          onChange={this.saveFormData("username")}
          type="text"
          name="username"
        />
        密码：
        <input
          onChange={this.saveFormData("password")}
          type="password"
          name="password"
        />
        <button>登录</button>
      </form>
    );
  }
}

// 渲染组件
ReactDOM.render(<Login />, document.getElementById("test"));
```

### 不使用函数柯里化的实现

```jsx
class Login extends React.Component {
  // 初始化
  state = {
    username: "", // 用户名
    password: "", // 密码
  };

  // 保存表单数据到状态中
  saveFormData = (dataType, event) => {
    this.setState({ [dataType]: event.target.value });
  };

  // 表单提交的回调
  handleSubmit = (event) => {
    event.preventDefault();
    const { username, password } = this.state;
    alert(`你输入的用户名是：${username},你输入的密码是：${password}`);
  };

  render() {
    return (
      <form onSubmit={this.handleSubmit}>
        用户名：
        <input
          onChange={(event) => this.saveFormData("username", event)}
          type="text"
          name="username"
        />
        密码：
        <input
          onChange={(event) => this.saveFormData("password", event)}
          type="password"
          name="password"
        />
        <button>登录</button>
      </form>
    );
  }
}

// 渲染组件
ReactDOM.render(<Login />, document.getElementById("test"));
```
