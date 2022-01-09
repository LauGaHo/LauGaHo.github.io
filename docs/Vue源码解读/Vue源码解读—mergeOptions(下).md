# Vue源码解读—mergeOptions(下)

## 生命周期钩子 `hook` 选项的合并策略

**上篇解读完 `strats.data` 策略函数，紧接着了解生命周期钩子的合并策略。**

```javascript
/**
 * Hooks and props are merged as arrays.
 */
function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  return childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
}

LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})
```

**上方的这段代码就是用来合并生命周期钩子的，慢慢剖析这段代码：**

```javascript
LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})
```

**`LIFECYCLE_HOOK` 这个常量在 `shared/constants.js` 文件中定义：**

```javascript
export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured',
  'serverPrefetch'
]
```

**由此可知，`LIFECYCLE_HOOK` 常量实际上是由与生命周期钩子同名的字符串组成的数组。**

**所以那段 `forEach` 语句可知，作用就是：*在 `strats` 策略对象上添加用来合并各个生命周期钩子选项的策略函数，并且这些生命周期钩子选项的策略函数相同，都是 `mergeHook()` 函数。***

**直接看 `mergeHook()` 函数的代码：**

```javascript
function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  return childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
}
```

**整个函数体由三组三目运算符组成，上方代码的意思转换一下就是：**

```javascript
return (是否有 childVal，即判断组件的选项中是否有对应名字的生命周期钩子函数)
  ? 如果有 childVal 则判断是否有 parentVal
    ? 如果有 parentVal 则使用 concat 方法将二者合并为一个数组
    : 如果没有 parentVal 则判断 childVal 是不是一个数组
      ? 如果 childVal 是一个数组则直接返回
      : 否则将其作为数组的元素，然后返回数组
  : 如果没有 childVal 则直接返回 parentVal
```

**这里需要注意的是：*组件选项的生命周期钩子函数被合并成一个数组，第一个三目运算符判断是否有 `childVal`，即组件的选项是否写了生命周期钩子函数，如果没有直接返回 `parentVal`，这里存在一个问题：`parentVal` 一定是数组吗？答案是：如果有 `parentVal` 那么他一定是一个数组，如果没有 `parentVal` 那么 `strats[hooks]` 根本不会执行。以 `created` 生命周期钩子函数为例：***

```javascript
new Vue({
  created: function () {
    console.log('created')
  }
})
```

**以这段代码为例，对于 `strats.created` 策略函数来讲 (这里的 `strats.created` 就是 `mergeHooks()`)，`childVal` 就是例子中的 `created` 选项，他是一个函数。`parentVal` 就是 `Vue.options.created`，但是 `Vue.options.created` 是不存在的，所以最终经过 `strats.created` 函数的处理将返回一个数组。**

```javascript
options.created = [
  function () {
    console.log('created')
  }
]
```

**再看下边的例子：**

```javascript
const Parent = Vue.extend({
  created: function () {
    console.log('parentVal')
  }
})

const Child = new Parent({
  created: function () {
    console.log('childVal')
  }
})
```

**其中 `Child` 是使用 `new Parent()` 生成的，所以对于 `Child` 来讲，`childVal` 是：**

```javascript
created: function () {
  console.log('childVal')
}
```

**而 `parentVal` 已经不是 `Vue.options.created` 了，而是 `Parent.options.created`，那么 `Parent.options.created` 是通过 `Vue.extend()` 函数内部的 `mergeOptions()` 处理过的，所以它应该长这样的：**

```javascript
Parent.options.created = [
  created: function () {
    console.log('parentVal')
  }
]
```

**所以这里既有 `childVal`，又有 `parentVal`，则根据 `mergeHooks()` 函数的逻辑：**

```javascript
function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  return childVal
    ? parentVal
      // 这里，合并且生成一个新数组
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
}
```

**`parentVal.concat(childVal)` 将 `parentVal` 和 `childVal` 合并成一个数组，最终结果如下：**

```javascript
[
  created: function () {
    console.log('parentVal')
  },
  created: function () {
    console.log('childVal')
  }  
]
```

**另外还有第三个三目运算符：**

