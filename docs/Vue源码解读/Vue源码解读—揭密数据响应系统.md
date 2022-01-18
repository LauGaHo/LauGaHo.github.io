# Vue源码解读—揭密数据响应系统

## 实例对象代理访问数据 `data`

**`initData()` 函数和 `initState()` 函数定义在同一个文件中，即在 `src/core/instance/state.js` 文件中，`initData()` 函数的开头几行代码如下：**

```javascript
let data = vm.$options.data
data = vm._data = typeof data === 'function'
  ? getData(data, vm)
  : data || {}
```

**定义 `data` 变量，它是 `vm.$options.data` 的引用。在前边的文章中，可以得知 `vm.$options.data` 被处理成了一个函数，并且该函数的执行结果才是真正的结果，但是上边的代码中，仍然是对 `data` 进行了一个 `typeof` 的判断，这里的意义在于，因为 `beforeCreate` 生命周期钩子函数是在 `mergeOptions()` 函数之后和 `initData()` 函数之前被调用，如果在 `beforeCreate` 生命周期钩子函数中修改了 `vm.$options.data` 的值，那么在 `initData()` 对 `vm.$option.data` 类型判断就显得很有必要。**

**如果 `vm.$options.data` 的类型是 `function` 的话，那么将调用 `getData()` 函数获取真正的数据，下边是 `getData()` 函数的源码：**

```javascript
export function getData (data: Function, vm: Component): any {
  // #7573 disable dep collection when invoking data getters
  pushTarget()
  try {
    return data.call(vm, vm)
  } catch (e) {
    handleError(e, vm, `data()`)
    return {}
  } finally {
    popTarget()
  }
}
```

**`getData()` 函数接收两个参数：一个是 `data` 选项 (类型为 `function`)，另一个是 Vue 实例对象。`getData()` 函数的作用就是通过调用 `data()` 函数获取真正的数据对象并将其返回，即：`data.call(vm)`，而且 `data.call(vm)` 被包裹在 `try...catch` 语句中，这里是为了捕获 `data()` 函数中的可能出现的错误，如果发生错误，则返回一个空对象作为数据对象：`return {}`。**

**另外在 `getData()` 函数开头使用了 `pushTarget()` 函数，并且在 `finally` 语句块中调用了 `popTarget()`，这样做的目的是防止使用 `props` 数据初始化 `data` 数据时收集冗余依赖，总结：*`getData()` 函数的作用是通过调用 `data` 选项的函数获取数据对象。***

**回到 `initData()` 函数中：**

```javascript
data = vm._data = getData(data, vm)
```

**当通过 `getData()` 函数拿到最终的数据对象后，将该返回的对象赋值给 `vm._data` 属性，同时重写了 `data` 变量，至此 `data` 变量和 `vm.$options.data` 不再是函数，而是变成了一个具体的数据对象了。**

**紧接着就是一个 `if` 判断语句：**

```javascript
if (!isPlainObject(data)) {
  data = {}
  process.env.NODE_ENV !== 'production' && warn(
    'data functions should return an object:\n' +
    'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
    vm
  )
}
```

**上方代码使用了 `isPlainObject()` 函数判断变量 `data` 是否为一个纯对象，如果不是纯对象，且在非生产环境下会打印警告信息。如下方例子：**

```javascript
new Vue({
  data () {
    return '不返回对象'
  }
})
```

**上方代码中的 `data()` 函数返回了一个字符串，而不是返回对象，因此需要判断 `data` 函数的返回值是否为 `PlainObject` 类型 (是否是一个对象)。**

**再往下就是这段代码了：**

```javascript
// proxy data on instance
const keys = Object.keys(data)
const props = vm.$options.props
const methods = vm.$options.methods
let i = keys.length
while (i--) {
  const key = keys[i]
  if (process.env.NODE_ENV !== 'production') {
    if (methods && hasOwn(methods, key)) {
      warn(
        `Method "${key}" has already been defined as a data property.`,
        vm
      )
    }
  }
  if (props && hasOwn(props, key)) {
    process.env.NODE_ENV !== 'production' && warn(
      `The data property "${key}" is already declared as a prop. ` +
      `Use prop default value instead.`,
      vm
    )
  } else if (!isReserved(key)) {
    proxy(vm, `_data`, key)
  }
}
```

**上方代码：*先获取 `data` 属性中对象的所有 `key` 并赋值给 `keys` 变量。然后分别将 `vm.$options.props` 和 `vm.$options.methods` 赋值给 `props` 和 `methods`。接着开启一个 `while` 循环，该循环是用于遍历 `keys` 变量对应的数组。循环的目的是：判断 `data` 中的 `key` 是否在 `methods` 和 `props` 对象中的 `key` 发生冲突，如果发生了冲突，就会打印一个警告，这里还有一个优先级：`props` > `data` > `methods`，由此可见如果一个 `key` 定义在了 `props`，那么在 `methods` 和 `data` 属性上就不可以出现对应的 `key`。如果一个 `key` 在 `data` 中出现了，那么在 `methods` 就不能出现了。最后在 `else if` 分支中，有一个条件：`!isReserved(key)`，该条件的意思是：判断定义在 `data` 属性上的 `key` 是否为保留键。`isReserved()` 函数通过判断一个字符串的第一个字符串是不是以 `$` 或 `_` 来决定是否保留，Vue 是不会代理那些键名以 `$` 或 `_` 开头的字段，因为 Vue 自身的属性和方法都是以 `$` 或 `_` 开头，这样是为了避免和 Vue 自身属性和方法冲突。***

**如果 `key` 满足不是 `$` 和 `_` 开头，将执行 `proxy()` 函数，实现实例对象的代理访问：**

```javascript
proxy(vm, `_data`, key)
```

**接着看 `proxy()` 函数的源码：**

```javascript
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

**`proxy()` 函数的原理是通过 `Object.defineProperty()` 函数在实例对象 `vm` 上定义和 `data` 数据字段同名的访问器属性，并且这些属性代理的值是 `vm._data` 上对应属性的值，下边举例说明：**

```javascript
const test = new Vue({
  data: {
    a: 1
  }
})
```

**当我们访问 `test.a` 的时候，其实我实际上访问的是 `test._data.a`。`test._a` 才是真正的数据对象。**

**最后经过一系列的处理之后，`initData()` 函数来到了最后一句：**

```javascript
// observe data
observe(data, true/*asRootData*/)
```

**调用 `observe()` 函数将 `data` 数据转换成响应式，这一句话就是响应式系统的真正开端。**

**剖析 `observe()` 函数之前，总结一下 `initData()` 函数：**

- ***根据 `vm.$options.data` 选项获取真正想要的数据 (这里的 `vm.$options.data` 是一个函数！！！)***
- ***校验得到的数据是否是一个纯对象***
- ***检查数据对象 `data` 上的键是否和 `props` 对象上的键冲突***
- ***检查 `methods` 对象上的键是否和 `data` 对象上的键冲突***
- ***在 Vue 实例对象上添加代理访问对象的同名属性***
- ***最后一行代码调用 `observe()` 函数开启响应式***



## 数据响应系统基本思路

**剖析源码之前，先对流程有一个基本的了解，这样方便后续的阅读源码。**

**在 Vue 中，可以使用 `$watch` 观察一个字段，当字段的值发生变化，则执行指定的观察者，如下：**

```javascript
const ins = new Vue({
  data: {
    a: 1
  }
})

