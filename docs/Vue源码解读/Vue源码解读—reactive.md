# Vue源码解读—reactive

**Vue 响应式原理是通过对数据进行一个数据劫持和数据代理进而实现的，本文将深入 Vue 源码来以此探讨 Vue 响应式的原理**

## Observe

**先看到代码的入口：**

```javascript
export function initState (vm: Component) {
  // 初始化一个 _wathcers 数组，作用我们后面看
  vm._watchers = []
  const opts = vm.$options

  // 初始化props
  if (opts.props) initProps(vm, opts.props)

  // 初始化函数
  if (opts.methods) initMethods(vm, opts.methods)

  // 初始化data
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }

  // 初始化computed
  if (opts.computed) initComputed(vm, opts.computed)

  // 初始化watch
  // 这个 nativeWatch 定义在 src/core/util/env.js 中
  // nativeWatch = ({}).watch，注释说的是 Firefox 中对象原型有这么一个方法，但是我没有打印出来，囧
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

**我们先聚焦 Vue 如何对 `data` 进行数据劫持和代理：**

```javascript
function initData (vm: Component) {
  // 获取合并 data 的处理函数，在mergeField时会合并，栗子 🌰 中的 parentVal 是 undefined，所以这里的 data 就是我们看到的 data
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  
  // ... 省略一大段报警信息 🚔

    // 判断key的首字符是不是 $ 或 _
    // 不是将属性代理到 vue 实例中
    } else if (!isReserved(key)) {

      // 代理到实例下面
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  // 调用 observe 方法观测整个 data 的变化，把 data 变成响应式
  observe(data, true /* asRootData */)
}
```

**上方的代码，最终会调用 `observe` 把 `data` 数据变成响应式：**

```javascript
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void

  // 如果有 __ob__ 属性并且是 Observer 的实例，说明已经是个响应式对象，就直接返回
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  }
	// 没有的话实例一个ob对象
	else if (
    shouldObserve &&  // 定义的变量，这里是true，在initProps会改变该值，后面分析 props 的过程再看
    !isServerRendering() && // 服务端渲染
    (Array.isArray(value) || isPlainObject(value)) && // 数组或纯对象
    Object.isExtensible(value) && // 可扩展
    !value._isVue // 不是 vue 实例
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

**`observe(value, asRootData)` 方法通过判断 `value` 中是否含有 `__ob__` 属性和 `__ob__` 属性是否为 `Observer` 类型来确定数据是否为响应式，如果是响应式则直接返回，如果不是响应式，则新建一个 `Observer` 对象 (即相当于：实例化一个 `ob`)**

```javascript
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value

    // 实例化一个 Dep 对象
    this.dep = new Dep()
    this.vmCount = 0

    // 把自身实例添加到数据对象 value 的 __ob__ 属性上
    // 定义在 src/core/util/lang.js
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  /**
   * Walk through each property and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  // 将每个属性转换成 getter/setter，注意函数参数是对象
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {

    // 遍历数组再次调用 observe
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

**`Observer` 构造函数中把参数赋值给私有属性 `value`，然后调用 `def` 函数在对象上挂上 `__ob__` 属性，最后判断 `value` 如果是数组，循环调用 `observe`，如果是对象，那依次对每一个 `key` 执行 `defineReactive`。回到例子中，执行完 `def` 之后，我们的 `value` 就会变成下图所示：**

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/sXxJB2.png)

**在我们的例子当中，`value` 是对象，因此会执行 `walk(obj: Object)` 函数，我们先来看一下 `observe` 的重点 — `defineReactive()`：**

```javascript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {

  /*在闭包内存储一个Dep对象*/
  const dep = new Dep()

  // 拿到 obj 的属性描述符
  const property = Object.getOwnPropertyDescriptor(obj, key)

  // 判断，如果是不能配置的属性
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  // 如果你之前就定义了 getter 和 setter，这里获取定义值
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // 对子对象递归调用 observe 方法，这样就保证了无论 obj 的结构多复杂，它的所有子属性也能变成响应式的对象
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // ...
    },
    set: function reactiveSetter (newVal) {
	  // ...
    }
  })
}
```

**响应式的第一步工作给 `data` 里的每一个属性都通过 `Object.defineProperty` 挂上 `get` 和 `set`，小节开头的两个问题也就迎刃而解了，上面的 `defineReactive` 代码逻辑中，将 `get` 和 `set` 的逻辑去掉了，下面我们就赖补充这部分的逻辑。**



## 依赖收集

**`observe` 好了数据之后，程序主流程就会进入到挂载阶段 — `$mount`。这时例子就会经过生成 `render` 函数过程。**

**有了 `render` 函数，就会进入 `mountComponent`：**

```javascript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  // ...
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    // ...
  } else {
    updateComponent = () => {

      // vm._render()生成虚拟节点
      // 调用 vm._update 更新 DOM
      vm._update(vm._render(), hydrating)
    }
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined

  // 实例化 watcher
  new Watcher(
    vm,
    updateComponent,
    noop,
    {
      before () {
        if (vm._isMounted) {
          callHook(vm, 'beforeUpdate')
        }
      }
    },
    true /* isRenderWatcher */)

  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook

  // vm.$vnode 表示 Vue 实例的父虚拟 Node，所以它为 Null 则表示当前是根 Vue 的实例
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```

**这里我们先不管，先看一下 `Watcher` 类**

```javascript
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  computed: boolean;
  sync: boolean;
  dirty: boolean;
  active: boolean;
  dep: Dep;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  before: ?Function;
  getter: Function;
  value: any;

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
      
    // 第5个参数标志是不是渲染watcher，整个Vue有3种watcher：renderWatcher、computedWatcher、userWatcher，在执行mountComponent时，这里是true的
    if (isRenderWatcher) {
        
      // 把渲染 watcher 挂在实例的 _watcher 上
      vm._watcher = this
    }
      
    // vm._watchers 在前面 initState 时初始化了
    vm._watchers.push(this)
    // 这里的 options 是上面说到的 3 种 watcher 对应的一些属性
    // 对于渲染watcher过程，这里有 before 函数，其他都是false
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.computed = !!options.computed
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.computed = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.computed // for computed watchers

    // 跟dep相关的一些属性
    // 表示 Watcher 实例持有的 Dep 实例的数组
    this.deps = []
    this.newDeps = []

    // 代表 this.deps 和 this.newDeps 的 id Set
    this.depIds = new Set()
    this.newDepIds = new Set()

    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {

      // 这里的 getter 是 updateComponent 函数
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

    // computed watcher并不会立刻求值
    if (this.computed) {
      this.value = undefined
      this.dep = new Dep()
    } else {

      // 执行watch的get函数
      this.value = this.get()
    }
  }

  // ...
}
```

**先看构造函数，构造函数是为各种属性复制，并且最终会执行到：**

```javascript
this.value = this.get();
```

**先看一下 `get()` 函数的定义：**

```javascript
get () {
    // 把当前的 watcher 对象 push 到 targetStack 中
    // 然后赋值给 Dep.target，这个作用后面会讲到
    pushTarget(this)
    let value
    const vm = this.vm
    try {

        // this.getter 对应就是 updateComponent 函数
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
        // deep watcher
        if (this.deep) {
            traverse(value)
        }
        popTarget()
        this.cleanupDeps()
    }
    return value
}
```

> **`pushTarget` 和 `popTarget` 的定义在 `src/core/observer/dep.js`**

```javascript
// the current target watcher being evaluated.
// this is globally unique because there could be only one
// watcher being evaluated at any time.
Dep.target = null
const targetStack = []

