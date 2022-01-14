# Vue源码解读—Vue初始化

## 用于初始化的最终选项 `$options`

**从一个例子开始，例子如下：**

```javascript
const vm = new Vue({
  el: '#app',
  data: {
    test: 1
  }
})
```

**根据这个例子，来讲解实例化 Vue 的过程，`Vue.prototype._init()` 方法被第一个执行，这个方法的位置在 `src/core/instance/init.js` 文件中，分析 `_init()` 方法的时候，就碰上了如下代码：**

```javascript
vm.$options = mergeOptions(
	resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

**在前边 `mergeOptions解读(上)、(下)` 中，了解到了 `mergeOptions()` 函数是如何对父子选项进行合并处理的。**

**`mergeOptions()` 函数中的最后一句代码：`return options`。**

**这说明了 `mergeOptions()` 函数最终将合并处理后的选项返回，并以该返回值作为 `vm.$options` 的值。`vm.$options` 在 Vue 的官方文档是这样描述：**

> ***`$options` 用于当前 Vue 实例的初始化选项。需要在选项中包含自定义属性时会有用处。***

**并且给定了一个例子，如下：**

```javascript
new Vue({
  customOption: 'foo',
  created: function () {
    console.log(this.$options.customOption)	// => 'foo'
  }
})
```

**上方的例子中，在创建 Vue 实例的传递了一个自定义选项：`customOption`，在 Vue 实例中，可以通过 `this.$options.customOption` 进而访问，原理就是：*使用 `mergeOptions()` 函数对自定义选项进行合并操作，但是由于没有指定 `customOption` 选项使用什么合并策略，所以将会使用默认的策略函数 `defaultStrat` 对其进行合并处理。***

**另外，Vue 中也提供了 `Vue.config.optionMergeStrategies` 全局配置，我们可以通过该属性为自定义属性指定特定的合并策略，比如为上方的 `customOption` 选项指定一个合并策略，只需要在 `Vue.config.optionMergeStrategies` 上添加和选项同名的策略函数即可：**

```javascript
Vue.config.optionMergeStrategies.customOption = function (parentVal, childVal) {
  return parentVal ? (parentVal + childVal) : childVal
}
```

**上方的代码中，我们添加了自定义选项 `customOption` 的合并策略，具体为：如果没有 `parentVal` 则直接返回 `childVal`，否则返回两者的和。**

**举例说明：**

```javascript
// 创建子类
const Sub = Vue.extend({
    customOption: 1
})
// 以子类创建实例
const v = new Sub({
    customOption: 2,
    created () {
        console.log(this.$options.customOption) // 3
    }
})
```

**最终，在实例的 `created()` 方法中将打印数字 `3`。**

**回到开头的例子：**

```javascript
const vm = new Vue({
  el: '#app',
  data: {
    test: 1
  }
})
```

**这个时候 `mergeOptions` 函数将会 `Vue.options` 作为父选项，把我们传递的实例选项作为子选项进行合并，合并的结果可以通过 `console.log(vm.$option)` 得知。`el` 选项将使用默认合并策略合并，最终的值就是字符串 `'#app'`，而 `data` 选项将会变成一个函数，且这个函数执行结果就是合并后的数据，即：`{test: 1}`。**



## 渲染函数的作用域代理

**接着继续看 `_init()` 函数中的代码，了解 Vue 的初始化工作。**

**`_init()` 方法中，经过了 `mergeOptions()` 合并处理选项之后，紧接着就是这段代码：**

```javascript
/* istanbul ignore else */
if (process.env.NODE_ENV !== 'production') {
    initProxy(vm)
} else {
    vm._renderProxy = vm
}
```

**这段代码是是一个判断分支，如果是非生产环境的话，则执行 `initProxy(vm)` 函数，如果在生产模式下，则在实例上添加 `_renderProxy` 实例属性，该属性的值就是当前实例。**

**`initProxy()` 函数的作用：*在实例对象 `vm` 上添加 `_renderProxy` 属性。在上方的代码中，生产环境直接执行：***

```javascript
vm._renderProxy = vm
```

**在非生产环境下其实也应该执行这句代码，但实际上却调用了 `initProxy()` 函数，所以 `initProxy()` 函数作用之一必然是在实例对象 `vm` 上添加 `_renderProxy` 属性，看 `initProxy()` 函数源码，一探究竟，`initProxy()` 函数的源码位于：`src/core/instance/proxy.js`。**

```javascript
/* not type checking this file because flow doesn't play well with Proxy */