ins.$watch('a', () => {
  console.log('修改了 a')
})
```

**经过上方代码后，当修改属性 `a` 的值：`ins.a = 2` 时，控制台就会打印 `'修改了 a'`。将这个问题抽象一下，假设有数据对象 `data`，如下：**

```javascript
const data = {
  a: 1
}
```

**还有一个名为 `$watch()` 的函数：**

```javascript
function $watch () {...}
```

**`$watch()` 函数接收两个参数，第一个参数是需要观测的字段，第二个参数是当该字段的值发生改变后需要执行的函数，如下：**

```javascript
$watch('a', () => {
  console.log('修改了 a')
})
```

**要实现上方的功能，第一个问题是：*如何知道属性被修改了？*这个时候就需要依赖上文所说的 `Object.defineProperty()` 函数，调用该函数为对象的每个属性都设置一对 `getter/setter`，从而得知属性被读取和被设置，如下：**

```javascript
Object.defineProperty(data, 'a', {
  set () {
    console.log('设置了属性 a')
  },
  get () {
    console.log('读取了属性 a')
  }
})
```

**这样就可以实现对属性 `a` 的设置和获取操作的拦截，再进一步思考：*能否在获取属性 `a` 的时候收集依赖，在设置属性 `a` 的时候触发之前收集的依赖？*收集依赖需要一个容器，获取属性时，将依赖放进容器中，设置属性时，从容器中取出相应的依赖执行即可，代码如下：**

````javascript
// dep 数组就是我们所谓的“容器”
const dep = []
Object.defineProperty(data, 'a', {
  set () {
    // 当属性被设置的时候，将“容器”里的依赖都执行一次
    dep.forEach(fn => fn())
  },
  get () {
    // 当属性被获取的时候，把依赖放到“容器”里
    dep.push(fn)
  }
})
````

**在上方的代码中，定义了一个名为 `dep` 的常量数组，也就是所谓的容器。上方代码中，就是获取属性 `a` 的时候，将依赖添加到 `dep` 数组中，设置属性 `a` 的时候就会在 `dep` 数组中取出依赖，然后依序执行依赖。**

**随即产生了一个新的问题：*如何在获取属性 `a` 的时候收集依赖？*这里就借助了 `$watch` 函数了：**

```javascript
const data = {
  a: 1
}

const dep = []
Object.defineProperty(data, 'a', {
  set () {
    dep.forEach(fn => fn())
  },
  get () {
    // 此时 Target 变量中保存的就是依赖函数
    dep.push(Target)
  }
})

// Target 是全局变量
let Target = null
function $watch (exp, fn) {
  // 将 Target 的值设置为 fn
  Target = fn
  // 读取字段值，触发 get 函数
  data[exp]
}
```

**上方代码中，定义了一个名为 `Target` 的全局变量，然后在 `$watch` 中将 `Target` 的值设置为 `fn` (这里的 `fn` 也就是上文所说到的依赖)，然后读取字段的值 `data[exp]` 触发属性的 `get()` 函数，这时就会把此时全局变量 `Target` 的值添加到 `dep` 数组中，从而完成了依赖收集的过程。**

**添加如下测试代码：**

```javascript
$watch('a', () => {
  console.log('第一个依赖')
})
$watch('a', () => {
  console.log('第二个依赖')
})
```

**此时设置 `data.a = 3` ，在控制台将分别打印 `第一个依赖` 和 `第二个依赖`。此时的响应系统还是存在着许多的缺陷，多字段的情况就无法满足，因此，起码需要一个循环包着定义访问器属性的代码，如下：**

```javascript
const data = {
  a: 1,
  b: 1
}

for (const key in data) {
  const dep = []
  Object.defineProperty(data, key, {
    set () {
      dep.forEach(fn => fn())
    },
    get () {
      dep.push(Target)
    }
  })
}
```

**这样就可以使用 `$watch()` 函数观察任意一个 `data` 对象下的字段了，此时还有一个问题，因为访问器属性中的 `get()` 函数并没有返回任何值，所以 `console.log(data.a)` 会得到 `undefined` 的结果，解决这个问题只需要：**

```javascript
for (let key in data) {
  const dep = []
  let val = data[key] // 缓存字段原有的值
  Object.defineProperty(data, key, {
    set (newVal) {
      // 如果值没有变什么都不做
      if (newVal === val) return
      // 使用新值替换旧值
      val = newVal
      dep.forEach(fn => fn())
    },
    get () {
      dep.push(Target)
      return val  // 将该值返回
    }
  })
}
```

**只要在 `Object.defineProperty()` 函数定义访问器属性之前缓存一下原来的值，即 `val`，然后在 `get()` 函数中将 `val` 返回即可，除此之外还需要在 `set()` 函数中使用新值 (`newVal`) 重写旧值 `val`。**

**但是此时还有一个问题，就是当面对数据对象是嵌套对象的情况时，代码只能够检测出第一层对象的属性，如果数据对象如下：**

```javascript
const data = {
  a: {
    b: 1
  }
}
```

**对于上方的对象结构，程序只能够将 `data.a` 字段转换成响应式属性，而 `data.a.b` 仍然不是响应式属性，解决这个问题，只需要一个递归结构即可：**

```javascript
function walk (data) {
  for (let key in data) {
    const dep = []
    let val = data[key]
    // 如果 val 是对象，递归调用 walk 函数将其转为访问器属性
    const nativeString = Object.prototype.toString.call(val)
    if (nativeString === '[object Object]') {
      walk(val)
    }
    Object.defineProperty(data, key, {
      set (newVal) {
        if (newVal === val) return
        val = newVal
        dep.forEach(fn => fn())
      },
      get () {
        dep.push(Target)
        return val
      }
    })
  }
}

