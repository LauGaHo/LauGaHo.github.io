# Vue源码解读—渲染函数的观察者和进阶的数据响应系统(下)

## `$watch` 和 `watch` 选项的实现

**前边的章节讲了许多关于 `Watcher` 类的内容，接下来看一下 `$watch` 函数以及 `watch` 选项的实现。实际上无论是 `$watch` 函数还是 `watch` 选项，他们的实现都是基于 `Watcher` 的封装。首先先看 `$watch` 函数，它的定义在 `src/core/instance/state.js` 文件的 `stateMixin()` 函数中，如下：**

```javascript
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {
  const vm: Component = this
  if (isPlainObject(cb)) {
    return createWatcher(vm, expOrFn, cb, options)
  }
  options = options || {}
  options.user = true
  const watcher = new Watcher(vm, expOrFn, cb, options)
  if (options.immediate) {
    cb.call(vm, watcher.value)
  }
  return function unwatchFn () {
    watcher.teardown()
  }
}
```

**`$watch()` 方法允许开发者观察数据对象的某个属性，当属性变化时执行回调。所以 `$watch()` 方法至少接收两个参数，一个要观察的属性，以及一个回调函数。通过上方的代码可以发现，`$watch()` 函数接收三个参数，除了前边介绍的两个参数之后还接收第三个参数，它是一个选项参数，比如是否立即执行回调或者是否深度观测等。这三个参数和 `Watcher` 类的构造函数中的是三个参数相匹配，如下：**

```javascript
export default class Watcher {
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    // 省略...
  }
}
```

**`$watch()` 函数的实现本质就是创建了一个 `Watcher` 实例对象。另外通过官方文档的介绍可知 `$watch()` 函数的第二个参数既可以是一个回调函数，也可以是一个纯对象，这个对象中可以包含 `handler` 属性，该属性的值将作为回调函数，同时该对象还可以包含其他属性作为选项参数，如 `immediate` 和 `deep`。**

**假设传递给 `$watch()` 函数的第二个参数是一个函数，观察其实现过程，在 `$watch()` 函数中内部先执行的代码如下：**

```javascript
const vm: Component = this
if (isPlainObject(cb)) {
  return createWatcher(vm, expOrFn, cb, options)
}
```

**定义了 `vm` 变量，它是当前组件实例对象，接着检测传递给 `$watch()` 函数的是否是纯对象，由于假设参数 `cb` 是一个函数，所以这段 `if` 语句块内的代码不会执行。再往下就是这段代码：**

```javascript
options = options || {}
options.user = true
const watcher = new Watcher(vm, expOrFn, cb, options)
```

**如果没有传递 `options` 选项参数，那么会给其一个默认的空对象，接着将 `options.user` 的值设置为 `true`，`options.user` 代表该观察者实例对象是用户创建的，然后到了关键的一步：*创建 `Watcher` 实例对象，很简单的一行代码。***

**再往下就是一段 `if` 语句块：**

```javascript
if (options.immediate) {
  cb.call(vm, watcher.value)
}
```

**`immediate` 属性表示在属性或函数被侦听后立即调用回调，如上代码就是其实现原理，如果 `options.immediate` 选项为 `true`，则会执行回调函数，不过此时回调函数的参数只有新值没有旧值。同时取值的方式是通过前边创建的观察者实例对象的 `watch.value` 属性。观察者实例对象的 `value` 属性，保存着被观察属性的值。**

**最后 `$watch()` 函数还有一个返回值，如下：**

```javascript
return function unwatchFn () {
  watcher.teardown()
}
```

**`$watch()` 函数返回一个函数，这个函数的执行会解除当前观察者对属性的观察。原理是：*通过调用观察者实例对象的 `watcher.teardown()` 函数实现。*`teardown()` 函数源码如下：**

```javascript
export default class Watcher {
  // 省略...
  /**
   * Remove self from all dependencies' subscriber list.
   */
  teardown () {
    if (this.active) {
      // remove self from vm's watcher list
      // this is a somewhat expensive operation so we skip it
      // if the vm is being destroyed.
      if (!this.vm._isBeingDestroyed) {
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}
```

