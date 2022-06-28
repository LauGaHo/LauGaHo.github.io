# 句法分析—生成真正的AST(二)

## 彻底理解解析属性值的方式

接下来要讲解的就是 `processElement` 函数中调用的最后一个函数，它就是 `processAttrs` 函数，这个函数是用来处理元素描述对象的 `el.attrsList` 数组中剩余的所有属性的。到目前为止已经讲解过的属性有：

- `v-pre`
- `v-for`
- `v-if`、`v-else-if`、`v-else`
- `v-once`
- `key`
- `ref`
- `slot`、`slot-scope`、`scope`、`name`
- `is`、`inline-template`

以上这些属性的解析已经在上一节中全部讲解过了，能够发现一些规律，比如在获取这些属性的值得时候，要么使用 `getAndRemoveAttr` 函数，要么使用 `getBindingAttr` 函数，但是无论使用哪个函数，其共同行为都是：**在获取到特定属性值的同时，还会将该属性从 `el.attrsList` 数组中移除**。所以在调用 `processAttrs` 函数的时候，以上列出来得属性都已经从 `el.attrsList` 数组中移除了。但是 `el.attrsList` 数组中仍然可能存在其他属性，所以这个时候就需要使用 `processAttrs` 函数处理 `el.attrsList` 数组中剩余的属性。

在讲解 `processAttrs` 函数之前，先来回顾一下现在掌握的知识。比如上述列出的属性为例，下表中总结了特定的属性和获取该属性值的方式：

| 属性                          | 获取属性值的方式   |
| ----------------------------- | ------------------ |
| `v-pre`                       | `getAndRemoveAttr` |
| `v-for`                       | `getAndRemoveAttr` |
| `v-if`、`v-else-if`、`v-else` | `getAndRemoveAttr` |
| `v-once`                      | `getAndRemoveAttr` |
| `key`                         | `getBindingAttr`   |
| `ref`                         | `getBindingAttr`   |
| `name`                        | `getBindingAttr`   |
| `slot-scope`、`scope`         | `getAndRemoveAttr` |
| `slot`                        | `getBindingAttr`   |
| `is`                          | `getBindingAttr`   |
| `inline-template`             | `getAndRemoveAttr` |

可以发现凡是以 `v-` 开头的属性，在获取属性值的时候都是通过 `getAndRemoveAttr` 函数获取的。而对于没有 `v-` 开头的特性，如 `key`、`ref` 等，在获取这些属性的值时，是通过 `getBindingAttr` 函数获取的，不过 `slot-scope`、`scope` 和 `inline-template` 这三个属性虽然没有以 `v-` 开头，但仍然使用 `getAndRemoveAttr` 函数获取其属性值。但这并不是关键，关键是需要知道使用 `getAndRemoveAttr` 和 `getBindingAttr` 这两个函数获取属性值的时候到底有什么区别。

可以知道类似于 `v-for` 或 `v-if` 这类以 `v-` 开头的属性，在 Vue 中称之为指令，并且这些属性的属性值是默认情况下被当做表达式处理的，比如：

```html
<div v-if="a && b"></div>
```

如上代码在执行的时候 `a` 和 `b` 都会被当做变量，并且 `a && b` 是具有完整意义的表达式，而非普通字符串。并且在解析阶段，如上 `div` 标签的元素描述对象的 `el.attrsList` 属性将是如下数组：

```javascript
el.attrsList = [
  {
    name: 'v-if',
    value: 'a && b'
  }
]
```

这时，当使用 `getAndRemoveAttr` 函数获取 `v-if` 属性值时，得到的就是字符串 `'a && b'`，但不要忘了这个字符串最终是要运行在 `new Function()` 函数中的，假设是如下代码：

```javascript
new Function('a && b')
```

那么这行代码等价于：

```javascript
function () {
  a && b
}
```

可以看到，此时的 `a && b` 已经不再是普通的字符串了，而是表达式。

这就意味着 `slot-scope`、`scope`、`inline-template` 这三个属性的值，最终也将会被作为表达式处理，而非普通字符串。如下：

```html
<div slot-scope="slotProps"></div>
```

如上代码是使用作用域插槽的典型例子，可以知道这里的 `slotProps` 确实是变量，而非字符串。