walk(data)
```

**上方代码，将定义访问器属性的逻辑放在了函数 `walk()` 上，并且增加了一段判断逻辑，如果某个属性的值是对象的话，则递归调用 `walk()` 函数，这样就可以实现深度定义访问器属性。**

**虽然经过了上方的改造，`data.a.b` 已经是访问器属性了，但是下方的这段代码仍然不能够正确执行：**

```javascript
$watch('a.b', () => {
  console.log('修改了字段 a.b')
})
```

**详细看一下当前 `$watch()` 函数的代码：**

```javascript
function $watch (exp, fn) {
  Target = fn
  // 读取字段值，触发 get 函数
  data[exp]
}
```

**读取字段值的时候，直接使用 `data[exp]`，按照上方的调用 `$watch('a.b', fn)`，那么 `$watch()` 函数中的 `data[exp]` 就等价于 `data['a.b']`，这里显然是不正确的，正确的读取方式应该是：`data['a']['b']`，所以这里需要做一些改造：**

```javascript
const data = {
  a: {
    b: 1
  }
}

function $watch (exp, fn) {
  Target = fn
  let pathArr,
      obj = data
  // 检查 exp 中是否包含 .
  if (/\./.test(exp)) {
    // 将字符串转为数组，例：'a.b' => ['a', 'b']
    pathArr = exp.split('.')
    // 使用循环读取到 data.a.b
    pathArr.forEach(p => {
      obj = obj[p]
    })
    return
  }
  data[exp]
}
```

**这里对 `$watch()` 函数进行一些改造，先检查需要读取的字段是否包含 `.`，如果包含了 `.` 说明读取的是嵌套对象的字段，这个时候就使用字符串的 API `split('.')` 函数将字符串转换成数组，因此访问的路径： `a.b` 转换后的数组就是 `['a', 'b']`，然后使用一个煦暖从而读取到嵌套对象的属性值，这里需要注意的是：*读取到嵌套对象的属性值之后应该立即 `return`，不需要执行后边的代码。***

**`$watch()` 函数的原理：*其实 `$watch()` 函数就是想尽一切办法，访问到我们需要观察的字段，从而触发该字段的 `get()` 函数，从而在对应的 `get()` 函数进行依赖的收集，此时传递给 `$watch()` 函数的第一个参数是一个字符串，代表要访问数据的某一个字段属性，然而第一个参数除了可以是一个字符串，还可以是一个函数，下边举例说明：***

```javascript
const data = {
  name: '刘嘉豪',
  age: 24
}

function render () {
  return document.write(`姓名：${data.name}; 年龄：${data.age}`)
}
```

**上方代码中，`render()` 函数中依赖了数据对象 `data`，则 `render()` 函数的执行会触发 `data.name` 和 `data.age` 这两个字段的属性访问器中的 `get()` 拦截器，所以可以将 `render()` 函数作为 `$watch()` 函数的第一个参数。**

```javascript
$watch(render, render)
```

**为了让 `$watch()` 函数可以正常执行，可以对 `$watch()` 函数进行如下修改：**

```javascript
function $watch (exp, fn) {
  Target = fn
  let pathArr,
      obj = data
  // 如果 exp 是函数，直接执行该函数
  if (typeof exp === 'function') {
    exp()
    return
  }
  if (/\./.test(exp)) {
    pathArr = exp.split('.')
    pathArr.forEach(p => {
      obj = obj[p]
    })
    return
  }
  data[exp]
}
```

**上方的 `$watch()` 函数中，检查了 `exp` 的类型，如果是 `function`，则直接执行该函数，由于 `render()` 函数的执行会触发数据字段的 `get()` 拦截器。所以依赖会被收集。同时还需要注意传递给 `$watch()` 函数的第二个参数：**

```javascript
$watch(render, render)
```

**`$watch()` 函数第二个参数仍然是 `render()` 函数，说明当依赖发生改变的时候，会重新触发 `render()` 函数，这样就实现了当数据发生了变化，可以将变化自动应用到 DOM 中。但是这里还是留有问题，关于如何避免收集冗余的依赖，这些问题将在后边讲述。上边所说的思路是为了在进入 Vue 源码前，给大家铺垫一下。**



## `observe()` 工厂函数

**了解了响应系统的基本思路之后，可以深入到 Vue 源码，回到 `initData()` 函数的最后一行代码：**

```javascript
// observe data
observe(data, true/*asRootData*/)
```

**调用了 `observe()` 函数观察数据，`observe()` 函数来自 `src/core/observe/index.js` 文件中，源码如下：**

```javascript
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

**`observe()` 函数接收两个参数，第一个参数是需要被观察的数据，第二个参数是一个布尔值，代表被观察的数据是否为根级数据。**

```javascript
if (!isObject(value) || value instanceof VNode) {
  return
}
```

**`observe()` 函数的开头就是一个判断语句：*判断需要观察的数据，如果不是一个对象或者是 `VNode` 实例，则直接 `return`。***

```javascript
if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
  ob = value.__ob__
} else if (
  shouldObserve &&
  !isServerRendering() &&
  (Array.isArray(value) || isPlainObject(value)) &&
  Object.isExtensible(value) &&
  !value._isVue
) {
  ob = new Observer(value)
}
```

**上方的代码中，首先通过一个 `if` 判断分支，判断数据对象 `value` 上自身是否含有 `__ob__` 属性，并且 `__ob__` 属性应该是 `Observer` 的实例。如果为 `true`，则直接将数据对象自身的 `__ob__` 属性作为 `ob` 的值：`ob = value.__ob__`。`__ob__` 的作用：*当一个数据对象被观察之后，将会在该对象上定义 `__ob__` 属性，所以 `if` 分支的作用就是避免重复观察一个数据对象。***

**然后就是 `else if` 分支，如果数据对象上没有定义 `__ob__` 属性，那么说明该数据对象还没有被观察过，然后就会进入到 `else if` 判断分支，如果 `else if` 分支为 `true`，则会执行 `ob = new Observer(value)` 对数据对象进行观察。只有当数据对象满足 `else if` 中所有的分支条件才会被观察，接下来会详细剖析条件：**

- **第一个条件：`shoulObserve` 必须为 `true`。`shouldObserver` 变量定义在 `src/core/observer/index.js` 文件中：**

  ```javascript
  /**
   * In some cases we may want to disable observation inside a component's
   * update computation.
   */
  export let shouldObserve: boolean = true
  
  export function toggleObserving (value: boolean) {
    shouldObserve = value
  }
  ```

  **该变量的初始值为 `true`，在 `shouldObserver` 变量的下边定义了一个名为 `toggleObserving()` 的函数，该函数接收一个布尔值参数，用来切换 `shouldObserver` 变量的真假值，当 `shouldObserver` 为 `true` 时，可以对数据进行观察，当 `shouldObserver` 变量为 `false` 时，数据不允许被观察。**