export function pushTarget (_target: ?Watcher) {

  // Dep.target 赋值为当前的渲染 watcher 并压栈（为了恢复用）
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

export function popTarget () {
  Dep.target = targetStack.pop()
}
```

**`get()` 函数对应的 `value = this.getter.call(vm, vm)` 最终会执行到 `updateComponent`，回到 `mountComponent` 中查看这部分的逻辑：**

```javascript
updateComponent = () => {
    // vm._render()生成虚拟节点
    // 调用 vm._update 更新 DOM
    vm._update(vm._render(), hydrating)
}
```

**`vm._render()` 会执行 `render` 函数，生成 `VNode`：**

```javascript
Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    // ...
    let vnode
    try {

        // 写 render 函数的第一个参数 h，就是 vm.$createElement
        // vm._renderProxy 就是 vm，在 init 阶段定义了
        vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
        // ...
    }
    // ...
    // set parent
    vnode.parent = _parentVnode
    return vnode
}
```

**例子生成的 `VNode` 如下：**

```javascript
with (this) {
        return _c('div', {
            attrs: {
                "id": "app"
            }
        }, [_c('div', [_v(_s(message))]), _v(" "), _l((relationship), function(item) {
            return _c('div', {
                key: item.name
            }, [_c('span', [_v("name：" + _s(item.name))]), _v(" "), _c('span', [_v("relation：" + _s(item.relation))])])
        }), _v(" "), _c('div', [_c('button', {
            on: {
                "click": addRelatives
            }
        }, [_v("添加亲戚")])]), _v(" "), _c('p', [_v("\n      Province: " + _s(address.children.name) + "\n    ")])], 2)
    }
}
```

> **生成的 `render` 函数有一堆小矮人，`_v`、`_l`、`_s`，这些小矮人定义在 `src/core/instance/render-helpers/index.js`：**

```javascript
import { toNumber, toString, looseEqual, looseIndexOf } from 'shared/util'
import { createTextVNode, createEmptyVNode } from 'core/vdom/vnode'
import { renderList } from './render-list'
import { renderSlot } from './render-slot'
import { resolveFilter } from './resolve-filter'
import { checkKeyCodes } from './check-keycodes'
import { bindObjectProps } from './bind-object-props'
import { renderStatic, markOnce } from './render-static'
import { bindObjectListeners } from './bind-object-listeners'
import { resolveScopedSlots } from './resolve-slots'