那如果使用 `getBindingAttr` 函数获取 `slot-scope` 属性的值会产生什么效果呢？由于 `slot-scope` 并非 `v-bind:slot-scope` 或 `:slot-scope`，所以在使用 `getBindingAttr` 函数获取 `slot-scope` 属性值的时候，将会得到使用 `JSON.stringify` 函数处理后的结果，即：

```javascript
JSON.stringify('slotProps')
```

这个值就是字符串 `'"slotProps"'`，把这个字符串拿到 `new Function()` 中，如下：

```javascript
new Function('"slotProps"')
```

如上这行代码等价于：

```javascript
function () {
  "slotProps"
}
```

可以发现此时函数体内只有一个字符串 `"slotProps"`，而非变量。

但并不是说使用了 `getBindingAttr` 函数获取的属性值最终是字符串，如果该属性是绑定的属性 (使用了 `v-bind` 或 `:`)，则该属性的值仍然具有 `javascript` 语言的能力。否则该属性的值就是一个普通的字符串。

## `processAttrs` 处理剩余属性值的方式

`processAttrs` 函数是 `processElement` 函数中调用的最后一个 `process*` 系列函数，在这之前已经调用了很多其他的 `process*` 系列函数对元素进行了处理，并且每当处理一个属性时，都会将该属性从元素描述对象的 `el.attrsList` 数组中移除，但 `el.attrsList` 数组中仍然保存着未被处理的属性，而 `processAttrs` 函数就是用来处理这些剩余属性的。

既然 `processAttrs` 函数用来处理剩余未被处理的属性，那么首先确定的是 `el.attrsList` 数组中都包含了哪些剩余属性，如下是前面已经处理过的属性：

- `v-pre`
- `v-for`
- `v-if`、`v-else-if`、`v-else`
- `v-once`
- `key`
- `ref`
- `slot`、`slot-scope`、`scope`、`name`
- `is`、`inline-template`

如上属性中包含了部分 Vue 内置的指令 (`v-` 开头的属性)，可以对照一下 Vue 的官方文档，查看其内置的指令，可以发现之前的讲解中不包含对以下指令的解析：

- `v-text`、`v-html`、`v-show`、`v-on`、`v-bind`、`v-model`、`v-cloak`

除了这些指令之外，还有部分属性的处理也没有讲到，比如 `class` 属性和 `style` 属性，这两个属性比较特殊，因为 Vue 对它们做了增强，实际上在“中置处理” (`transforms` 数组) 中有对于 `class` 属性和 `style` 属性的处理，这个后面会统一讲解。

还有就是一些普通属性的处理了，如下 `html` 代码所示：

```html
<div :custom-prop="someVal" @custom-event="handleEvent" other-prop="static-prop"></div>
```

如上代码所示，其中 `:custom-prop` 是自定义的绑定属性，`@custom-event` 是自定义的事件，`other-prop` 是自定义的非绑定属性，对于这些内容的处理都是由 `processAttrs` 函数完成的。其实处理自定义绑定属性本质上就是处理 `v-bind` 指令，而处理自定义事件就是处理 `v-on` 指令。

接着具体看一下 `processAttrs` 函数的源码，看如何处理这些剩余未被处理的指令的。如下时简化后的代码：

```javascript
function processAttrs (el) {
  const list = el.attrsList
  let i, l, name, rawName, value, modifiers, isProp
  for (i = 0, l = list.length; i < l; i++) {
    // 省略...
  }
}
```

可以看到在 `processAttrs` 函数内部，首先定义了 `list` 常量，它是 `el.attrsList` 数组的引用。接着又定义了一系列变量待使用，然后开启了一个 `for` 循环，循环的目的就是遍历 `el.attrsList` 数组，所以这里能够想到在循环内部就是逐个处理 `el.attrsList` 数组中那些剩余的属性。

`for` 循环内部的代码被一个 `if...else` 语句块分成了两部分，如下：

```javascript
for (i = 0, l = list.length; i < l; i++) {
  name = rawName = list[i].name
  value = list[i].value
  if (dirRE.test(name)) {
    // 省略...
  } else {
    // 省略...
  }
}
```

在 `if...else` 语句块之前，分别为 `name`、`rawName` 以及 `value` 变量赋了值，其中 `name` 和 `rawName` 变量中保存的是属性的名字，而 `value` 变量中则保存着属性的值。然后才执行了 `if...else` 语句块，来看一下 `if` 语句块的判断条件：

```javascript
if (dirRE.test(name))
```

