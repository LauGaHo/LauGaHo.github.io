# Vue源码解读—mergeOptions

## `mergeOptions` 函数的三个参数

> **`mergeOptions` 函数来自于 `core/util/options.js` 文件，事实上不仅是 `mergeOptions` 函数，整个文件所做的一切都为了选项的合并。**

**在深入 `mergeOptions` 函数之前，必须搞清楚 `mergeOptions` 的三个参数是什么**

```javascript
vm.$options = mergeOptions(
	resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

**其中第一个参数是通过调用一个函数得到的，这个函数是 `resolveConstructorOptions`，并将 `vm.constructor` 作为参数传递进去。第二个参数 `options` 就是我们调用 Vue 构造函数时传进来的对象，第三个参数是当前的 Vue 实例，我们逐一研究。**

**`resolveConstructorOptions` 是一个函数，这个函数就声明在 `core/instance/init.js` 文件中，如下：**

```javascript
export function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  if (Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    if (superOptions !== cachedSuperOptions) {
      // super option changed,
      // need to resolve new options.
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options (#4976)
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      }
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}
```

**这个函数的作用就是他的名字所说：解析构造函数的 `options`，先看第一句：**

```javascript
let options = Ctor.options
```

**其中 `Ctor` 即传递进来的参数 `vm.constructor`，在我们的例子中他就是 Vue 构造函数，但是当我们使用 `Vue.extend` 创造一个子类并使用子类创造实例时，那么 `vm.constructor` 就不是 Vue 构造函数，而是子类，比如：**

```javascript
const Sub = Vue.extend()
const s = new Sub()
```

**那么 `s.constructor` 自然就是 `Sub` 而非 `Vue`，这里的 `Ctor` 就是 Vue 的构造函数，有关 `Vue.extend()` 的东西，后面会专门讨论。**

**所以在这里，`Ctor.options` 就是 `Vue.options`，然后看到 `resolveConstructorOptions` 的返回值是 `options`。**

```javascript
return options
```

**也就是把 `Vue.options` 返回回去了，所以这个函数就像他的名字那样，是用来获取构造者的 `options` 的，`resolveConstructorOptions` 函数第一句和最后一句中间的 `if` 语句块中的代码，这坨代码的作用：这里比较复杂，比如 `if` 语句中的条件 `Ctor.super` 这个 `super` 是子类才有的属性，如下：**

```javascript
const Sub = Vue.extend()
console.log(Sub.super)	// Vue
```

**这里可以知道，`super` 属性是和 `Vue.extend` 有关系的，除此之外，判断分支中的第一句代码：**

```javascript
const superOptions = resolveConstructorOptions(Ctor.super)
```

**上面的代码又递归调用了 `resolveConstructorOptions` 函数，只不过此时的参数是构造者的父类，之后的代码中，还有一些关于父类的 `options` 属性是否被改变过的判断和操作，但是不论走不走进 `if` 分支，最终返回的都是 `options`**

**根据我们的例子，`resolveConstructorOptions` 函数目前并不会走 `if` 判断分支，即此时这个函数相当于：**

```javascript
export function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  return options
}
```

**所以，此时 `mergeOptions` 函数的第一个参数就是 `Vue.options`，而 `Vue.options` 如下：**

```javascript
Vue.options = {
  components: {
    KeepAlive,
    Transition,
    TransitionGroup,
  },
  directives: {
    model,
    show
  },
  filters: Object.create(null),
  _base: Vue
}
```

**接下来，再看看第二个参数 `options`，这个参数实际上就是我们调用 Vue 构造函数的时候传进来的选项，所以例子中的 `options` 的值如下：**

```javascript
{
  el: '#app',
  data: {
    test: 1
  }
}
```

**而第三个参数 `vm` 就是 Vue 实例本身，综上所述，最终的代码如下：**

```javascript
Vue.options = {
	components: {
		KeepAlive
		Transition,
    TransitionGroup
	},
	directives:{
		model,
    show
	},
	filters: Object.create(null),
	_base: Vue
}
```

**`mergeOptions` 的参数搞懂了，现在直接看 `mergeOptions` 的本体了**



## `mergeOptions` 函数本体

**`mergeOptions` 函数上有一段注释：**

```javascript
/**
 * Merge two option objects into a new one.
 * Core utility used in both instantiation and inheritance.
 */
