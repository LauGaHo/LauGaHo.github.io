# Vue 笔记

## `Object.defineProperty` 方法

- 通过 `Object.defineProperty` 方法为对象生成的属性是不可被枚举的 (也就是不可以被遍历的) ，不可被修改，不可被删除
- 若想通过 `Object.defineProperty` 方法为对象生成的属性可以被枚举，就需要进行以下的操作

```javascript
Object.defineProperty(person, 'age', {
  value: 18,
  enumerable: true //默认值为 false
})
```

- 若想通过 `Object.defineProperty` 方法为对象生成的属性可以被修改，就需要进行以下的操作

```javascript
Object.defineProperty(person, 'age', {
  value: 18,
  enumerable: true, //默认值为 false
  writable: true, //默认值为 false
})
```

- 若想通过 `Object.defineProperty` 方法为对象生成的属性可以被删除，就需要进行以下的操作

```javascript
Object.defineProperty(person, 'age', {
  value: 18,
  enumerable: true, //默认值为 false
  writable: true, //默认值为 false
  configurable: true, //默认值为 false
})
```

- 若想通过 `Object.defineProperty` 方法为对象的属性绑定一个对象的地址，需要进行以下操作

```javascript
let number = 18;
Object.defineProperty(person, 'age', {
  // 当有人读取 person.age 就会调用 get 函数，且返回值是 age 的值
  get: function() {
    return number;
  }
  // 当有人修改 person.age 就会调用 set 函数，且会收到修改的具体值
  set: function(value) {
  	number = value;
	}
})
```



## Vue 中的数据代理

```javascript
const vm = new Vue({
  el: '#root',
  data: {
    name: '尚硅谷',
    address: '宏福科技园'
  }
})
// 创建了数据代理之后，就可以通过下面的方式来访问 data 中的数据
console.log(vm.name);
console.log(vm.address);
```

- 上方的代码中，初始化 Vue 实例的参数对象中的 `data` 属性对应的对象地址是存放在 Vue 实例中的 `_data` 属性当中的。然后读取 `_data` 中的属性，为里边的属性创建数据代理。在上方的例子中就是在 Vue 实例中创建 `name` 属性和 `address` 属性的数据代理，意思就是在 Vue 中增加了 `name` 和 `address` 属性。Vue 创建数据代理的方式就是使用 `Object.defineProperty` 这个方法。总结来说 Vue 的数据代理就是通过 vm 对象来代理 `data` 对象中的属性操作。



## Vue 事件处理

- 使用 `v-on:xxx` 或 `@xxx` 绑定事件，其中  `xxx` 是事件名
- 事件的回调需要配置在 `methods` 对象中，最终会在 vm 上
- `methods` 中配置的函数，不要用箭头函数，否则 `this` 就不是 vm 了
- `methods` 中配置的函数，都是被 Vue 所管理的函数，`this` 的指向是 vm 或组件实例对象
- `@click = "demo"` 和 `@click = "demo($event)"` 效果一致，但后者可以传参



## Vue 事件修饰符

- prevent：阻止默认事件
- stop：阻止事件冒泡
- once：事件只触发一次
- capture：使用事件的捕获模式
- self：只有 `event.target` 是当前操作的元素才触发事件
- passive：事件的默认行为立即执行，无需等待事件回调执行完毕



## Vue 计算属性

- 计算属性：需要用到的属性不存在，要通过已有属性计算得来

- 原理：底层借助了 `Object.defineProperty` 方法提供的 `getter` 和 `setter`

- `get` 函数在初次读取时会执行一次；当依赖数据发生改变时会被再次调用

- 与 `methods` 相比，计算属性具有缓存机制，效率更高

- 计算属性最终会出现在 vm 上，直接读取使用即可

- 如果计算属性要被修改，那必须写 `set` 函数去响应修改，且 `set` 中要引起计算时依赖的数据发生改变

- Vue 计算属性响应式更新原理：

  - 计算属性更新过程

    - 首先 b 属性会被处理成存取器属性，访问 b 就会触发其 `get` 函数，从而会执行 `this.b`，于是就触发了 b 的 `get` 函数。
    - b 的 `get` 函数会添加 b 属性的依赖项，而刚才在处理计算属性的过程中，a 已经作为依赖项传给了一个全局变量，b 的 `get` 函数会检测到这个全局变量，并将其添加到自身的订阅者列表中。
    - 对 b 赋予新的值时，会触发其 `set` 函数，`set` 函数中会遍历执行订阅者，a 的值就是在这个时候更新的。

  - 计算属性初始化及响应式更新代码

    - **实现 `defineReactive`**

      - 它用于初始化 `data` 中的数据，转为存取器。`get`：该函数通过闭包，维护每一个属性单独的 `deps`，定义计算属性时，计算属性绑定的函数引用了哪些 `data` 中的属性，就会触发对应的 `get` 函数，向该属性的 `deps` 中添加响应函数。设置 `data` 值时，会将闭包的 `deps` 中所有函数，全部遍历执行一遍

        ```javascript
        // 定义全局属性，用于向dep传送计算属性函数
        var Dep = null
        
        function defineReactive(obj, key, val) {
          var deps = [];
          Object.defineProperty(obj, key, {
            get: function () {
              if (Dep) {
                deps.push(Dep)
              }
              return val
            },
            set: function (newVal) {
              val = newVal;
              deps.forEach(func => func())
            }
          })
        }
        ```

    - 实现 `defineComputed`

      - 它用于在定义计算属性时，如果绑定函数引用了 Vue 实例 `data` 对象中的属性，则会向该属性 `deps` 中添加绑定函数的函数

        ```javascript
        function defineComputed(obj, key, func) {
          func = func.bind(obj);
          let value;
          // 首次定义计算属性，会将第一次的函数执行结果返回给该计算属性，
          // 此后由于闭包，以后该计算属性的value值，会由deps中的函数返回值来决定。
          Dep = function () {
            value = func();
            console.log('Dep中value',value)
          };
          // 执行一次func，函数引用了哪些data中属性，就会触发get方法，向该属性的dep中添加响应函数
          // 在这又使用了闭包，保存计算结果，在get中返回出去
          value = func();
          console.log('defineComputed中value',value)
          // 销毁Dep
          Dep = null;
          // 获取计算属性时，返回函数计算结果
          Object.defineProperty(obj, key, {
            get: function () {
              // 返回函数结果
              return value
            }
          })
        }
        ```

        




