# Vue源码解读—渲染函数的观察者和进阶的数据响应系统 (上)

## `$mount()` 挂载函数

**在 `src/core/instance/init.js` 文件中找到 `Vue.prototype._init()` 函数，如下代码：**

```javascript
Vue.prototype._init = function (options?: Object) {
  // 省略...

  // expose real self
  vm._self = vm
  initLifecycle(vm)
  initEvents(vm)
  initRender(vm)
  callHook(vm, 'beforeCreate')
  initInjections(vm) // resolve injections before data/props
  initState(vm)
  initProvide(vm) // resolve provide after data/props
  callHook(vm, 'created')

  // 省略...

  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```

**以上时简化后的代码，注意代码 `vm.$mount(vm.$options.el)`，这行代码是 `_init()` 函数的最后一行代码，这行代码执行之前完成了所有初始化的工作。**

**`$mount()` 函数的定义出现在两个地方，第一个在 `src/platforms/web/runtimes/index.js` 文件中，如下：**

```javascript
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

**`src/platform/web/runtime/index.js` 文件是运行版 Vue 的入口文件。`$mount()` 函数接收两个参数：**

- **第一个参数 `el` 可以是字符串也可以是一个 DOM 元素**
- **第二个参数 `hydrating` 是用于补充 `Virtual DOM` 补丁算法的**

**`$mount()` 函数的第一行代码如下：**

```javascript
el = el && inBrowser ? query(el) : undefined
```

**首先检查是否传递了 `el` 选项，如果传递了 `el` 选项则会直接判断 `inBrowser` 是否为真，即当前宿主环境是否是浏览器，如果在浏览器中则将 `el` 传给 `query()` 函数，并用返回值重写 `el` 变量，否则 `el` 将会被重写为 `undefined`。其中 `query()` 函数来自 `src/platforms/web/util/index.js` 文件，用来根据给定的参数在 DOM 中查找对应的元素并返回。如果在浏览器环境下，`el` 变量将存储着 DOM 元素 (理想情况下)。**

**接着来到了 `$mount()` 函数的第二行代码：**

```javascript
return mountComponent(this, el, hydrating)
```

**调用了 `mountComponent()` 函数完成真正的挂载工作，并返回其运行结果，以上就是运行时版 Vue 的 `$mount()` 函数所做的事情。**

**第二个定义 `$mount()` 函数的地方是 `src/platforms/web/entry-runtime-with-compiler.js` 文件，这个文件是完整版 Vue 的入口文件，在该文件中重新定义了 `$mount()` 函数，但是保留了运行时 `$mount()` 的功能，并在此基础上为 `$mount()` 函数添加了模板编译的能力，接下来详细讲述完整版 `$mount()` 函数的实现，完整版的 `$mount()` 位于 `src/platforms/web/entry-runtime-with-compiler.js` 文件，如下：**

```javascript
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  // 省略...
  return mount.call(this, el, hydrating)
}
```

**上方代码中，首先使用 `mount` 常量缓存了运行时版的 `$mount()` 函数，然后重新定义了 `Vue.prototype.$mount()` 函数并在重新定义的 `$mount()` 函数体中调用了缓存下来的运行时版的 `$mount()` 函数，另外重新定义前后 `$mount()` 函数所接收的参数是不变的。之所以重写 `$mount()` 函数，是因为为了给运行时版的 `$mount()` 函数增加模板编译能力，函数开头如下：**

```javascript
el = el && query(el)