```javascript
: Array.isArray(childVal)
	? childVal
	: [childVal]
```

**由此可以知道，生命周期钩子是可以写成数组，如下：**

```javascript
new Vue({
  created: [
    function () {
      console.log('first')
    },
    function () {
      console.log('second')
    },
    function () {
      console.log('third')
    }
  ]
})
```

**钩子函数将顺序执行。**



## 资源 `assets` 选项的合并策略

**在 Vue 中 `directives`、`filters`、`components` 被认为是资源，指令、过滤器、组件都是可以作为第三方应用来提供的。**

**紧接着看如何合并 `directives`、`filters`、`components` 资源选项：**

```javascript
/**
 * Assets
 *
 * When a vm is present (instance creation), we need to do
 * a three-way merge between constructor options, instance
 * options and parent options.
 */
function mergeAssets (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): Object {
  const res = Object.create(parentVal || null)
  if (childVal) {
    process.env.NODE_ENV !== 'production' && assertObjectType(key, childVal, vm)
    return extend(res, childVal)
  } else {
    return res
  }
}

ASSET_TYPES.forEach(function (type) {
  strats[type + 's'] = mergeAssets
})
```

**和生命周期钩子的合并处理策略基本一致，首先 `ASSET_TYPES` 是一个来自 `shared/constants.js` 常量，出处如下：**

```javascript
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
```

**代码中遍历 `ASSET_TYPES` 常量，然后在每一个字母的后边加上个 `s` 就变成了资源选项的名字了。接下来就看 `mergeAssets()` 代码如下：**

```javascript
function mergeAssets (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): Object {
  const res = Object.create(parentVal || null)
  if (childVal) {
    process.env.NODE_ENV !== 'production' && assertObjectType(key, childVal, vm)
    return extend(res, childVal)
  } else {
    return res
  }
}
```

**上方代码的意思：*首先以 `parentVal` 为原型创建 `res` ，然后判断是否有 `childVal`，如果有的话，使用 `extend()` 函数将 `childVal` 上的属性混合到 `res` 对象上并返回。如果没有 `childVal` 则直接返回 `res`。***

**Vue 中本身也存在着各种内置的组件，例如：`<transition/>` 或者 `<keep-alive/>` 组件，但是我们并没有显式声明这些组件，这里实现的地方就在 `mergeAssets()` 中，下方将详细解释：**

```javascript
const vm = new Vue({
  el: '#app',
  components: {
    ChildComponent: ChildComponent
  }
})
```

**上方的代码中，创建了一个 Vue 实例，并注册了一个子组件 `ChildComponent`，此时 `mergeAssets()` 方法中的 `childVal` 就是本例中的 `components` 选项：**

```javascript
components: {
  ChildComponent: ChildComponent
}
```

**而 `parentVal` 就是 `Vue.options.components`，`Vue.options` 如下：**

```javascript
Vue.options = {
  components: {
    KeepAlive,
    Transition,
    TransitionGroup
  },
  directives: Object.create(null),
  directives: {
    model,
    show
  },
  filter: Object.create(null),
  _base: Vue
}
```

**由此可得：`Vue.options.components` 如下：**

```javascript
{
  KeepAlive,
  Transition,
  TransitionGroup
}
```

**即：*`parentVal` 是如上包含三个内置组件的对象，经过下边的这行代码后：***

```javascript
const res = Object.create(parentVal || null)
```

**你可以通过 `res.KeepAlive` 访问到 `KeepAlive` 对象，虽然 `res` 对象上自身没有 `KeepAlive`，但是他的原型上有。**

**然后再经过 `return extend(res, childVal)` 这行代码后，`res` 变量就会被添加 `ChildComponent` 属性，最后 `res` 如下：**

```javascript
res = {
  ChildComponent,
  __proto__: {
    KeepAlive,
    Transition,
    TransitionGroup
  }
}
```

**这就是不用声明 Vue 的内置组件依然能够使用他们的原因，通过 `Vue.extend()` 创建出来的子类也是一样的道理，一层一层通过原型进行组件的查找。**

**还有一个注意点：`mergeAssets()` 函数中的这句话：**