## Vue 监视属性

- 当监视的属性变化时，回调函数自动调用，进行相关操作
- 监视的属性必须存在，才能进行监视，`data` 属性和 `computed` 属性都可以被监视
- 监视属性的两种写法：`new Vue` 时传入 `watch` 配置；通过 `vm.$watch` 监视



## Vue 深度监视

- Vue 中的 `watch` 默认不监视对象内部值的改变 (一层)
- 配置 `deep: true` 可以监视对象内部值的改变 (多层)
- Vue 自身可以监测对象内部值的改变，但 Vue 提供的 `watch` 默认不可以
- 使用 `watch` 时根据数据的具体结构，决定是否采用深度监视



## Vue watch 对比 computed

- `computed` 能完成的功能，`watch` 都可以完成
- `watch` 能完成的功能，`computed` 不一定能完成，例如：`watch` 可以进行异步操作
- 所有被 Vue 管理的函数，最好写成普通函数，这样 `this` 的指向才是 `vm` 或者组件示例对象
- 所有不被 Vue 所管理的函数 (定时器的回调函数、ajax 的回调函数等) ，最好写成箭头函数，这样 `this` 的指向才是 `vm` 或者组件实例对象



## Vue style 和 class 绑定

- class 样式：
  - 字符串写法：适用于类名不确定，要动态获取
  - 对象写法：适用于要绑定多个样式，个数不确定，名字也不确定
  - 数组写法：适用于要绑定多个样式，个数确定，名字也确定，但不确定用还是不用
- style 样式：
  - `:style = "{fontSize: xxx}"` 其中 xxx 是动态值
  - `:style = "[a, b]"` 其中 a、b 是样式对象



## Vue 条件渲染

- `v-if`
  - `v-if = "xxx"`
  - `v-else-if = "xxx"`
  - `v-else = xxx`
  - 适用于切换频率比较低的场景，不展示的 DOM 元素直接被移除掉，配合使用时不能被打断
- `v-show`
  - `v-show = "xxx"`
  - 适用于切换频率较高的场景，不展示的 DOM 元素不会被移除，仅仅是使用样式隐藏掉
- 使用 `v-if` 的时候，元素可能无法获取到，但使用 `v-show` 就能被获取到



## Vue 列表渲染

- `v-for` 
  - 用于展示列表数据
  - 语法：`v-for = "(item, index) in xxx" :key = "yyy"`
  - 可遍历：数组、对象、字符串 (用得少)、指定次数 (用得少)



## Vue 中 key 的作用与原理

- 虚拟 DOM 中 key 的作用
  - key 是虚拟 DOM 对象的标识，当数据发生变化时，Vue 会根据新数据生成新的虚拟 DOM，随后 Vue 进行新虚拟 DOM 与旧虚拟 DOM 的差异比较。比较规则在下面说明
- 对比规则
  - 旧虚拟 DOM 中找到了与新虚拟 DOM 相同的 key
    - 若虚拟 DOM 中内容没有发生改变，直接使用之前的真实 DOM
    - 若虚拟 DOM 中内容发生了改变，则直接生成新的真实 DOM，随后替换掉页面中之前的真实 DOM
  - 旧虚拟 DOM 中未找到与新虚拟 DOM 相同的 key
    - 创建新的真实 DOM，随后渲染到页面
- 用 index 作为 key 可能会引发的问题
  - 若对数据进行：逆序添加、逆序删除等操作会产生没有必要的真实 DOM 更新，导致效率低下
  - 如果结构中还包含输入类 DOM 会产生错误 DOM 更新，导致界面有问题
- 开发中如何选择 key
  - 最好使用每条数据的唯一标识作为 key，比如 id、手机号、身份证号、学号等唯一值
  - 如果不存在对数据的逆序添加、逆序删除等破坏顺序操作，仅用于渲染列表用于展示，使用 index 作为 key 是没有问题的。



## Vue 监视数据的原理

- Vue 会监视 data 中所有层次的数据
- 如何监测对象中的数据
  - 通过 `setter` 实现监测，且要在 `new Vue` 时就传入要监测的数据
    - 对象中后追加的属性，Vue 默认不做响应式处理
    - 如需给后添加的属性做响应式，请使用如下的 API：`Vue.set(target, propertyName/index, value)` 或 `vm.$set(target, propertyName/index, value)`
- 如何监测数组中的数据
  - 通过包裹数组更新元素的方法实现，本质上就是做了两件事情
    - 调用原生对应的方法对数组进行更新
    - 重新解析模板，进而更新页面
- 在 Vue 修改数组中的某个元素一定要用如下方法
  - 使用这些 API：`push(), pop(), shift(), unshift(), splice(), sort(), reverse()`
  - 或者使用：`Vue.set()` 或者 `vm.$set()`
  - 特别注意，`Vue.set()` 和 `vm.$set()` 不能给 vm 或者 vm 的根数据对象添加属性



## 收集表单数据

- `<input type="text"/>`。则 `v-model` 收集的是 value 值，用户输入的就是 value 值
- `<input type="radio"/>`。则 `v-model` 收集的是 value 值，且要给标签配置 value 值。
- `<input type="checkbox"/>`
  - 没有配置 `input` 的 value 值，那么收集的就是 checked (布尔值)
  - 配置 `input` 的 value 属性：
    - `v-model` 的初始值是非数组，那么收集的就是 checked (布尔值)
    - `v-model` 的初始值是数组，那么收集的就是 value 组成的数组
  - 备注：`v-model` 的三个修饰符
    - `lazy`：失去焦点再收集数据
    - `number`：输入字符串转为有效的数字
    - `trim`：输入首尾空格过滤



## 过滤器

- 定义：对要显示的数据进行特定格式化后再显示 (适用于一些简单逻辑的处理)
- 语法：
  - 注册过滤器：`Vue.filter(name, callback)` 或 `new Vue(filters: {})`
  - 使用过滤器：`{{xxx | 过滤器名}}` 或 `v-bind: 属性 = "xxx | 过滤器名"`
- 备注
  - 过滤器也可以接收额外的参数、多个过滤器也可以串联
  - 并没有改变原来的数据，是产生新的对应的数据



## 已学的 Vue 指令