使用 `dirRE` 正则去匹配属性名 `name`，`dirRE` 正则在前面已经讲解过了，它用来匹配一个字符串是否以 `v-`、`@`、`:` 开头，所以如果匹配成功则说明该属性是指令，此时 `if` 语句块内的内容会被执行，否则将执行 `else` 语句块内的代码。举个例子，如下 `html` 片段所示：

```html
<div :custom-prop="someVal" @custom-event="handleEvent" other-prop="static-prop"></div>
```

其中 `:custom-prop` 属性和 `@custom-event` 属性将会被 `if` 语句块内的代码处理，而对于 `other-prop` 属性则会被 `else` 语句块内的代码处理。

接下来优先看如果属性是一个指令，那么在 `if` 语句块内是如何对该指令进行处理，如下代码：

```javascript
if (dirRE.test(name)) {
  // mark element as dynamic
  el.hasBindings = true
  // modifiers
  modifiers = parseModifiers(name)
  if (modifiers) {
    name = name.replace(modifierRE, '')
  }
  if (bindRE.test(name)) { // v-bind
    // 省略...
  } else if (onRE.test(name)) { // v-on
    // 省略...
  } else { // normal directives
    // 省略...
  }
} else {
  // 省略...
}
```

如果代码执行到了这里，能够确认的是该属性是一个指令，如上高亮的三行代码所示，这是一个 `if...else if...else` 语句块，不难发现 `if` 语句的判断条件是在检测该指令是否是 `v-bind` (包括缩写 `:`) 指令，`else if` 语句的判断条件是在检测该指令是否是 `v-on` (包括缩写 `@`)，而对于其他指令则会执行 `else` 语句块的代码。后面会对这三个分支内的代码做详细讲解，不过在之前再来看一下如下高亮的代码：

```javascript
if (dirRE.test(name)) {
  // mark element as dynamic
  el.hasBindings = true
  // modifiers
  modifiers = parseModifiers(name)
  if (modifiers) {
    name = name.replace(modifierRE, '')
  }
  if (bindRE.test(name)) { // v-bind
    // 省略...
  } else if (onRE.test(name)) { // v-on
    // 省略...
  } else { // normal directives
    // 省略...
  }
} else {
  // 省略...
}
```

一个完整的指令包含指令的名称、指令的参数、指令的修饰符以及指令的值，以上高亮代码的作用就是用来解析指令中的修饰符。首先既然元素使用了指令，那么该指令的值就是表达式，既然是表达式就涉及到动态的内容，所以此时会在元素描述对象上添加 `el.hasBindings` 属性，并将其值设置为 `true`，标识着当前元素是一个动态的元素。接着执行了如下这行代码：

```javascript
modifiers = parseModifiers(name)
```

调用 `parseModifiers` 函数，该函数接收整个指令字符串作为参数，作用就是解析指令中的修饰符，并将解析结果赋值给 `modifiers` 变量。找到 `parseModifiers` 函数的代码，如下：

```javascript
function parseModifiers (name: string): Object | void {
  const match = name.match(modifierRE)
  if (match) {
    const ret = {}
    match.forEach(m => { ret[m.slice(1)] = true })
    return ret
  }
}
```

在 `parseModifiers` 函数内部首先使用指令字符串的 `match` 方法匹配正则 `modifierRE`，`modifierRE` 正则在上一章节讲解过了，它是用来全局匹配字符串中字符 `.` 以及 `.` 后面的字符，也就是修饰符，举个例子，假设指令字符串为：`'v-bind:some-prop.sync'`，则使用该字符串去匹配正则 `modifierRE` 最终将会得到一个数组：`[".sync"]`。一个指令有几个修饰符，则匹配的结果数组中就包含了几个元素。如果匹配失败则会得到 `null`。回到上方代码，定义了 `match` 常量，它保存着匹配结果。接着是一个 `if` 语句块，如果匹配成功则会执行 `if` 语句块内的代码，在 `if` 语句块内首先定义了 `ret` 常量，它是一个空对象，并且能够发现 `ret` 常量将作为匹配成功时的返回结果，`ret` 常量的作用，可以来看这行代码：

```javascript
match.forEach(m => { ret[m.slice(1)] = true })
```

使用 `forEach` 循环遍历了 `match` 数组，然后将每一项都作为 `ret` 对象的属性，并将其值设置为 `true`。注意由于 `match` 数组中的每个修饰符中都包含了字符 `.`，所以如上代码中使用 `m.slice(1)` 将字符 `.` 去掉。假设指令字符串为：`':v-bind:some-prop.sync'`，则最终 `parseModifiers` 会返回一个对象：

