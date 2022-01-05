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