- `v-bind` ：单向绑定解析表达式，可简写为 `:xxx`
- `v-model`：双向数据绑定
- `v-for`：遍历数组、对象、字符串
- `v-on`：绑定事件监听，可简写为 `@xxx`
- `v-if`：条件渲染 (动态控制节点是否存在)
- `v-else`：条件渲染 (动态控制节点是否存在)
- `v-show`：条件渲染  (动态控制节点是否展示)



## v-html  指令

- 作用：向指定节点中渲染包含 html 结构的内容
- 与插值语法的区别
  - `v-html` 会替换掉节点中所有的内容，`{{xx}}` 则不会
  - `v-html` 可以识别 html 结构
- 严重注意，`v-html` 有安全性问题
  - 在网站上动态渲染任意 html 是非常危险的，容易导致 XSS 攻击
  - 一定要在可信的内容上使用 v-html，永不要用在用户提交的内容上



## v-cloak 指令

- 本质是一个特殊属性，Vue 实例创建完毕并接管容器后，会删掉 `v-cloak` 属性
- 使用 css 配合 `v-cloak` 可以解决网速慢时页面展示出 `{{xxx}}` 的问题



## v-once 指令

- `v-once` 所在节点在初次动态渲染后，就视为静态内容了
- 以后数据的改变不会引起 `v-once` 所在结构的更新，可以用于优化性能



## v-pre 指令

- 跳过其所在节点的编译过程
- 可利用它跳过：没有使用指令语法、没有使用插值语法的节点，会加快编译



## Vue 模板语法

- 插值语法
  - 功能：用于解析标签体内容
  - 写法：`{{xxx}}` 其中 `xxx` 是 JS 表达式，且可以直接读取到 data 中的所有属性
- 指令语法
  - 功能：用于解析标签 (包括：标签属性、标签体内容、绑定事件)
  - 举例：`v-bind:href="xxx"` 或简写为 `:href="xxx"`，`xxx` 同样要写 JS 表达式，且可以直接读取到 data 中的所有属性
  - 备注：Vue 中有很多指令，且形式都是 `v-??????`，此处我们只是拿  `v-bind` 举个例子



## 自定义指令

- 定义语法

  - 局部指令

    ```javascript
    new Vue({
    	directives: {
    		指令名: 配置对象
    	}
    })
    ```

    或者

    ```javascript
    new Vue({
      directives() {
        
      }
    })
    ```

  - 全局指令

    ```javascript
    Vue.directive(指令名, 配置对象) 或者 Vue.directive(指令名, 回调函数)
    ```

- 配置对象中常用的 3 个回调
  - `.bind`：指令和元素成功绑定时调用
  - `.inserted`：指令所在元素被插入页面时调用
  - `.update`：指令所在模板结构被重新解析时调用
- 备注
  - 指令定义时不加 `v-`，但使用时要加 `v-`
  - 指令名如果是多个单词，要是用 `kekab-case` 命名方式，不要使用 `camelCase` 方式命名
  - 配置对象中的常用 3 个回调函数中的 `this` 指向都是指向 `window`



## 生命周期

- 生命周期又名生命周期回调函数、生命周期函数、生命周期钩子
- Vue 在关键时刻帮我们调用的一些特殊名称的函数
- 生命周期函数的名字不可更改，但函数的具体内容是程序员根据需求编写的
- 生命周期函数中的 this 指向是 vm 或 组件实例对象
- Vue 的生命周期：
  - 将要创建 ==> 调用 `beforeCreate` 函数
  - 创建完毕 ==> 调用 `created` 函数
  - 将要挂载 ==> 调用 `beforeMount`
  - <u>**(重要)**</u> 挂载完毕 ==> 调用 `mounted` 函数 ==========> 重要的钩子
  - 将要更新 ==> 调用 `beforeUpdate` 函数
  - 更新完毕 ==> 调用 `updated` 函数
  - **<u>(重要)</u>** 将要销毁 ==> 调用 `beforeDestroy` 函数 ==========> 重要的钩子
  - 销毁完毕 ==> 调用 `destroyed` 函数

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/生命周期.png)
 - 常用的生命周期钩子
   	- `mounted`：发送 ajax 请求、启动定时器、绑定自定义事件、订阅消息等 (初始化操作)
   	- `beforeDestroy`：清除定时器、解绑自定义事件、取消订阅消息等 (收尾工作)
 - 关于销毁 Vue 实例
   	- 销毁后借助 Vue 开发者工具看不到任何信息
   	- 销毁后自定义事件会失效，但原生 DOM 事件依然有效
   	- 一般不会在 `beforeDestroy` 操作数据，因为即便操作数据，也不会再触发更新流程了



## Vue 使用组件的三大步骤

- 定义组件
  - 使用 `Vue.extend(options)` 创建，其中 `options` 和 `new Vue(options)` 时传入的那个 `options` 几乎一样，但也有点区别，区别如下：
    - `el` 不要写 — 最终所有的组件都要经过一个 vm 的管理，由 vm 中的 `el` 决定服务哪个容器。
    - `data` 必须写成函数  — 避免组件复用时，数据存在引用关系
  - 备注：使用 `template` 可以配置组件结构
- 如何注册组件
  - 局部注册：靠 `new Vue` 的时候传入 `component` 选项
  - 全局注册：靠 `Vue.component('组件名', 组件)`
- 编写组件标签
  - `<school></school>`



## Vue 组件注意的点

- 关于组件名
  - 一个单词
    - 第一种写法 (首字母小写)：school
    - 第二种写法 (首字母大写)：School
  - 多个单词
    - 第一种写法 (kekab-case)：my-school
    - 第二种写法 (CamelCase)：MySchool (需要 Vue 脚手架支持)
  - 备注
    - 组件名尽可能回避 HTML 中已有的元素名称，例如：h2 和 H2 都不行
    - 可以使用 name 配置项指定组件在开发者工具中呈现的名字
- 关于组件标签
  - 第一种写法：`<school></school>`
  - 第二种写法：`</school>`
  - 备注：不使用脚手架时，`</school>` 会导致后续组件不能渲染
- 一个简写方式
  - `const school = Vue.extend(options)` 可简写为：`const school  = options`



## 关于 VueComponent