/* istanbul ignore if */
if (el === document.body || el === document.documentElement) {
  process.env.NODE_ENV !== 'production' && warn(
    `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
  )
  return this
}
```

**如果传递了 `el` 参数，那么使用 `query()` 函数获取指定的 DOM 元素并重新赋值给 `el` 变量，这个元素被称为挂载点。接着是一段 `if` 语句块，检测挂载点是不是 `<body>` 元素或者 `<html>` 元素，如果是的话，并且当前处于非生产环境下，会打印警告信息，警告不要挂载到 `<body>` 元素或者 `<html>` 元素。因为挂载点的本意是组件挂载的占位，它将会被组件自身的模板替换掉，而 `<body>` 元素和 `<html>` 元素显然不能被替换掉。**

**继续看代码，如下是对 `$mount()` 函数剩余代码的简化：**

```javascript
const options = this.$options
// resolve template/el and convert to render function
if (!options.render) {
  // 省略...
}
return mount.call(this, el, hydrating)
```

**定义了 `options` 常量，该常量是 `$options` 的引用，其实也就是 `vm.$options` 的引用，然后使用一个 `if` 语句判断是否包含 `render` 选项，即是否包含渲染函数。如果渲染函数存在则不需要做任何事情，直接调用运行时的 `$mount()` 函数即可。运行时版本的 `$mount()` 函数仅仅是两句代码，真正的挂载所需的必要条件是 `mountComponent()` 函数完成，所以 `mountComponent()` 完成挂载的必须条件就是：*提供渲染函数给 `mountComponent()`。***

**如果 `options.render` 选项不存在，这个时候会执行 `if` 语句块的代码，其所做的事情只有一个：*使用 `template` 或 `el` 选项构建渲染函数。*下边详细解析构建渲染函数的过程。`if` 语句块的第一段代码：**

```javascript
let template = options.template
if (template) {
  if (typeof template === 'string') {
    if (template.charAt(0) === '#') {
      template = idToTemplate(template)
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && !template) {
        warn(
          `Template element not found or is empty: ${options.template}`,
          this
        )
      }
    }
  } else if (template.nodeType) {
    template = template.innerHTML
  } else {
    if (process.env.NODE_ENV !== 'production') {
      warn('invalid template option:' + template, this)
    }
    return this
  }
} else if (el) {
  template = getOuterHTML(el)
}
```

**首先定义了 `template` 变量，初始值是 `options.template` 选项的值，在没有 `render` 渲染函数的情况下会优先使用 `template` 选项，并尝试将 `template` 选项编译成渲染函数，但开发者未必传递了 `template` 选项，这时会检测 `el` 是否存在，如果存在，则使用 `el.outerHTML` 作为 `template` 的值。**

**这里进行一个插播，说明一下 `innerHTML` 和 `outerHTML` 的区别，我们需要一段代码进行分析：**

```html
<div id="target">
  <span>You know what I mean</span>
</div>
```

**此时打印 `document.getElementById("target").innerHTML` 则结果是：**

```html
<span>You know what I mean</span>
```

**此时打印 `document.getElementById("target").outerHTML` 则结果是：**

```html
<div id="target">
  <span>You know what I mean</span>
</div>
```

**从上可知，`innerHTML` 和 `outerHTML` 的差别是：`innerHTML` 不包含选中元素自身的 HTML，只是包含其内部的 HTML。**

**而 `outerHTML` 是选中元素自身也包含进去。**

**上方的 `if` 代码分支较多，但目标只有一个，即获取合适的内容作为模板 (`template`)，下边总结阐述了获取模板 (`template`) 的过程：**

- **如果 `template` 选项不存在，那么使用 `el` 元素的 `outerHTML` 作为模板的内容**
- **如果 `template` 选项存在：**
  - **且 `template` 选项存在：**
    - **如果第一个字符是 `#`，那么会把该字符串作为 `CSS` 选择符去选中对应的元素，并把该元素的 `innerHTML` 作为模板**
    - **如果第一个字符不是 `#`，那什么都不做，就用 `template` 自身的字符串值作为模板**
  - **且 `template` 的类型是元素节点 (`template.nodeType` 存在)**
    - **则使用该元素的 `innerHTML` 作为模板**
  - **若 `template` 既不是字符串，又不是元素节点，那么在非生产环境下会提示开发者传递的 `template` 选项无效**

**经过以上逻辑的处理之后，理想状态下此时 `template` 变量应该是一个模板字符串，将来用于渲染函数的生成。但这个 `template` 存在为空字符串的情况，所以即便经过了以上逻辑的处理，后续还要对其进行判断。**

**另外上边的代码中使用了两个工具函数，分别是 `idToTemplate()` 和 `getOuterHTML()` 函数，这两个函数都定义在当前文件中，其中 `idToTemplate()` 函数的源码如下：**

```javascript
const idToTemplate = cached(id => {
  const el = query(id)
  return el && el.innerHTML
})
```

**`getOuterHTML()` 函数源码如下：**

```javascript
function getOuterHTML (el: Element): string {
  if (el.outerHTML) {
    return el.outerHTML
  } else {
    const container = document.createElement('div')
    container.appendChild(el.cloneNode(true))
    return container.innerHTML
  }
}
```

**上方函数接收一个 DOM 元素作为参数，并返回该元素的 `outerHTML`。上方代码先判断了 `outerHTML` 是否存在，也就是说一个元素的 `outerHTML` 属性未必存在，实际上 `IE 9-11` 中 `SVG` 标签元素是没有 `innerHTML` 和 `outerHTML` 这两个属性的。解决这个问题的方法很简单，将 `SVG` 元素放到一个新创建的 `<div>` 标签中，这样 `<div>` 元素的 `innerHTML` 属性就等价于 `SVG` 标签的 `outerHTML` 的值了，这就是上方代码中 `else` 分支中的内容了。**

**然后继续接着看代码：**

```javascript
if (template) {
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    mark('compile')
  }

  const { render, staticRenderFns } = compileToFunctions(template, {
    shouldDecodeNewlines,
    shouldDecodeNewlinesForHref,
    delimiters: options.delimiters,
    comments: options.comments
  }, this)
  options.render = render
  options.staticRenderFns = staticRenderFns

  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    mark('compile end')
    measure(`vue ${this._name} compile`, 'compile', 'compile end')
  }
}
```

**上方代码是最关键的地方，在处理完 `options.template` 选项之后，`template` 变量中存储着最终用来生成渲染函数的字符串，但是正如前边所提到的，`template` 变量有可能是一个空字符串，所以上方代码中的 `if (template)` 就是对 `template` 进行判断，只有在 `template` 存在才执行 `if` 语句块中的代码，而 `if` 语句块中的代码的作用就是使用 `compileToFunction()` 函数将 `template` 编译成渲染函数 (`render`)，并将渲染函数添加到 `vm.$options` 选项中 (`options` 是 `vm.$options` 的引用)。对于 `compileToFunctions()` 函数会在之后讲解 Vue 编译器的时候进行解析说明。实际上在 `src/platforms/web/entry-runtime-with-compiler.js` 文件中可以看到这样一句代码：**

```javascript
Vue.compile = compileToFunctions
```

**`Vue.compile()` 函数是 Vue 暴露给开发者的工具函数，它能够将字符串编译为渲染函数。而上边的这行代码证明了 `Vue.compile()` 函数就是 `compileToFunction()` 函数。**

**最后总结：*实际上，完整版的 `$mount()` 函数需要做的核心事情就是编译模板 (`template`) 字符串为渲染函数，并将渲染函数赋值给 `vm.$options.render` 选项，这个选项会在真正挂载组件的 `mountComponent()` 函数中发挥真正的作用。***



## 渲染函数的观察者

**不管完整版 Vue 的 `$mount()` 函数还是运行时版 Vue 的 `$mount()` 函数，最终都是通过 `mountComponent()` 函数去真正的挂载组件，接着细看 `mountComponent()` 函数中做了什么，`mountComponent()` 函数位于 `src/core/instance/lifecycle.js` 文件中，如下：**

```javascript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  // 省略...
}
```

**`mountComponent()` 函数接收三个参数，分别是组件实例 `vm`，挂载元素 `el` 以及传进来的 `hydrating` 参数。`mountComponent()` 函数的第一行代码如下：**

```javascript
vm.$el = el
```

**在组件实例对象上添加 `$el` 属性，其值为挂载元素 `el`。但是根据我们所知道的 Vue 基础，可以得知 `$el` 的值是组件模板根元素的引用，如下代码：**

```vue
<div id="foo"></div>

<script>
const new Vue({
  el: '#foo',
  template: '<div id="bar"></div>'
})
</script>
```

**上方代码中，挂载元素是一个 `id` 为 `foo` 的 `<div>` 元素，而组件模板是一个 `id` 为 `bar` 的 `<div>` 元素。那么此时 `vm.$el` 应该是 `id` 为 `bar` 的 `<div>` 引用。这是因为 `vm.$el` 始终是组件模板的根元素。由于我们传递了 `template` 选项指定了模板，`vm.$el` 自然是 `id` 为 `bar` 的 `<div>` 引用。假设没有传递 `template` 选项，根据前边的分析，`el` 选项指定的挂载点将被作为组件模板，这个时候 `vm.$el` 则是 `id` 为 `foo` 的 `<div>` 元素的引用。**

**现在结合 `mountComponent()` 函数体中的这一行代码：`vm.$el = el`，有的同学可能此时就会相当疑惑，这里将 `el` 挂载元素赋值给 `vm.$el`，那么 `vm.$el` 怎么可能引用的是 `template` 选项指定的模板的根元素？？？其实这里仅仅是暂时赋值而已，这是为了给虚拟 DOM 的 `patch` 算法使用的，实际上 `vm.$el` 会被 `patch` 算法的返回值重写了，打开 `src/core/instance/lifecycle.js` 文件找到 `Vue.prototype._update()` 方法，如下代码所示：**

```javascript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  // 省略...

  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  
  // 省略...
}
```

**正如上方代码所示，`vm.$el` 的值将会被 `vm.__patch__()` 函数的返回值重写。不过现在或许对 `Vue.prototype._update()` 函数不太了解，后边会详细讲述。**

**继续查看 `mountComponent()` 函数的代码，接下来是一段 `if` 语句块：**

```javascript
if (!vm.$options.render) {
  vm.$options.render = createEmptyVNode
  if (process.env.NODE_ENV !== 'production') {
    /* istanbul ignore if */
    if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
      vm.$options.el || el) {
      warn(
        'You are using the runtime-only build of Vue where the template ' +
        'compiler is not available. Either pre-compile the templates into ' +
        'render functions, or use the compiler-included build.',
        vm
      )
    } else {
      warn(
        'Failed to mount component: template or render function not defined.',
        vm
      )
    }
  }
}
```

**这段 `if` 条件语句块首先检测渲染函数是否存在，即 `vm.$options.render` 是否为真，如果不为真说明渲染函数不存在，这时将会执行 `if` 语句块内的内容，在 `if` 语句块内首先将 `vm.$options.render` 的值设置为 `createEmptyVNode()` 函数，即此时渲染函数的作用将仅仅渲染一个空的 `vnode` 对象，然后在非生产环境下会根据相应的情况打印警告信息。**

**在上方这段 `if` 语句块的下边，执行了 `callHook()` 函数，触发了 `beforeMount` 生命周期钩子：**

```javascript
callHook(vm, 'beforeMount')
```

**在触发了 `beforeMount` 生命周期钩子之后，组件将开始挂载工作，首先是如下这段代码：**

```javascript
let updateComponent
/* istanbul ignore if */
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
  updateComponent = () => {
    const name = vm._name
    const id = vm._uid
    const startTag = `vue-perf-start:${id}`
    const endTag = `vue-perf-end:${id}`

    mark(startTag)
    const vnode = vm._render()
    mark(endTag)
    measure(`vue ${name} render`, startTag, endTag)

    mark(startTag)
    vm._update(vnode, hydrating)
    mark(endTag)
    measure(`vue ${name} patch`, startTag, endTag)
  }
} else {
  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }
}
```

**上方代码的作用只有一个，即定义并初始化 `updateComponent()` 函数，这个函数将用作创建 `Watcher` 实例时传递给 `Watcher` 构造函数的第二个参数，这也将是第一次真正接触 `Watcher` 构造函数，不过需要先把 `updateComponent()` 函数搞清楚，在上方代码中，首先定义了 `updateComponent` 变量，虽然是一个 `if ... else ...` 语句块，其中 `if` 语句块的条件遇到过了很多次了，满足该条件的情况下会做一些性能统计，可以看到在 `if` 语句块中分别统计了 `vm._render()` 函数以及 `vm._update()` 函数的运行性能。也就是说无论是执行 `if` 语句块还是 `else` 语句块，最终 `updateComponent()` 函数的功能是不变的。**

**既然功能相同，直接看 `else` 语句块中的代码，毕竟简洁很多：**

```javascript
let updateComponent
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
  // 省略...
} else {
  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }
}
```

**可以看到，`updateComponent()` 是一个函数，该函数的作用是以 `vm._render()` 函数的返回值作为第一个参数调用 `vm._update()` 函数。对于这两个函数，可以这样简单理解：**

- **`vm._render()`：调用 `vm.$options.render()` 函数并返回生成的虚拟节点 (`vnode`)。**
- **`vm._update()`：把 `vm._render()` 函数生成的虚拟节点渲染成真正的 DOM。**

***现在可以简单理解 `updateComponent()` 函数的作用就是：把渲染函数生成的虚拟 DOM 渲染成真正的 DOM，其实在 `vm._update()` 内部是通过虚拟 DOM 的补丁算法 (`patch`) 来完成，后边再讲述。***

**再往下，将遇到创建观察者 (`Watcher`) 实例的代码：**

```javascript
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

**前边说过，这将是第一次真正意义上的遇到观察者构造函数 `Watcher`，在之前的文章提到过，正是因为 `Watcher` 对表达式的求值，触发了数据属性的 `get()` 拦截器函数，从而收集到了依赖，当数据变化时能够触发响应。在上方的代码中 `Watcher` 观察者实例将对 `updateComponent()` 函数进行求值，`updateComponent()` 函数的执行会间接触发渲染函数 (`vm.$options.render`) 的执行，而渲染函数的执行则会数据属性的 `get()` 拦截器函数，从而将依赖 (观察者) 收集，当数据变化时将重新执行 `updateComponent()` 函数，这就完成了重新渲染。同时我们将上方代码实例化的观察者称为：*渲染函数的观察者。***



## 初识 `Watcher`

**接下来将以渲染函数的观察者对象为例，顺着源码的脉络了解 `Watcher` 类，`Watcher` 类定义在 `src/core/observer/watcher.js` 文件中，如下是 `Watcher` 类的全部：**

```javascript
export default class Watcher {

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {

  }

  get () {
    // 省略...
  }

  addDep (dep: Dep) {
    // 省略...
  }

  cleanupDeps () {
    // 省略...
  }

  update () {
    // 省略...
  }

  run () {
    // 省略...
  }

  getAndInvoke (cb: Function) {
    // 省略...
  }

  evaluate () {
    // 省略...
  }

  depend () {
    // 省略...
  }

  teardown () {
    // 省略...
  }
}
```

**通过 `Watcher` 类的 `constructor` 方法可以知道在创建 `Watcher` 实例时可以传递五个参数，分别是：组件实例对象 `vm`、要观察的表达式 `expOrFn`、当被观察的表达式的值变化时的回调函数 `cb`、一些传递给当前观察者对象的选项 `options` 以及一个不布尔值 `isRenderWatcher` 用来标识该观察者实例是否是渲染函数的观察者。**

**如下是在 `mountComponent()` 函数中创建渲染函数观察者实例的代码：**

```javascript
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

**可以看到在创建渲染函数观察者实例对象时传递了全部的五个参数:**

- **第一个参数 `vm`：显然就是当前组件实例对象。**
- **第二个参数 `updateComponent`：就是被观察的目标，在这里是一个函数。**
- **第三个参数 `noop`：是一个空函数。**
- **第四个参数 `object`：是一个包含 `before()` 函数的对象。**
- **第五个参数 `boolean`：是一个 `true` 的 `boolean` 值。该参数标识该观察者实例是否为渲染函数的观察者。**

**这里有几个问题需要注意，如下：**

- **被观察的表达式是一个函数，即 `updateComponent()` 函数，`Watcher` 的原理是通过对“被观察目标”的求值，触发数据属性的 `get()` 拦截器函数从而收集依赖的，至于“被观察目标”到底是表达式还是函数或者其他形式的内容都不重要，重要的是“被观察目标”能否触发数据属性的 `get()` 拦截器函数，显然函数是具备这个能力。**
- **`Watcher` 构造函数的第三个参数 `noop` 是一个空函数，它什么事情都不做，有的人可能会产生迷惑，“不是说好的当数据发生变化的时候重新渲染吗，现在怎么什么都不做了？”，实际上数据的变化不仅仅会执行回调，还会重新对“被观察对象”求值，也就是说 `updateComponent()` 也会被调用，所以不需要通过执行回调去重新渲染。这里也有一个提醒的地方，再次执行 `updateComponent()`，Vue 会避免收集重复依赖，后边会讲解到。**

**接下来从 `constructor` 开始，看一下创建渲染函数观察者实例对象的过程，进一步了解观察者，如下是 `constructor` 函数的开头一段代码：**

```javascript
this.vm = vm
if (isRenderWatcher) {
  vm._watcher = this
}
vm._watchers.push(this)
```

**首先将当前组件实例对象 `vm` 赋值给该观察者实例的 `this.vm` 属性，也就说每一个观察者实例对象都有一个 `vm` 实例属性，该属性指明了这个观察者是属于哪一个组件的。接着使用 `if` 条件判断 `isRenderWatcher` 是否为真，`isRenderWatcher` 标识着是否是渲染函数的观察者，只有在 `mountComponent()` 函数中创建渲染函数观察者时这个参数为真，如果 `isRenderWatcher` 为真则会将当前观察者实例赋值给 `vm._watcher` 属性，也就说组件实例的 `_watcher` 属性的值引用着该组件的渲染函数观察者。`_watcher` 属性的初始化是在 `initLifecycle()` 函数中进行的，其初始值为 `null`。在 `if` 语句块的后边将当前观察者实例对象 `push` 到 `vm._watchers` 数组中，也就说属于该组件实例的观察者都会被添加到该组件实例对象的 `vm._watchers` 数组中，包括渲染函数的观察者和非渲染函数的观察者。组件实例的 `vm._watchers` 属性是在 `initState()` 函数中初始化的，初始值是一个空数组。**

**再往下是这样一段代码：**

```javascript
if (options) {
  this.deep = !!options.deep
  this.user = !!options.user
  this.computed = !!options.computed
  this.sync = !!options.sync
  this.before = options.before
} else {
  this.deep = this.user = this.computed = this.sync = false
}
```

**这时一个 `if...else...` 语句块，判断是否传递了 `options` 参数，如果没有传递，则 `else` 语句块的代码被执行，可以看到 `else` 语句块中将当前观察者实例对象的四个属性 `this.deep`、`this.user`、`this.computed` 以及 `this.sync` 全部初始化为 `false`。如果传递了 `options` 参数，那么这四个属性的值则会使用 `options` 对象中同名属性值的真假来初始化。通过 `if` 语句块中的代码可以知道在创建一个观察者对象时，可以传递五个选项，分别是：**

- **`options.deep`：用来告诉当前观察者实例对象是否深度观测。平时使用 `watch` 选项或者 `vm.$watch` 函数去观测某个数据时，可以通过 `deep` 选项的值为 `true` 来深度观测该数据。**
- **`options.user`：用来标识当前观察者实例对象是开发者定义还是内部定义的。实际上无论是 Vue 的 `watch` 选项和 `vm.$watch` 函数，它们的实现都是通过实例化 `Watcher` 类来完成的。除了内部定义的观察者 (如：渲染函数的观察者、计算属性的观察者等) 之外，所有观察者都被认为是开发者定义的，这时 `options.user` 会自动被设置为 `true`。**
- **`options.computed`：用来标识当前观察者实例对象是否是计算属性的观察者。这里需要明确的是，计算属性的观察者并不是指一个观察某个计算属性变化的观察者，而是指 Vue 内部在实现这个计算属性功能时，为计算属性创建的观察者，之后详细讲解。**
- **`options.sync`：用来告诉观察者当数据变化时是否同步求值并执行回调。默认情况下，当数据变化时，不会同步求值并执行回调，而是将需要重新求值并执行回调的观察者放到一个异步队列当中，当所有数据的变化结束之后，统一求值并执行回调，这样做的好处有很多，后边细讲。**
- **`options.before`：可以理解为 `Watcher` 的钩子，当数据变化之后，触发更新之前，调用在创建渲染函数的观察者实例对象时传递的 `before` 选项。**

**如下代码：**

```javascript
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

**可以看到当数据变化之后，触发更新之前，如果 `vm._isMounted` 属性值为 `true`，则会调用 `beforeUpdate` 生命周期钩子。**

**再往下又定义了一些实例属性，如下：**

```javascript
this.cb = cb
this.id = ++uid // uid for batching
this.active = true
this.dirty = this.computed // for computed watchers
```

**如上代码所示，定义了 `this.cb` 属性，值为 `cb` 回调函数。定义了 `this.id` 属性，他是观察者实例对象的位移标识。定义了 `this.active` 属性，它标识着该观察者实例对象是否是激活状态，默认值为 `true` 代表激活。定义了 `this.dirty` 属性，该属性的值和 `this.computed` 属性的值相同，也就是说只有计算属性的观察者实例对象的 `this.dirty` 属性值才为 `true`。因为计算属性是惰性求值。**

**继续往下看代码：**

```javascript
this.deps = []
this.newDeps = []
this.depIds = new Set()
this.newDepIds = new Set()
```

**这四个属性两两一组，`this.deps` 和 `this.depIds` 为一组；`this.newDeps` 和 `this.newDepIds` 为一组。这两组属性的作用：*为了避免收集重复依赖，且移除无用依赖。*注意这四个属性的数据结构：`this.deps` 和 `this.newDeps` 被初始化为数组，而 `this.depIds` 和 `this.newDepIds` 则被初始化为 `Set` 实例对象。**

**再往下，代码如下：**

```javascript
this.expression = process.env.NODE_ENV !== 'production'
  ? expOrFn.toString()
  : ''
```

**定义了 `this.expression` 属性，在非生产环境下该属性的值为表达式 (`expOrFn`) 的字符串表示，在生产环境下其值为空字符串，所以 `this.expression` 这个属性是给非生产环境下使用的。**

**再往下，来到了一段 `if...else...` 语句块，代码如下：**

```javascript
if (typeof expOrFn === 'function') {
  this.getter = expOrFn
} else {
  this.getter = parsePath(expOrFn)
  if (!this.getter) {
    this.getter = function () {}
    process.env.NODE_ENV !== 'production' && warn(
      `Failed watching path: "${expOrFn}" ` +
      'Watcher only accepts simple dot-delimited paths. ' +
      'For full control, use a function instead.',
      vm
    )
  }
}
```

**这段代码检测了 `expOrFn` 属性的类型，如果 `expOrFn` 是函数，那么直接使用 `expOrFn` 作为 `this.getter` 属性的值。如果 `expOrFn` 不是函数，那么将 `expOrFn` 传给 `parsePath()` 函数，并以 `parsePath()` 函数的返回值作为 `this.getter` 属性的值。`parsePath()` 函数位于 `src/core/util/lang.js` 文件中，源码如下：**

```javascript
const bailRE = /[^\w.$]/
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```

**首先需要知道 `parsePath()` 函数接收的参数是什么，如下是平时使用 `$watch()` 函数的例子：**

```javascript
// 函数
const expOrFn = function () {
  return this.obj.a
}
this.$watch(expOrFn, function () { /* 回调 */ })

// 表达式
const expOrFn = 'obj.a'
this.$watch(expOrFn, function () { /* 回调 */ })
```

**以上两种用法实际上是等价的，当 `expOrFn` 不是函数时，比如上方例子中的 `obj.a` 是一个字符串，这时就会将字符串传递给 `parsePath()` 函数，其实 `parsePath()` 函数的返回值是另一个函数，返回的函数的作用就是：*触发 `'obj.a'` 的 `get()` 拦截器函数，同时新函数将 `'obj.a'` 的值返回。***

**接着具体看一下 `parsePath()` 函数的具体实现，先看一下在 `parsePath()` 函数定义之前定义的正则：`bailRE`：**

```javascript
const bailRE = /[^\w.$]/
```

**同时在 `parsePath()` 函数开头有一段 `if` 语句，使用该正则来匹配传递给 `parsePath()` 参数 `path`，如果匹配则直接返回 (`return`)，且返回值是 `undefined`，也就是说如果 `path` 匹配正则 `bailRE` 那么最终 `this.getter` 将不是一个函数，而是 `undefined`。这个正则将匹配一个位置，这个位置满足三个条件：**

- **不是 `\w`，也就是说这个位置不能是 `字母` 或 `数字` 或 `下划线`**
- **不是字符 `.`**
- **不是字符 `$`**

**举几个例子如：`obj~a`、`obj/a`、`obj+a` 等，这些字符串中的 `~`、`/`、`+` 字符都能成功匹配到 `bailRE` 正则，这时 `parsePath()` 函数返回 `undefined`，也就是解析失败。实际上这些字符串在 JS 中不是一个合法的访问对象属性的语法，按照 `bailRE` 正则只有如下几种字符串才能解析成功：`obj.a`、`this.$watch` 等。**

**回过头来看，如果参数 `path` 不满足正则 `bailRE`，那么如下代码中 `if` 分支后边的内容就会被执行：**

```javascript
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```

**首先定义了 `segments` 常量，它的值是通过字符 `.` 分割字符串产生的数组，随后 `parsePath()` 函数将返回一个函数，该函数的作用是遍历 `segments` 数组循环访问 `path` 指定的属性值。这样就触发了数据属性的 `get()` 拦截器函数。但是需要注意 `parsePath()` 返回的新函数将作为 `this.getter` 的值，只有当 `this.getter` 被调用的时候，这个函数才会被执行。**

**看完了 `parsePath()` 函数，回到如下代码中：**

```javascript
if (typeof expOrFn === 'function') {
  this.getter = expOrFn
} else {
  this.getter = parsePath(expOrFn)
  if (!this.getter) {
    this.getter = function () {}
    process.env.NODE_ENV !== 'production' && warn(
      `Failed watching path: "${expOrFn}" ` +
      'Watcher only accepts simple dot-delimited paths. ' +
      'For full control, use a function instead.',
      vm
    )
  }
}
```

**从上方的代码可以得知，观察者实例对象的 `this.getter` 函数终将是一个函数，如果不是函数，根据如上代码，只有一种可能，那就是 `parsePath()` 函数解析失败了。这个时候在非生产环境下会打印警告信息，告诉开发者：*`Watcher` 只接受简单的点 (`.`) 分割路径，如果需要用全部的 JS 语法特性，直接观察一个函数即可。***

**再往下就是 `constructor` 函数的最后一段代码：**

```javascript
if (this.computed) {
  this.value = undefined
  this.dep = new Dep()
} else {
  this.value = this.get()
}
```

**通过这段代码可以发现，计算属性的观察者和其他观察者实例对象的处理方式不一样，对于计算属性的观察者之后会详细讲解，除了计算属性的观察者之外的所有观察者实例对象都将执行上方代码的 `else` 分支语句，即调用 `this.get()` 方法。**



## 依赖收集过程

**`this.get()` 是我们遇到的第一个观察者对象的实例方法，它的作用两个字可以描述：*求值。*求值的目的有两个：**

- **触发访问器属性的 `get()` 拦截器函数，能够触发访问器属性的 `get()` 拦截器函数是依赖被收集的关键**
- **能够获取被观察目标的值**

**接下来看看 `this.get()` 方法的源码实现：**

```javascript
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    value = this.getter.call(vm, vm)
  } catch (e) {
    if (this.user) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    } else {
      throw e
    }
  } finally {
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) {
      traverse(value)
    }
    popTarget()
    this.cleanupDeps()
  }
  return value
}
```

**如上是 `this.get()` 方法的全部代码，`this.get()` 方法中一上来就调用了 `pushTarget(this)` 函数，并将当前观察者实例对象作为参数传递，这里的 `pushTarget()` 函数来自于 `src/core/observer/dep.js` 文件，如下代码所示：**

```javascript
export default class Dep {
  // 省略...
}