**检查 `this.active` 属性是否为 `true`，如果为 `false` 则说明该观察者已经不处于激活状态了，什么都不需要做，如果为 `true` 则会执行 `if` 语句块内的代码，在 `if` 语句块中先执行这段代码：**

```javascript
if (!this.vm._isBeingDestroyed) {
  remove(this.vm._watchers, this)
}
```

**先补充一个小知识：*每一个组件都有一个 `vm._isBeingDestroy` 属性，它是一个标识，为 `true` 说明该组件实例已经被销毁了，为 `false` 说明该组件还没销毁。*所以以上代码意为：如果组件没有销毁，那么将当前观察者实例对象从组件实例对象中的 `vm._watchers` 数组中移除，`vm._watchers` 数组中包含了该组件所有的观察者实例对象，所以将当前观察者实例对象从 `vm._watchers` 数组中移除是解除属性和观察者实例对象之间关系的第一步。由于这个参数的性能开销比较大，所以仅在组件没有被销毁的情况下才会执行此操作。**

**将观察者实例对象从 `vm._watchers` 数组中移除之后，会执行如下代码：**

```javascript
let i = this.deps.length
while (i--) {
  this.deps[i].removeSub(this)
}
```

**当一个属性和一个观察者建立联系之后，属性的 `Dep` 实例对象会收集到该观察者实例对象，同时观察者实例对象也会将该 `Dep` 实例对象收集，这是一个双向过程，并且一个观察者可以同时观察多个属性，这些属性的 `Dep` 实例对象都会被收集到该观察者实例对象的 `this.deps` 数组中，所以解除属性和观察者之间的关系第二步就是将当前观察者实例对象从所有的 `Dep` 实例对象中移除，实现方法如上代码所示。**

**最后会将当前观察者实例对象的 `active` 属性设置为 `false`，代表该观察者实例对象已经处于非激活状态：**

```javascript
this.active = false
```

**以上就是 `$watch()` 函数的实现，以及如何解除观察的实现。上述的源码讲解是建立于一个前提：*传递 `$watch()` 函数的第二个参数是一个函数。*如果是一个纯对象，这时如下代码就会执行：**

```javascript
if (isPlainObject(cb)) {
    return createWatcher(vm, expOrFn, cb, options)
}
```

**当参数 `cb` 不是函数，而是一个纯对象，则会调用 `createWatcher()` 函数，并将参数传递下去。`createWatcher()` 函数定义在 `src/core/instance/state.js` 文件中，如下是 `createWatcher()` 函数源码：**

```javascript
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

**其实 `createWatcher()` 函数的作用就是将纯对象形式的参数规范一下，然后再通过 `$watch()` 函数创建观察者。`createWatcher()` 函数最后一行代码就是通过调用 `vm.$watch()` 函数并将其返回。看 `createWatcher()` 函数的第一段代码：**

```javascript
if (isPlainObject(handler)) {
  options = handler
  handler = handler.handler
}
```

**检测参数 `handler` 是否是纯对象，在 `$watch()` 函数中曾检测过一次，在 `createWatcher()` 再次检测的原因是因为 `createWatcher()` 函数除了 `$watch()` 函数调用，还会用于 `watch` 选项，这个时候就需要进行检查了。总之，如果 `handler` 是一个纯对象，那么将变量 `handler` 的值赋给 `options`，然后使用 `handler.handler` 的值重写 `handler` 变量的值。举例说明：**

```javascript
vm.$watch('name', {
  handler () {
    console.log('change')
  },
  immediate: true
})
```

**上方代码对于 `createWatcher()` 函数来讲，`handler` 参数为：**

```javascript
handler = {
  handler () {
    console.log("change")
  },
  immediate: true
}
```

**所以如下这段代码：**

```javascript
if (isPlainObject(handler)) {
  options = handler
  handler = handler.handler
}
```

**等价于：**

```javascript
if (isPlainObject(handler)) {
  options = {
    handler () {
      console.log("change")
    },
    immediate: true
  }
  handler = handler () {
    console.log("change")
  }
}
```

**这样就可以正常通过 `$watch()` 函数创建观察者了。另外 `createWatcher()` 函数中还有如下这段代码：**

```javascript
if (typeof handler === 'string') {
    handler = vm[handler]
}
```

**上方这段代码说明 `handler` 除了可以使一个纯对象还可以是一个字符串，当 `handler` 是一个字符串的时候，会读取组件实例对象的 `handler` 属性的值，并用该值重写 `handler` 的值。然后再通过调用 `$watch()` 函数创建观察者，举例说明，如下：**

```javascript
watch: {
  name: 'handleNameChange'
},
methods: {
  handleNameChange () {
    console.log('name change')
  }
}
```

**上方代码中在 `watch` 选项中观察了 `name` 属性，但没有指定回调函数，而是指定了一个 `handleNameChange` 字符串，相当于指定了 `methods` 选项中同名函数作为回调函数。**

**接着了解 `watch` 选项如何进行初始化。源码入口位于 `initState()` 函数中：**

```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