export function installRenderHelpers (target: any) {
  target._o = markOnce
  target._n = toNumber
  target._s = toString
  target._l = renderList
  target._t = renderSlot
  target._q = looseEqual
  target._i = looseIndexOf
  target._m = renderStatic
  target._f = resolveFilter
  target._k = checkKeyCodes
  target._b = bindObjectProps
  target._v = createTextVNode
  target._e = createEmptyVNode
  target._u = resolveScopedSlots
  target._g = bindObjectListeners
}
```

**在生成 `VNode` 的过程中，一定会访问到我们的数据 (`this.message`、`this.relationship` 等等)，这时就会触发数据的 `getter`，此时就把上一小节省略掉的 `get()` 函数放回来：**

```javascript
get: function reactiveGetter () {
    
    // 当前值，比如 message 就是 this is a message
    const value = getter ? getter.call(obj) : val
    
    // 还记得大明河畔的夏雨荷吗？
    // 在执行 Watcher 的 get 时，执行了 pushTarget，当时给 Dep.target 赋值了渲染 Watcher
    if (Dep.target) {

        // 依赖收集
        dep.depend()
        
        // 如果有子ob，就一起收集了
        if (childOb) {
            childOb.dep.depend()
            if (Array.isArray(value)) {
                dependArray(value)
            }
        }
    }
    return value
},
```

**终于到了依赖收集的重点了，我们在 `defineReactive()` 函数中，实例化了一个依赖实例：**

```javascript
export function defineReactive(...) {
  ......
  const dep = new Dep();
  ......
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // ...
    },
    set: function reactiveSetter (newVal) {
	  // ...
    }
  })
}
```

**调用 `dep.depend()` 进行依赖收集，这时我们看一下 `Dep` 类：**

```javascript
export default class Dep {
  // 静态属性 target，这是一个在任何时间都是全局唯一 Watcher
  static target: ?Watcher;  
  id: number;