Dep.target = null
const targetStack = []

export function pushTarget (_target: ?Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

export function popTarget () {
  Dep.target = targetStack.pop()
}
```

**在 `src/core/observer/dep.js` 文件中定义了 `Dep` 类。之前的文章讲述过：每个响应式数据的属性都通过闭包引用着一个用来收集自身依赖的容器，实际上，这个容器就是 `Dep` 类的实例对象。**

**现在着眼来看 `pushTarget()` 函数的作用，在上方这段代码中能够看到 `Dep` 类拥有一个静态属性，即 `Dep.target` 属性，该属性初始值为 `null`，其实 `pushTarget()` 函数的作用就是用来为 `Dep.target` 属性赋值的，`pushTarget()` 函数会将接收到的参数赋值给 `Dep.target` 属性，传递给 `pushTarget()` 函数的参数就是调用该函数的观察者实例对象，所以 `Dep.target` 属性保存着一个观察者实例对象，这个观察者实例对象就是即将要收集的目标。**

**回到 `this.get()` 函数，如下是简化后的代码：**

```javascript
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    value = this.getter.call(vm, vm)
  } catch (e) {
    // 省略...
  } finally {
    // 省略...
  }
  return value
}
```

**在调用了 `pushTarget()` 函数之后，定义了 `value` 变量，该变量的值为 `this.getter()` 函数的返回值。前边提到观察者实例对象的 `this.getter` 属性是一个函数，这个函数的执行意味着对观察目标的求值，并将得到的值赋值给 `value` 变量，而且可以看到 `this.get()` 函数最后将 `value` 返回。**

```javascript
constructor (
  vm: Component,
  expOrFn: string | Function,
  cb: Function,
  options?: ?Object,
  isRenderWatcher?: boolean
) {
  // 省略...
  if (this.computed) {
    this.value = undefined
    this.dep = new Dep()
  } else {
    this.value = this.get()
  }
}
```

**`this.value = this.get()` 这行代码将 `this.get()` 函数的返回值赋值给了观察者实例对象的 `this.value` 属性。也就是 `this.value` 属性保存着被观察目标的值。**

**`this.get()` 函数除了对被观察目标求值之外，还会触发依赖的收集。这里需要强调一下：*正因为对观察目标的求值才得以触发数据属性的 `get()` 拦截器函数。*还是以渲染函数的观察者为例，假设有如下模板：**

```html
<div id="demo">
  <p>{{name}}</p>