- school  组件的本质是一个名为 `VueComponent`  的构造函数，且不是程序员定义的，是 `Vue.extend` 生成的
- 我们只需要写 `<school></school>` 或 `</school>`，Vue 解析时会为我们创建 school 组件的实例对象，即 Vue 帮我们执行 `new VueComponent(options)`
- 特别注意：每次调用 `Vue.extend` ，返回的都是一个全新的 `VueComponent`！！！！！！！！！！
- 关于 `this` 指向：
  - 组件配置中：`data`、`methods`、`watch`、`computed` 等函数他们的 `this` 指向都是 `VueComponent` 实例对象
  - `new Vue()` 配置中：`data`、`methods`、`watch`、`computed` 等函数他们的 `this` 指向都是 `Vue` 实例对象
- `VueComponent` 的实例对象，以后简称 `vc` 。`Vue` 的实例对象，以后简称 `vm` 。



## 一个重要的内置关系 

- `VueComponent.prototype.__proto__ === Vue.prototype`
- 上述的关系是为了能够让组件实例 (vc) 可以访问到 Vue 原型上的属性和方法



## 关于不同版本的 Vue

- `vue.js` 和 `vue.runtime.xxx.js` 的区别：
  - `vue.js` 是完整版的 Vue，包含：核心功能和模板解析器
  - `vue.runtime.xxx.js` 是运行版的 Vue，只包含核心功能，没有模板解析器
- 因为 `vue.runtime.xxx.js` 没有模板解析器，所以不能使用 `template` 配置项，需要使用 `render` 函数接收到的 `createElement` 函数去指定具体内容



## ref 属性 (相当于 Angular 中的 @ViewChild)

- 被用来给元素或子组件注册引用信息 (id 的替代者)
- 应用在 HTML 标签上获取的是真实 DOM 元素，应用在组件标签上是组件实例对象 (vc)
- 使用方式：
  - 打标识：`<h1 ref="xxx">......</h1>` 或 `<School ref="xxx"></School>`
  - 获取：`this.$refs.xxx`



## props 配置项 (相当于 Angular 中的 @Input)

- 功能：让组件接收外部传过来的数据

  - 传递数据

    - `<Demo name="xxx"/>` 或者 `<Demo :name="xxx">`。两者的区别在于：前者传入的是一个字符串，后者传入的是一个名为 xxx 的变量

  - 接收数据

    - 第一种方式 (只接收)：

      ```javascript
      props: ['name']
      ```

    - 第二种方式 (限制类型)：

      ```javascript
      props: {
        name: String
      }
      ```

    - 第三种方式 (限制类型、限制必要性、指定默认值)

      ```javascript
      props: {
        name: {
          type: String, //类型
        	required: true,	//必要性
          default: '老王'	//默认值
        }
      }
      ```

- `props` 是只读的，Vue 底层会检测你对 `props` 的修改，如果进行了修改，就会发出警告，若业务需求确实需要修改，那么请复制 `props` 的内容到 `data` 中一份，然后去修改 `data` 中的数据。`VueComponent` 加载的过程中，是先加载 `props` 然后再加载 `data` 的，所以在初始化加载 `data` 的时候，已经能够获取对应 `props` 的值了。



## mixin 混入 (相当于 Angular 中子类继承父类)

- 功能：可以把多个组件共用的配置提取成一个混入对象

- 使用方式：

  - 第一步定义混合，例如：

    ```javascript
    var myMixin = {
      data() {...},
      methods: {...},
      ...
    }
    ```

  - 第二步使用混合，例如：

    - 全局混入：`Vue.mixin(xxx)`
    - 局部混入：`mixins: ['xxx']`
  
- 选项合并

  - 当组件和混入对象含有同名选项时，这些选项将以恰当的方式进行“合并”。

    比如，数据对象 (data) 在内部会进行递归合并，并在发生冲突时以组件数据优先
  
    ```javascript
    var mixin = {
      data: function () {
        return {
          message: 'hello',
          foo: 'abc'
        }
      }
    }
    
    new Vue({
      mixins: [mixin],
      data: function () {
        return {
          message: 'goodbye',
          bar: 'def'
        }
      },
      created: function () {
        console.log(this.$data)
        // => { message: "goodbye", foo: "abc", bar: "def" }
      }
    })
    ```
  
    同名钩子函数将合并成为一个数组，因此都将被调用。另外，混入对象的钩子将在组件自身钩子之前调用。
  
    ```javascript
    var mixin = {
      created: function () {
        console.log('混入对象的钩子被调用')
      }
    }
    
    new Vue({
      mixins: [mixin],
      created: function () {
        console.log('组件钩子被调用')
      }
    })
    
    // => "混入对象的钩子被调用"
    // => "组件钩子被调用"
    ```
  
    值为对象的选项，例如 `method`、`components`、`directives`，将被合并为同一个对象。两个对象键名冲突时，取组件对象的键值对。
  
    ```javascript
    var mixin = {
      methods: {
        foo: function () {
          console.log('foo')
        },
        conflicting: function () {
          console.log('from mixin')
        }
      }
    }
    
    var vm = new Vue({
      mixins: [mixin],
      methods: {
        bar: function () {
          console.log('bar')
        },
        conflicting: function () {
          console.log('from self')
        }
      }
    })
    
    vm.foo() // => "foo"
    vm.bar() // => "bar"
    vm.conflicting() // => "from self"
    ```
  
    注意：`Vue.extend()` 也是用同样的策略进行合并。
  



## 插件

- 用途：

  - 添加全局方法或 property。如：vue-custom-element
  - 添加全局资源：指令/过滤器/过渡.如 vue-touch
  - 通过全局混入来添加一些组件选项。如 vue-router
  - 添加 Vue 实例方法，通过把他们添加到 `Vue.prototype` 上实现

- 使用插件

  - 通过全局方法 `Vue.use()` 使用插件。他需要在你调用 `new Vue()` 启动应用之前完成：

    ```javascript
    // 调用 `MyPlugin.install(Vue)`
    Vue.use(MyPlugin)
    
    new Vue({
     	// ......组件选项
    })
    ```

    也可以传入一个可选的选项对象：

    ```javascript
    Vue.use(MyPlugin, { someOption: true })
    ```

    `Vue.use` 会自动阻止多次注册相同组件，届时即使多次调用也只会注册一次该插件。