  // 订阅数组，收集订阅对象
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
        
      // Dep.target 指渲染 watcher
      Dep.target.addDep(this)
    }
  }
  
  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```

**`Dep.target.addDep(this)` 就会回到 `Watcher` 中执行 `addDep` 逻辑：**

```javascript
addDep (dep: Dep) {
    const id = dep.id
    
    // 防止重复添加，去重
    if (!this.newDepIds.has(id)) {

        // 把当前的 watcher 订阅到这个数据持有的 dep 的 subs 中
        // 这个目的是为后续数据变化时候能通知到哪些 subs 做准备
        this.newDepIds.add(id)
        this.newDeps.push(dep)
        if (!this.depIds.has(id)) {
            
            // 把当前渲染 watcher 添加到依赖实例的 subs 数组中
            dep.addSub(this)
        }
    }
}
```

**这里将 `dep` 添加到 `watcher` 中的数组 `newDeps` 中，同时也将 `watcher` 添加到 `dep` 中的 `subs` 中。**

**`render` 函数执行完，还有一小部分逻辑**

```javascript
// 处理 deep 的 user watcher
if (this.deep) {
    traverse(value)
}
// 恢复成上一个状态，当前vm的依赖收集完了，对应的渲染也要更新
// 在栗子 🌰 中把 Dep.target 变成 undefined
popTarget()
this.cleanupDeps()
```

**看一下最后的 `cleanupDeps()` 函数执行了什么：**

```javascript
cleanupDeps () {
    let i = this.deps.length
    while (i--) {
        const dep = this.deps[i]
        if (!this.newDepIds.has(dep.id)) {
            // 移除对Watcher的订阅
            dep.removeSub(this)
        }
    }
    
    // 交换 newDepIds 和 depIds
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

**因为每次依赖收集的情况有可能跟上一轮依赖收集的情况都可能会有所不同，所以每次依赖收集完之后，都会将每个 `watcher` 对应所需最新的依赖存放在 `newDeps` 和 `newDepIds` 当中，如果旧的依赖 `deps` 中存在着某些不存在与 `newDeps` 中的依赖，那么就会进入到对应的 `dep` 中，在 `dep` 中对应的 `subs` 中移除当前的 `watcher`。这样就可以让 `dep.subs` 属性保持最新的一个状态。避免更新渲染时做无用功。**



**依赖收集会发生在 `render` 函数生成 `VNode` 的过程中。因为会访问到具体的数据，从而触发数据的 `get`，然后当前的渲染 `wacher` 会把数据当做依赖添加到对应的 `newDeps` 和 `newDepIds` 当中，同时，`dep` 也会将渲染 `watcher` 存到 `subs` 属性当中，当我们修改数据时，就会用收集的结果去触发渲染 `watcher` 更新，接下来我们看一下派发更新的详细过程。**



## 派发更新

**Vue 中的派发更新是通过触发 `Dep` 类中的 `notify()` 函数来进行触发的：**

```javascript
notify () {
    const subs = this.subs.slice()
    // 依次触发渲染 watcher 的 update
    for (let i = 0, l = subs.length; i < l; i++) {
        subs[i].update()
    }
}
```

> **上方的 `update()` 方法的位置在 `src/core/observer/watch.js`。**

```javascript
update () {
    // ...
    // 这里省略掉 computed watcher 和同步 user watcher 的逻辑，直接看栗子 🌰会执行到的 queueWatcher
    queueWatcher(this)
}
```

> **`queueWatcher` 在 `src/core/observer/scheduler.js` 中定义：**

```javascript
const queue: Array<Watcher> = []
let waiting = false
let flushing = false
let index = 0

export function queueWatcher (watcher: Watcher) {
  const id = watcher.id

  // 确保一个 watcher 只添加一次
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      // 把当前 watcher 加入到队列中
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

    // 保证只执行一次nextTick
    if (!waiting) {
      waiting = true
      nextTick(flushSchedulerQueue)
    }
  }
}
```

**`queueWatcher` 可以看到是把渲染 `watcher` 加入到队列中，并不是每次更新都实时执行。然后执行到 `nextTick(flushSchedulerQueue)`：**

```javascript
export function withMacroTask (fn: Function): Function {
  return fn._withTask || (fn._withTask = function () {
    useMacroTask = true
    const res = fn.apply(null, arguments)
    useMacroTask = false
    return res
  })
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve

  // 把传入的函数压入callbacks数组，需要callbacks而不是直接执行cb是因为多次执行nextTick时，能用同步顺序在下一个tick中执行，而不需要开启多个宏/微任务
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
  if (!pending) {
    pending = true
    // 对于手动触发的一些数据更新，比如栗子 🌰 中的 click 事件，强行走宏任务 
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // $flow-disable-line
  // 当nextTick不传函数时，提供一个promise化的调用

  // 不传cb直接返回一个promise的调用
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

**看 `nextTick` 之前，我们先看宏任务 `macroTimerFunc` 和微任务 `microTimerFunc` 的实现：**

```javascript
// 宏任务的实现：先看是否支持 setImmediate，然后判断是否支持 MessageChannel，最终 setTimeout 兜底
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

// Determine microtask defer implementation.
/* istanbul ignore next, $flow-disable-line */
// 微任务实现：先判断是否支持Promise，否则直接指向 macroTimerFunc
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

**回到 `nextTick`，把回调函数存到 `callbacks`，通过 `macroTimerFunc` 触发 `flushCallbacks`：**

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

**然后依次调用 `callbacks` 中的函数—`flushSchedulerQueue`：**

```javascript
function flushSchedulerQueue () {
  flushing = true
  let watcher, id

  /**
   * 1.组件的更新由父到子；因为父组件的创建过程是先于子的，所以 watcher 的创建也是先父后子，执行顺序也应该保持先父后子。
   * 2.用户的自定义 watcher 要优先于渲染 watcher 执行；因为用户自定义 watcher 是在渲染 watcher 之前创建的。
   * 3.如果一个组件在父组件的 watcher 执行期间被销毁，那么它对应的 watcher 执行都可以被跳过，所以父组件的 watcher 应该先执行。
   */
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  // 这里不缓存队列长度的原因是在 watcher.run() 的时候，很可能用户会再次添加新的 watcher
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    // 执行 before 函数，也就是 beforeUpdated 钩子
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  // 重置调度队列的状态
  resetSchedulerState()

  // call component updated and activated hooks
  // 执行一些钩子函数，比如keep-alive的actived和组件的updated钩子
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```

**上面的代码主要操作是：先对 `watcher` 做了排序，然后依次执行组件的 `beforeUpdated` 的钩子函数。依次调用 `watcher.run()` 触发更新：**

```javascript
run () {
    if (this.active) {
        this.getAndInvoke(this.cb)
    }
}

getAndInvoke (cb: Function) {
    // 对于渲染 watcher 而言，调用 get 会重新触发 updateComponent，就又会重新执行依赖收集，不一样的是更新时是有老的VNode，会做对比再重新patch
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

                // 这就是为什么我们写自定义watcher时能拿到新值和旧值
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

**上面的代码主要操作：执行 `getAndInvoke()` 函数，其实也就是执行 `this.get()` 获取当前的值，如果满足新老值不相等，新值是对象、`deep` 模式下的任一条件，就执行回调 `cb`。回调中的参数是新老值，这也是为什么我们写自定义 `watcher` 有新老值参数的原因，对于例子而言，这里的 `this.get` 重新触发 `updateComponent()`。**



## 总结

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/SSxrOG.png)

**根据上图做一个总结：在 `render` 阶段，会读取到响应式数据，触发数据的 `getter`，然后会通过 `Dep` 做 `render watcher` 的收集。当修改响应式数据时，会触发数据的 `setter`，从而触发 `Dep` 的 `notify`。让所有的 `render watcher` 异步更新，从而重新渲染页面。重新渲染页面会做 `VNode` 的 `diff`，然后高效地渲染页面。**