import config from 'core/config'
import { warn, makeMap } from '../util/index'

// 声明 initProxy 变量
let initProxy

if (process.env.NODE_ENV !== 'production') {
  // ... 其他代码
  
  // 在这里初始化 initProxy
  initProxy = function initProxy (vm) {
    if (hasProxy) {
      // determine which proxy handler to use
      const options = vm.$options
      const handlers = options.render && options.render._withStripped
        ? getHandler
        : hasHandler
      vm._renderProxy = new Proxy(vm, handlers)
    } else {
      vm._renderProxy = vm
    }
  }
}

// 导出
export { initProxy }
```

**上方的代码是经过了简化的，在文件的末尾将 `initProxy` 变量进行了导出，由此可以得知，在非生产环境下，`initProxy` 变量是一个函数，在生产环境下，`initProxy` 变量是 `undefined`。**

**接着看 `initProxy` 变量是函数的源代码：**

```javascript
initProxy = function initProxy (vm) {
    if (hasProxy) {
        // determine which proxy handler to use
        const options = vm.$options
        const handlers = options.render && options.render._withStripped
        ? getHandler
        : hasHandler
        vm._renderProxy = new Proxy(vm, handlers)
    } else {
        vm._renderProxy = vm
    }
}
```

**这个函数接收一个 `Vue` 实例对象，总览这个函数，可以看出这个函数无论是走 `if` 还是 `else` 分支，最终的效果都是在 `vm` 对象上添加 `_renderProxy` 属性，而 `if` 判断分支中的条件 `hasProxy` 就是判断宿主环境是否支持 `JS` 原生的 `Proxy` 特性，如果支持 `Proxy` 特性，则直接执行：**

```javascript
vm._renderProxy = new Proxy(vm, handlers)
```

**如果不存在，则直接赋值：**

```javascript
vm._renderProxy = vm
```

**因此这里可以得知 `initProxy()` 函数的作用就是对实例对象 `vm` 的代理，通过原生的 `Proxy` 实现：**

**而 `hasProxy` 的定义也是在该文件中：**

```javascript
const hasProxy = typeof Proxy !== 'undefined' && Proxy.toString().match(/native code/)
```

**上面的代码是判断当前宿主环境是否支持原生 `Proxy`。**

**下方讨论是形成代理的过程：**

```javascript
initProxy = function initProxy (vm) {
    if (hasProxy) {
        // determine which proxy handler to use
        // options 就是 vm.$options 的引用
        const options = vm.$options
        // handlers 可能是 getHandler 也可能是 hasHandler
        const handlers = options.render && options.render._withStripped
            ? getHandler
            : hasHandler
        // 代理 vm 对象
        vm._renderProxy = new Proxy(vm, handlers)
    } else {
        // ...
    }
}
```

**如果 `Proxy` 存在，那么将使用 `Proxy` 对 `vm` 做一层代理，代理对象赋值给 `vm._renderProxy` 属性，因此之后对 `vm._renderProxy` 的访问，如果存在代理的话，那么访问就会被拦截。我们使用 `handlers` 对代理对象进行配置，但是这里的 `handlers` 有可能是 `getHandler` 也有可能是 `hasHandler`，两者之间使用哪个是通过这个判断条件来判断的：**

```javascript
options.render && options.render._withStripped
```

**如果上方的条件为 `true`，则使用 `getHandler`，否则使用 `hasHandler`，先说一个结论，一般情况下，`options.render._withStripped` 属性都为 `false`，所以一般情况下都是使用 `hasHandler` 作为配置对象。**

**`hasHandler` 的定义就在当前文件中，如下：**

```javascript
const hasHandler = {
    has (target, key) {
        // has 常量是真实经过 in 运算符得来的结果
        const has = key in target
        // 如果 key 在 allowedGlobals 之内，或者 key 是以下划线 _ 开头的字符串，则为真
        const isAllowed = allowedGlobals(key) || (typeof key === 'string' && key.charAt(0) === '_')
        // 如果 has 和 isAllowed 都为假，使用 warnNonPresent 函数打印错误
        if (!has && !isAllowed) {
            warnNonPresent(target, key)
        }
        return has || !isAllowed
    }
}
```

**`has()` 操作可以拦截这些操作：**

- **属性查询：`foo in proxy`**
- **继承属性查询：`foo in Object.create(proxy)`**
- **`with` 检查：`with(proxy) { (foo); }`**
- **`Reflect.has()`**

**其中最关键的，就是 `has()` 可以拦截 `with` 语句中对变量的访问。**

**在 `hasHandler` 中定义的 `has()` 函数中调用了两个函数：`allowedGlobals()` 和 `warnNonPresent()` 函数，分别来看看这两个函数的源码：**

**`allowedGlobals()`：**

```javascript
const allowedGlobals = makeMap(
    'Infinity,undefined,NaN,isFinite,isNaN,' +
    'parseFloat,parseInt,decodeURI,decodeURIComponent,encodeURI,encodeURIComponent,' +
    'Math,Number,Date,Array,Object,Boolean,String,RegExp,Map,Set,JSON,Intl,' +
    'require' // for Webpack/Browserify
)
```

**`allowedGlobals()` 函数是通过 `makeMap()` 生成的函数，他的作用是：判断给定的 `key` 是否出现上上述字符串中定义的关键字中，这些关键字都是在 `javascript` 中可以全局访问的。**

**`warnNonPresent()`：**

```javascript
const warnNonPresent = (target, key) => {
    warn(
        `Property or method "${key}" is not defined on the instance but ` +
        'referenced during render. Make sure that this property is reactive, ' +
        'either in the data option, or for class-based components, by ' +
        'initializing the property. ' +
        'See: https://vuejs.org/v2/guide/reactivity.html#Declaring-Reactive-Properties.',
        target
    )
}
```

**`warnNonPresent()` 函数通过 `warn()` 打印一段警告信息，提示在渲染的时候引用了 `key`，但是对象中并没有定义该属性或者方法。**

**举例说明：**

```javascript
const vm = new Vue({
  el: '#app',
  template: '<div>{{a}}</div>',
  data: {
    test: 1
  }
})
```

**此时在模板中使用了 `a` 属性，但是在 `data` 中并没有定义这个属性，这个时候就会得到报错。**

**之所以会有这种情况的出现，是因为在 `src/core/instance/render.js` 文件中，找到 `Vue.prototype._render` 方法，里边有一行代码：**

```javascript
vnode = render.call(vm._renderProxy, vm.$createElement)
```

**在调用 `render()` 函数的时候，使用 `call()` 指定了函数的执行环境为 `vm._renderProxy` (其实也就是指定了 `this` 为 `vm._renderProxy`)，而渲染函数源码如下：**

```javascript
vm.$option.render = function () {
  // render 函数的 this 指针指向 _renderProxy
  with(this) {
    // 这里访问 a，就相当于访问 vm._renderProxy.a
    return _c('div', [_v(_s(a))])
  }
}
```

**上述代码中，在 `with` 语句中访问 `a`，相当于访问 `this.a`，而 `this` 在 `call()` 函数中被指定为了 `vm._renderProxy`，经过推导，其实就是相当于访问 `vm._renderProxy.a`，`vm._renderProxy` 是一个经过 `Proxy` 的对象，上边说过，在 `with` 语句中访问变量会被 `Proxy` 的 `has()` 代理所拦截，所以自然就执行了 `has()` 函数中的代码了。**

**还有一个 `getHandler`。**

```javascript
options.render && options.render._withStripped
```

**上边的条件为 `true`，`getHandler` 才会被使用，在 `test/unit/features/instance/render-proxy.spec.js` 文件中有如下代码：**

```javascript
it('should warn missing property in render fns without `with`', () => {
    const render = function (h) {
        // 这里访问了 a
        return h('div', [this.a])
    }
    // 在这里将 render._withStripped 设置为 true
    render._withStripped = true
    new Vue({
        render
    }).$mount()
    // 应该得到警告
    expect(`Property or method "a" is not defined`).toHaveBeenWarned()
})
```

**这个情况下就会触发 `getHanler` 设置的 `get()` 拦截，`getHandler()` 代码如下：**

```javascript
const getHandler = {
  get (target, key) {
    if (typeof key === 'string' && !(key in target)) {
      warnNonPresent(target, key)
    }
    return target[key]
  }
}
```

**实现效果：如果访问了一个不存在的属性，就给出一个警告，并且这个效果是需要设置 `render._withStripped = true` 才会触发。**

**之所以会这样设计，是因为使用 `webpack` 配合 `vue-loader` 的环境中，`vue-loader` 会借助 `vuejs@component-compiler-utils` 将 `template` 编译为不使用 `with` 语句包裹的遵循严格模式的 JavaScript，并为编译后的 `render` 方法设置 `render._withStripped = true`。在不使用 `with` 语句时，访问对象属性都是通过 `vm['a']` 或者 `vm.a` 来访问的，这里就相当于使用了 `getHandler` 拦截了属性的访问。可以进行一些检查的操作。**

**还有一些细节，下边这段代码：**

```javascript
// has 变量是真实经过 in 运算符得来的结果
const has = key in target
// 如果 key 在 allowedGlobals 之内，或者 key 是以下划线 _ 开头的字符串，则为真
const isAllowed = allowedGlobals(key) || (typeof key === 'string' && key.charAt(0) === '_')
// 如果 has 和 isAllowed 都为假，使用 warnNonPresent 函数打印错误
if (!has && !isAllowed) {
    warnNonPresent(target, key)
}
```

**上边这段代码的判断语句时：`!has && !isAllowed`，其中 `!has` 意为访问了一个没有定义在实例对象上 (或原型链上) 的属性。除此之外还要满足 `isAllowed`，`isAllowed` 的意思是如果 `key` 不是全局对象，而且 `key` 是形如 `_c`、`_s`、`_v` 这些全局的渲染函数。**

**总结这段代码的意思：如果访问了一个没有定义在实例对象上 (或原型链上) 的属性，并且访问的属性不是全局对象，也不是以 `_` 开头的渲染函数，如：`_c`、`_v` 这些全局的渲染函数。**

**就如下边这个例子：**

```html
<template>
  {{Number(b) + 2}}