- 开发插件

  - Vue 的插件应该暴露一个 `install` 方法。这个方法的第一个参数是 `Vue` 构造器，第二个参数是一个可选的选项对象：

    ```javascript
    MyPlugin.install = function (Vue, options) {
      // 1. 添加全局方法或 property
      Vue.myGlobalMethod = function () {
        // 逻辑...
      }
    
      // 2. 添加全局资源
      Vue.directive('my-directive', {
        bind (el, binding, vnode, oldVnode) {
          // 逻辑...
        }
        ...
      })
    
      // 3. 注入组件选项
      Vue.mixin({
        created: function () {
          // 逻辑...
        }
        ...
      })
    
      // 4. 添加实例方法
      Vue.prototype.$myMethod = function (methodOptions) {
        // 逻辑...
      }
    }
    ```

    

## scoped 样式

- 作用：让样式在局部生效，防止冲突
- 写法：`<style scoped></style>`
- 注意：一般情况下，`AppComponent` 中不是用局部样式，`AppComponent` 中一般使用的都是全局样式



## 总结 TodoList 案例

- 组件化编码流程
  - 拆分静态组件：组件要按照功能点进行拆分，命名不要和 `html` 元素冲突
  - 实现动态组件：考虑好数据的存放位置，数据是一个组件在用还是一些组件在用：
    - 一个组件在用：放在组件自身即可
    - 一些组件再用：放在他们共同的父组件上 (*状态提升*)
  - 实现交互：从绑定事件开始
- `props` 使用于
  - 父组件 ==> 子组件
  - 子组件 ==> 父组件 (要求：需要父组件先给子组件传一个函数，具体可以参考 ezlearn 项目中的 `crossGet`)
- 使用 `v-model` 时需要切记：`v-model` 绑定的值不能是 `props` 传过来的值，因为 `props` 是不可以修改的！(其实这里 Vue 跟 Angular 是有点类似的，在 Angular 中如果存在 `@Input` 这种父传值给子的情况，一旦在子组件中改变对应 `@Input` 的值就很容易会导致 Angular 变更检测的时候无法检测到数据的变化，所以两个框架的道理都是相通的，如果存在父传子的数据绑定，一定需要在数据源的地方进行修改数据，不要直接在子组件对数据进行修改！！！！！！！！这个是代码规范的问题！！！！)
- `props` 传过来的若是类型的值，修改对象中的属性时，Vue 不会报错，但是不推荐这样做，因为这样违背了上一条中所说的规范，关于所有 父传子的绑定，都需要在数据源的地方进行修改，切勿随意在子组件对数据进行修改



## WebStorage

- 存储内容大小一般支持 5MB 左右 (不同浏览器可能还不一样)
- 浏览器通过 Window.sessionStorage 和 Window.localStorage 属性来实现本地存储机制
- 相关 API
  - `xxxxxStorage.setItem('key', 'value')` ：该方法接收一个键和值作为参数，会把键和值添加到存储中，如果键名存在，则更新其对应的值
  - `xxxxxStorage.getItem('person')` ：该方法接收一个键名作为参数，返回键名对应的值
  - `xxxxxStorage.removeItem('key')` ：该方法接收一个键名作为参数，并把键名从存储中删除
  - `xxxxxStorage.clear()` ： 该方法会清空存储中的所有数据
- 备注：
  - SessionStorage 中存储的内容会随着浏览器窗口关闭而消失
  - LocalStorage 中存储的内容，需要手动清除才会消失
  - `xxxxxStorage.getItem(xxx)` 如果对应 xxx 的 value 获取不到，那么 `getItem` 的返回值是 null
  - `JSON.parse(null)` 的结果依然是 null



## 组件的自定义事件

- 一种组件间的通信方式，适用于：子组件 ===> 父组件

- 使用场景：A 是父组件，B 是子组件，B 想给 A 传数据，那么就要在 A 中给 B 绑定自定义事件 (事件回调在 A 中)

- 绑定自定义事件：

  - 第一种方式：在父组件中 `<Demo @atguigu="test"/>` 或 `<Demo v-on:atguigu="test"/>`

  - 第二种方式：在父组件

    `````vue
    <Demo ref="demo"/>
    .............
    mounted() {
    	this.$ref.xxx.$on('atguigu', this.test);
    }
    `````

  		-  若想自定义事件只触发一次，可以使用 `once` 修饰符，或 `$once` 方法

- 触发自定义事件：`this.$emit('atguigu', data)`

- 解绑自定义事件：`this.$off('atguigu')`

- 组件上也可以绑定原生 DOM 事件，需要使用 `native` 修饰符

- 注意：通过 `this.$ref.xxx.$on('atguigu', callback)` 绑定自定义事件时，回调如果要使用 `function(){...}` 这种方式定义的话，就需要定义在 `methods` 属性当中，才能确保 `this` 指针正常，又或者可以直接使用箭头函数



## 全局事件总线 (GlobalEventBus)

- 一种组件间通信的方式，适用于任意组件间通信

- 安装全局事件总线

  ```javascript
  new Vue({
    ...
    beforeCreate() {
    	Vue.prototype.$bus = this // 安装全局事件总线，$bus 就是当前应用的 vm
  	}
    ...
  })
  ```

- 使用事件总线

  - 接收数据：A 组件想接收数据，则在 A 组件中给 `$bus` 绑定自定义事件，事件的回调留在 A 组件自身

    ```vue
    methods() {
      demo(data) {...}
    }
    ......
    mounted() {
    	this.$bus.$on('xxx', this.demo)
    }
    ```

  - 提供数据：`this.$bus.$emit('xxx', data)`

- 最好再 `beforeDestroy` 钩子中，用 `$off` 去解绑当前组件所用到的事件



## 消息订阅与发布 (pubsub)

- 一种组件间通信的方式，适用于任意组件间的通信

- 使用步骤

  - 安装 pubsub：`yarn add pubsub-js`

  - 引入：`import pubsub from 'pubsub-js'`

  - 接收数据：A 组件想接收数据，则在 A 组件中订阅消息，订阅的回调留在 A 组件自身

    ```vue
    methods: {
    	demo(data) {...}
    }
    ...
    mounted() {
    	this.pid = pubsub.subscribe('xxx', this.demo) // 订阅消息
    }
    ```

  - 提供数据：`pubsub.publish('xxx', data)`

  - 最好再 `beforeDestroy` 钩子中，用 `Pubsub.unsubscribe(pid)` 去取消订阅



## nextTick

- 语法：

  ```vue
  this.$nextTick(function(){
  	......
  });
  ```

- 作用：在下一次 DOM 更新结束后执行其指定的回调

- 什么时候用：当更改数据之后，要基于更新后的新 DOM 进行某些操作时，要在 `nextTick()` 所指定的回调函数中执行



