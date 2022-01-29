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