- **第二个条件：`!isServerRendering()` 必须为 `true`。`isServerRendering()` 函数的返回值是一个布尔值，用来判断是否是服务端渲染。只有当不是服务端渲染的时候才会观察数据。**
- **第三个条件：`(Array.isArray(value) || isPlainObject(value))` 必须为 `true`。只有当数据是纯对象或者是数组，才需要对数据进行观察。**
- **第四个条件：`Object.isExtensible(value)` 必须为 `true`。意为被观察的数据必须是可扩展的，一个普通对象默认就是可扩展的，以下三个方法都会使得一个对象不可扩展：`Object.preventExtensions()`、`Object.freeze()`、`Object.seal()`。**
- **第五个条件：`!value._isVue` 必须为 `true`。Vue 实例对象拥有 `_isVue` 属性，所以这个条件用来避免 Vue 实例对象被观察。**

**当一个对象满足了上边的五个条件之后，就会执行 `else if` 语句块中的代码：*创建一个 `Observer` 实例：***

```javascript
ob = new Observer(value)
```



## `Observer` 构造函数

**真正将数据对象转换成响应式数据的是 `Observer` 构造函数。定义在 `src/core/observer/index.js` 文件中：**

```javascript
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    // 省略...
  }

  walk (obj: Object) {
    // 省略...
  }
  
  observeArray (items: Array<any>) {
    // 省略...
  }
}
```

**从上方代码可以看到，`Observer` 类的实例拥有三个实例属性，分别是 `value`、`dep`、`vmCount` 及两个实例方法 `walk` 和 `observeArray`。`Observer` 类的构造函数接收一个参数，即数据对象。下边从 `constructor()` 开始，研究实例化一个 `Observer` 类需要做什么。**

### 数据对象的 `__ob__` 属性

**`constructor()` 函数源码如下：**

```javascript
constructor (value: any) {
  this.value = value
  this.dep = new Dep()
  this.vmCount = 0
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
```

**`constructor()` 函数中传递了一个 `value` (即数据对象)，然后将数据对象赋值给 `Observer` 类实例中的 `value` 属性。其次通过创建一个 `Dep` 类实例来实例对象的 `dep` 属性。**

```javascript
this.value = value
this.dep = new Dep()
```

**这里的 `Dep` 相当于上边所说的用于收集依赖的一个容器。实例对象的 `vmCount` 属性被设置为 `0`：`this.vmCount = 0`。**

**初始化完三个实例属性之后，使用 `def()` 函数，为数据对象定义一个 `__ob__` 属性，这个属性就是当前的 `Observer` 实例对象。其中 `def()` 函数的原理就是通过 `Object.defineProperty()` 函数的简单封装，在这里使用 `def()` 函数定义 `__ob__` 属性，是因为使用 `def()` 函数定义 `__ob__` 属性可以定义不可枚举的属性，这样后边遍历数据对象的时候就可以防止遍历到 `__ob__` 属性了。**

**举例说明：**

```javascript
const data = {
  a: 1
}
```

**经过了 `def()` 函数进行处理之后，`data` 对象应该变成如下：**

```javascript
const data = {
  a: 1,
  // __ob__ 是不可枚举的属性
  __ob__: {
    value: data, // value 属性指向 data 数据对象本身，这是一个循环引用
    dep: dep实例对象, // new Dep()
    vmCount: 0
  }
}
```



### 响应式数据之纯对象的处理

**经过了 `def()` 函数处理完之后，接着就会进入到一个 `if...else` 判断分支：**

```javascript
if (Array.isArray(value)) {
  const augment = hasProto
    ? protoAugment
    : copyAugment
  augment(value, arrayMethods, arrayKeys)
  this.observeArray(value)
} else {
  this.walk(value)
}
```

**上边的 `if...else` 分支用于判断数据对象是一个数组还是一个纯对象，因为这两种的处理方式是不一样的。从容易的入手，先看数据对象为纯对象的情况，也就是 `else` 分支的内容，他会执行 `this.walk(value)` 函数，下边是该函数的源码：**

```javascript
walk (obj: Object) {
  const keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    defineReactive(obj, keys[i])
  }
}
```

**`walk()` 函数先获取纯对象中可以枚举的属性，然后使用 `for` 循环来为每个属性调用 `defineReactive()` 函数。**



### `defineReactive()` 函数

**`defineReactive()` 函数定义在 `src/core/observer/index.js` 文件，源码如下：**

```javascript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // 省略...
    },
    set: function reactiveSetter (newVal) {
      // 省略...
    }
  })
}
```

**`defineReactive()` 函数核心：*将数据对象的数据属性转换为访问器属性。*即为数据对象的属性设置一对 `getter/setter`，其中做了许多边界条件的处理。`defineReactive()` 函数接收五个参数，但是在 `walk()` 函数中调用 `defineReactive()` 函数时只传递了两个参数 (数据对象和属性的键名)。**

**现在细看 `defineReactive()` 函数体：**

```javascript
const dep = new Dep()
```

**首先定义了一个 `dep` 常量，是一个 `Dep` 的实例对象。**

**在上边讲解 `Observer` 的 `constructor` 的时候，为数据对象定义了一个 `__ob__` 属性，该属性是一个 `Observer` 的实例对象，并且该对象包含着一个 `Dep` 实例对象：**

```javascript
const data = {
  a: 1,
  __ob__: {
    value: data,
    dep: dep实例对象, // new Dep() , 包含 Dep 实例对象
    vmCount: 0
  }
}
```

**这里 `__ob__.dep` 的这个 `Dep` 实例对象的作用和上方讲解数据响应系统基本思路上一节所说的容器的作用不同，他的作用在后边会讲述到。而与上一节讲解数据响应系统中所说的容器作用相同的 `Dep` 实例对象是在 `defineReactive()` 函数一开始定义的 `dep` 常量，即如下：**

```javascript
const dep = new Dep()
```

**这里在 `defineReactive()` 函数中第一行所定义的常量 `dep` 对应的 `Dep` 实例对象才跟前文所说的容器相同。这里定义的 `dep` 在访问器属性的 `getter/setter` 中被闭包引用，如下：**

```javascript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  // 省略...

  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        // 这里闭包引用了上面的 dep 常量
        dep.depend()
        // 省略...
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      // 省略...

      // 这里闭包引用了上面的 dep 常量
      dep.notify()
    }
  })
}
```

**上方代码中，在访问器属性的 `getter/setter` 中，通过闭包引用了 `defineReactive()` 函数中第一行定义的 `dep` 常量。这里有一个需要注意的点：*每一个数据字段都通过闭包引用着属于自己的 `dep` 常量。*因为在 `walk()` 函数中通过循环遍历了所有数据对象的属性，并且调用了 `defineReactive()` 函数，而每次调用 `defineReactive()` 函数定义访问器属性时，该属性的 `setter/getter` 都通过闭包引用了一个属于自己的容器。并且由于这个 `dep` 属性是通过闭包的方式被 `getter/setter` 引用的，所以只能够通过 `getter/setter` 才能访问或者修改他，除此之外，其他方式都不可以访问或修改。**