## Vue 脚手架配置代理

- **方法一**

  在 `vue-config.js` 中添加如下配置：

  ```vue
  devServer: {
  	proxy: "http://localhost:5000"
  }
  ```

  - **优点：**配置简单，请求资源时直接发送给 `localhost: 8080` 即可
  - **缺点：**不能配置多个代理，不能灵活控制是否走代理
  - **工作方式：**若按照上述配置代理，当请求了 `localhost: 8080` 中不存在的资源，那么就会请求转发给 `localhost: 5000` (优先匹配前端资源，在这里也就是优先匹配 `localhost: 8080` 的资源)

- **方法二**

  编写 `vue.config.js` 配置具体代理规则：

  ```javascript
  module.exports = {
    devServer: {
      proxy: {
        // 匹配所有以 '/api1' 开头的请求路径
        '/api1': {
          // 代理目标的基础路径
          target: "http://localhost: 5000",
          changeOrigin: true,
          pathRewrite: {'^/api1': ''}
        },
        // 匹配所有以 '/api2' 开头的请求路径
        '/api2': {
          // 代理目标的基础路径
          target: "http://localhost: 5001",
          changeOrigin: true,
          pathRewrite: {'^/api2': ''}
        }
      }
    }
  }
  /**
  		changeOrigin 设置为 true 时，服务器收到的请求头中的 host 为：localhost: 5000
  		changeOrigin 设置为 false 时，服务器收到的请求头中的 host 为：localhost: 8080
  		changeOrigin 默认值为 true
  **/
  ```

  - **优点：**可以配置多个代理，且可以灵活的控制请求是否走代理
  - **缺点：**配置稍微繁琐，请求资源时必须加前缀



## 插槽

- 作用：让父组件可以向子组件指定位置插入 HTML 结构，也是一种组件间通信的方式，适用于 <span style="color: red">父组件 ===> 子组件</span>

- 分类：默认插槽、具名插槽、作用域插槽

- 使用方式：

  - 默认插槽

    ```html
    <!--父组件-->
    <Category>
      <div>
        html结构1
      </div>
    </Category>
    
    <!--子组件-->
    <template>
    	<div>
        <!--定义插槽-->
        <slot>插槽默认内容</slot>
      </div>
    </template>
    ```

  - 具名插槽

    ```html
    <!--父组件-->
    <Category>
    	<template slot="center">
      	<div>
          html结构1
        </div>
      </template>
      
      <template v-slot:footer>
      	<div>
          html结构2
        </div>
      </template>
    </Category>
    
    <!--子组件-->
    <template>
    	<div>
        <!--定义插槽-->
      	<slot name="center">插槽默认内容...</slot>
      	<slot name="footer">插槽默认内容...</slot>
      </div>
    </template>
    ```

  - **作用域插槽**

    - **理解：**<span style="color: red">数据在组件的本身，也就是定义插槽的组件本身，但是数据生成HTML结构需要组件的使用者来决定。</span>(games 数据在 Category 组件中，但使用数据所遍历出来的结构由 App 组件决定)

    - **具体代码：**

      ```vue
      <!--父组件-->
      <Category>
      	<template scope="scopeData">
        	<!--生成的是 ul 列表-->
      		<ul>
            <li v-for="g in scopeData.games" :key="g">{{g}}</li>
          </ul>
        </template>
      </Category>
      
      <Category>
      	<template slot-scope="scopeData">
        	<!--生成的是 h4 标题-->
      		<h4 v-for="g in scopeData.games" :key="g">
            {{g}}
          </h4>
        </template>
      </Category>
      
      <!--子组件-->
      <template>
      	<div>
          <slot :games="games"></slot>
        </div>
      </template>
      
      <script>
      	export default {
          name: "Category",
          props: ['title'],
          // 数据在子组件自身
          data() {
            return {
              games: ['红色警戒', '穿越火线', '劲舞团', '超级玛丽']
            }
          }
        }
      </script>
      ```

      

## Vuex

- **概念**
  - 在 Vue 中实现集中式状态 (数据) 管理的一个 Vue 插件，对 Vue 应用中多个组件的共享状态进行集中式管理 (读/写) ，也是一种组件间通信的方式，且适用于任何组件间的通信



- **何时使用**
  - 多个组件需要共享数据时



- **搭建 Vuex 环境**

  - 创建文件：`src/store/index.js`

    ```javascript
    // 引入 Vue 核心库
    import Vue from 'vue'
    // 引入 Vuex
    import Vuex from 'vuex'
    // 应用 Vuex 插件
    Vue.use(Vuex);
    
    // 准备 actions 对象来响应组件中用户的动作
    const actions = {};
    // 准备 mutations 对象来修改 state 中的数据
    const mutations = {};
    // 准备 state 对象来保存具体的数据
    const state = {};
    
    // 创建并导出 store
    export default new Vuex.Store({
      actions,
      mutations,
      state
    })
    ```

    **注意：**在上方的代码中，由于应用插件 `Vue.use(Vuex)` 需要再创建 `store` (需要注意：创建 `store` 语句是在 `./store/index.js` 中的 `new Vuex.Store()`)，所以无法在 `main.js` 中 `import store from ./store` 和 进行 `Vue.use(Vuex)`，因为这两者的顺序必须是先进行 `Vue.use(Vuex)` 然后再进行 `import store from ./store` (即 `new Vuex.Store()`)，不然就会报错，但是由于在文件中，无论 `import ... from ...` 语句在哪一行出现，首先执行的都是 `import ... from ...` 中导入的 JS 文件，在此处就是对应的 `./store/index.js`，相当于 JS 中的 `var` 变量提升， 所以需要将 `Vue.use(Vuex)` 放到 `./store` 当中，如上方所示，这样在 `main.js` 执行 `import store from './store'` 时，就可以保证先进行 `Vue.use(Vuex)` 再进行 `new Vuex.Store()`

  - 在 `main.js` 中创建 vm 时传入 `store` 配置项

    ```javascript
    ......
    // 引入 store
    import store from './store'
    ......
    
    // 创建 vm
    new Vue({
      el: '#app',
      render: h => h(App),
      store,
    })
    ```

  