```

**这段注释的意思是：合并两个选项对象为一个新的对象，这个函数在实例化和继承的时候都有用到，这里需要注意两点**

- **这个函数将会产生一个新的对象**
- **这个函数不仅在实例化对象 (即 `_init` 方法中) 的时候用到，在继承 (`Vue.extend`) 中也有用到，所以这个函数应该是一个用来合并两个选项对象为一个新对象的通用方法**

**`mergeOptions` 函数的本体如下：**

```javascript
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)

  // Apply extends and mixins on the child options,
  // but only if it is a raw options object that isn't
  // the result of another mergeOptions call.
  // Only merged options has the _base property.
  if (!child._base) {
    if (child.extends) {
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }

  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```



## `mergeOptions` 中的 `checkComponents`

**我们逐段分析，先看第一段：**

```javascript
if (process.env.NODE_ENV !== 'production') {
  checkComponents(child)
}
```

**在非生产环境下，会以 `child` 为参数调用 `checkComponents` 方法，`checkComponents` 方法同样定义在 `core/util/options.js` 中：**

```javascript
/**
 * Validate component names
 */
function checkComponents (options: Object) {
  for (const key in options.components) {
    validateComponentName(key)
  }
}
```

**从注释可以得知，这个方法是用来校验组件的名字是否符合要求的，该方法使用一个 `for in` 循环遍历 `options.component` 选项，将每个子组件的名字作为参数传递给 `validateComponentName` 函数，所以 `validateComponentName` 函数才是真正校验名字的函数，该函数就定义在 `checkComponents` 函数下方，源码如下：**

```javascript
export function validateComponentName (name: string) {
  if (!/^[a-zA-Z][\w-]*$/.test(name)) {
    warn(
      'Invalid component name: "' + name + '". Component names ' +
      'can only contain alphanumeric characters and the hyphen, ' +
      'and must start with a letter.'
    )
  }
  if (isBuiltInTag(name) || config.isReservedTag(name)) {
    warn(
      'Do not use built-in or reserved HTML elements as component ' +
      'id: ' + name
    )
  }
}
```

**`validateComponentName` 函数由两个 `if` 语句块组成，所以组件的名字需要满足这两个条件才可以：**

- **组件的名字需要满足正则表达式：`/^[a-zA-Z][\w-]*$/`**
- **需要满足条件：`isBuiltInTag(name) || config.isReservedTag(name)` 不成立，也就是说组件名字不能是 Vue 内置的标签 (`isBuiltInTag(name)`) 以及不能够是保留标签 (`config.isReservedTag(name)`)**



## 允许合并另一个构造函数的 `options`

**继续看下一段代码**

```javascript
if (typeof child === 'function') {
  child = child.options
}
```

**这里说明 `child` 参数除了可以是普通的选项对象外，还可以是一个函数，如果是函数的话，就取函数的 `options` 属性作为新的 `child`，我们所知道的除了 Vue 构造函数本身就具有这个属性之外，其实通过 `Vue.extend` 创造出来的子类 (Vue 组件构造器) 也拥有这个属性，说明进行选项合并的时候，允许合并一个 Vue 组件构造器的选项**

**接着看下一段代码，是用来规范化 `options` 的函数调用：**

```javascript
normalizeProps(child, vm)
normalizeInject(child, vm)
normalizeDirectives(child)
```



## 规范化 `props` (`normalizeProps`)

**`normalizeProps` 是用来规范化 `props` 的，我们在 Vue 中，使用 `props` 的时候有两种写法，一种是字符串数组，如下：**

```javascript
const ChildComponent = {
  props: ['someData']
}
```

**另外一种是使用对象语法：**

```javascript
const ChildComponent = {
  props: {
    someData: {
      type: Number,
      default: 0
    }
  }
}
```

**我们看一下 `normalizeProps` 函数的源码：**

```javascript
/**
 * Ensure all props option syntax are normalized into the
 * Object-based format.
 */
function normalizeProps (options: Object, vm: ?Component) {
  const props = options.props
  if (!props) return
  const res = {}
  let i, val, name
  if (Array.isArray(props)) {
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val)
        res[name] = { type: null }
      } else if (process.env.NODE_ENV !== 'production') {
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  options.props = res
}
```

**根据注释可以知道，这个函数最终将 `props` 规范为对象的形式，比如当 `props` 是一个字符串数组的时候：**

```javascript
props: ["data"]
```

**经过这个函数之后，就会规范成：**

```javascript
props: {
  data: {
    type: null
  }
}
```

**如果对象是这样的：**

```javascript
props: {
  data1: Number,
  data2: {
    type: String,
    default: ''
  }
}
```

**经过这个函数之后就会变成：**

```javascript
props: {
  data1: {
    type: Number
  },
  data2: {
    type: String,
    default: ''
  }
}
```



## 规范化 `inject` (`normalizeInject`)

**接着看下一个规范化函数 `normalizeInject`，源码如下：**

```javascript
/**
 * Normalize all injections into Object-based format
 */
function normalizeInject (options: Object, vm: ?Component) {
  const inject = options.inject
  if (!inject) return
  const normalized = options.inject = {}
  if (Array.isArray(inject)) {
    for (let i = 0; i < inject.length; i++) {
      normalized[inject[i]] = { from: inject[i] }
    }
  } else if (isPlainObject(inject)) {
    for (const key in inject) {
      const val = inject[key]
      normalized[key] = isPlainObject(val)
        ? extend({ from: key }, val)
        : { from: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "inject": expected an Array or an Object, ` +
      `but got ${toRawType(inject)}.`,
      vm
    )
  }
}
```

**其中，我们知道，Vue 中定义 `inject` 可以使用数组或者对象两种方式进行定义的，对应的 `normalizeInject` 就需要对这两种情况进行相应的处理，下面我们举例子来说明：**

```javascript
// 子组件
const ChildComponent = {
  template: '<div>child component</div>',
  created: function () {
    // 这里的 data 是父组件注入进来的
    console.log(this.data)
  },
  inject: ['data']
}

// 父组件
var vm = new Vue({
  el: '#app',
  // 向子组件提供数据
  provide: {
    data: 'test provide'
  },
  components: {
    ChildComponent
  }
})
```

**在上面的代码中，在父组件中定义了 `provide` 属性，在子组件中通过使用数组的方式来定义了 `inject` 属性，同样的，我们也可以用对象的方式来定义 `inject` 属性，如下：**

```javascript
// 子组件
const ChildComponent = {
  template: '<div>child component</div>',
  created: function () {
    console.log(this.d)
  },
  // 对象的语法类似于允许我们为注入的数据声明一个别名
  inject: {
    d: 'data'
  }
}
```

**下面我们看一下规范化后的数据形式是怎样的**

**如果 `inject` 属性是以数组的形式展示：**

```javascript
inject: ['data1', 'data2']
```

**那么他将会被规范为：**

```javascript
inject = {
  'data1': { from: 'data1' },
  'data2': { from: 'data2' }
}
```

**如果 `inject` 属性是以对象的形式展示：**

```javascript
let data1 = 'data1'

inject: {
  data1,
  d2: 'data2',
  data3: { someProperty: 'someValue' }
}
```

**规范化后的数据形式如下：**

```javascript
inject: {
  'data1': { from: 'data1' },
  'd2': { from: 'data2' },
  'data3': { from: 'data3', someProperty: 'someValue' }
}
```



## Vue 选项合并

**前边我们了解了 Vue 对选项的规范化，接下来才是真实的合并阶段，接着看 `mergeOptions` 函数的代码如下：**

```javascript
const options = {}
let key
for (key in parent) {
  mergeField(key)
}
for (key in child) {
  if (!hasOwn(parent, key)) {
    mergeField(key)
  }
}
function mergeField (key) {
  const strat = strats[key] || defaultStrat
  options[key] = strat(parent[key], child[key], vm, key)
}
return options
```

**这段代码的第一句和最后一句说明了 `mergeOptions` 函数确实返回了一个新的对象，因为上方的代码块中的第一句定义了一个常量 `options`，而最后一句代码将其返回，所以中间的代码就是用来填充 `options` 的，`options` 就是最后合并之后的选项，围观一下它的产生过程：**

```javascript
for (key in parent) {
  mergeField(key)
}
```

**这段 `for in` 用来遍历 `parent`，并且将 `parent` 对象的键作为参数传递给 `mergeField` 函数，假设 `parent` 就是 `Vue.options`：**

```javascript
Vue.options = {
  components: {
      KeepAlive,
      Transition,
      TransitionGroup
  },
  directives:{
      model,
      show
  },
  filters: Object.create(null),
  _base: Vue
}
```

**那么 `key` 就应该分别是：`components`、`directives`、`filters` 以及 `_base`，除了 `_base` 其他字段都可以理解为是 Vue 提供的选项的名字。**

**而第二段 `for in` 代码：**

```javascript
for (key in child) {
  if (!hasOwn(parent, key)) {
    mergeField(key)
  }
}
```

**第二段代码中遍历的是 `child` 对象，且增加了一个判断条件：**

```javascript
if (!hasOwn(parent, key))
```

**`hasOwn()` 函数的作用是用来判断一个属性是否是对象自身的属性 (不包括原型上的)，所以这个判断语句具体的意思是：如果 `child` 对象的 `key` 在 `parent` 上出现，那么不要再调用 `mergeField(key)` 方法了，因为在上一个循环中已经调用过了一次了，避免了重复调用，增加不必要的开支。**

**这两个 `for in` 循环的目的是将 `child` 和 `parent` 中的 `key` 作为参数调用 `mergeField(key)` 函数，真正的合并操作是发生在 `mergeField(key)` 中的，而 `mergeField(key)` 的源码如下：**

```javascript
function mergeField (key) {
  const strat = strats[key] || defaultStrat
  options[key] = strat(parent[key], child[key], vm, key)
}
```

**`mergeField` 函数只有两行代码，第一行定义了一个常量 `strat`，他的值是通过使用指定的 `key` 去访问对应的 `strats` 对象得到的，当 `strats` 对象中没有 `key` 对应的 `value`，则会使用 `defaultStrat` 作为值。**

> **`strats` 的定义在 `src/core/util/options.js` 文件中，如下：**

```javascript
/**
 * Option overwriting strategies are functions that handle
 * how to merge a parent option value and a child option
 * value into the final value.
 */
const strats = config.optionMergeStrategies
```

**这行代码就定义了 `strats` 变量的值，它是一个名为 `config.optionMergeStrategies`，这个 `config` 对象是全局配置对象，来自于 `src/core/config.js` 文件，此时 `config.optionMergeStrategies` 只是一个空对象，上边代码的注释的中文意思是：`config.optionMergeStrategies` 是一个合并选项的策略对象，这个对象中包含很多函数，这些函数是合并特定选项的策略。这就可以让不同的选项使用不同的合并策略，如果使用自定义选项，那么也可以自定义该选项的合并策略，只需要在 `Vue.config.optionMergeStrategies` 对象添加和自定义选项同名的函数即可。**



## 选项 `el`、`propsData` 的合并策略

**紧接着将探讨这个策略对象中有哪些策略，看下边的代码：**

>**`src/core/util/options.js` 的开头几行代码**

```javascript
/**
 * Options with restrictions
 */
if (process.env.NODE_ENV !== 'production') {
  strats.el = strats.propsData = function (parent, child, vm, key) {
    if (!vm) {
      warn(
        `option "${key}" can only be used during instance ` +
        'creation with the `new` keyword.'
      )
    }
    return defaultStrat(parent, child)
  }
}
```

**上述代码，非生产环境下在 `strats` 策略对象上添加两个策略 (一个属性对应一个策略) 分别是 `el` 属性和 `propsData` 属性对应的策略，并且这个两个策略对应的是同一个函数，由此可知，合并 `el` 和合并 `propsData` 的方式是一样，毕竟他们的策略都是一致的。**

**首先是一段 `if` 判断语句，判断是否有传递 `vm` 参数：**

```javascript
if (!vm) {
  warn(
    `option "${key}" can only be used during instance ` +
    'creation with the `new` keyword.'
  )
}
```

**如果没有传递这个参数，便会弹出一个警告，提示 `el` 和 `propsData` 只能在使用 `new` 操作符创建实例的时候使用，比如下面的代码：**

```javascript
// 子组件
var ChildComponent = {
  el: '#app2',
  created: function () {
    console.log('child component created')
  }
}

// 父组件
new Vue({
  el: '#app',
  data: {
    test: 1
  },
  components: {
    ChildComponent
  }
})
```

**上面的代码中在父组件中使用了 `el` 选项，这里没有问题，但是在子组件中也使用了 `el` 属性，这里就会喜提警告，说明了一个问题，如果在策略函数中如果拿不到 `vm` 参数，就说明处理的是子组件选项。为了搞清楚这个结论的出处，我们来仔细研究一下整个流程，首先我们一层一层往外看，看看 `vm` 来自哪里，先看 `mergeField(key)` 函数：**

```javascript
function mergeField (key) {
  const strat = strats[key] || defaultStrat
  options[key] = strat(parent[key], child[key], vm, key)
}
```

**而 `mergeField(key)` 函数体中的 `vm` 是来自于 `mergeOptions()` 函数的第三个参数。所以当调用 `mergeOptions()` 函数时不传第三个参数，那么在策略中就会获取不到 `vm` 参数。所以我们可以猜测到一件事，那就是 `mergeOptions()` 函数除了在 `_init()` 函数中被调用之外，还在其他地方被调用，并且是没有传递第三个参数，先解开谜底，就是在 `Vue.extend()` 方法中被调用的，至于 `Vue.extend()` 就来自于 `src/global-api/extend.js` 文件，我们在 `Vue.extend()` 方法体中就能够看到有这么一段代码：**

```javascript
Sub.options = mergeOptions(
  Super.options,
  extendOptions
)
```

**由此可知，此时调用 `mergeOptions()` 被调用时并没有传递第三个参数，也就说通过 `Vue.extend()` 创建子类构造器时 `mergeOptions()` 会被调用，这时 `mergeOptions()` 就会拿不到第三个参数。**

**因此得出结论，通过判断是否存在 `vm` 就能够知道 `mergeOptions()` 是在实例化的时候调用 (使用 `new` 操作符走 `_init()` 方法) 还是在继承时调用 (`Vue.extend()`)，而子组件的实现方式是通过实例化子类完成的，子类有时通过 `Vue.extend()` 创造出来的，所以通过对 `vm` 的判断就可以得知是否是子组件了。**

**结论：如果策略函数中拿不到 `vm` 参数，那么处理的就是子组件的选项**

**`strats.el` 和 `strats.propsData` 策略函数的代码中，最终是调用了 `defaultStrat()` 函数并将 `defaultStrat()` 函数的返回值返回。**

```javascript
return defaultStrat(parent, child)
```

**`defaultStrat()` 函数的定义在 `src/core/util/option.js` 文件中：**

```javascript
/**
 * Default strategy.
 */
const defaultStrat = function (parentVal: any, childVal: any): any {
  return childVal === undefined
    ? parentVal
    : childVal
}
```

**实际上 `defaultStrat()` 函数就如同他的名字一样，是一个默认策略，当一个选项不需要特殊处理的时候就使用默认的合并策略，逻辑很简单：只要子选项不是 `undefined` 就使用子选项，否则使用父选项。**

**有一个地方需要注意点：`strats.el` 和 `strats.propsData` 这两个策略函数只有在非生产环境才有，在生产环境下访问这个两个函数将会得到 `undefined`，这个时候 `mergeField()` 函数中的第一行就起作用了：**

```javascript
// 当一个选项没有对应的策略函数时，使用默认策略函数
const strat = strats[key] || defaultStrat
```

**因此在生产环境下将直接使用默认的策略函数 `defaultStrat()` 函数来处理 `el` 和 `propsData` 这两个选项。**



## 选项 `data` 的合并策略

**接着看 `src/core/util/option.js` 文件的代码，紧接着定义了两个函数：`mergeData()` 以及 `mergeDataOrFn()`，他们都是关于合并 `data` 属性的，但这里直接先看 `strats.data`，从他入手了解 `mergeData()` 和 `mergeDataOrFn()` 这两个函数。**

```javascript
strats.data = function (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    if (childVal && typeof childVal !== 'function') {
      process.env.NODE_ENV !== 'production' && warn(
        'The "data" option should be a function ' +
        'that returns a per-instance value in component ' +
        'definitions.',
        vm
      )

      return parentVal
    }
    return mergeDataOrFn(parentVal, childVal)
  }

  return mergeDataOrFn(parentVal, childVal, vm)
}
```

**这段代码的作用是在 `strats` 策略对象上添加 `data` 策略函数，用来合并处理 `data` 选项。现在对 `data` 策略函数抽丝剥茧分析，首先是第一个判断分支：**

```javascript
if (!vm) {
  ...
}
```

**和 `el`、`propsData` 这两个策略函数相同，先判断是否传递了 `vm` 参数，如果没有 `vm` 参数，处理的是子组件的选项，`if` 判断语句块的代码如下：**

```javascript
if (childVal && typeof childVal !== 'function') {
  process.env.NODE_ENV !== 'production' && warn(
    'The "data" option should be a function ' +
    'that returns a per-instance value in component ' +
    'definitions.',
    vm
  )

  return parentVal
}
return mergeDataOrFn(parentVal, childVal)
```

**首先判断是否传递了子组件的 `data` 选项 (即：`childVal`)，并且检测 `childVal` 的类型是不是 `function`，如果 `childVal` 的类型不是 `function` 则会给出一个警告，也就是说 `childVal` 应该是一个函数，如果不是函数会提示 `data` 的类型必须是一个函数，这就是我们之前一直知道的一个规律：子组件中的 `data` 必须是返回对象的函数。如果不是函数，除了会给出一段警告之外，还会直接返回 `parentVal`。**

**如果 `childVal` 是函数类型，那么说明满足了子组件的 `data` 选项需要是一个函数的要求，那么久直接返回 `mergeDataOrFn()` 函数的执行结果：**

```javascript
return mergeDataOrFn(parentVal, childVal)
```

**上面的情况是在 `strats.data` 策略函数拿不到 `vm` 参数时的情况，如果拿到了 `vm` 参数，那么说明处理的选项不是子组件的选项，而是正常使用 `new` 操作符创建实例时的选项，这个时候则会直接返回 `mergeDataOrFn()` 函数的执行结果，但是会多传一个参数 `vm`**

```javascript
return mergeDataOrFn(parentVal, childVal, vm)
```

**通过上面的分析我们得知一件事，即 `strats.data` 策略函数无论在合并处理的是子组件的选项还是非子组件的选项，最终都是调用 `mergeDataOrFn()` 函数进行处理的，并且以 `mergeDataOrFn()` 函数的返回值作为策略函数的最终返回值。有一点不同的是在处理非子组件选项的时候调用的 `mergeDataOrFn()` 函数多传递一个参数 `vm`。所以我们紧接着就是看 `mergeDataOrFn()` 策略函数的源码，因为 `mergeDataOrFn()` 函数的返回值就等于 `strats.data` 策略函数的返回值：**

```javascript
/**
 * Data
 */
export function mergeDataOrFn (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    // in a Vue.extend merge, both should be functions
    if (!childVal) {
      return parentVal
    }
    if (!parentVal) {
      return childVal
    }
    // when parentVal & childVal are both present,
    // we need to return a function that returns the
    // merged result of both functions... no need to
    // check if parentVal is a function here because
    // it has to be a function to pass previous merges.
    return function mergedDataFn () {
      return mergeData(
        typeof childVal === 'function' ? childVal.call(this, this) : childVal,
        typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
      )
    }
  } else {
    return function mergedInstanceDataFn () {
      // instance merge
      const instanceData = typeof childVal === 'function'
        ? childVal.call(vm, vm)
        : childVal
      const defaultData = typeof parentVal === 'function'
        ? parentVal.call(vm, vm)
        : parentVal
      if (instanceData) {
        return mergeData(instanceData, defaultData)
      } else {
        return defaultData
      }
    }
  }
}
```

**这个函数整体由 `if` 判断分支语句块组成，首先对 `vm` 进行判断，我们知道无论是子组件还是非子组件选项 `strats.data` 策略函数都是通过调用 `mergeDataOrFn()` 函数来完成处理的，且处理非子组件选项的时候要比处理子组件选项时多传递一个参数 `vm`，这就使得 `mergeDataOrFn()` 也能通过是否有 `vm` 来区分处理的对象是子组件选项还是非子组件选项，实际上我们可以看到这里有一段注释：**

```javascript
// in a Vue.extend merge, both should be functions
```

**这段注释的意思是：选项是在调用 `Vue.extend()` 函数时进行合并处理的时候，此时父子 `data` 选项都应该是函数。**

**这里再次说明，当拿不到 `vm` 这个参数的时候，合并操作是在 `Vue.extend()` 函数中进行的，也就是处理子组件的选项。而且此时 `childVal` 和 `parentVal` 都应该是函数，这里的确可以保证 `parentVal` 和 `childVal` 都是函数，之后会讲述到。**

**上面的注释之下，是一段代码：**

```javascript
if (!childVal) {
  return parentVal
}
if (!parentVal) {
  return childVal
}
```

**第一个 `if` 语句块，如果没有 `childVal`，也就是说子组件选项中没有 `data` 选项，那么直接返回 `parentVal`，比如下方的代码：**

```javascript
Vue.extent({})
```

**我们使用 `Vue.extend()` 函数创建子类的时候传递的子组件选项时一个空对象，既没有 `data` 选项，那么此时 `parentVal` 实际上就是就是 `Vue.options`，由于此时 `Vue.options` 上也没有 `data` 这个属性，所以根本上不可能会执行 `strats.data` 策略函数，也就更加不可能会执行 `mergeDataFn()` 函数，此时可能会有一个种想法，对于子组件选项来说，这里的 `return parentVal` 是不是属于多余选项，这里其实并不多余，因为确实存在在子组件选项中，`parentVal` 存在有值的情况 (这里描述情况是：`childVal` 不存在，而 `parentVal` 存在的情况)，下面的代码将描述这种情况：**

```javascript
const Parent = Vue.extend({
  data: function() {
    return {
      test: 1
    }
  }
})

const Child = Parent.extend({})
```

**上面的代码中 `Parent` 类继承了 `Vue`，而 `Child` 类又继承了 `Parent` 类，这里关键就在于我们使用 `Parent.extend` 创建 `Child` 子类的时候，对于 `Child` 子类来讲，`childVal` 不存在，因为我们没有传递 `data` 选项，但是 `parentVal` 的的确确是存在，这里可能会有人对 `parentVal` 为什么会存在，在哪里生成会有一个大大的黑人问号。实际上就是 `Vue.extend()` 函数内使用 `mergeOptions()` 函数生成的，所以此时的 `parentVal` 必定是一个函数，因为 `strats.data` 策略函数在处理 `data` 选项后返回的始终都是一个函数。 **

**所以这里也就是解答了上边的提出的问题：”在处理子组件中的选项中，能够保证 `childVal` 和 `parentVal` 这两个属性如果存在，那么他们的存在方式必定是以 `function` 类型的方式。“也就是解答了下边的这段代码和注释的内容：**

```javascript
// in a Vue.extend merge, both should be functions
```

```javascript
if (!childVal) {
  return parentVal
}
if (!parentVal) {
  return childVal
}
```

**由于 `childVal` 和 `parentVal` 必定会有一个，否则就不会执行 `strats.data` 策略函数，所以上面判断的意思是：*如果没有子选项就使用父选项，如果没有父选项就使用子选项，并且这两个选项都能够保证是函数，如果父子选项都存在的话，那么代码将继续执行，执行下边的代码：***

```javascript
// when parentVal & childVal are both present,
// we need to return a function that returns the
// merged result of both functions... no need to
// check if parentVal is a function here because
// it has to be a function to pass previous merges.
return function mergedDataFn () {
  return mergeData(
    typeof childVal === 'function' ? childVal.call(this, this) : childVal,
    typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
  )
}
```

**上方代码的意思是：*如果父子选项同时存在，则返回一个函数 `mergeDataFn()` 函数，*注意：此时代码运行就结束了，因为函数已经 `return` 了，至于 `mergeDataFn()` 函数中返回了 `mergeData()` 函数的执行结果的这一句代码并不是当前处理的。**

**以上就是 `strats.data` 策略函数在处理子组件的 `data` 选项时所做的事，可以发现 `mergeDataOrFn()` 函数在处理子组件选项时总是返回一个函数，这导致了 `strats.data` 策略函数在处理子组件选项时返回的总是一个函数。**

**讨论完处理子组件选项的情况，再看一下处理非子组件选项的情况，也就是使用 `new` 操作符创建 Vue 实例的情况，此时程序直接执行 `strats.data` 函数的最后一句代码：**

```javascript
return mergeDataOrFn(parentVal, childVal, vm)
```

**这里发现同样是调用 `mergeDataOrFn()` 函数，只不过这个时候传递了 `vm` 参数，也就是说这将会执行 `mergeDataOrFn()` 函数中的 `else` 分支：**

```javascript
if (!vm) {
  ...
}
else {
  return function mergedInstanceDataFn () {
    // instance merge
    const instanceData = typeof childVal === 'function' ? childVal.call(vm, vm) : childVal
    const defaultData = typeof parentVal === 'function' ? parentVal.call(vm, vm) : parentVal
    
    if (instanceData) {
      return mergeData(instanceData, defaultData)
    }
    else {
      return defaultData
    }
  }
}
```

**如果走了 `else` 分支，那么直接返回 `mergedInstanceFn()` 函数，此时 `mergedInstanceFn()` 函数同样也是还没执行，它是作为 `mergeDataOrFn()` 函数的返回值。**

**所以这里得出了一个结论：*`mergeDataOrFn()` 函数永远返回一个函数。***

**这里结合例子来说明：**

```javascript
let vm = new Vue({
  el: '#app',
  data: {
    test: 1
  }
})
```

**`data` 选项在经过 `mergeOptions()` 函数处理之后就会变成一个函数，而且根据分析，他应该就是 `mergedInstanceDataFn()` 函数，在控制台打印如下信息：**

```javascript
console.log(vm.$options)
```

**输出的结果如下图：**

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/IY0DRH.png)

**从中确实可以看到 `data` 选项被 `mergeOptions()` 函数处理成了一个函数，并且当 `data` 选项为非子组件的选项时，该函数就是 `mergedInstanceDataFn()`。**

**一个简单的总结：*`data` 选项最终被 `mergeOptions()` 函数处理成了一个函数，当合并处理的是子组件的选项时，`data` 函数可能是以下三者之一：***

- **`data` 本身。因为子组件的 `data` 选项本身就是一个函数，如下 `mergeDataOrFn()` 函数的代码段所示：**

  ```javascript
  export function mergeDataOrFn (
    parentVal: any,
    childVal: any,
    vm?: Component
  ): ?Function {
    if (!vm) {
      ...
      // 返回子组件的 data 选项本身
      if (!parentVal) {
        return childVal
      }
      ...
    } else {
      ...
    }
  }
  ```

- **父类的 `data` 选项。如下代码段所示：**

  ```javascript
  export function mergeDataOrFn (
    parentVal: any,
    childVal: any,
    vm?: Component
  ): ?Function {
    if (!vm) {
      ...
      // 返回父类的 data 选项
      if (!childVal) {
        return parentVal
      }
      ...
    } else {
      ...
    }
  }
  ```

- **`mergedDataFn` 函数，如下代码段所示：**

  ```javascript
  export function mergeDataOrFn (
    parentVal: any,
    childVal: any,
    vm?: Component
  ): ?Function {
    if (!vm) {
      ...
      // 返回 mergedDataFn 函数
      return function mergedDataFn () {
        return mergeData(
          typeof childVal === 'function' ? childVal.call(this, this) : childVal,
          typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
        )
      }
    } else {
      ...
    }
  }
  ```

**当处理的是非子组件的选项时，`data` 函数为 `mergedInstanceDataFn()` 函数，如下代码所示：**

```javascript
export function mergeDataOrFn (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    ...
  } else {
    // 当合并处理的是非子组件的选项时 `data` 函数为 `mergedInstanceDataFn` 函数
    return function mergedInstanceDataFn () {
      // instance merge
      const instanceData = typeof childVal === 'function'
        ? childVal.call(vm, vm)
        : childVal
      const defaultData = typeof parentVal === 'function'
        ? parentVal.call(vm, vm)
        : parentVal
      if (instanceData) {
        return mergeData(instanceData, defaultData)
      } else {
        return defaultData
      }
    }
  }
}
```

**因此，再次强调结论：*`data` 选项最终被处理成一个函数。但是根据之前的分析可以得知，函数是分为几种情况的，但是他们都有一个共同的特点：这些函数的执行结果就是最终的 `data` 数据！！！这里指的是最终数据！！！注意最终和数据这两个词语的意思！！！***

**从源码处可以发现 `mergedDataFn()` 函数和 `mergedInstanceDataFn()` 函数都有一个共同的特点：*内部都调用了 `mergeData()` 处理数据并返回，先看一下 `mergedDataFn()` 函数，源码如下：***

```javascript
return function mergedDataFn () {
  return mergeData(
  	typeof childVal === 'function' ? childVal.call(this, this) : childVal,
    typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
  )
}
```

**这个函数直接返回了 `mergeData()` 函数的执行结果，再看看 `mergedInstanceDataFn()` 函数，源码如下：**

```javascript
return function mergedInstanceDataFn () {
  // instance merge
  const instanceData = typeof childVal === 'function'
    ? childVal.call(vm, vm)
    : childVal
  const defaultData = typeof parentVal === 'function'
    ? parentVal.call(vm, vm)
    : parentVal
  if (instanceData) {
    return mergeData(instanceData, defaultData)
  } else {
    return defaultData
  }
}
```

**这里可以注意到 `mergedDataFn()` 和 `mergedInstanceDataFn()` 这两个函数都有类似这样的代码：**

```javascript
typeof childVal === 'function' ? childVal.call(this, this) : childVal
typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
```

**`childVal` 要么是子组件的选项，要么是 `new` 操作符创建实例时的选项，无论是哪种情况，`childVal` 要么是函数，要么是一个纯对象。所以如果是函数的话就通过执行该函数从而得到一个纯对象，所以类似上面那段代码中判断 `childVal` 和 `parentVal` 的类型是否是函数的目的只有一个：*获取数据对象 (纯对象)。*所以 `mergedDataFn()` 和 `mergedInstanceDataFn()` 函数内部调用 `mergeData` 方法时传递的两个参数就是两个纯对象 (简单理解就是两个 JSON 对象)**

**通过看 `mergeData` 的源码，可以看到究极终极版合并策略，如下：**

 