**上方代码中的 `initWatch()` 就是用于初始化 `watch` 选项，`initWatch()` 函数的源码如下：**

```javascript
function initWatch (vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```

**从上方代码可知，`initWatch()` 函数是对 `watch` 选项进行遍历，然后通过 `createWatcher()` 函数创建观察者对象的，需要注意的是上方代码中有一个判断条件：`if (Array.isArray(handler))`，通过这个条件可以发现 `handler` 常量是一个数组，其值是：`watch[key]`，就是说在使用 `watch` 选项时可以通过传递数组来实现创建多个观察者，如下：**

```javascript
watch: {
  name: [
    function () {
      console.log('name 改变了1')
    },
    function () {
      console.log('name 改变了2')
    }
  ]
}
```

**总结：在 `Watcher` 类的基础上，实现 `$watch()` 还是实现 `watch` 选项都显得十分从容，十分自然，丝滑一般。**



## 深度观察的实现

**这小节将讨论深度观察的实现，回顾一下数据响应的原理，响应式数据的关键在于数据的属性是访问器属性，使得开发者可以拦截对该属性的读写操作，从而有机会收集依赖并触发响应。思考如下代码：**

```javascript
watch: {
  a () {
    console.log('a 改变了')
  }
}
```

**上方这段代码使用了 `watch` 选项观察了数据对象的 `a` 属性，`watch` 选项内部是通过创建 `Watcher` 实例对象来实现观察的，在创建 `Watcher` 实例对象时会读取 `a` 的值从而触发属性 `a` 的 `get()` 拦截器函数，最终将依赖收集。但是如果属性 `a` 是一个对象，如下：**

```javascript
data () {
  return {
    a: {
      b: 1
    }
  }
},
watch: {
  a () {
    console.log('a 改变了')
  }
}
```

**如上方代码所示，数据对象 `data` 的属性 `a` 是一个对象，当实例化 `Watcher` 对象并观察属性 `a` 时，会读取属性 `a` 的值，这样的确能够触发属性 `a` 的 `get()` 拦截器函数，但是因为没有读取属性 `a.b` 属性的值，所以对于 `b` 来说是没有收集到任何观察者。这就是所谓的浅观察，直接修改属性 `a` 的值能触发依赖，而修改 `a.b` 的值则触发不了响应。**

**深度观察就是用来解决这个问题，原理很简单，属性 `a.b` 中没有收集到观察者，就主动读取 `a.b` 的值，这样就能触发属性 `a.b` 的 `get()` 拦截器函数从而收集到观察者。只要你将 `deep` 选项参数设置为 `true`，就会告知 `Watcher` 类进行深度观察了，找到 `Watcher` 类中的 `get()` 函数，如下：**

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