**举例说明：**

```javascript
const data = {
  a: 1,
  b: 2
}
```

**`data` 经过了 `defineReactive()` 函数之后，`data.a` 和 `data.b` 都通过闭包引用属于自己独一份的 `Dep` 实例对象。每个字段的 `Dep` 对象都被用来收集属于自己 (该字段) 的依赖。**

**定义完了 `dep` 常量之后，紧接着就是这一段代码：**

```javascript
const property = Object.getOwnPropertyDescriptor(obj, key)
if (property && property.configurable === false) {
  return
}
```

**通过 `Object.getOwnPropertyDescriptor()` 函数获取该字段已有的属性描述对象，并将该对象保存在 `property` 常量中，紧接着是一个 `if` 语句块，判断该字段是否可配置的，如果不可配置的，即：`property.configurable === false`，那么直接 `return`，即不会继续执行 `defineReactive()` 函数，这样做的原因是：一个不可配置的属性不能使用，也没有必要使用 `Object.defineProperty()` 函数来改变其属性定义。**

**往下是这段代码：**

```javascript
// cater for pre-defined getter/setters
const getter = property && property.get
const setter = property && property.set
if ((!getter || setter) && arguments.length === 2) {
  val = obj[key]
}

let childOb = !shallow && observe(val)
```

**上方的这段代码的前两句定义了 `getter` 和 `setter` 常量，分别保存了来自 `property` 对象的 `get` 和 `set` 函数，`property` 对象是属性的描述对象，一个对象的属性可能已经是一个访问器属性，所以该属性可能存在 `get` 和 `set` 方法，又因为接下来会使用 `Object.defineProperty()` 函数重新定义属性的 `getter/setter`，这会导致属性原有的 `set` 和 `get` 函数被覆盖，所以先把属性原有的 `getter/setter` 事先缓存起来，并在重新定义的 `set` 和 `get` 函数中调用原来缓存的函数，这样就会不影响属性原有的读写操作。相当于做了一层 AOP 一样！！**

**上边这段代码 `if` 条件语句稍微比较难理解：**

```javascript
(!getter || setter) && arguments.length === 2
```

**`arguments.length == 2` 意为：当只传递两个参数时，即没有传递第三个参数 `val`，此时就需要根据 `key` 主动去对象上获取相应的值，即执行：`val = obj[key]`。而 `(!getter || setter)` 的意思后边会讲述。**

**在 `if` 语句块的下边，是这一行代码：**

```javascript
let childObj = !shallow && observe(val)
```

**定义了 `childObj` 常量，在 `if` 语句块中获取到了对象属性的值 `val`，但是 `val` 本身也有可能是一个对象，如果是一个对象，将继续调用 `observe(val)` 函数观察该对象从而深度观察数据对象。不过这里有一个前提条件 `shallow` 应该是 `false`。即 `!shallow` 为 `true` 的时候才会继续调用 `observe()` 函数深度观察。由于在 `walk()` 函数中调用 `defineReactive()` 函数没有传递 `shallow` 参数，所以该参数是 `undefined`。简单来讲，Vue 默认是深度观察对象的。而非深度观察的场景其实也经历过，那就是在 `initRender()` 函数中在 Vue 实例对象上定义 `$attrs` 属性和 `$listeners` 属性时就是非深度观察，如下：**

```javascript
defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true) // 最后一个参数 shallow 为 true
defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
```

**需要注意一个问题：*使用 `observe(val)` 深度观察数据对象时，`val` 未必是有值的，因为必须满足条件 `(!getter || setter) && arguments.length === 2` 时才会触发取值的动作：`val = obj[key]`，所以一旦满足不了条件，就算属性有值，但是由于没有触发取值的动作，所以 `val` 仍然是 `undefined`。这样就导致了深度观察无效。***



### 被观察后的数据对象的样子

**一个数据对象经过了 `observe()` 函数处理之后变成的样子，将举例说明：**

```javascript
const data = {
  a: {
    b: 1
  }
}

observe(data)
```

**数据对象 `data` 拥有一个 `a` 属性，并且 `a` 属性是一个对象，该对象有一个 `b` 属性，经过 `observe` 处理之后，`data` 和 `data.a` 两个对象都被定义了 `__ob__` 属性，并且访问器属性 `a` 和 `b` 的 `setter/getter` 都通过闭包引用属于自己的 `Dep` 实例对象和 `childOb` 对象：**

```javascript
const data = {
  // 属性 a 通过 setter/getter 通过闭包引用着 dep 和 childOb
  a: {
    // 属性 b 通过 setter/getter 通过闭包引用着 dep 和 childOb
    b: 1
    __ob__: {a, dep, vmCount}
  }
  __ob__: {data, dep, vmCount}
}
```

**从梳理整个代码流程可以看出，属性 `a` 闭包引用的 `childOb` 实际上就是 `data.a.__ob__`。而属性 `b` 闭包引用的 `childOb` 是 `undefined`，因为属性 `b` 是基本数据类型值，并不是对象也不是数组，所以在 `observe()` 函数中的第一个 `if` 判断分支就直接返回了，如下：**

```javascript
if (!isObject(value) || value instanceof VNode) {
  return
}
```



### 在 `get()` 函数中收集依赖

**接着看 `defineReactive()` 函数的源码，这部分是 `defineReactive()` 函数的关键代码：*使用 `Object.defineProperty()` 函数定义访问器属性：***

```javascript
Object.defineProperty(obj, key, {
  enumerable: true,
  configurable: true,
  get: function reactiveGetter () {
    // 省略...
  },
  set: function reactiveSetter (newVal) {
    // 省略...
})
```

**以上代码执行完，理论上 `defineProperty()` 函数的代码就已经执行完了。对于访问器属性的 `get()` 和 `set()` 函数是不会执行的，因为没有触发属性的读取或设置操作。但是这里重点研究 `get()` 和 `set()` 函数分别执行了什么操作。**

**先从 `get()` 函数开始进行分析：**

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

**首先是判断是否存在 `getter()`，从上方的代码可以知道，`getter()` 是属性原来的属性访问器中的 `get()` 函数，如果存在 `getter()` 则直接调用 `getter()` 函数。并以该函数的返回值作为属性的值。保证属性的原有读取操作正常运作。如果 `getter()` 不存在，则使用 `val` 作为属性的返回值。**