```javascript
process.env.NODE_ENV !== 'prodection' && assertObjectType(key, childVal, vm)
```

**在非生产环境下，会调用 `assertObjectType()` 函数，这个函数其实是用来检测 `childVal` 是否是一个纯对象，如果不是一个纯对象则会给出一个警告，源码如下：**

```javascript
function assertObjectType (name: string, value: any, vm: ?Component) {
  if (!isPlainObject(value)) {
    warn(
      `Invalid value for option "${name}": expected an Object, ` +
      `but got ${toRawType(value)}.`,
      vm
    )
  }
}
```



## 监视器 `watch` 的合并策略

**紧接着就是 `watch` 监视器的合并策略了：**

```javascript
/**
 * Watchers.
 *
 * Watchers hashes should not overwrite one
 * another, so we merge them as arrays.
 */
strats.watch = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  // work around Firefox's Object.prototype.watch...
  if (parentVal === nativeWatch) parentVal = undefined
  if (childVal === nativeWatch) childVal = undefined
  /* istanbul ignore if */
  if (!childVal) return Object.create(parentVal || null)
  if (process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = {}
  extend(ret, parentVal)
  for (const key in childVal) {
    let parent = ret[key]
    const child = childVal[key]
    if (parent && !Array.isArray(parent)) {
      parent = [parent]
    }
    ret[key] = parent
      ? parent.concat(child)
      : Array.isArray(child) ? child : [child]
  }
  return ret
}
```

**这段代码为  `strats` 策略对象添加 `watch` 策略函数。所以 `strats.watch` 策略函数是处理合并 `watch` 选项的。先看开头两行：**

```javascript
// work around Firefox's Object.prototype.watch...
if (parentVal === nativeWatch) parentVal = undefined
if (childVal === nativeWatch) childVal = undefined
```

**其中 `nativeWatch` 来自于 `src/core/util/env.js` 文件，在 `Firefox` 浏览器 中，`Object.prototype` 拥有原生的 `watch` 函数，所以即便一个普通的对象没有定义 `watch` 属性，依然能够通过原型链访问到原生的 `watch` 属性，这里就会跟 Vue 中的 `watch` 属性造成混淆，因为 Vue 也提供了一个名为 `watch` 的属性，如果开发者没有配置 `watch` 选项，但是当需要在某个对象中查找开发者是否有定义 `watch` 属性时， Vue 可以通过原型链访问到原生的 `watch`。所以上述两行代码的作用就是：*当发现组件选项时浏览器原生的 `watch`，说明用户没有提供 Vue 的 `watch` 选项，直接重置为 `undefined`。***

**接着就是这句代码：**

```javascript
if (!childVal) return Object.create(parentVal || null)
```

**检测是否有 `childVal`，即组件选项是否有 `watch` 选项，如果没有的话，直接以 `parentVal` 为原型创建对象并返回 (如果有 `parentVal` 的话)。**

**如果组件选项中有 `watch` 选项，即存在 `childVal`，则代码继续往下执行。**

```javascript
if (process.env.NODE_ENV !== 'production') {
  assertObjectType(key, childVal, vm)
}
if (!parentVal) return childVal
```

**由于此时 `childVal` 存在，所以在非生产环境下使用 `assertObjectType()` 函数对 `childVal` 进行类型检测，检测其是否是一个纯对象，因为 Vue 的 `watch` 选项是需要一个纯对象。接着判断是否有 `parentVal`，如果没有的话则直接返回 `childVal`，即直接使用组件选项的 `watch`。**

**如果存在 `parentVal`，那么代码继续执行，此时 `parentVal` 以及 `childVal` 都将存在，那么就需要做合并处理了，如下代码：**

```javascript
// 定义 ret 常量，其值为一个对象
const ret = {}
// 将 parentVal 的属性混合到 ret 中，后面处理的都将是 ret 对象，最后返回的也是 ret 对象
extend(ret, parentVal)
// 遍历 childVal
for (const key in childVal) {
  // 由于遍历的是 childVal，所以 key 是子选项的 key，父选项中未必能获取到值，所以 parent 未必有值
  let parent = ret[key]
  // child 是肯定有值的，因为遍历的就是 childVal 本身
  const child = childVal[key]
  // 这个 if 分支的作用就是如果 parent 存在，就将其转为数组
  if (parent && !Array.isArray(parent)) {
    parent = [parent]
  }
  ret[key] = parent
    // 最后，如果 parent 存在，此时的 parent 应该已经被转为数组了，所以直接将 child concat 进去
    ? parent.concat(child)
    // 如果 parent 不存在，直接将 child 转为数组返回
    : Array.isArray(child) ? child : [child]
}
// 最后返回新的 ret 对象
return ret
```