</div>
```

**这段模板被编译将生成如下渲染函数：**

```javascript
// 编译生成的渲染函数是一个匿名函数
function anonymous () {
  with (this) {
    return _c('div',
      { attrs:{ "id": "demo" } },
      [_v("\n      "+_s(name)+"\n    ")]
    )
  }
}
```

**上方代码稍微有点晦涩难懂，像女人的心思一般。不过所幸，我们只需要注意这一行代码：`[_v("\n      "+_s(name)+"\n    ")]`。从这行代码，可以发现渲染函数的执行会读取数据属性 `name` 的值，这样就会触发 `name` 属性的 `get()` 拦截器函数，如下代码截取自 `defineReactive()` 函数：**

```javascript
get: function reactiveGetter () {
  const value = getter ? getter.call(obj) : val
  if (Dep.target) {
    dep.depend()
    if (childOb) {
      childOb.dep.depend()
      if (Array.isArray(value)) {
        dependArray(value)
      }
    }
  }
  return value
}
```

**这段代码在之前的章节已经进行过了详细的讲解了，它是数据属性的 `get()` 拦截器函数，由于渲染函数读取了 `name` 属性的值，所以 `name` 属性的 `get()` 拦截器函数将被执行，注意摘自于上方代码的两行代码：**

```javascript
if (Dep.target) {
    dep.depend()
    // ...不重要的代码省略掉啦，免得影响同学们理解
  }
```

**首先判断了 `Dep.target` 是否存在，如果存在则调用 `dep.depend()` 函数收集依赖。这里的 `Dep.target` 是存在值的，因为 `pushTarget()` 会先一步执行。然后现在将我们的注意力转到 `dep.depend()` 函数，源码如下：**

```javascript
depend () {
  if (Dep.target) {
    Dep.target.addDep(this)
  }
}
```

**在 `dep.depend()` 函数内部再次判断 `Dep.target` 属性中是否有值，有的人觉得多重判断会显得有点多余，但这里并不，毕竟大把地方调用这个 `dep.depend()` 函数，不见得每个地方在调用之前都进行了判断。在 `depend()` 函数中并没有真正执行收集依赖的动作，而是调用了观察者实例对象的 `addDep()` 方法：`Dep.target.addDep(this)`，并以当前的 `Dep` 实例对象作为参数。为了了解这样绕着操作的目的，看到 `addDep()` 函数的源码，如下：**

```javascript
addDep (dep: Dep) {
  const id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      dep.addSub(this)
    }
  }
}
```

**上方代码中，`addDep()` 函数接收一个参数，这个参数是一个 `Dep` 对象，在 `addDep()` 函数中，首先定义了常量 `id`，他的值是 `Dep` 实例对象的唯一 `id` 值，然后是一个 `if` 语句块，这个 `if` 语句块中的内容就是用来：*避免收集重复依赖。*既然是避免收集重复依赖的，那就免不了使用 `newDepIds`、`newDeps`、`depIds`、`deps`。**

**为了各位老大更好理解这个骚操作，思考一下 `addDep()` 函数能否改成如下这样：**

```javascript
addDep (dep: Dep) {
  dep.addSub(this)
}
```

**先行科普一下，`addSub()` 函数的源码如下：**

```javascript
addSub (sub: Watcher) {
  this.subs.add(sub)
}
```

**`addSub()` 函数接收观察者实例对象作为参数，并将接收到的观察者实例对象添加到 `Dep` 实例对象的 `subs` 数组中，其实 `addSub` 函数才是真正用来收集观察者的函数，并且收集到的观察者实例对象都会被添加到 `subs` 数组中。**

**科普完之后，回到如下这段代码：**

```javascript
addDep (dep: Dep) {
  dep.addSub(this)
}
```

**上方的这一小段代码其实存在一个问题，下方用一个例子进行讲解：**

```html
<div id="demo">
  {{name}}{{name}}