</template>
```

**对于 `proxy.js` 文件内的代码，还有一段没有渗透：**

```javascript
if (hasProxy) {
    // isBuiltInModifier 函数用来检测是否是内置的修饰符
    const isBuiltInModifier = makeMap('stop,prevent,self,ctrl,shift,alt,meta,exact')
    // 为 config.keyCodes 设置 set 代理，防止内置修饰符被覆盖
    config.keyCodes = new Proxy(config.keyCodes, {
        set (target, key, value) {
            if (isBuiltInModifier(key)) {
                warn(`Avoid overwriting built-in modifier in config.keyCodes: .${key}`)
                return false
            } else {
                target[key] = value
                return true
            }
        }
    })
}
```

**上边这段代码先检测宿主环境是否支持 `Proxy`，内部代码会使用 `makeMap` 函数生成一个 `isBuiltInModifier()` 函数，该函数用来检测给定的值是否内置的事件修饰符。然后为了 `config.keyCodes` 设置了一个 `set` 代理，目的是为了防止开发者在自定义键位别名的时候，覆盖了内置的修饰符，如下：**

```javascript
Vue.config.keyCodes.shift = 16
```

**因为 `shift` 是内置的修饰符，所以这行代码就会得到警告。**



## 初始化之 `initLifecycle`

**`_init` 函数在执行完 `initProxy` 之后，执行的是 `initLifecycle` 函数：**

```javascript
vm._self = vm
initLifecycle(vm)
```

**在 `initLifecycle()` 函数执行完之前，执行了 `vm._self = vm` 这行之后，这行代码会在 Vue 实例对象 `vm` 上添加 `_self` 属性，指向真实的实例本身。这里 `vm._self` 和 `vm._renderProxy` 不同。**

**紧接着就是执行 `initLifecycle` 函数，同时将 Vue 实例 `vm` 作为参数传递。`initLifecycle` 函数位于 `src/core/instance/lifecycle.js` 文件中，源码如下：**

```javascript
export function initLifecycle (vm: Component) {
  // 定义 options，它是 vm.$options 的引用，后面的代码使用的都是 options 常量
  const options = vm.$options

  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```

**下边会对这段代码进行一个解释：**

```javascript
// locate first non-abstract parent (查找第一个非抽象的父组件)
// 定义 parent，它引用当前实例的父实例
let parent = options.parent
// 如果当前实例有父组件，且当前实例不是抽象的
if (parent && !options.abstract) {
  // 使用 while 循环查找第一个非抽象的父组件
  while (parent.$options.abstract && parent.$parent) {
    parent = parent.$parent
  }
  // 经过上面的 while 循环后，parent 应该是一个非抽象的组件，将它作为当前实例的父级，所以将当前实例 vm 添加到父级的 $children 属性里
  parent.$children.push(vm)
}

// 设置当前实例的 $parent 属性，指向父级
vm.$parent = parent
// 设置 $root 属性，有父级就是用父级的 $root，否则 $root 指向自身
vm.$root = parent ? parent.$root : vm
```

**上述代码的总结：*将当前实例添加到父实例的 `$children` 属性当中，并设置当前实例的 `$parent` 属性指向父实例。*这里是通过这种方式寻找父级的：**

```javascript
// 定义 parent，它引用当前实例的父组件
let parent = options.parent
```

**看下边的例子：**

```javascript
// 子组件本身并没有指定 parent 选项
var ChildComponent = {
  created () {
    // 但是在子组件中访问父实例，能够找到正确的父实例引用
    console.log(this.$options.parent)
  }
}

var vm = new Vue({
    el: '#app',
    components: {
      // 注册组件
      ChildComponent
    },
    data: {
        test: 1
    }
})
```

**Vue 中提供了 `parent` 选项，使得可以手动指定一个组件的父实例，但是在上边的例子中，没有指定 `parent` 选项，但是子组件中仍然还能找到正确的父实例。Vue 寻找父实例的时候是自动检测的。在 `src/core/vdom/create-component.js` 文件，有一个函数 `createComponentInstanceForVnode` 源码如下：**

```javascript
export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
  parentElm?: ?Node,
  refElm?: ?Node
): Component {
  const vnodeComponentOptions = vnode.componentOptions
  const options: InternalComponentOptions = {
    _isComponent: true,
    parent,
    propsData: vnodeComponentOptions.propsData,
    _componentTag: vnodeComponentOptions.tag,
    _parentVnode: vnode,
    _parentListeners: vnodeComponentOptions.listeners,
    _renderChildren: vnodeComponentOptions.children,
    _parentElm: parentElm || null,
    _refElm: refElm || null
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnodeComponentOptions.Ctor(options)
}
```

**这个函数的作用通过一个例子来说明：**

```javascript
// 子组件
var ChildComponent = {
  created () {
    console.log(this.$options.parent)
  }
}

var vm = new Vue({
    el: '#app',
    components: {
      // 注册组件
      ChildComponent
    },
    data: {
        test: 1
    }
})
```

**上方的代码中，子组件 `ChildComponent` 是一个 `JSON` 对象，或者可以说组件选项对象，在父组件的 `components` 选项中把这个子组件选项对象注册进去，实际上在 Vue 内部，会以子组件选项对象作为参数通过 `Vue.extend()` 函数创建一个子类出来，然后再通过实例化子类来创建子组件，只不过这个过程是通过虚拟 DOM 的 `patch` 算法进行。在 `createComponentInstanceForVnode()` 函数中有一段代码：**

```javascript
const options: InternalComponentOptions = {
  _isComponent: true,
  parent,
  propsData: vnodeComponentOptions.propsData,
  _componentTag: vnodeComponentOptions.tag,
  _parentVnode: vnode,
  _parentListeners: vnodeComponentOptions.listeners,
  _renderChildren: vnodeComponentOptions.children,
  _parentElm: parentElm || null,
  _refElm: refElm || null
}
```

**这是实例化子组件时的组件选项，第二个值就是 `parent`，它是 `createComponentInstanceForVnode()` 函数的形参，`createComponentInstanceForVnode()` 函数的调用位置就在 `src/core/vdom/create-component.js` 文件中的 `componentVNodeHooks` 钩子对象的 `init()` 钩子函数内，源码如下：**

```javascript
// hooks to be invoked on component VNodes during patch
const componentVNodeHooks = {
  init (
    vnode: VNodeWithData,
    hydrating: boolean,
    parentElm: ?Node,
    refElm: ?Node
  ): ?boolean {
    if (!vnode.componentInstance || vnode.componentInstance._isDestroyed) {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance,
        parentElm,
        refElm
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    } else if (vnode.data.keepAlive) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    }
  },

  prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    ...
  },

  insert (vnode: MountedComponentVNode) {
    ...
  },

  destroy (vnode: MountedComponentVNode) {
    ...
  }
}
```

**在 `init()` 函数中有一段代码中：**

```javascript
const child = vnode.componentInstance = createComponentInstanceForVnode(
	vnode,
  activeInstance,
  parentElm,
  refElm
)
```

**第二个参数 `activeInstance` 就是形参 `parent`，而 `activeInstance` 通过顶部的 `import` 可知 `activeInstance` 来自于 `src/core/instance/lifecycle.js` 文件，也就是在 `initLifecycle()` 函数的上边，源码如下：**

```javascript
export let activeInstance: any = null
```

**这个变量总保存着当前正在渲染的实例的引用，所以它就是当前实例 `components` 下注册的子组件的父实例，Vue 实际上就是这样实现自动侦测父级。**

```javascript
// 定义 parent，它引用当前实例的父组件
let parent = options.parent
// 如果当前实例有父组件，且当前实例不是抽象的
if (parent && !options.abstract) {
  // 使用 while 循环查找第一个非抽象的父组件
  while (parent.$options.abstract && parent.$parent) {
    parent = parent.$parent
  }
  // 经过上面的 while 循环后，parent 应该是一个非抽象的组件，将它作为当前实例的父级，所以将当前实例 vm 添加到父级的 $children 属性里
  parent.$children.push(vm)
}
```

**上边的这段代码还有一些需要注意的细节：拿到了父实例 `parent` 之后，进入一个判断分支，条件是：`parent && !options.abstract`，*即父实例存在，且当前实例不是抽象，实际上 `Vue` 内部有一些选项时没有暴露给开发者的，就比如当前这里的 `abstract`，通过设置这个选项为 `true`，可以指定该组件为抽象，通过该组件的构造函数创建的实例也是抽象的，比如：***

```javascript
AbsComponents = {
  abstract: true,
  create () {
    console.log('我是一个抽象组件')
  }
}
```

**抽象组件的特点：抽象组件一般不渲染真实 DOM，像 Vue 内置了了一些全局组件比如：`keep-alive` 或者 `transition`，这两个组件是不会渲染 DOM 到页面的，但是他仍然提供了开发者需要的功能，打开 `keep-alive` 组件的源码，他们位于 `src/core/components/keep-alive.js` 中：**

```javascript
export default {
  name: 'keep-alive',
  abstract: true,
  ......
}
```

**抽象组件除了不渲染真实 DOM 之外，还有一个特点，那就是他不会出现在父子关系的路径上。重新回来看这段代码：**

```javascript
// locate first non-abstract parent (查找第一个非抽象的父组件)
// 定义 parent，它引用当前实例的父组件
let parent = options.parent
// 如果当前实例有父组件，且当前实例不是抽象的
if (parent && !options.abstract) {
  // 使用 while 循环查找第一个非抽象的父组件
  while (parent.$options.abstract && parent.$parent) {
    parent = parent.$parent
  }
  // 经过上面的 while 循环后，parent 应该是一个非抽象的组件，将它作为当前实例的父级，所以将当前实例 vm 添加到父级的 $children 属性里
  parent.$children.push(vm)
}