**`get()` 函数除了要正确返回属性值，还要收集依赖，`get()` 函数中第一行和最后一行代码中间的所有代码都是用来进行依赖收集，接下来根据源码讲解收集依赖的过程，`dep.depend()` 这行代码这里先简单理解为依赖被收集了。源码如下：**

```javascript
if (Dep.target) {
  dep.depend()
  if (childOb) {
    childOb.dep.depend()
    if (Array.isArray(value)) {
      dependArray(value)
    }
  }
}
```

**首先判断 `Dep.target` 是否存在，这里的 `Dep.target` 和上文讲的*数据响应系统基本思路*所讲的 `Target` 的作用一致，所以 `Dep.target` 就是要被收集的依赖 (观察者)。如果 `Dep.target` 存在，就说明有依赖需要被收集。将会执行 `if (Dep.target)` 判断分支中的代码块。**

**在 `if (Dep.target)` 语句块中的第一行代码中就是：`dep.depend()`，执行 `dep` 对象中的 `depend()` 方法将依赖收集到 `dep` 这个依赖中。这里的 `dep` 对象就是属性 `getter/setter` 中通过闭包引用的容器 (这个 `dep` 是在 `defineReactive()` 函数中定义)。紧接着判断 `childOb` 是否存在，如果存在就执行 `childOb.dep.depend()`，要搞懂 `if (childOb)` 分支中的内容，需要明确 `childOb` 是什么。举个例子：**

```javascript
const data = {
  a: {
    b: 1
  }
}
```

**经过了观察处理之后，将被添加 `__ob__` 属性，如下：**

```javascript
const data = {
  a: {
    b: 1,
    __ob__: { value, dep, vmCount }
  },
  __ob__: { value, dep, vmCount }
}
```

**属性 `a` 的 `getter/setter` 通过闭包引用了一个 `Dep` 实例对象，除此之外，还通过闭包引用着 `childOb`，且在这里有 `childOb === data.a.__ob__` 成立。由此可知，`childOb === data.a.__ob__.dep`。也就是 `childOb.dep.depend()` 除了要将依赖收集到属性 `a` 闭包引用的容器 `dep` 之外，还要将同样的依赖收集到 `data.a.__ob__.dep` 容器当中。之所以会将依赖放进两个不同的容器当中，是因为这两个容器中的依赖触发的时机是不一样的。**

- **属性 `a` 闭包引用的 `dep` 容器：在属性 `a` 自身的 `set()` 函数中修改了属性就会触发：`dep.notify()`。**

- **`data.a.__ob__.dep` 容器：这个容器触发的时机是在使用 `$set()` 或者使用 `Vue.set()` 给数据添加新字段时触发的，因为 JS 在没有劫持数据之前是无法拦截给对象添加属性的操作的，所以这里就通过了数据对象的 `__ob__` 属性进行操作。因为 `__ob__.dep` 这个容器收集了和闭包引用的 `dep` 一样的依赖。假设 `Vue.set()` 函数代码如下：**

  ```javascript
  Vue.set = function (obj, key, val) {
    defineReactive(obj, key, val)
    obj.__ob__.dep.notify()
  }
  ```

  **当使用 `Vue.set()` 为 `data.a` 对象添加新属性：**

  ```javascript
  Vue.set(data.a, 'c', 1)
  ```

  **上方代码之所以能够触发依赖，是因为 `Vue.set()` 函数中触发了 `data.a.__ob__.dep` 的这个容器中的依赖。**

  ```javascript
  Vue.set = function (obj, key, val) {
    defineReactive(obj, key, val)
    obj.__ob__.dep.notify() // 相当于 data.a.__ob__.dep.notify()
  }
  
  Vue.set(data.a, 'c', 1)
  ```

  **所以 `__ob__` 属性以及 `__ob__.dep` 主要作用就是为了添加、删除属性的时候有能力触发依赖，这就是 `Vue.set()` 和 `Vue.delete()` 的原理。**

  **在 `childOb.dep.depend()` 这句话下边还有一个 `if` 语句，如下：**

  ```javascript
  if (Array.isArray(value)) {
    dependArray(value)
  }
  ```

  **这里因为数组和对象的响应式处理不一样，所以需要分开来进行处理。**



### 在 `set()` 函数中触发依赖

**在 `get()` 函数中收集了依赖之后，需要再 `set()` 函数中触发依赖，当属性被修改的时候触发依赖：**

```javascript
set: function reactiveSetter (newVal) {
  const value = getter ? getter.call(obj) : val
  /* eslint-disable no-self-compare */
  if (newVal === value || (newVal !== newVal && value !== value)) {
    return
  }
  /* eslint-enable no-self-compare */
  if (process.env.NODE_ENV !== 'production' && customSetter) {
    customSetter()
  }
  if (setter) {
    setter.call(obj, newVal)
  } else {
    val = newVal
  }
  childOb = !shallow && observe(newVal)
  dep.notify()
}
```

**`get()` 函数有两个工作：**

- **返回正确的属性值**
- **收集依赖**

**`set()` 函数同样也有两个工作：**

- **为属性设置新值**
- **触发相应的依赖**

**在 `set()` 函数中接收一个参数 `newVal`，该属性被设置的新值，在函数体内，先执行了一行：**

```javascript
const value = getter ? getter.call(obj) : val
```

**`set()` 函数中第一行代码和 `get()` 函数中第一行代码一样，都是提取属性原有的值，用属性原有的值和新的值进行一个比较，当原有的值和新设置的值不一致的时候才需要触发依赖和重新设置属性值，否则视属性值没有改变，不做处理。如下代码：**

```javascript
/* eslint-disable no-self-compare */
if (newVal === value || (newVal !== newVal && value !== value)) {
  return
}
```

**`(newVal === value)` 表示新值和旧值相同，直接返回。`(newValue !== newValue && value !== value)` 表示新值和新值自身不全等，旧值和旧值自身也不全等，JS 中只有一种情况满足这种条件，那就是：**

```javascript
NaN === NaN	// false
```

**综上可知：`(newValue !== newValue && value !== value)` 表示新值是 `NaN`，旧值也是 `NaN`，即属性的值没有变化，所以直接返回了。**

**紧接着是一个 `if` 语句块：**

```javascript
/* eslint-enable no-self-compare */
if (process.env.NODE_ENV !== 'production' && customSetter) {
  customSetter()
}
```

**上方代码的意思：`customSetter()` 存在，那么在非生产环境下会执行 `customSetter()` 函数。其中 `customSetter()` 函数是 `defineReactive()` 函数的第四个参数。这里的 `customSetter()` 的作用是用来打印辅助信息，如下例子：**

```javascript
defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
  !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
}, true)
```

**上方的代码中的箭头函数就是 `customSetter()` 函数，就是用来打印打印辅助信息的。**