</div>
```

**这段模板中，两次使用了 `name` 属性，对应的渲染函数变成如下：**

```javascript
function anonymous () {
  with (this) {
    return _c('div',
      { attrs:{ "id": "demo" } },
      [_v("\n      "+_s(name)+_s(name)+"\n    ")]
    )
  }
}
```

**从上方的渲染函数可知，其执行将读取两次数据对象 `name` 属性的值，这必然触发两次 `name` 属性的 `get()` 拦截器函数，同样的道理，`dep.depend()` 函数将被触发两次，最后导致 `dep.addSub()` 函数被执行两次，且参数一毛一样，产生出一个问题：*同一个观察者实例对象被收集多次。*所以不能够像如上那样修改 `addDep()` 函数的代码，所以下边的这段代码的含义应该有思路理解了叭：**

```javascript
addDep (dep: Dep) {
  const id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      dep.addSub(this)
    }
  }
}
```

**`newDepIds` 属性用来避免在一次求值的过程中收集重复的依赖，而 `depIds` 属性是用来在多次求值中避免收集重复的依赖。所谓的多次求值就是指当数据变化时重新求值的过程。这里需要注意：*重新求值的时候不可以用 `newDepIds` 属性来避免收集重复的依赖，原因在于每一次求值之后，`newDepIds` 属性都会被清空，也就是说每次重新求值的时候对于观察者实例对象来说 `newDepIds` 属性始终都是全新的。虽然每次求值之后会清空 `newDepIds` 属性的值，但在清空之前会把 `newDepIds` 属性的值以及 `newDeps` 属性的值赋值给 `depIds` 属性和 `deps` 属性，这样重新求值的时候 `depIds` 属性和 `dep` 属性将会保存着上一次求值中 `newDepIds` 属性以及 `newDeps` 属性的值。为了证明这一点，来看一下观察者对象的求值方法，即 `get()` 函数：***

```javascript
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    value = this.getter.call(vm, vm)
  } catch (e) {
    // 省略...
  } finally {
    // 省略...
    popTarget()
    this.cleanupDeps()
  }
  return value
}
```

**从上方代码中可以看到，在 `finally` 语句块中调用了观察者对象的 `cleanupDeps()` 函数，这个函数的作用就像之前所说那样，每次求值完毕之后都会使用 `depIds` 属性和 `deps` 属性保存 `newDepIds` 属性和 `newDeps` 属性的值，然你再清空 `newDepIds` 属性和 `newDeps` 属性的值，如下是 `cleanupDeps()` 函数的源码：**

```javascript
cleanupDeps () {
  let i = this.deps.length
  while (i--) {
    const dep = this.deps[i]
    if (!this.newDepIds.has(dep.id)) {
      dep.removeSub(this)
    }
  }
  let tmp = this.depIds
  this.depIds = this.newDepIds
  this.newDepIds = tmp
  this.newDepIds.clear()
  tmp = this.deps
  this.deps = this.newDeps
  this.newDeps = tmp
  this.newDeps.length = 0
}
```

**在 `cleanupDeps()` 函数中，首先是一个 `while` 循环，`while` 循环下边是一段代码，先聚焦这一段代码。这段代码是典型的引用类型变量交换值的过程，最终的结果就是 `newDepIds` 属性和 `newDeps` 属性被清空，并且在清空之前将值分别赋给了 `deps` 属性和 `depIds` 属性，这两个属性将会在下一次求值时避免依赖的重复收集。**

**总结：**

- **`newDepIds` 属性用来在一次求值中避免收集重复的观察者**
- **每次求值并收集观察者完成之后会清空 `newDepIds` 和 `newDeps` 这两个属性的值，并且在清空之前会把值分别赋给 `depIds` 和 `deps` 这两个属性**
- **`depIds` 属性用来避免重复求值时收集重复的观察者**

**`newDepIds` 和 `newDeps` 这两个属性的值所存储的总是当前求值所收集到的 `Dep` 实例对象，而 `depIds` 和 `deps` 这两个属性的值所存储的总是上一次求值过程中所收集到的 `Dep` 实例对象。**

**除了上述三点之外，其实 `deps` 属性还能够用来移除废除的观察者，`cleanupDeps()` 函数中开头的那段 `while` 循环实现的就是这个功能，如下代码所示：**

```javascript
cleanupDeps () {
  let i = this.deps.length
  while (i--) {
    const dep = this.deps[i]
    if (!this.newDepIds.has(dep.id)) {
      dep.removeSub(this)
    }
  }
  // 省略...
}
```

**这段 `while` 循环就是对 `deps` 数组进行循环遍历，也就是对上一次求值所收集到的 `Dep` 对象进行遍历，然后在循环内部检查上一次求值所收集到的 `Dep` 实例对象是否存在于当前这次求值所收集到的 `Dep` 实例对象中，如果不存在则说明该 `Dep` 实例对象已经和该观察者不存在依赖关系了，这时就会调用 `dep.removeSub(this)` 函数并以该观察者实例对象作为参数传递，从而将该观察者对象从 `Dep` 实例对象中移除。**

**`Dep` 类中的 `removeSub()` 函数如下：**

```javascript
removeSub (sub: Watcher) {
  remove(this.subs, sub)
}
```

**上方代码中的内容很简单，接收一个要被移除的观察者作为参数，然后使用 `remove()` 工具函数，将该观察者从 `this.subs` 数组中移除。其中 `remove()` 函数来自于 `src/shared/utils.js` 文件中。**



## 触发依赖的过程

**上一小节中提到，每次求值并收集完观察者之后，会将当前求值收集到的观察者保存到另外的一组属性中，即 `deps` 和 `depIds` 这两个属性中，并将当前求值所收集到的观察者的属性清空，即清空 `newDeps` 和 `newDepIds` 这两个属性。这样做的目的其实就是为了对比当次求值和上一次求值所收集到的观察者的变化情况，并做出相对应的纠正工作，比如移除那些已经没有关联关系的观察者等。本小节将以数据属性的变化为切入点，讲解重新求值的过程。**

**假设有如下模板：**

```html
<div id="demo">
  {{name}}
</div>
```

**上方的这段模板会被编译成渲染函数，接着创建一个渲染函数的观察者，从而对渲染函数求值，在求值过程中会触发数据对象 `name` 属性的 `get()` 拦截器函数，进而将该观察者者收集到 `name` 属性中通过闭包引用的容器中，即 `dep` (`Dep` 实例对象) 中。这个 `dep` 属于 `name` 属性自身所拥有的，这样尝试修改数据对象 `name` 属性的值时就会触发 `name` 属性的 `set()` 拦截器函数，这样就有机会调用 `Dep` 实例对象的 `notify()` 函数，从而触发了响应，如下代码摘自 `defineReactive()` 函数中的 `set()` 拦截器函数：**

```javascript
set: function reactiveSetter (newVal) {
  // 省略...
  dep.notify()
}
```

**如上代码所示，当属性值发生变化时确实被 `set()` 拦截器函数拦截了，并且调用了 `Dep` 实例对象的 `notify()` 函数，这个函数就是用来通知变化的，查看 `Dep` 类的 `notify()` 函数，如下：**

```javascript
export default class Dep {
  // 省略...

  constructor () {
    this.id = uid++
    this.subs = []
  }