**如上代码所示，在 `Watcher` 类中的 `get()` 函数用于求值，在 `get()` 函数内部通过调用 `this.getter()` 函数对被观察的属性进行求值，并将求得的值赋值给变量 `value`，同时在 `finally` 语句块中，如果属性 `this.deep` 为 `true` 则说明是深度观察，此时会将被观察属性的值 `value` 作为参数传递给 `traverse()` 函数，其中 `traverse()` 函数的作用就是递归读取被观察属性的子属性的值，这样观察属性的所有子属性都将会收集到观察者，从而达到深度观察的目的。**

**`traverse()` 函数来自 `src/core/observer/traverse.js` 文件，如下：**

```javascript
const seenObjects = new Set()

/**
 * Recursively traverse an object to evoke all converted
 * getters, so that every nested property inside the object
 * is collected as a "deep" dependency.
 */
export function traverse (val: any) {
  _traverse(val, seenObjects)
  seenObjects.clear()
}
```

**上方代码定义了 `traverse()` 函数，这个函数将接收被观察属性的值作为参数，然后在其内部调用 `_traverse()` 函数完成递归遍历。其中 `_traverse()` 函数定义在 `traverse()` 函数的下方，如下是 `_traverse()` 函数的签名：**

```javascript
function _traverse (val: any, seen: SimpleSet) {
  // 省略...
}
```

**`_traverse()` 函数接收两个参数，第一个参数是被观察属性的值，第二个参数是一个 `Set` 数据结构的实例，在 `traverse()` 函数中调用 `_traverse()` 函数时传递的第二个参数 `seenObjects` 就是一个 `Set` 数据结构的实例，它定义在文件头部：`const seenObjects = new Set()`。**

**`_traverse()` 函数源码如下：**

```javascript
function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  /////////////////////////////////
  if (val.__ob__) {
    const depId = val.__ob__.dep.id
    if (seen.has(depId)) {
      return
    }
    seen.add(depId)
  }
  /////////////////////////////////
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```

**对上方代码进行一下精简，去掉上方注释包围的代码，得出 `_traverse()` 的如下代码：**

```javascript
function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```

**上方的这段代码就是降低了复杂度的代码，现在细看 `_traverse()` 函数的实现。在 `_traverse()` 函数的开头声明了两个变量，分别是 `i` 和 `keys`，这两个变量后面会使用到，接着检查参数 `val` 是否是数组，并将检测结果存放在常量 `isA` 中，再往下就是一段 `if` 语句块：**

```javascript
if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
  return
}
```

**这段代码是对参数 `val` 的检测，后边统一称 `val` 为被观察属性的值，既然是深度观察，所以被观察属性的值要么是一个对象，要么是一个数组，并且该值不能是冻结的，也不应该是 `VNode`。只有当被观察属性的值满足这些条件时，才会对其进行深度观察，只要有一项条件不满足，`_traverse()` 就会 `return` 直接返回，结束流程。所以上方这段 `if` 语句可以理解为是在检测被观察属性的值能否进行深度观察，一旦能够深度观察将会继续执行之后的代码，如下：**

```javascript
if (isA) {
  i = val.length
  while (i--) _traverse(val[i], seen)
} else {
  keys = Object.keys(val)
  i = keys.length
  while (i--) _traverse(val[keys[i]], seen)
}
```

**这段代码将检测被观察属性的值是数组还是对象，无论是数组还是对象都会通过 `while` 循环对其进行遍历，并递归调用 `_traverse()` 函数，这段代码的关键在于递归调用 `_traverse()` 函数时所传递的第一个参数是 `val[i]` 还是 `val[keys[i]]`。这两个参数实际上是在读取子属性的值，这将触发子属性的 `get()` 拦截器函数，保证子属性能够收集到观察者。**

**`_traverse()` 函数初步解析完成，但是还有一段被简化掉的代码，如下：**

```javascript
if (val.__ob__) {
  const depId = val.__ob__.dep.id
  if (seen.has(depId)) {
    return
  }
  seen.add(depId)
}
```

**这段代码用于解决循环引用导致死循环的问题，下边举例说明：**