**定义了 `ret` 常量，最后返回的也是 `ret` 常量，用 `extend()` 函数将 `parentVal` 属性融进 `ret` 对象中，然后一个 `for in` 循环遍历 `childVal`，这个循环的目的：*检测子选项中的值是否也存在于父选项中，如果存在的话，将父子选项合并到数组中，否则直接将子选项变成一个数组返回。***

**举例说明该过程：**

```javascript
// 创建子类
const Sub = Vue.extend({
  // 检测 test 的变化
  watch: {
    test: function () {
      console.log('extend: test change')
    }
  }
})

// 使用子类创建实例
const v = new Sub({
  el: '#app',
  data: {
    test: 1
  },
  // 检测 test 的变化
  watch: {
    test: function () {
      console.log('instance: test change')
    }
  }
})

// 修改 test 的值
v.test = 2
```

**上方的代码中，当我们修改 `v.test` 的值时，两个观察 `test` 变化的函数都将被执行。**

**使用子类 `Sub` 创建了实例 `v`，对于实例 `v` 来讲，其 `childVal` 就是组件选项的 `watch`：**

```javascript
watch: {
  test: function () {
    console.log('instance: test change')
  }
}
```

**而其 `parentVal` 就是 `Sub.options`，实际上就是：**

```javascript
watch: {
  test: function () {
    console.log('extend: test change')
  }
}
```

**最终这两个 `watch` 选项将被合并成一个数组：**

```javascript
watch: {
  test: [
    function () {
      console.log('extend: test change')
    },
    function () {
      console.log('instance: test change')
    }
  ]
}
```

**`watch.test` 并不一定总是数组，只有父选项 (`parentVal`) 也存在对该字段的观测，他才是数组，如下：**

```javascript
// 创建实例
const v = new Vue({
  el: '#app',
  data: {
    test: 1
  },
  // 检测 test 的变化
  watch: {
    test: function () {
      console.log('instance: test change')
    }
  }
})

// 修改 test 的值
v.test = 2
```

**我们直接使用 Vue 创建实例，这个时候对于实例 `v` 来说，父选项是 `Vue.options`，由于 `Vue.options` 并没有 `watch` 选项，所以逻辑将直接在 `strats.watch` 函数的这一行直接返回，不会执行下方的代码：**

```javascript
if (!parentVal) return childVal
```

**没有 `parentVal` 即父选项中没有 `watch` 选项，则直接返回 `childVal` (相当于直接返回子选项的 `watch` 选项)，就如上方的例子一般。**

**结论：*被合并处理后的 `watch` 选项下的每一对 `key-value` 有可能是一个数组，也有可能是一个函数。***



## 选项 `props`、`methods`、`inject`、`computed` 的合并策略

**先看一段代码：**

```javascript
/**
 * Other object hashes.
 */
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  if (childVal && process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = Object.create(null)
  extend(ret, parentVal)
  if (childVal) extend(ret, childVal)
  return ret
}
```

**这段代码：*在 `strats` 策略对象上添加 `props`、`methods`、`inject` 以及 `computed` 策略函数，他们的合并策略相同。***

**对于 `props`、`methods`、`inject` 以及 `computed` 这四个选项有一个共同点，他们的结构都是纯对象，即使我们书写的时候有可能写成数组等形式，但是 Vue 会帮我们进行 `normalize` 操作变成一个对象，接着看看 Vue 如果处理这些对象散列。**

**策略函数如下：**