  // 省略...

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```

**对上方的代码进行简化一下，`notify()` 函数将变成：**

```javascript
notify () {
	// stabilize the subscriber list first
  const subs = this.subs.slice()
  for (let i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```

**`notify()` 函数只做了一件事：遍历当前 `Dep` 实例对象的 `subs` 属性只能够所保存的的所有观察者，并逐一调用观察者实例对象的 `update()` 方法，这就是触发响应的实现机制，由此可以猜想出来，重新求值的操作应该是在 `update()` 方法中进行的，找到 `update()` 方法的源码如下：**

```javascript
update () {
  /* istanbul ignore else */
  if (this.computed) {
    // 省略...
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

**在 `update()` 方法中，拆分成了三个部分：**

- **`if (this.computed)` 语句块：如果该观察者实例对象是计算属性的观察者实例对象，将执行其内部代码，此部分将在讲解计算属性时进行讲解**

- **`else if (this.sync)` 语句块：`this.sync` 表示当变化发生时，是否同步更新变心，对于渲染函数的观察者实例对象，并不是同步更新变化的，而是将变化放到一个异步更新队列中。**

- **`else` 语句块：`queueWatcher()` 会将当前观察者实例对象放到一个异步更新队列中，这个队列会在调用栈被清空之后按照一定的顺序执行。关于更多的异步更新队列的内容，后续会讲解。这里需要明确一个知识即可：*无论是同步更新变化还是将更新变化的操作放到异步更新队列，真正的更新变化操作都是通过调用观察者实例对象的 `run()` 函数实现的。所以此时将目光转向 `run()` 函数，如下：***

  ```javascript
  run () {
    if (this.active) {
      this.getAndInvoke(this.cb)
    }
  }
  ```

  **`run()` 函数的代码相当简短：*判断了当前观察者实例对象的 `this.active` 属性是否为真，其中 `this.active` 属性用来标识一个观察者实例对象是否处于激活状态，或者可用状态。如果观察者处于激活状态，那么 `this.active` 的值为 `true`。此时会调用观察者实例对象的 `getAndInvoke()` 函数，并以 `this.cb` 作为参数，`this.cb` 属性是一个函数，也就是传说中的回调函数，当变化发生时会触发，但是对于渲染函数的观察者来说，`this.cb` 属性为 `noop`，即什么都不做。***

**所以可以得知，`getAndInvoke()` 函数是更新变化的根源，源码如下：**

```javascript
getAndInvoke (cb: Function) {
  const value = this.get()
  if (
    value !== this.value ||
    // Deep watchers and watchers on Object/Arrays should fire even
    // when the value is the same, because the value may
    // have mutated.
    isObject(value) ||
    this.deep
  ) {
    // set new value
    const oldValue = this.value
    this.value = value
    this.dirty = false
    if (this.user) {
      try {
        cb.call(this.vm, value, oldValue)
      } catch (e) {
        handleError(e, this.vm, `callback for watcher "${this.expression}"`)
      }
    } else {
      cb.call(this.vm, value, oldValue)
    }
  }
}
```

**在 `getAndInvoke()` 函数中，第一行代码中就调用了 `this.get()` 函数，这意味着重新求值。对于渲染函数的观察者来讲，重新求值相当于就是重新执行渲染函数，最终结果就是重新生成了虚拟 DOM 并更新真实 DOM，这样就完成了整个重新渲染的过程。在重新调用 `this.get()` 函数之后是一个 `if` 语句块，实际上对于渲染函数的观察者来讲，并不会执行这个 `if` 语句块，因为 `this.get()` 函数的返回值其实就等价于 `updateComponent()` 函数的返回值，这个值永远都是 `undefined`。实际上 `if` 语句块中的代码是为了非渲染函数类型的观察者所准备的，它是用来对比新旧两次求值的结果，当值不相等的时候会调用参数传递过来的回调。先看一下判断条件，如下：**

```javascript
const value = this.get()
if (
  value !== this.value ||
  // Deep watchers and watchers on Object/Arrays should fire even
  // when the value is the same, because the value may
  // have mutated.
  isObject(value) ||
  this.deep
) {
  // 省略...
}
```

**首先对比新值 `value` 和旧值 `this.value` 是否相等，只有在不相等的情况下才需要执行回调，但是两个值相等，也不代表不需要执行回调，这个时候需要检测第二个条件是否成立，即 `isObject(value)`，判断新值是否是对象，如果是对象，几时值不变，也需要执行回调，注意：*此处的值“不变”，指代的是引用不变！！！*如下代码所示：**

```javascript
const data = {
  obj: {
    a: 1
  }
}
const obj1 = data.obj
data.obj.a = 2
const obj2 = data.obj

console.log(obj1 === obj2) // true
```

**上方代码中，由于 `obj1` 和 `obj2` 具有相同的引用，所以他们总是相等，但是其实数据已经发生变化了，这就是判断 `isObject(value)` 为 `true` 则执行回调的原因。**

**接下来看 `if` 语句块中的代码：**

```javascript
const oldValue = this.value
this.value = value
this.dirty = false
if (this.user) {
  try {
    cb.call(this.vm, value, oldValue)
  } catch (e) {
    handleError(e, this.vm, `callback for watcher "${this.expression}"`)
  }
} else {
  cb.call(this.vm, value, oldValue)
}
```

**代码如果执行到了 `if` 语句块中，说明应该执行观察者的回调函数了。首先定义了 `oldValue` 常量，它的值是旧值，紧接着使用新值更新 `this.value` 的值。可以看到如上代码是这样执行回调的，如下：**

```javascript
cb.call(this.vm, value, oldValue)
```

**将回调函数的作用域修改为当前 Vue 组件对象，然后传递了两个参数，分别是新值和旧值。**

**另外还有一行代码需要注意：`this.dirty = false`，将观察者实例对象的 `this.dirty` 属性设置为 `false`，实际上 `this.dirty` 属性也是为计算属性准备的，由于计算属性时惰性求值的，所以在实例化计算属性的时候，`this.dirty` 属性的值会被设置为 `true`，代表还没有求值，后面当真正对计算属性进行求值时，也就是执行如上代码时才会将 `this.dirty` 设置为 `false`，代表已经求过值了。**

**除此之外，注意如下代码：**

```javascript
if (this.user) {
  try {
    cb.call(this.vm, value, oldValue)
  } catch (e) {
    handleError(e, this.vm, `callback for watcher "${this.expression}"`)
  }
} else {
  cb.call(this.vm, value, oldValue)
}
```

**在调用回调函数的时候，如果观察者实例对象的 `this.user` 为 `true` 意味着这个观察者实例对象是开发者定义的，所谓开发者定义就是指通过 `watch` 选项或者 `$watch()` 函数定义的观察者，这些观察者的特点就是回调函数由开发者编写的，所以这些回调函数在执行的过程中其行为是不可预见的，所以很有可能会出现错误，这时将其放在一个 `try...catch` 语句块中，这样当错误发生时，就能够给开发者一个友好的提示。并且提示信息中包含了 `this.expression` 属性，前边说过该属性是被观察目标 (`expOrFn`) 的字符串表示，这样开发者就能够清楚哪里发生了错误了。**



## 异步更新队列

### 异步更新的意义

**接着讨论一下 Vue 中的异步更新队列，上一节中讲解了触发依赖的过程，举例说明：**

```vue
<div id="app">
  <p>{{name}}</p>
</div>

<script>
  new Vue({
    el: '#app',
    data: {
      name: ''
    },
    mounted () {
      this.name = 'hcy'
    }
  })
</script>
```

**如上代码所示，在模板中使用了数据对象中的 `name` 属性，意味着 `name` 属性将会收集渲染函数的观察者作为依赖，接着在 `mounted()` 钩子中修改了 `name` 属性的值，这样就会触发响应：*渲染函数的观察者会重新求值，完成重渲染。***

**如果采用 *同步更新* 会导致每次属性值的变化都会引发一次重新渲染，假设需要修改两个属性值，那么同步更新就会导致两次重渲染。**

**这其实是一个致命缺陷，如果有一个复杂的业务场景，可能会同时修改多个属性值，如果每次属性值的变化都要重新渲染，就会导致出现严重的性能问题，而异步更新队列就是来解决这个问题的。**

**异步更新和同步更新的不同之处在于，每次修改属性的值之后并没有立即重新求值，而是将需要执行更新操作的观察者放到一个队列当中，当我们修改了 `name` 属性值时，由于 `name` 属性收集了渲染函数的观察者实例对象 (后面简称为 `renderWatcher`) 作为依赖，所以此时 `renderWatcher` 会被添加到队列当中，紧接着修改了 `age` 属性的值，由于 `renderWatcher` 已经存在于队列中，所以并不会重复添加，这样队列中将只会存在一个 `renderWatcher`。当所有的突变完成之后，再一次性执行队列中所有观察者的更新函数，同时清空队列，这样就达到了优化的目的。**

**阐述完简单的思路之后，紧接着将会从源码出发看一下具体实现，当修改了一个属性的值时，会通过执行该属性所收集的所有观察者实例对象的 `update()` 函数进行更新，`update()` 函数源码如下：**

```javascript
update () {
  /* istanbul ignore else */
  if (this.computed) {
    // 省略...
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

**如果没有指定观察者是同步更新 (`this.sync` 为 `false`)，这个观察者的更新就是异步的，这时调用观察者实例对象的 `update()` 函数，在 `update()` 函数中就会调用 `queueWatcher()` 函数，并将当前观察者实例对象作为参数传递，`queueWatcher()` 函数的作用就是将观察者实例对象放到一个队列中，等所有的突变都完成了，统一执行更新操作。**

**`queueWatcher()` 函数来自 `src/core/observer/scheduler.js` 文件，如下是 `queueWatcher()` 函数的全部代码：**

```javascript
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

**`queueWatcher()` 函数接收观察者实例对象作为参数，首先定义了 `id` 常量，其值是观察者实例对象的唯一 `id`，然后执行 `if` 判断语句，如下是简化版代码：**

```javascript
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    // 省略...
  }
}
```

**其中 `has` 变量定义在 `scheduler.js` 文件头部，是一个空对象：**

```javascript
let has: { [key: number]: ?true } = {}
```

**当 `queueWatcher()` 函数被调用之后，会尝试将该观察者实例对象放到队列中，并将该观察者的 `id` 值登记到 `has` 对象上作为 `has` 对象的属性同时将该属性值设置为 `true`。该 `if` 语句以及 `has` 变量的作用就是用来避免将相同的观察者实例对象重复放进队列中，在 `if` 语句块中执行了真正的添加到队列的操作，如下方代码所示：**

```javascript
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // 省略...
    }
    // 省略...
  }
}
```

**`queue.push(watcher)` 就是将观察者实例对象添加到队列的真正操作。其中 `queue` 常量也定义在 `scheduler.js` 文件的头部：**

```javascript
const queue: Array<Watcher> = []
```

**`queue` 常量是一个数组，添加到队列就直接调用该数组的 `push()` 函数将观察者实例对象添加到数组的尾部。在添加到队列之前有一个 `flushing` 变量的判断，`flushing` 变量的定义也在 `scheduler.js` 文件的头部，初始值是 `false`：**

```javascript
let flushing = false
```

**`flushing` 变量是一个标志，放进队列 `queue` 中的所有观察者将会在突变完成之后统一执行更新，当更新开始时会将 `flushing` 变量的值设置为 `true`，代表此时正在执行更新，所以根据判断条件 `if (!flushing)` 可知只有当队列没有执行更新时才会简单地将观察者实例对象追加到队列的尾部，实际上，在队列执行更新的时候还会有观察者进入队列的操作，最典型就是计算属性，比如队列执行更新时常常会执行渲染函数观察者实例对象的更新，渲染函数中很可能有计算属性的存在，由于计算属性在实现方式上和普通响应式属性有所不同，所以当触发计算属性的 `get()` 拦截器函数时会有观察者入队的行为，这个时候就需要特殊处理，也就是 `else` 分支的代码，如下：**

```javascript
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // 省略...
  }
}
```

**如上代码所示，当变量 `flushing` 为 `true` 时，说明队列正在执行更新，这时如果有观察者实例对象进入队列则会执行 `else` 分支中的代码，这段代码的作用是为了保证观察者实例对象的执行顺序，现在只需要知道观察者实例对象会被放进 `queue` 队列即可，其他的后边再讨论。**

**接着看如下代码：**

```javascript
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    // 省略...
    // queue the flush
    if (!waiting) {
      waiting = true
      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

**如上代码中有这样一段 `if` 条件语句：**

```javascript
if (process.env.NODE_ENV !== 'production' && !config.async) {
  flushSchedulerQueue()
  return
}
```

**下边的讲解会先忽略掉这段代码，之后的讲解再进行补充。**

**回到上方的代码，这段代码是一个 `if` 语句块，其中变量 `waiting` 变量同样是一个标志，它也是定义在 `scheduler.js` 文件的头部，初始值为 `false`：**

```javascript
let waiting = false
```

**在 `if` 语句块中先将 `waiting` 变量的值设置为 `true`，这意味着无论调用多少次 `queueWatcher()` 函数，该 `if` 语句块中的代码都只会执行一次。紧接着调用 `nextTick()` 并以 `flushSchedulerQueue` 函数作为参数，其中 `flushSchedulerQueue` 函数的作用之一就是用来将队列中的观察者实例对象统一执行更新的。对于 `nextTick()` 函数最好的理解方式，就是把它当做 `setTimeout(fn, 0)`，如下：**

```javascript
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    // 省略...
    // queue the flush
    if (!waiting) {
      waiting = true
      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      setTimeout(flushSchedulerQueue, 0)
    }
  }
}
```

**在这里完全可以使用 `setTimeout` 来替换 `nextTick`，只需要执行一次 `setTimeout` 语句即可，而在这里 `waiting` 变量就保证了 `setTimeout` 语句只会执行一次，这样 `flushSchedulerQueue` 函数将会在下一次事件循环开始时立即调用，但是这里依然不使用 `setTimeout` 代替 `nextTick`，因为 `setTimeout` 不是一个最优选择，`nextTick` 的意义在于他会选择一条最优的解决方案，紧接着会讨论 `nextTick` 如何实现。**

**提醒：*根据上边的代码，这里提出一个需要注意的点。当 `setTimeout` 或者 `nextTick` 执行的之前，会将 `waiting` 置为 `true` 保证 `setTimeout` 或者 `nextTick` 只执行一次，当 `flushSchedulerQueue` 执行的时候会将 `flushing` 置为 `true`。这就是两个变量的不同之处。***



### `nextTick` 的实现

**`nextTick()` 函数来自于 `src/core/util/next-tick.js` 文件，`$nextTick()` 函数实际上就是对 `nextTick()` 函数的封装，如下：**

```javascript
export function renderMixin (Vue: Class<Component>) {
  // 省略...
  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }
  // 省略...
}
```

**`$nextTick()` 函数是在 `renderMixin()` 函数中挂载到 Vue 原型上的，可以看到 `$nextTick()` 函数体中只有一行调用 `nextTick()` 的代码，这说明 `$nextTick()` 确实是对 `nextTick()` 函数的简单包装。**

**前边提到过，`nextTick()` 函数的作用相当于 `setTimeout(fn, 0)`，这里有几个概念需要了解一下，即调用栈、任务队列、事件循环，JS 是一种单线程的语言，一切都建立在这三个概念为基础之上。**

**任务队列中并非只有一个队列，任务队列中可以分为 `microtask` 和 `(macro)task`，并且这两个队列的行为还要依据不同的浏览器的具体实现来进行讨论。这里只讨论被广泛认同和接受的队列执行行为。当调用栈空闲后每次事件循环只会从 `(macro)task` 中读取一个任务并执行，而在同一次事件循环中，会将 `microtask` 队列中所有的任务全部执行完毕，且要先于 `(macro)task`。另外 `(macro)task` 中两个不同的任务之间可能会穿插着 UI 的重渲染，那么只需要在 `microtask` 中把所有在 UI 重渲染之前需要更新的数据全部更新，这样只需要一次重渲染就能够得到最新的 DOM 了。恰好 Vue 是一个数据驱动的框架，如果能在 UI 重渲染之前更新所有数据状态，这对性能的提升是一个很大的帮助，所以要优先使用 `microtask` 去更新数据状态而不是 `(macro)task`，这就是不使用 `setTimeout` 的原因，因为 `setTimeout` 会将回调放到 `(macro)task` 队列中，而不是 `microtask` 队列，所以理论上最优的选择是使用 `Promise`，当浏览器不支持 `Promise` 时再降级为 `setTimeout`，如下是 `next-tick.js` 文件中的一段代码：**

```javascript
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    // in problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
} else {
  // fallback to macro
  microTimerFunc = macroTimerFunc
}
```

**其中变量 `microTimeFunc` 定义在文件头部，初始值为 `undefined`，上方代码中首先检测当前宿主环境是否支持原生的 `Promise`，如果支持则优先使用 `Promise` 注册 `microtask`，做法很简单，首先定义一个常量 `p`，它的值是一个立即 `resolve` 的 `Promise` 实例对象，接着将变量 `microTimeFunc` 定义为一个箭头函数，这个箭头函数的执行将会把 `flushCallbacks` 函数注册为 `microtask`。另外注意这行代码：**

```javascript
if (isIOS) setTimeout(noop)
```

**注释上讲述：*这是一个解决怪异问题的变通方法，在一些 `UIWebViews` 中存在很奇怪的问题，即 `microtask` 没有被刷新，对于这个问题的解决方案就是让浏览器做一些其他的事情，比如注册一个 `(macro)task` 即使这个 `(macro)task` 什么都不做，这样就能够简洁触发 `microtask` 的刷新。***

**使用 `Promise` 是最理想的方案，但是如果宿主环境不支持 `Promise`，就需要降级处理，即注册 `(macro)task`，这就是 `else` 语句块中的所做的事情：**

```javascript
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  // 省略...
} else {
  // fallback to macro
  microTimerFunc = macroTimerFunc
}
```

**将 `macroTimerFunc` 的值赋值给 `microTimerFunc`。`microTimerFunc` 用来将 `flushCallbacks` 函数注册成 `microtask`，而 `macroTimerFunc` 则是用来将 `flushCallbacks` 函数注册成 `(macro)task` 的，来看如下代码：**

```javascript
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  /* istanbul ignore next */
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