```javascript
const obj1 = {}
const obj2 = {}

obj1.data = obj2
obj2.data = obj1
```

**上方代码定义了两个对象，分别是 `obj1` 和 `obj2`，并且 `obj1.data` 属性引用了 `obj2`，而 `obj2.data` 引用了 `obj1`，这是一个典型的循环应用。如果使用 `obj1` 或 `obj2` 这两个对象中的任意一个对象出现在 Vue 的响应式数据中，如果不做防循环引用的处理，将会导致死循环，如下代码：**

```javascript
function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```

**如果被观察属性的值 `val` 是一个循环引用的对象，那么上方代码将导致死循环，为了避免这种情况的出现，可以使用一个变量来存储那些已经被遍历过的对象，当再次遍历该对象时，程序会发现该对象已经被遍历过了，这时就会跳过遍历，从而避免死循环，如下代码所示：**

```javascript
function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  /////////////////////////////////
  if (val.__ob__) {
    const depId = val.__ob__.dep.id
    if (seen.has(depId)) {
      return
    }
    seen.add(depId)
  }
  ////////////////////////////////
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```

**注释包含的这段代码，是一段 `if` 语句块，用来判断 `val.__ob__` 是否有值，如果一个响应式数据是对象或者数组，它会包含一个叫做 `__ob__` 属性，这时读取 `val.__ob__.dep.id` 作为一个唯一的 `id` 值，并将它放到 `seenObjects` 变量中：`seen.add(depId)`，这样即使 `val` 是一个拥有循环引用的对象，当下一次遇到该对象时，就能够发现该对象已经遍历过了：`seen.has(depId)`，这样函数直接返回 `return` 即可。**

**以上就是深度观察的实现以及避免循环引用造成的死循环的解决方案。**



## 计算属性的实现

### 计算属性的初始化

**现在对响应式数据系统了解得足够深了，轮到研究计算属性了，计算属性本质上是一个惰性求值的观察者。计算属性的初始化是在 `src/core/instance/state.js` 文件中的 `initState()` 函数中：**

```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

**上方代码通过 `initComputed(vm, opts.computed)` 来实现计算属性的初始化，找到 `initComputed()` 函数源码如下：**

```javascript
function initComputed (vm: Component, computed: Object) {
  // 省略...
}
```

**和其他初始化响应式数据相关函数一样，都接收两个参数，第一个参数是组件实例对象，第二个参数是对应选项。在 `initComputed()` 函数中开头定义了两个常量：**

```javascript
// $flow-disable-line
const watchers = vm._computedWatchers = Object.create(null)
// computed properties are just getters during SSR
const isSSR = isServerRendering()
```

**其中 `watchers` 常量和组件实例的 `vm._computedWatchers` 属性拥有相同的引用，并且初始值都是通过 `Object.create(null)` 创建的空对象，`isSSR` 常量是用来判断是否服务端渲染的布尔值。接着开启一个 `for...in` 循环，后续的代码都写在了这个 `for...in` 循环中：**

```javascript
for (const key in computed) {
  // 省略...
}
```

**这个 `for...in` 循环用来遍历 `computed` 选项对象，在循环的内部首先是一段这样的代码：**

```javascript
const userDef = computed[key]
const getter = typeof userDef === 'function' ? userDef : userDef.get
if (process.env.NODE_ENV !== 'production' && getter == null) {
  warn(
    `Getter is missing for computed property "${key}".`,
    vm
  )
}
```

**定义了 `userDef` 常量，它的值是计算属性对象中相应的属性值，计算属性有两种写法，计算属性可以是一个函数，如下：**

```javascript
computed: {
  someComputedProp () {
    return this.a + this.b
  }
}
```

**如果使用了上方的写法，那么 `userDef` 的值就是一个函数：**

```javascript
userDef = someComputedProp () {
  return this.a + this.b
}
```

**另外计算属性也可以写成对象，如下：**

```javascript
computed: {
  someComputedProp: {
    get: function () {
      return this.a + 1
    },
    set: function (v) {
      this.a = v - 1
    }
  }
}
```

**如果使用如上这种写法，那么 `userDef` 常量的值就是一个对象：**

```javascript
userDef = {
  get: function () {
    return this.a + 1
  },
  set: function (v) {
    this.a = v - 1
  }
}
```

**在 `userDef` 常量的下边定义了 `getter` 常量，它的值是根据 `userDef` 常量的值决定的：**

```javascript
const getter = typeof userDef === 'function' ? userDef : userDef.get
```

**如果计算属性使用函数的写法，那么 `getter` 常量的值就是 `userDef` 本身，即函数。如果计算属性使用的是对象写法，那么 `getter` 的值将会是 `userDef.get()` 函数。反正 `getter` 常量总是一个函数！！！**

**在 `getter` 常量的下边做了一个检测：**

```javascript
if (process.env.NODE_ENV !== 'production' && getter == null) {
  warn(
    `Getter is missing for computed property "${key}".`,
    vm
  )
}
```

**在非生产环境下如果发现 `getter` 不存在，则直接打印警告信息，提示计算属性没有对应的 `getter`。也就是说计算属性的函数写法实际上是对象写法的简化，如下这两种写法是等价的：**

```javascript
computed: {
  someComputedProp () {
    return this.a + this.b
  }
}