**紧接着就是这段代码：**

```javascript
if (setter) {
  setter.call(obj, newVal)
}
else {
  val = newVal
}
```

**上方代码：*先判断 `setter()` 是否存在，`setter` 变量所存储的函数是属性原有的 `set()` 函数，所以如果一个属性自身原本拥有 `set()` 函数，那么应该继续使用函数来设置属性的值，从而保证属性原有的设置操作不受影响。如果属性原本没有 `set()` 函数的，那就将 `val` 设置为 `newVal` 即：`val = newVal`。***

**紧接着就是最后 `set()` 函数的最后两行代码：**

```javascript
childOb = !shallow && observe(newVal)
dep.notify()
```

**由于属性被设置了一个新的值，如果设置的新的值是一个数组或者是一个纯对象，那么该数组或纯对象是未被 Vue 观测的，因此需要对新值进行一个观察的操作，这就是第一行代码的作用，当然了这些都是在 `!shallow` 为 `true` 的情况下，即需要进行深度观察的时候才会执行。最后就是触发依赖了，这里的容器 `dep` 指代的是 `set()` 函数中闭包引用的 `dep`。而触发依赖就是将容器 `dep` 中的依赖都执行一下，也就是：`dep.notify()`。**



### 保持定义相应数据行为的一致性

**本小节解读 `defineReactive()` 函数中最让人迷惑的代码：**

```javascript
if ((!getter || setter) && arguments.length === 2) {
  val = obj[key]
}
```

**这个判断分支中的条件，有两个条件：**

- **`(!getter || setter)`**
- **`arguments.length === 2`**

**只有这两个条件同时满足了，才会根据 `key` 去对象 `obj` 上取值：`val = obj[key]`，否则就不会触发取值的动作，触发不了取值的动作，就意味着 `val` 的值为 `undefined`，这会导致 `if` 语句块下方的深度观察的代码无效，即不会发生深度观察对象：**

```javascript
// val 是 undefined，不会深度观测
let childOb = !shallow && observe(val)
```

**对于第二个条件，只要传递参数的数量为 `2` 时，说明没有传递第三个参数 `val`，那就需要通过执行 `val = obj[key]` 去获取属性值。**

**对于第一个条件则比较难理解，即：`(!getter || setter)`，其实最初的 Vue 源码是没有上边的这段 `if` 代码的，在 `walk()` 函数中是这样调用 `defineReactive()` 函数的：**

```javascript
walk (obj: Object) {
  const keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    // 这里传递了第三个参数
    defineReactive(obj, keys[i], obj[keys[i]])
  }
}
```

**从上方的代码可以知，调用 `defineReactive()` 函数的时候传递了第三个参数，即属性值。这个是最初的实现，后来变成成下边这个样子：**

```javascript
walk (obj: Object) {
  const keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    // 在 walk 函数中调用 defineReactive 函数时暂时不获取属性值
    defineReactive(obj, keys[i])
  }
}

// ================= 分割线 =================

// 在 defineReactive 函数内获取属性值
if (!getter && arguments.length === 2) {
  val = obj[key]
}
```

**在 `walk()` 函数中调用了 `defineReactive()` 函数时去掉了第三个参数，而是在 `defineReactive()` 函数中增加了一段 `if` 语句的判断体，当发现调用 `defineReactive()` 函数的时候传递了两个参数，并且在属性原本没有 `getter()` 函数的情况下，才会通过 `val = obj[key]` 取值。**

**当原本的属性中存在着 `getter()` 拦截函数时，在初始化的时候不触发 `getter()` 函数，只有当真正获取该属性的值的时候，再通过缓存下来的属性原本的 `getter()` 函数进行取值即可。所以由此可知，如果数据对象上的某一个属性原本就拥有了自己的 `getter()` 函数，那么这个属性就不会被深度观察了。因为属性原本有 `getter()` 的话，`val = obj[key]` 这行代码是不会执行的。所以 `val` 是 `undefined` 了，导致后边语句传递给 `observe()` 函数的参数就是 `undefined`。**

**举例说明：**

```javascript
const data = {
  getterProp: {
    a: 1
  }
}

new Vue({
  data,
  watch: {
    'getterProp.a': () => {
      console.log('这句话会输出')
    }
  }
})
```

**上方代码中，定义了数据对象 `data`，`data` 是一个嵌套对象，在 `watch` 选项中观察了属性 `getterProp.a`，当修改了 `getterProp.a` 的值，以上代码是能够正常输出的。**

**再看如下代码：**

```javascript
const data = {}
Object.defineProperty(data, 'getterProp', {
  enumerable: true,
  configurable: true,
  get: () => {
    return {
      a: 1
    }
  }
})

const ins = new Vue({
  data,
  watch: {
    'getterProp.a': () => {
      console.log('这句话不会输出')
    }
  }
})
```

**仅修改了定义数据对象 `data` 的方式，此时 `data.getterProp` 本身就已经是一个访问器属性了，并且拥有 `getter()` 方法，这个时候当修改 `getterProp` 属性的值时，在 `watch` 选项中观察 `getterProp.a` 的函数不会被执行。因为属性 `getterProp` 是一个拥有 `getter()` 的访问器属性，当 Vue 发现拥有原本的 `getter()` 时，是不会进行深度观察的。**

**Vue 之所以不会对拥有 `getter()` 的访问器属性进行深度观察的原因：**

- **由于当前属性存在原本的 `getter()`，在深度观察 (`let childOb = !shallow && observe(val)`) 之前不会取值，所以在深度观察语句执行之前取不到属性值，从而无法深度观察。**
- **之所以在深度观察之前不取值是因为属性原本的 `getter()` 由开发者来定义，开发者有可能在 `getter()` 中做任何意想不到的事情，这样做是出于避免引发不可预见问题的考虑。**

**现在回过头来看这段 `if` 语句块：**

```javascript
if (!getter && arguments.length === 2) {
  val = obj[key]
}
```

**这样做，其实同样存在一些问题的，当数据对象中的某一个属性只拥有 `getter` 而不拥有 `setter` 的时候，此时属性是不会被深度观察的，但是经过了 `defineReactive()` 函数的处理之后，在 `defineReactive()` 函数中定义的 `set()` 函数体中有那么一段代码，如下：**

```javascript
if (getter && !setter) return
if (setter) {
  setter.call(obj, newVal)
}
else {
  val = newVal
}
childOb = !shallow && observe(newVal)
```