**将一个回调函数注册成 `(macro)task` 的方式有很多，如：`setTimeout`、`setInterval` 以及 `setImmediate` 等，但不同方案之间有区别，通过上方的代码可以看到 `setTimeout` 被作为最后的备选方案，如下：**

```javascript
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // 省略...
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  // 省略...
} else {
  /* istanbul ignore next */
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

**而首选方案是 `setImmediate`：**

```javascript
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  // 省略...
} else {
  // 省略...
}
```

**如果宿主环境支持原生的 `setImmediate` 函数，则使用 `setImmediate` 注册 `(macro)task`，之所以首选 `setImmediate` 是因为 `setImmediate` 拥有比 `setTimeout` 更好的性能。`setTimeout` 在将回调注册为 `(macro)task` 之前要不停地做超时检测，而 `setImmediate` 则不需要。但是 `setImmediate` 的缺陷也很明显，就是他的兼容性问题，到目前为止只有 IE 浏览器实现了它，所以为了兼容非 IE 浏览器，需要做兼容处理，不过此时还没轮到 `setTimeout` 登场，而是使用 `MessageChannel`：**

```javascript
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // 省略...
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  // 省略...
}
```

**这里对 `MessageChannel` 进行科普：一个 `MessageChannel` 实例对象拥有两个属性 `port1` 和 `port2`，只需要让其中一个 `port` 监听 `onmessage` 事件，然后使用另外一个 `port` 的 `postMessage` 向前一个 `port` 发送消息即可，这样一个 `port` 的 `onmessage` 回调就会被注册为 `(macro)task`，由于它不需要做任何检测工作，所以性能也会优于 `setTimeout`。总而言之，`macroTimerFunc` 函数的作用就是将 `flushCallbacks` 注册为 `(macro)task`。**

**现在对 `nextTick()` 进行一下剖析，为了更融入理解 `nextTick()` 函数，需要从 `$nextTick()` 函数入手，如下：**

```javascript
export function renderMixin (Vue: Class<Component>) {
  // 省略...
  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }
  // 省略...
}
```

**`$nextTick()` 函数只接收一个回调函数作为参数，但在内部调用 `nextTick()` 函数的时候，除了把回调函数 `fn` 传进去之外，第二个参数是硬编码为当前组件实例对象 `this`。使用 `$nextTick()` 函数时是可以省略回调函数这个参数，这时 `$nextTick()` 函数会返回一个 `Promise` 实例对象。这些功能实际上都是由 `nextTick()` 函数提供的，如下是 `nextTick()` 函数的签名：**

```javascript
export function nextTick (cb?: Function, ctx?: Object) {
  // 省略...
}
```

**`nextTick()` 函数接收两个参数，第一个参数是一个回调函数，第二个参数指定一个作用域。接下来逐个分析传递回调函数和不传回调函数这两种使用场景的场景，先看传递回调函数的情况，此时参数 `cb` 就是回调函数，如下代码：**

```javascript
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  // 省略
}
```

**`nextTick()` 函数会在 `callbacks` 数组中添加一个新的函数，`callbacks` 数组定义在文件头部：`const callbacks = []`。这里需要注意：*并不是将 `cb` 回调函数直接添加到 `callbacks` 数组中，但这个被添加到 `callbacks` 数组中的函数的执行会间接调用 `cb` 回调函数，并且可以看到在调用 `cb` 函数时使用 `.call()` 方法将函数 `cb` 的作用域设置为 `ctx`，也就是 `nextTick()` 函数的第二个参数。所以对于 `$nextTick()` 函数来讲，传递给 `$nextTick()` 函数的回调函数的作用域就是当前组件实例对象，前提是回调函数不能是箭头函数，其实在平时使用时，回调函数使用箭头函数也没关系，只要能够达到目的即可。*这里需要强调：*此时回调函数并没有执行，当调用 `$nextTick()` 函数并传递回调函数时，会使用一个新的函数包裹回调函数并将新函数添加到 `callbacks` 数组中。***

**继续看 `nextTick()` 函数的代码，如下：**

```javascript
export function nextTick (cb?: Function, ctx?: Object) {
  // 省略...
  if (!pending) {
    pending = true
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // 省略...
}
```

**在将回调函数添加到 `callbacks` 数组中，会进行一个 `if` 条件判断，判断变量 `pending` 的真假，`pending` 变量也定义在文件头部：`let pending = false`，它是一个标识，代表回调队列是否处于等待刷新的状态，初始值是 `false` 代表回调队列为空不需要等待刷新，加入此时在某个地方调用了 `$nextTick()` 函数，那么 `if` 语句块中的代码将会被执行，在 `if` 语句块中优先将变量 `pending` 的值设置为 `true`，代表此时回调队列不为空，正在等待刷新。这个时候刷新队列就用到了前边所讲的 `microTimerFunc` 或者 `macroTimerFunc` 函数，这两个函数的作用就是将 `flushCallbacks` 函数分别注册为 `microtask` 和 `(macro)task`。但是无论哪种任务类型，他们都会等待调用栈清空之后才执行。如下：**

```javascript
created () {
  this.$nextTick(() => { console.log(1) })
  this.$nextTick(() => { console.log(2) })
  this.$nextTick(() => { console.log(3) })
}
```

**上方代码中，在 `create()` 钩子中连续调用了三次 `$nextTick()` 函数，但只有第一次调用 `$nextTick()` 函数时才会执行 `microTimerFunc` 函数将 `flushCallbacks` 注册为 `microtask`，但此时 `flushCallbacks` 函数并不会执行，因为它需要等待接下来的两次 `$nextTick()` 函数的调用语句执行完之后才会执行，或者准确的说，是等待调用栈被清空之后才会执行。也就说，当 `flushCallbacks` 函数执行的时候，`callbacks` 回调队列中将包含本次事件循环中通过 `$nextTick()` 函数注册的回调函数，而接下来的任务就是在 `flushCallbacks` 函数中将这些回调全部执行并清空。如下是 `flushCallbacks` 函数的源码：**

```javascript
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```

**上方 `flushCallbacks` 的源码很好理解，首先将变量 `pending` 重置为 `false`，接着开始执行回调，但需要注意的是在执行 `callbacks` 队列的回调函数时没有直接遍历 `callbacks` 数组，而是使用 `copies` 常量保存一份 `callbacks` 的复制，然后遍历 `copies` 数组，并且在遍历 `copies` 数组之前将 `callbacks` 数组清空：`callbacks.length = 0`。这样做的原因将在下边这个例子中阐明：**

```javascript
created () {
  this.name = 'HcySunYang'
  this.$nextTick(() => {
    this.name = 'hcy'
    this.$nextTick(() => { console.log('第二个 $nextTick') })
  })
}
```

**上方代码中在外层的 `$nextTick()` 函数中再次调用了 `$nextTick()` 函数，理论上外层的 `nextTick()` 函数中的回调函数不应该跟内层的 `$nextTick()` 函数中的回调函数在同一个 `microtask` 任务中被执行，而是两个不同的 `microtask` 任务，虽然在结果上看或许没有什么差别，但是从设计角度上就应该这样做。**

**上方代码中修改了两次 `name` 属性的值 (假设他是响应式数据)，首先将 `name` 属性的值改为字符串 `HcySunYang`，前边讲述过，这样会导致依赖于 `name` 属性的渲染函数观察者被添加到 `queue` 队列中，这个过程是通过调用 `src/core/observer/scheduler.js` 文件中的 `queueWatcher()` 函数完成的。同时在 `queueWatcher()` 函数中会使用 `nextTick()` 函数将 `flushSchedulerQueue` 函数添加到 `callbacks` 数组中，所以此时 `callbacks` 数组如下：**

```javascript
callbacks = [
  flushSchedulerQueue // queue = [renderWatcher]
]
```

**同时会将 `flushCallbacks` 函数注册成 `microtask`，所以此时 `microtask` 队列如下：**

```javascript
// microtask 队列
[
  flushCallbacks
]
```

**接着会调用第一个 `$nextTick()` 函数，`$nextTick()` 函数会将其回调函数添加到 `callbacks` 数组中，此时的 `callbacks` 数组将如下：**

```javascript
callbacks = [
  flushSchedulerQueue, // queue = [renderWatcher]
  () => {
    this.name = 'hcy'
    this.$nextTick(() => { console.log('第二个 $nextTick') })
  }
]
```

**接下来主线程处于空闲状态 (调用栈清空)，开始执行 `microtask` 队列中的任务，即执行 `flushCallbacks()` 函数，`flushCallbacks()` 函数会按照顺序执行 `callbacks` 数组中的函数，首先会执行 `flushSchedulerQueue()` 函数，这个函数会遍历 `queue` 中的所有观察者并重新求值，完成重新渲染 (`re-render`)，在完成渲染之后，本次更新队列已经清空，`queue` 会被重置为空数组，一切状态还原。接着还会执行如下函数：**

```javascript
() => {
  this.name = 'hcy'
  this.$nextTick(() => { console.log('第二个 $nextTick') })
}
```

**这个函数是第一个 `$nextTick()` 函数中的回调函数，由于在执行该回调函数之前已经完成了重新渲染，所以该回调函数内的代码是能够访问更新后的 DOM 的，到目前为止一切都很正常，继续往下看，在该回调函数中再次修改了 `name` 属性的值为字符串 `hcy`，这会再次触发响应，同样的会调用 `nextTick()` 函数将 `flushSchedulerQueue` 添加到 `callbacks` 数组中，但是由于在执行中的 `flushCallbacks()` 函数将 `pending` 变量置为了 `false`，所以 `nextTick()` 函数会将 `flushCallbacks()` 函数注册为一个新的 `microtask`，此时 `microtask` 队列将包含两个 `flushCallbacks` 函数：**

```javascript
// microtask 队列
[
  flushCallbacks, // 第一个 flushCallbacks
  flushCallbacks  // 第二个 flushCallbacks
]
```

**现在目的达到了，拥有了两个 `microtask` 任务。**

**关于 `flushCallbacks()` 函数需要了解：*除了将 `pending` 的值重置为 `false` 之外，`flushCallbacks()` 函数遍历的不是 `callbacks` 数组本身，而是它的复制品 `copies` 数组，并且 `flushCallbacks()` 函数一开头就把 `callbacks` 数组清空了。所以上边的代码中，第一个 `flushCallbacks()` 和第二个流程上是一样的。***

**接着讲述，当调用 `$nextTick()` 函数不传递回调函数时，是如何实现返回 `Promise` 实例对象的，实现在 `nextTick()` 函数的源码，如下：**

```javascript
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 省略...
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

**如上代码所示，当 `nextTick()` 函数没有接收到 `cb` 参数时，会检测当前宿主环境是否支持 `Promise`，如果支持则直接返回一个 `Promise` 实例对象，并且将 `resolve` 函数赋值给 `_resolve` 变量，`_resolve` 变量声明在 `nextTick()` 函数的顶部。同时再来看如下代码：**

```javascript
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  // 省略...
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

**当 `flushCallbacks` 函数开始执行 `callbacks` 数组中的函数时，如果没有传递 `cb` 参数，则直接调用 `_resolve` 函数，这个函数就是返回的 `Promise` 实例对象的 `resolve` 函数。这样就实现了 `Promise` 方式的 `$nextTick()` 函数。**