- **基本使用**

  - **初始化数据、配置 `actions`、配置 `mutations`、操作文件 `store.js`**

    ```javascript
    // 引入 Vue 核心库
    import Vue from 'vue'
    // 引入 Vuex
    import Vuex from 'vuex'
    // 使用 Vuex
    Vue.use(Vuex);
    
    // 初始化 actions
    const actions = {
      // 响应组件中加的动作
      jia(context, value) {
        context.commit('JIA', value);
      },
    };
    
    // 初始化 mutations
    const mutations = {
      // 执行 actions 中 commit 的函数
      JIA(state, value) {
        state.sum += value;
      },
    };
    
    // 初始化数据
    const state = {
      sum: 0,
    };
    
    // 创建并暴露 store
    export default new Vuex.Store({
      actions,
      mutations,
      state,
    });
    ```

  - 组件中读取 vuex 中的数据：`$store.state.sum`

  - 组件中修改 vuex 中的数据：`$store.dispatch('actions 中的方法名', 数据)` 或 `$store.commit('mutations 中的方法名', 数据)`

  - **备注：**

    - 若没有网络请求或其他业务逻辑，组件中也可以越过 `actions` ，即不写 `dispatch` ，直接编写 `commit`
    - 在 `store` 中，切记代码规范，业务逻辑的代码，或者请求后台的代码，统一写在 `actions` 当中，修改数据的代码就写在 `mutations` 中，切勿将业务代码写在 `mutations` 中，`mutations` 只允许编写操作 `state` 中数据的相关代码！！！！！！



- **getter 的使用 (此处的 getters 其实有点类似计算属性 computed 是对原数据进行加工后显示)**

  - **概念：**当 `state` 中的数据需要经过加工后再使用时，可以使用 `getter` 进行加工

  - **在 `store.js` 中追加 `getter` 配置**

    ```javascript
    ......
    
    const getters = {
      bigSum(state) {
        return state.sum * 10;
      }
    }
    
    // 创建并导出 store
    export default new Vuex.Store({
      ......
      getters,
    })
    ```

  - **组件中读取数据：**`$store.getters.bigSum`



- 四个 map 方法的使用

  - **mapState 方法：**用于帮助我们映射 `state` 中的数据为计算属性

    ```javascript
    computed: {
      // 借助 mapState 生成：sum、school、subject 计算属性(对象写法)
      ...mapState({sum: 'sum', school: 'school', subject: 'subject'}),
        
      // 借助 mapState 生成：sum、school、subject 计算属性(数组写法)
      ...mapState(['sum', 'school', 'subject']),
    }
    ```

  - **mapGetters 方法：**用于帮助我们映射 `getters` 中的数据为计算属性

    ```javascript
    computed: {
      // 借助 mapGetters 生成：bigSum 计算属性(对象写法)
      ...mapAGetters({bigSum: 'bigSum'}),
        
      // 借助 mapGetters 生成：bigSum 计算属性(数组写法)
      ...mapGetters(['bigSum'])
    }
    ```

  - **mapActions 方法：**用于帮助我们生成和 `actions` 对话的方法，即：包含 `$store.dispatch(xxx)` 的函数

    ```javascript
    methods: {
      // 靠 mapActions 生成：incrementOdd、incrementWait 函数 (对象形式)
      ...mapActions({incrementOdd: 'jiaOdd', incrementWait: 'jiaWait'})
      
      // 靠 mapActions 生成：jiaOdd、jiaWait 函数 (数组形式)
      ...mapActions(['jiaOdd', 'jiaWait'])
    }
    ```

  - **mapMutations 方法：**用于帮助我们生成和 `mutations` 对话的方法，即：包含 `$store.commit(xxx)` 的参数

    ```javascript
    methods: {
      // 靠 mapActions 生成：increment、decrement 函数 (对象形式)
      ...mapMutations({increment: 'JIA', decrement: 'JIAN'}),
        
      // 靠 mapMutations 生成 JIA、JIAN 函数 (数组形式)
      ...mapMutations(['JIA', 'JIAN']),
    }
    ```

  - **备注：**mapActions 和 mapMutations 使用时，若需要传参数，则需要：在模板中绑定事件时传递好参数，否则参数是事件参数！！！！！！！



- **模块化 + 命名空间**

  - **目的：**让代码更好维护，让多种数据分类更加明确

  - **修改 `store.js`**

    ```javascript
    const countAbout = {
      // 开启命名空间
      namespace: true,
      state: {x: 1},
      mutations: {...}, 
      actions: {...},
      getters: {
        bigSum(state) {
          return state.sum * 10;
        }
      }
    }
                  
    const personAbout = {
      // 开启命名空间
      namespace: true,
      state: {...},
      mutations: {...},
      actions: {...}
    }
                  
    const store = new Vuex,Store({
      modules: {
      	countAbout,
        personAbout,
      }            
    })             
    ```

  - **开启命名空间后，组件中读取 `state` 数据**

    ```javascript
    // 方式一：自己直接读取
    this.$store.state.personAbout.list;
    // 方式而：在 computed 中借助 mapState 读取
    ...mapState('countAbout', ['sum', 'school', 'subject']),
    ```

  - **开启命名空间后，组件中读取 `getters` 数据**

    ```javascript
    // 方式一：自己读取
    this.$store.getters['personAbout/firstPersonName'];
    // 方式二：在 computed 中借助 mapGetters 读取
    ...mapGetters('countAbout', ['bigSum']);
    ```

  - **开启命名空间后，组件中调用 `dispatch`**

    ```javascript
    // 方式一：自己直接 dispatch
    this.$store.dispatch('personAbout/addPersonWang', person);
    // 方式二：在 methods 中借助 mapActions 生成对应函数
    ...mapActions('countAbout', {incrementOdd: 'jiaOdd', incrementWait: 'jiaWait'}),
    ```

  - **开启命名空间后，组件中调用 `commit`**

    ```javascript
    // 方式一：自己直接 commit
    this.$store.commit('personAbout/ADD_PERSON', person);
    // 方式二：在 methods 中借助 mapMutations 生成对应函数
    ...mapMutations('countAbout', {increment: 'JIA', decrement: 'JIAN'}),
    ```



## 路由 Router

- **安装 `vue-router`，命令：`yarn add vue-router`**

- **应用插件：`Vue.use(VueRouter)`**

- **编写 `router` 配置项**

  ```javascript
  // 引入 VueRouter
  import VueRouter from 'vue-router'
  // 引入路由组件
  import About from '../components/About'
  import Home from '../components/Home'
  
  // 创建 router 实例对象，去管理一组一组的路由规则
  const router = new VueRouter({
    routes: [
      {
        path: '/about',
        component: About
      },
      {
        path: '/home',
        component: Home
      }
    ]
  })
  
  // 导出 router
  export default router
  ```