```javascript
{
  sync: true
}
```

当然了，如果指令字符串中不包含修饰符，则 `parseModifiers` 函数没有返回值，或者说其返回值为 `undefined`。

再回到如下这段代码，注意代码，如下：

```javascript
if (dirRE.test(name)) {
  // mark element as dynamic
  el.hasBindings = true
  // modifiers
  modifiers = parseModifiers(name)
  if (modifiers) {
    name = name.replace(modifierRE, '')
  }
  if (bindRE.test(name)) { // v-bind
    // 省略...
  } else if (onRE.test(name)) { // v-on
    // 省略...
  } else { // normal directives
    // 省略...
  }
} else {
  // 省略...
}
```

在使用 `parseModifiers` 函数解析完指令中的修饰符之后，会使用 `modifiers` 变量保存解析结果，如果解析成功，将会执行如下代码：

```javascript
if (modifiers) {
  name = name.replace(modifierRE, '')
}
```

这行代码的作用很简单，就是将修饰符从指令字符串中移除，也就是说此时的指令字符串 `name` 中已经不包含修饰符部分了。

### 解析 `v-bind` 指令

处理完了修饰符，将进入对于指令的解析，解析环节分为三部分，分别是对于 `v-bind` 指令的解析，对于 `v-on` 指令的解析，以及对于其他指令的解析。如下代码所示：

```javascript
if (dirRE.test(name)) {
  // mark element as dynamic
  el.hasBindings = true
  // modifiers
  modifiers = parseModifiers(name)
  if (modifiers) {
    name = name.replace(modifierRE, '')
  }
  if (bindRE.test(name)) { // v-bind
    // 省略...
  } else if (onRE.test(name)) { // v-on
    // 省略...
  } else { // normal directives
    // 省略...
  }
} else {
  // 省略...
}
```

如上高亮的代码所示，该 `if...else if...else` 语句块分别用来处理 `v-bind` 指令、`v-on` 指令以及其他指令。先来看 `if` 语句块：

```javascript
if (bindRE.test(name)) {
  // 省略...
}
```

该 `if` 语句的判断条件是使用 `bindRE` 去匹配指令字符串，如果一个指令以 `v-bind:` 或 `:` 开头，则说明该指令为 `v-bind` 指令，这时 `if` 语句块内的代码将被执行，如下：

```javascript
if (bindRE.test(name)) { // v-bind
  name = name.replace(bindRE, '')
  value = parseFilters(value)
  isProp = false
  // 省略...
}
```

首先使用 `bindRE` 正则将指令字符串中的 `v-bind` 或 `:` 去除掉，此时 `name` 字符串已经从一个完整的指令字符串变为绑定属性的名字了，举个例子，假如原本的指令字符串为 `'v-bind:some-prop.sync'`，由于之前已经把该字符串中修饰符的部分去掉了，所以指令字符串将变为了 `'v-bind:some-prop'`，接着如上第一行代码又将指令字符串找那个的 `v-bind:` 去掉，所以此时指令字符串将变为 `'some-prop'`，可以发现该字符串就是绑定属性的名字，或者说是 `v-bind` 指令的参数。

接着调用 `parseFilters` 函数处理绑定属性的值，可以知道 `parseFilters` 函数的作用是用来将表达式与过滤器整合在一起的，前边已经做了详细的讲解，但凡涉及到能够使用过滤器的地方都要使用 `parseFilters` 函数去解析，并将解析后的新表达式返回。如上第二行代码所示，使用 `parseFilters` 函数的返回值重新赋值 `value` 变量。

第三行代码将 `isProp` 变量初始化为 `false`，`isProp` 变量标识着该绑定的属性是否是原生 DOM 对象的属性，所谓原生 DOM 对象的属性就是能够通过 DOM 元素对象直接访问的有效 API，比如 `innerHTML` 就是一个原生 DOM 对象的属性。

再往下将进入一段 `if` 条件语句，该 `if` 语句块的作用是用来处理修饰符的：

```javascript
if (modifiers) {
  if (modifiers.prop) {
    isProp = true
    name = camelize(name)
    if (name === 'innerHtml') name = 'innerHTML'
  }
  if (modifiers.camel) {
    name = camelize(name)
  }
  if (modifiers.sync) {
    addHandler(
      el,
      `update:${camelize(name)}`,
      genAssignmentCode(value, `$event`)
    )
  }
}
```