// 等价于

computed: {
  someComputedProp: {
    get () {
      return this.a + this.b
    }
  }
}
```

**再往下，是一段 `if` 条件语句，如下：**

```javascript
if (!isSSR) {
  // create internal watcher for the computed property.
  watchers[key] = new Watcher(
    vm,
    getter || noop,
    noop,
    computedWatcherOptions
  )
}
```

**只有在非服务端渲染时才会执行 `if` 语句块内的代码，因为服务端中计算属性的实现本质上和使用 `methods` 选项差不多。这里着重讲解费服务端渲染的实现，这时 `if` 语句块中的代码会被执行，可以看到在 `if` 语句块中创建了一个观察者实例对象，称之为计算属性的观察者，同时会把计算属性的观察者添加到 `watchers` 常量对象中，键值是对应计算属性的名字，由于 `watchers` 常量和 `vm._computedWatchers` 属性具有相同的引用，所以对 `watchers` 常量的修改相当于对 `vm._computedWatchers` 属性的修改，因此，`vm._computedWatchers` 对象就是用来存储计算属性观察者的。**

**另外还有需要注意：*首先创建计算属性观察者时传递的第二个参数是 `getter` 参数，也就是说计算属性观察者的求值对象是 `getter` 函数。传递的第四个参数是 `computedWatcherOptions` 常量，是一个对象，定义在 `initComputed` 函数的上方：***

```javascript
const computedWatcherOptions = { computed: true }
```

**传递给 `Watcher` 类的第四个参数是观察者的选项参数，选项参数对象可以包含如 `deep`、`sync` 等选项，当然也包括 `computed` 选项，通过如上这行代码可以得知创建计算属性的观察者实例对象时 `computed` 选项为 `true`，它的作用就是用来标识一个观察者对象是计算属性的观察者，计算属性的观察者和非计算属性的观察者的行为是不一致的。**

**再往下就是 `for...in` 循环中最后一段代码，如下：**

```javascript
if (!(key in vm)) {
  defineComputed(vm, key, userDef)
} else if (process.env.NODE_ENV !== 'production') {
  if (key in vm.$data) {
    warn(`The computed property "${key}" is already defined in data.`, vm)
  } else if (vm.$options.props && key in vm.$options.props) {
    warn(`The computed property "${key}" is already defined as a prop.`, vm)
  }
}
```

**这段代码首先检测计算属性的名字是否已经存在于组件实例对象中，因为在初始化计算属性之前，就已经初始化了 `props`、`methods`、`data` 选项，并且这些选项数据都会定义在组件实例对象上，由于计算属性也需要定义在组件实例对象上，所以需要使用计算属性的名字检测组件实例对象上是否已经有同名的定义，如果该名字已经定义在组件实例对象，那么有可能是 `data` 数据或 `methods` 数据或 `props` 数据之一，对于 `data` 和 `props` 来讲，它们是不允许被 `computed` 选项中的同名属性覆盖的，所以在非生产环境中还要检查计算属性中是否存在和 `data` 和 `props` 选项同名的属性，如果有则会打印警告信息。如果没有则调用 `defineComputed()` 定义计算属性。**

**`defineComputed()` 函数就定义在 `initComputed()` 函数的下方，如下是 `defineComputed()` 函数的签名及最后一句代码：**

```javascript
export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  // 省略...
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