- **实现切换 (`active-class` 可配置高亮样式)**

  ```html
  <router-link active-class="active" to="/about">About</router-link>
  ```

- **指定展示位置**

  ```html
  <router-view></router-view>
  ```

- **几个注意点**

  - 路由组件通常存放在 `pages` 文件夹，一般组件通常在 `components` 文件夹
  - 通过切换，“隐藏”了的路由组件，默认是被销毁的，需要的时候再去挂载
  - 每个组件都有自己的 `$route` s属性，里面存储着自己的路由信息
  - 整个应用只有一个 `router`，可以通过组件的 `$router` 属性获取到



## 多级路由 (嵌套路由)

- **配置路由规则，使用 `children` 配置项**

  ```javascript
  routes: [
    {
      path: '/about',
      component: About,
    },
    {
      path: '/home',
      component: Home,
      // 通过 children 配置子路由
      children: [
        {
          // 此处一定不要写：/news
          path: 'news',
          component: News,
        },
        {
          // 此处一定不要写：/message
          path: 'message',
          component: Message,
        }
      ]
    }
  ]
  ```

- **跳转 (要写完整路径)：**

  ```vue
  <router-link to="/home/news">News</router-link>
  ```



## 路由的 query 参数

- **传递参数**

  ```vue
  <!-- 跳转并携带 query 参数，to 的字符串写法 -->
  <router-link :to="`/home/message/detail?id=${id}&title=${title}`">跳转</router-link>
  
  <!-- 跳转并携带 query 参数，to 的对象写法 -->
  <router-link :to="{
                   		path: '/home/message/detail',
                   		query: {
                   				id: 666,
                   				title: '你好'
                   		}
                   }"
  >跳转</router-link>
  ```

- **接收参数**

  ```vue
  $route.query.id
  $route.query.title
  ```



## 命令路由

- **作用：可以简化路由的跳转**

- **使用方法**：

  - **给路由命名：**

    ```javascript
    {
      path: '/demo',
      component: Demo,
      children: [
        {
          path: 'test',
          component: Test,
          children: [
            {
              // 给路由命名
              name: 'hello',
              path: 'welcome',
              component: Hello,
            }
          ]
        }
      ]
    }
    ```

  - **简化跳转**：

    ```vue
    <!-- 简化前，需要写完整的路径 -->
    <router-link to="/demo/test/welcome">跳转</router-link>
    
    <!-- 简化后，直接通过名字跳转 -->
    <router-link :to="{name: 'hello'}">跳转</router-link>
    
    <!-- 简化写法配合传递参数 -->
    <router-link :to="{
                     		name: 'hello',
                     		query: {
                     				id: 666,
                     				title: '你好'
                     		}
                     }"
    >跳转</router-link>
    ```



## 路由的 params 参数

- **配置路由，声明接收 params 参数**

  ```javascript
  {
    path: '/home',
    component: Home,
    children: [
      {
        path: 'news',
        component: News,
      },
      {
        component: Message,
        children: [
          {
            name: 'xiangqing',
  					// 使用占位符声明接收 params 参数
            path: 'detail/:id/:title',
            component: Detail
          }
        ]
      },
    ]
  }
  ```

- **传递参数**

  ```vue
  <!-- 跳转并携带 params 参数，to 的字符串写法 -->
  <router-link :to="/home/message/detail/666/你好">跳转</router-link>
  
  <!-- 跳转并携带 params 参数，to 的对象写法 -->
  <router-link :to="{
                   		name: 'xiangqing',
                   		params: {
                   				id: 666,
                   				title: '你好',
                   		}
                   }"
  >跳转</router-link>
  ```

  > **特别注意：路由携带 params 参数时，若使用 to 的对象写法，则不能使用 path 配置项，必须使用 name 配置项！！！**

- **接收参数**

  ```javascript
  $route.params.id
  $route.params.title
  ```



## 路由的 props 配置

- **作用：让路由组件更方便的收到参数**

  ```javascript
  {
    name: "xiangqing",
    path: 'detail/:id',
    component: Detail,
    
    // 第一种写法：props 值为对象，该对象中所有的 key-value 的组合最终都会通过 props 传给 Detail 组件
    props: {a: 90},
      
    // 第二种写法：props 值为布尔值，布尔值为 true，则把路由收到的所有 params 参数通过 props 传给 Detail 组件
    props: true,
      
    // 第三种写法：props 值为函数，该函数返回的对象中每一组 key-value 都会通过 props 传给 Detail 组件
    props(route) {
      return {
        id: route.query.id,
        title: route.query.title,
      }
    }
  }
  ```

  

## `<route-link>` 的 replace 属性

- **作用：**控制路由跳转时操作浏览器历史记录的模式
- **浏览器的历史记录有两种写入方式：**分别是 `push` 和 `replace` ，`push` 是追加历史记录，`replace` 是替换当前的记录。路由跳转时候，默认开启的是 `push` 方式
- **如何开启 `replace` 模式：**`<router-link replace ......>News</router-link>`



## 编程式路由导航

- **作用：**不借助 `<router-link>` 实现路由跳转，让路由跳转更加灵活

- **具体编码：**

  ```javascript
  // $router 的两个 API
  this.$router.push({
    name: 'xiangqing',
    params: {
      id: xxx,
      title: xxx,
    }
  })
  
  this.$router.replace({
    name: 'xiangqing',
    params: {
      id: xxx,
      title: xxx
    }
  })
  
  // 前进
  this.$router.forward();
  // 后退
  this.$router.back();
  // 可前进也可后退
  this.$router.go();
  ```



## 缓存路由组件

- **作用：**让不展示的路由组件保持挂载，不被销毁

- **具体编码：**

  ```vue
  <!-- 缓存一个组件 -->
  <keep-alive include="News">
  	<router-view></router-view>
  </keep-alive>
  
  <!-- 缓存多个组件 -->
  <keep-alive :include="['News', 'Message']">
  	<router-view></router-view>
  </keep-alive>
  ```



## 两个新的生命周期钩子

- **作用：**路由组件所独有的两个钩子，用于捕获路由组件的激活状态
- **具体名字：**
  - `activated` 路由组件被激活时触发
  - `deactivated` 路由组件失活时触发