当然了，如果没有给 `v-bind` 属性提供修饰符，则这段 `if` 语句的代码将被忽略。`v-bind` 属性为开发者提供了三个修饰符，分别是 `prop`、`camel`、`sync`，这恰好对应如上代码的三段 `if` 语句块。首先看第一段 `if` 语句块：

```javascript
if (modifiers.prop) {
  isProp = true
  name = camelize(name)
  if (name === 'innerHtml') name = 'innerHTML'
}
```

这段 `if` 语句块的代码用来处理使用了 `prop` 修饰符的 `v-bind` 指令，既然使用了 `prop` 修饰符，则意味着该属性将被作为原生 DOM 对象的属性，所以首先会将 `isProp` 变量设置为 `true`，接着使用 `camelize` 函数将属性名驼峰化，最后还会检查驼峰化之后的属性名是否等于字符串 `'innerHtml'`，如果属性名全等于该字符串则将属性名重写为字符串 `'innerHTML'`，可以知道 `'innerHTML'` 是一个特例，它的 `HTML` 四个字符串权威大写。以上就是对于使用了 `prop` 修饰符的 `v-bind` 指令的处理，如果一个绑定属性使用了 `prop` 修饰符则 `isProp` 变量会设置为 `true`，并且会把属性名字驼峰化。之所以要将 `isProp` 变量设置为 `true` 是因为答案如下：

```javascript
if (bindRE.test(name)) { // v-bind
  name = name.replace(bindRE, '')
  value = parseFilters(value)
  isProp = false
  if (modifiers) {
    if (modifiers.prop) {
      isProp = true
      name = camelize(name)
      if (name === 'innerHtml') name = 'innerHTML'
    }
    // 省略...
  }
  if (isProp || (
    !el.component && platformMustUseProp(el.tag, el.attrsMap.type, name)
  )) {
    addProp(el, name, value)
  } else {
    addAttr(el, name, value)
  }
}
```

如上代码所示，如果 `isProp` 为 `true` 则会执行该 `if` 语句块内的代码，即调用 `addProp` 函数，而 `else` 语句块内的 `addAttr` 函数是永远不会调用的。前边讲解过 `addAttr` 函数，它会将属性的名字和值以对象的形式添加到元素描述对象的 `el.attrs` 数组中，`addProp` 函数和 `addAttr` 函数类似，只不过 `addProp` 函数会把属性的名字和值以对象的形式添加到元素描述对象的 `el.props` 数组中。如下是 `addProp` 函数的源码，它来自 `src/compiler/helpers.js` 文件：

```javascript
export function addProp (el: ASTElement, name: string, value: string) {
  (el.props || (el.props = [])).push({ name, value })
  el.plain = false
}
```

总之 `isProp` 变量是一个重要的标识，它的值将会影响一个属性被添加到元素描述对象的位置，从而影响后续的行为。另外再说一句：**元素描述对象的 `el.props` 数组中存储的并不是组件概念中的 `prop`，而是原生 DOM 对象的属性**。在后面的章节中会看到，组件概念中的 `prop` 其实是在 `el.attrs` 数组中。

回过头来，明白了 `prop` 修饰符和 `isProp` 变量的作用之后，再来看一下对于 `camel` 修饰符的处理，如下代码：

```javascript
if (modifiers) {
  if (modifiers.prop) {
    // 省略...
  }
  if (modifiers.camel) {
    name = camelize(name)
  }
  if (modifiers.sync) {
    // 省略...
  }
}
```

如上代码所示，如果 `modifiers.camel` 为真，则说明该绑定的属性使用了 `camel` 修饰符，使用该修饰符的作用只有一个，那就是将绑定的属性驼峰化，如下代码所示：

```html
<svg :view-box.camel="viewBox"></svg>
```

有些人可能会说，直接写成驼峰就可以了：

```html
<svg :viewBox="viewBox"></svg>
```

不行，因为对于浏览器来讲，真正的属性名字是 `:viewBox` 而不是 `viewBox`，

### 解析 `v-on` 指令



### 解析其他指令



### 处理非指令属性



## `preTransformNode` 前置处理



## `transformNode` 中置处理



## 文本节点的元素描述对象



## `parseText` 函数解析字面量表达式



## 对结束标签的处理



## 注释节点的元素描述对象