**从上方代码可以看出，如果某个属性同时存在 `getter` 和 `setter` 两者，在 `defineReactive()` 中不会对该属性进行深度观察，但是当为该属性重新赋值时，根据上方代码可知，仍然会执行 `childOb = !shallow && observe(newVal)` 这行代码，也就是会对新赋值的对象进行深度观察，那么此时问题就来了，这不就产生了矛盾了么，也就是定义响应式数据时行为不一致。为了解决这个问题，采用的方法就是当属性拥有原本的 `setter` 时，就算拥有 `getter` 也要获取属性值，并进行深度观察，和上方的代码遥相呼应，各位慢慢理解哈，这个经典之中的经典哈！改动之后的代码如下，配合上方代码食用：**

```javascript
if ((!getter || setter) && arguments.length === 2) {
  val = obj[key]
}
```



### 响应式数据之数组的处理

**上边就是响应式数据对于纯对象的处理方式，紧接着会讨论响应式数组的处理方式，回到 `Observer` 类的 `constructor()` 函数的，看到如下代码：**

```javascript
if (Array.isArray(value)) {
  const augment = hasProto
    ? protoAugment
    : copyAugment
  augment(value, arrayMethods, arrayKeys)
  this.observeArray(value)
} else {
  this.walk(value)
}
```

**在 `if` 条件中，使用 `Array.isArray()` 函数检测被观测的值 `value` 是否为数组，如果是数组则执行 `if` 语句块中的代码，从而实现对数组的观测。处理数组和处理纯对象的方式不太一样，数组是一个特殊的数据结构，有很多实例方法，有些方法会改变数组自身的值，这些方法被称为：*变异方法，*这些方法有：`push`、`pop`、`shift`、`unshift`、`splice`、`sort` 以及 `reverse` 等。当开发者调用了这些变异方法改变数组时，需要触发依赖。所以需要知道开发者什么时候调用了这些变异方法，只有这样才能在这些方法被调用的时候做出反应。**



### 拦截数组变异方法的思路

**考虑一个问题，如下代码中 `sayHello()` 函数用来打印字符串 `hello`：**

```javascript
function sayHello () {
  console.log('hello')
}
```

**有一个需求，在不改动 `sayHello()` 函数源码情况下，在打印字符串 `'hello'` 之前先输出字符串 `'Hi'`。这时，可以这样子做：**

```javascript
const originalSayHello = sayHello
sayHello = function () {
  console.log('Hi')
  originalSayHello()
}
```

**上方的代码实现了需求，首先使用 `originalSayHello` 变量缓存原本的 `sayHello()` 函数，然后重新定义 `sayHello()` 函数，并在新定义的 `sayHello()` 函数中调用缓存下来的 `originalSayHello()` 函数，这样就保证在不改变 `sayHello()` 的前提下对其进行了功能的扩展，类似于 AOP，面向切面编程。**

**Vue 正是通过这种方式实现了对数据变异方法的拦截，即保持数组变异方法原有的功能不变对其功能进行扩展。数组的变异方法是来自于数组构造函数的原型，即可得 `arr.__proto__ === Array.prototype` 为 `true`。**

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/Kk45OU.png)

**由此可知一个思路就是：通过设置 `__proto__` 属性的值为一个新的对象，且该新对象的原型是数组构造函数原来的原型对象。如下图所示：**

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/aiwiQv.png)

**数组本身也是一个对象，既然是一个对象就可以访问其 `__proto__` 属性，上图数组实例的 `__proto__` 属性所指向的就是相当于`arrayMethods` 对象，同时 `arrayMethods` 对象上的 `__proto__` 属性指向了真正的数组原型对象。并且 `arrayMethods` 对象上定义了和数组变异方法同名的函数，这样当通过数组实例调用变异方法的时候，首先执行的就是 `arrayMethods` 对象上同名的函数，这样就实现了对数组方法的拦截，同时这也是实现 AOP 的一个方式。如下：**

```javascript
// 要拦截的数组变异方法
const mutationMethods = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

const arrayMethods = Object.create(Array.prototype) // 实现 arrayMethods.__proto__ === Array.prototype
const arrayProto = Array.prototype  // 缓存 Array.prototype

mutationMethods.forEach(method => {
  arrayMethods[method] = function (...args) {
    const result = arrayProto[method].apply(this, args)

    console.log(`执行了代理原型的 ${method} 函数`)

    return result
  }
})
```

**如上方代码所示，通过 `Object.create(Array.prototype)` 创建了 `arrayMethods` 对象，这样就保证了 `arrayMethods.__proto__ === Array.prototype`。然后通过一个循环在 `arrayMethods` 对象上定义了和数组变异方法同名的函数。并在这些函数内调用真正数组原型上的相应方法，测试代码如下：**

```javascript
const arr = []
arr.__proto__ = arrayMethods

arr.push(1)
```

**控制台中会打印一句话：`执行了代理原型的 push 函数`。但是由于低版本的 `IE` 不支持 `__proto__` 所以需要使用另外的兼容方案进行处理。**

**使用比较多的兼容方案就是直接在数组实例上定义和变异方法同名的函数，如下代码：**

```javascript
const arr = []
const arrayKeys = Object.getOwnPropertyNames(arrayMethods)

arrayKeys.forEach(method => {
  arr[method] = arrayMethods[method]
})
```

**上方代码中通过 `Object.getOwnPropertyNames()` 获取所有属于 `arrayMethods` 对象自身的键，然后通过一个循环在数组实例中定义和变异方法同名的函数，这样当调用 `push()` 方法的时候，就会优先执行定义在数组实例上的 `push()` 函数，这样就实现了兼容版本的拦截，不过上边这种直接在数组实例上定义的属性是可枚举的，因此更好的做法是使用 `Object.defineProperty()` 进行定义：**

```javascript
arrayKeys.forEach(method => {
  Object.defineProperty(arr, method, {
    enumerable: false,
    writable: true,
    configurable: true,
    value: arrayMethods[method]
  })
})
```



### 拦截数组变异方法在 Vue 中的实现

**上边小节讲解了拦截数组变异方法的思路，本小节着眼看 Vue 如何实现。**

**在 `Observer` 类中的 `constructor()` 函数中：**

```javascript
constructor (value: any) {
  this.value = value
  this.dep = new Dep()
  this.vmCount = 0
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
```

**有一个点需要注意：无论是纯对象还是数组，都将通过 `def()` 函数为其定义 `__ob__` 属性。紧接着查看 `if` 语句块中的代码将被执行，即如下代码：**

```javascript
const augment = hasProto
  ? protoAugment
  : copyAugment
augment(value, arrayMethods, arrayKeys)
this.observeArray(value)
```

**上方代码定义了一个 `augment` 常量，如果 `hasProto` 为 `true` 那么 `augment` 的值为 `protoAugment`，如果 `hasProto` 为 `false` 那么 `augment` 的值为 `copyAugment`。`hasProto` 是一个用于检测当前环境是否支持使用 `__proto__` 属性的值。**