// 设置当前实例的 $parent 属性，指向父级
vm.$parent = parent
// 设置 $root 属性，有父级就是用父级的 $root，否则 $root 指向自身
vm.$root = parent ? parent.$root : vm
```

**从上方的代码中可以知道，如果一个组件中的 `options.abstract` 为 `true`，那么说明当前组件实例是抽象的，所以他会跳过 `if` 分支的代码，直接设置 `vm.$parent` 和 `vm.$root` 的值等于 `options.parent` (也就是 `null`)。跳过 `if` 语句将导致该抽象实例不会被添加到父实例的 `$children` 中。如果 `abstract` 为 `false`，那说明当前实例不是抽象组件实例，只是一个普通组件实例，这个时候就会走 `while` 循环，这个 `while` 循环的目的就是沿着父实例链逐层向上寻找到一个不为抽象的实例作为 `parent` ，并且将自身添加到父实例中的 `$children` 属性中。**

**上方的代码中执行完毕之后，`initLifecycle()` 函数还会在当前实例上添加一些属性，即后边的代码：**

```javascript
vm.$children = []
vm.$refs = {}

vm._watcher = null
vm._inactive = null
vm._directInactive = false
vm._isMounted = false
vm._isDestroyed = false
vm._isBeingDestroyed = false
```

**其中 `$children` 和 `$refs` 都是熟悉的实例属性，他们都是在 `initLifecycle()` 函数中初始化，其中 `$children` 被初始化成一个数组，`$ref` 被初始化为一个空的 `JSON` 对象，除此之外还有一些内部使用的属性。这些属性都是跟生命周期有关的。继续回到 `_init` 函数。**



## 初始化之 `initEvents`

**`initLifecycle` 函数之后，执行的就是 `initEvents` 函数，它来自于 `src/core/instance/event.js` 文件，源码如下：**

```javascript
export function initEvents (vm: Component) {
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}
```

**上方代码中，为 `vm` 实例对象上添加两个实例属性 `_events` 和 `_hasHookEvent`，其中 `_events` 被初始化为一个空对象，`_hasHookEvent` 属性的初始值为 `false`。之后将执行这段代码：**

```javascript
// init parent attached events
const listeners = vm.$options._parentListeners
if (listeners) {
  updateComponentListeners(vm, listeners)
}
```

**其中 `vm.$options._parentListeners` 的 `_parentListeners` 在 `src/core/vdom/create-component.js` 文件中的 `createComponentInstanceForVnode()` 函数出现过：**

```javascript
export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
  parentElm?: ?Node,
  refElm?: ?Node
): Component {
  const vnodeComponentOptions = vnode.componentOptions
  const options: InternalComponentOptions = {
    _isComponent: true,
    parent,
    propsData: vnodeComponentOptions.propsData,
    _componentTag: vnodeComponentOptions.tag,
    _parentVnode: vnode,
    _parentListeners: vnodeComponentOptions.listeners,
    _renderChildren: vnodeComponentOptions.children,
    _parentElm: parentElm || null,
    _refElm: refElm || null
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnodeComponentOptions.Ctor(options)
}
```

**由此可见在创建子组件实例的时候会有这个参数选项。**



## 初始化之 `initRender`

**在 `initEvents()` 的后边，紧接着就是执行 `initRender()` 函数。`initRender()` 函数源码如下：**

```javascript
export function initRender (vm: Component) {
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null // v-once cached trees
  const options = vm.$options
  const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
  const renderContext = parentVnode && parentVnode.context
  vm.$slots = resolveSlots(options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

  // $attrs & $listeners are exposed for easier HOC creation.
  // they need to be reactive so that HOCs using them are always updated
  const parentData = parentVnode && parentVnode.data

  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
    }, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
    }, true)
  } else {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
  }
}
```

**我们逐行分析代码：**

```javascript
vm._vnode = null	// the root of the child tree
vm._staticTrees = null	// v-once cached trees
```

**首先先为 Vue 实例对象中添加两个实例属性，即 `_vnode` 和 `_staticTrees`，并初始化为 `null`，他们会在合适的地方被赋值并使用。**

**然后紧接着就是这段代码：**

```javascript
const options = vm.$options
const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
const renderContext = parentVnode && parentVnode.context
vm.$slots = resolveSlots(options._renderChildren, renderContext)
vm.$scopedSlots = emptyObject
```

**上方代码是解析并处理 `slot` 的，关于 `options._parentVnode`、`options._renderChildren`、`parentVnode.context` 这三个属性将会在本节的最后讲述。**

**上方的这段代码中的意思：为 Vue 实例添加了三个实例属性：**

```javascript
vm.$vnode
vm.$slots
vm.$scopedSlots
```

**再往下就是这段代码：**

```javascript
// bind the createElement fn to this instance
// so that we get proper render context inside it.
// args order: tag, data, children, normalizationType, alwaysNormalize
// internal version is used by render functions compiled from templates
vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
// normalization is always applied for the public version, used in
// user-written render functions.
vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
```

**这段代码在 Vue 实例对象上添加了两个方法：`vm._c` 和 `vm.$createElement`，这两个方法其实就是对内部函数 `createElement()` 函数的包装。其中 `vm.$createElement` 就是手写渲染函数的时候使用到。如下代码：**

```javascript
render: function (createElement) {
  return createElement('h2', 'Title')
}
```

**我们知道渲染函数的一个形参就是 `createElement()` 函数，这个函数是用于创建 `vnode`。实际上也可以这样子做：**

```javascript
render: function () {
  return this.$createElement('h2', 'Title')
}
```

**上方两段代码是完全等价的。对于 `vm._c` 方法，`vm._c` 和 `vm.$createElement` 的不同之处在于调用 `createElement()` 函数时传递的第六个参数不同。这里后边会讲解。**

**再往下就是 `initRender()` 函数的最后一段代码：**

```javascript
// $attrs & $listeners are exposed for easier HOC creation.
// they need to be reactive so that HOCs using them are always updated
const parentData = parentVnode && parentVnode.data

