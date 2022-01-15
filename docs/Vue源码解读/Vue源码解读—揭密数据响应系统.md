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