```javascript
// 如果存在 childVal，那么在非生产环境下要检查 childVal 的类型
if (childVal && process.env.NODE_ENV !== 'production') {
  assertObjectType(key, childVal, vm)
}
// parentVal 不存在的情况下直接返回 childVal
if (!parentVal) return childVal
// 如果 parentVal 存在，则创建 ret 对象，然后分别将 parentVal 和 childVal 的属性混合到 ret 中，注意：由于 childVal 将覆盖 parentVal 的同名属性
const ret = Object.create(null)
extend(ret, parentVal)
if (childVal) extend(ret, childVal)
// 最后返回 ret 对象。
return ret
```

**结论：*检测是否存在 `childVal` 和 `parentVal`。有 `childVal`，没有 `parentVal` 则直接返回 `childVal`，如果存在 `parentVal`，则将 `childVal` 和 `parentVal` 进行合并操作，并且 `childVal` 中的属性会覆盖 `parentVal` 中同名的 `key-value`。***



## 选项 `provide` 的合并策略

**最后一个选项的合并策略，就是 `provide` 选项的合并策略，只有一句代码，如下：**

```javascript
strats.provide = mergeDataOrFn
```

**`provide` 选项的合并策略和 `data` 选项的合并策略相同，都是使用 `mergeDataOrFn()` 函数。**



## 选项处理总结

- **对于 `el`、`propsData` 选项使用默认合并策略 `defaultStrat`。**
- **对于 `data` 选项，使用 `mergeDataOrFn()` 函数进行处理，最终结果是将 `data` 选项变成一个函数，并且该函数的执行结果为真正的数据对象。**
- **对于生命周期钩子选项，将合并成数组，父子选项中的钩子函数都能够被执行。**
- **对于 `directives`、`filters`、`components` 等资源选项，父子选项将以原型链的形式被处理，正是因为这样，我们才能在任何地方使用内置组件和内置指令。**
- **对于 `watch` 选项的合并处理，类似于生命周期钩子，如果父子选项都有相同的观测字段，将被合并成数组，这样观察者都将被执行。**
- **对于 `props`、`methods`、`inject`、`computed` 选项，父选项始终可用，但是子选项会覆盖同名的父选项字段。**
- **对于 `provide` 选项，合并策略使用与 `data` 选项相同的 `mergeDataOrFn()`。**
- **最后，没有提及到的选项都将使用默认选项 `defaultStrat`。**
- **`defaultStrat` 的合并策略：只要子选项不是 `undefined` 就使用子选项，否则使用父选项。**



## 再看 `mixins` 和 `extends`

**在上一章节中，讲到了 `mergeOptions()` 函数中有如下代码：**

```javascript
const extendsFrom = child.extends
if (extendsFrom) {
  parent = mergeOptions(parent, extendsFrom, vm)
}
if (child.mixins) {
  for (let i = 0, l = child.mixins.length; i < l; i++) {
    parent = mergeOptions(parent, child.mixins[i], vm)
  }
}
```

**当时没有深入讲述，因为当时对 `mergeOptions()` 函数的了解不深，现在回头看一下这段代码。**

**首先看 `mixins`，比如混入 `created` 生命周期钩子。**

```javascript
const consoleMixin = {
  created () {
    console.log('created:mixins')
  }
}

new Vue ({
  mixins: [consoleMixin],
  created () {
    console.log('created:instance')
  }
})
```

**上方代码运行，将打印两句话：**

```javascript
// created:mixins
// created:instance
```

**这是因为 `mergeOptions()` 函数在处理 `mixins` 选项的时候递归调用了 `mergeOptions()` 函数将 `mixins` 合并到了 `parent` 中，并将合并后生成的新对象作为新的 `parent`。**

```javascript
if (child.mixins) {
  for (let i = 0, l = child.mixins.length; i < l; i++) {
    parent = mergeOptions(parent, child.mixins[i], vm)
  }
}
```

**上方的例子中，我们只涉及到了 `created` 生命周期钩子的合并。透过该过程知道，所有写在 `mixins` 中的选项，都会使用 `mergeOptions()` 中相应的合并策略进行处理，这就是 `mixins` 的实现方式。**

**对于 `extends` 选项，和 `mixins` 相同，甚至由于 `extends` 选项只能是一个对象，不能是数组，实现起来比 `mixins` 还要简单。**