/* istanbul ignore else */
if (process.env.NODE_ENV !== 'production') {
  defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
    !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
  }, true)
  defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
    !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
  }, true)
} else {
  defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
  defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
}
```

**上边的代码主要作用是为 Vue 实例对象上定义两个属性：`vm.$atrrs` 以及 `vm.$listeners`。这两个属性在 Vue 中的文档是有说明的，这两个属性使得创建高阶组件变得容易。**

**上方代码中，为实例对象定义 `$attrs` 属性和 `$listeners` 属性时，使用了 `defineReactive()` 函数，该函数的作用就是为一个对象定义响应式的属性，另外，上方 `if` 语句上是对当前环境是否是生产环境进行了一个判断。在非生产环境中调用 `defineReactive()` 函数时传递的第四个参数是一个函数，这个函数是一个自定义的 `setter`，这个 `setter` 会在设置 `$attrs` 或 `$listeners` 属性时触发并执行。当程序试图设置该属性时，会执行该函数：**

```javascript
() => {
  !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
}
```

**当 `!isUpdatingChildComponent` 成立时，会提示 `$attrs` 或 `$listeners` 是只读属性，不应该手动设置他的值。**

**这里使用了 `isUpdatingChildComponent` 变量，这个变量来自 `lifecycle.js` 文件，在 `lifecycle.js` 文件中，有三个地方使用了这个变量：**

```javascript
// 定义 isUpdatingChildComponent，并初始化为 false
export let isUpdatingChildComponent: boolean = false

