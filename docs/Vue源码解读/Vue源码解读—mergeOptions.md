# Vue源码解读—mergeOptions

## mergeOptions 函数的三个参数

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



## mergeOptions 函数本体

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