**根据 `defineComputed()` 函数的最后一行代码可知，该函数的作用就是通过 `Object.defineProperty()` 函数在组件实例对象上定义和计算属性同名的组件实例属性，而且是一个访问器属性，属性的配置参数是 `sharedPropertyDefinition` 对象，`defineComputed()` 函数中除了最后一行代码之外，其余代码都是用来完善 `sharedPropertyDefinition` 对象。**

**`sharedPropertyDefinition` 对象定义在当前文件头部，如下：**

```javascript
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}
```

**紧接着看一下 `defineComputed()` 函数如何完善这个对象，在 `defineComputed()` 函数开头定义了 `shouldCache` 常量，它的值和 `initComputed()` 函数中定义的 `isSSR` 常量的值是取反的关系，也是一个布尔值，用来标识是否应该缓存值，也就是说只有在非服务端渲染的情况下计算属性才会缓存。**

**紧接着就是一段 `if...else` 语句块：**

```javascript
if (typeof userDef === 'function') {
  sharedPropertyDefinition.get = shouldCache
    ? createComputedGetter(key)
    : userDef
  sharedPropertyDefinition.set = noop
} else {
  sharedPropertyDefinition.get = userDef.get
    ? shouldCache && userDef.cache !== false
      ? createComputedGetter(key)
      : userDef.get
    : noop
  sharedPropertyDefinition.set = userDef.set
    ? userDef.set
    : noop
}
```

**这段 `if...else` 语句块的作用是为 `sharedPropertyDefinition.get()` 和 `sharedPropertyDefinition.set()` 赋予合适的值。首先检查 `userDef` 是否是函数，如果是函数则执行 `if` 语句块内的代码，如果 `userDef` 不是函数则说明是对象，此时会执行 `else` 分支的代码。假如 `userDef` 是函数，在 `if` 语句块中首先会使用三元运算符检查 `shouldCache` 是否为 `true`，如果为 `true` 说明不是服务端渲染，此时会调用 `createComputedGetter` 函数并将其返回值作为 `sharedPropertyDefinition.get` 的值。如果 `shouldCache` 为 `false` 则说明是服务端渲染，由于服务端渲染不需要缓存值，所以直接使用 `userDef` 函数作为 `sharedPropertyDefinition.get` 的值。另外由于 `userDef` 是函数，说明该计算属性并没有指定 `set` 拦截器函数，所以直接将其设置为空函数 `noop`：`sharedPropertyDefinition.set = noop`。**

**如果代码走到了 `else` 分支，那说明 `userDef` 是一个对象，如果 `userDef.get` 存在并且是在非服务端渲染的环境下，同时没有指定选项 `userDef.cache` 为 `false`，则此时会调用 `createComputedGetter` 函数并将其返回值作为 `sharedPropertyDefinition.get` 的值，否则 `sharedPropertyDefinition.get` 的值为 `userDef.get` 函数。同样的如果 `userDef.set` 函数存在，则使用 `userDef.set` 函数作为 `sharedPropertyDefinition.set` 的值，否则使用空函数 `noop` 作为其值。**

**总之，无论 `userDef` 是函数还是对象，在非服务端渲染的情况下，配置对象 `sharedPropertyDefinition` 最终会变成如下：**

```javascript
sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: createComputedGetter(key),
  set: userDef.set // 或 noop
}
```