// 省略中间代码 ...

export function updateChildComponent (
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
  if (process.env.NODE_ENV !== 'production') {
    isUpdatingChildComponent = true
  }

  // 省略中间代码 ...

  // update $attrs and $listeners hash
  // these are also reactive so they may trigger child update if the child
  // used them during render
  vm.$attrs = parentVnode.data.attrs || emptyObject
  vm.$listeners = listeners || emptyObject

  // 省略中间代码 ...

  if (process.env.NODE_ENV !== 'production') {
    isUpdatingChildComponent = false
  }
}
```

**这里需要注意的点只有一个：在 `updateChildComponent()` 函数执行的开头就把 `isUpdatingChildComponent` 变量设置为 `true`，在函数的结尾将 `isUpdatingChildComponent` 变量设置为 `false`。剩下的细节会在讲解 Virtual DOM 的时候详细讲述。**



## 生命周期钩子的实现方式

**`initRender()` 函数执行完之后，就到下边这段代码：**

```javascript
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created')
```

**上边这段代码中，`initInjections(vm)`、`initState(vm)`、`initProvide(vm)` 这三个函数被包裹在两个 `callHook()` 函数调用的语句中，`callHook()` 函数的作用：*调用生命周期钩子函数。*`callHook()` 函数来自于 `lifecycle.js` 文件，在源码对这个函数进行分析：**

```javascript
export function callHook (vm: Component, hook: string) {
  // #7573 disable dep collection when invoking lifecycle hooks
  pushTarget()
  const handlers = vm.$options[hook]
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm)
      } catch (e) {
        handleError(e, vm, `${hook} hook`)
      }
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}
```

**`callHook()` 函数接收两个参数：实例对象和要调用的生命周期钩子的名称。**

**`callHook()` 函数中以 `pushTarget()` 开头，并以 `popTarget()` 结尾，这里是为了避免在某些生命周期钩子中使用 `props` 数据导致收集冗余的依赖。**

**`callHook()` 函数首先获取要调用的生命周期钩子：**

```javascript
const handlers = vm.$options[hook]
```

**如：`callHook(vm, created)` 就相当于：**

```javascript
const handlers = vm.$options.created
```

**在上一个章节中，曾讲述过，对于生命周期钩子选项最终会被合并成一个数组，所以得到的 `handlers` 就是对应生命周期钩子的数组，接下来就是执行这段代码：**

```javascript
if (handlers) {
  for (let i = 0, j = handlers.length; i < j; i++) {
    try {
      handlers[i].call(vm)
    } catch (e) {
      handleError(e, vm, `${hook} hook`)
    }
  }
}
```

**为了保证生命周期钩子函数内可以通过 `this` 访问实例对象，所以使用 `.call(vm)` 执行这些函数。生命周期钩子函数是开发者编写，为了捕获可能出现的错误，使用 `try...catch` 语句块，并在 `catch` 语句块中使用 `handleError` 处理错误信息。其中 `handleError` 来自于 `srr/core/util/error.js` 文件中。**