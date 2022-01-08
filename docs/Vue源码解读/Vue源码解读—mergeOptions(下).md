# Vue源码解读—mergeOptions(下)

## 生命周期钩子选项的合并策略

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



## 资源 (assets) 选项的合并策略

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

