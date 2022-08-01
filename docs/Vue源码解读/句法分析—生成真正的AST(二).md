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

不行，因为对于浏览器来讲，真正的属性名字是 `:viewBox` 而不是 `viewBox`，所以浏览器在渲染时会认为这是一个自定义属性，对于任何自定义属性浏览器都会把它渲染成小写的形式，所以当 `Vue` 尝试获取这段字符串的时候，会得到如下字符串：

```javascript
'<svg :viewbox="viewBox"></svg>'
```

最终渲染的真实 `DOM` 将是：

```html
<svg viewbox="viewBox"></svg>
```

这将导致渲染失败，因为 `SVG` 标签只认 `viewBox`，却不知道 `viewbox` 是什么。

注意，这个问题仅存在于 `Vue` 需要获取被浏览器处理后的模板字符串时才会出现，所以如果使用了 `template` 选项代替 `Vue` 自动读取则不会出现这个问题：

```vue
new Vue({
  template: '<svg :viewBox="viewBox"></svg>'
})
```

当然了，使用单文件组件也不会出现这种问题，所以这些情况下，是不需要使用 `camel` 修饰符的。

接着来看一下对于最后一个修饰符的处理，即 `sync` 修饰符：

```javascript
if (modifiers) {
  if (modifiers.prop) {
    // 省略...
  }
  if (modifiers.camel) {
    // 省略...
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

如上代码所示，如果 `modifiers.sync` 为真，则说明该绑定的属性使用了 `sync` 修饰符。`sync` 修饰符实际上是一个语法糖，子组件不能够直接修改 `prop` 值，通常会在子组件中发射一个自定义事件，然后在父组件层面监听该事件并由父组件来修改状态。这个过程有时候过于繁琐，如下：

```vue
<template>
  <child :some-prop="value" @custom-event="handleEvent" />
</template>

<script>
export default {
  data () {
    value: ''
  },
  methods: {
    handleEvent (val) {
      this.value = val
    }
  }
}
</script>
```

为了简化该过程，可以在绑定属性时使用 `sync` 修饰符：

```vue
<child :some-prop.sync="value" />
```

这行代码等价于：

```vue
<template>
  <child :some-prop="value" @update:someProp="handleEvent" />
</template>

<script>
export default {
  data () {
    value: ''
  },
  methods: {
    handleEvent (val) {
      this.value = val
    }
  }
}
</script>
```

注意事件名称 `update:someProp` 是固定的，它由 `update:` 加上驼峰化的绑定属性名称组成的。所以在子组件中需要发射一个名字为 `update:someProp` 的事件才能使 `sync` 修饰符生效，这大大提高了开发者的开发效率。

在 `Vue` 内部，使用 `sync` 修饰符的绑定属性和没有使用 `sync` 修饰符的绑定属性之间差异就在于：使用了 `sync` 修饰符的绑定属性等价于多了一个事件侦听，并且事件名称为 `'update:${驼峰化命名}'`。回到源码：

```javascript
if (modifiers) {
  if (modifiers.prop) {
    // 省略...
  }
  if (modifiers.camel) {
    // 省略...
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

可以看到如果发现该绑定的属性使用了 `sync` 修饰符，则直接调用 `addHandler` 函数，在当前元素描述对象上添加事件侦听器。`addHandler` 函数的作用实际上就是将事件名称和该事件的侦听函数添加到元素描述对象的 `el.events` 属性或 `el.nativeEvents` 属性中。对于 `addHandler` 函数的实现将会在讲解的 `v-on` 指令的解析中讲解。这里需要注意一个公式：

```javascript
:some-prop.sync <==等价于==> :some-prop + @update:someProp
```

通过如下代码就能够知道事件名称的构成：

```javascript
if (modifiers.sync) {
  addHandler(
    el,
    `update:${camelize(name)}`,
    genAssignmentCode(value, `$event`)
  )
}
```

如上代码中，事件名称等于字符串 `'update:'` 加上驼峰化的绑定属性名称。另外可以注意到传递给 `addHandler` 函数的第三个参数，实际上 `addHandler` 函数的第三个参数就是当事件发生时的回调函数，而该回调函数是通过 `genAssignmentCode` 函数生成的。`genAssignmentCode` 函数来自 `src/compiler/directives/model.js` 文件，如下是源码：

```javascript
export function genAssignmentCode (
  value: string,
  assignment: string
): string {
  const res = parseModel(value)
  if (res.key === null) {
    return `${value}=${assignment}`
  } else {
    return `$set(${res.exp}, ${res.key}, ${assignment})`
  }
}
```

讲解 `genAssignmentCode` 函数将会牵扯到比较多的东西，实际上 `genAssignmentCode` 函数也被用在 `v-model` 指令，因为本质上 `v-model` 指令和绑定属性加上 `sync` 修饰符几乎相同，所以会在讲解 `v-model` 指令时再来讲解 `genAssignmentCode` 函数。这里只需要关注如上代码中 `genAssignmentCode` 的返回值即可，它返回的是一个代码字符串，可以看到如果这个代码字符串作为代码执行，其作用就是一个赋值工作。这样就免去了手动赋值的繁琐。

以上讲完了对于三个绑定属性可以使用的修饰符，接下来看处理绑定属性的最后一段代码：

```javascript
if (isProp || (
  !el.component && platformMustUseProp(el.tag, el.attrsMap.type, name)
)) {
  addProp(el, name, value)
} else {
  addAttr(el, name, value)
}
```

实际上这段代码已经讲到过了，这里要强调的是 `if` 语句的判断条件：

```javascript
isProp || (!el.component && platformMustUseProp(el.tag, el.attrsMap.type, name))
```

前面说过了如果 `isProp` 变量为 `true`，则说明了该绑定的属性是原生 `DOM` 对象的属性，但是如果 `isProp` 变量为 `false`，那么需要看第二个条件是否成立，如果第二个条件成立，则该绑定的属性还是会作为原生 `DOM` 对象的属性，第二个条件如下：

```javascript
!el.component && platformMustUseProp(el.tag, el.attrsMap.type, name)
```

首先 `el.component` 变量必须为 `false`，`el.component` 属性保存的是标签 `is` 的属性值，如果 `el.component` 属性值为假就能够保证该标签没有使用 `is` 属性。总结如下：

- `input`、`textarea`、`option`、`select`、`progress` 这些标签的 `value` 属性都应该使用元素对象的原生 `prop` 绑定。
- `option` 标签的 `selected` 属性应该使用元素对象的原生的 `prop` 绑定。
- `input` 标签的 `checked` 属性应该使用元素对象的原生的 `prop` 绑定。
- `video` 标签的 `muted` 属性应该使用元素对象的原生的 `prop` 绑定。

可以看到如果满足这些条件，则意味着即使在绑定以上属性时没有使用 `prop` 修饰符，那么它们依然会被当做原生 `DOM` 对象的属性。不过还是没有解释为什么要保证 `!el.component` 成立，这是因为 `platformMustUseProp` 函数在判断的时候需要标签的名字 (`el.tag`)，而 `el.component` 会在元素渲染阶段替换掉 `el.tag` 的值，所以如果 `el.component` 存在则会影响 `platformMustUseProp` 的判断结果。

最后对 `v-bind` 指令的解析做一个总结：

- 任何绑定的属性，最终要么会被添加到元素描述对象的 `el.attrs` 数组中，要么被添加到元素描述对象的 `el.props` 数组中。
- 对于使用了 `.sync` 修饰符的绑定属性，还会在元素描述对象的 `el.events` 对象中添加名字为 `'update:${驼峰化的属性名}'`。

### 解析 `v-on` 指令

接下来看一下 `processAttrs` 函数对于 `v-on` 指令的解析，如下代码所示：

```javascript
if (bindRE.test(name)) { // v-bind
  // 省略...
} else if (onRE.test(name)) { // v-on
  name = name.replace(onRE, '')
  addHandler(el, name, value, modifiers, false, warn)
} else { // normal directives
  // 省略...
}
```

和 `v-bind` 指令类似，使用 `onRE` 正则去匹配指令字符串，如果该指令字符串以 `@` 或 `v-on:` 开头，则说明该指令是事件绑定，此时 `else if` 语句块内的代码将会被执行，在 `else if` 语句块内，首先将指令字符串中的 `@` 字符或 `v-on:` 字符串去掉，然后直接调用 `addHandler` 函数。

打开 `src/compiler/helpers.js` 文件并找到 `addHandler` 函数，如下是 `addHandler` 函数签名：

```javascript
export function addHandler (
  el: ASTElement,
  name: string,
  value: string,
  modifiers: ?ASTModifiers,
  important?: boolean,
  warn?: Function
) {
  // 省略...
}
```

可以看到 `addHandler` 函数接收六个参数，分别是：

- `el`：当前元素描述对象。
- `name`：绑定属性的名字，即事件名称。
- `value`：绑定属性的值，这个值可能是事件回调函数名字，有可能是内联语句，有可能是函数表达式。
- `modifiers`：指令对象。
- `important`：可选参数，是一个布尔值，代表着添加的事件侦听函数的重要级别，如果为 `true`，则该侦听函数会被添加到该事件侦听函数数组的头部，否则将其添加到尾部。
- `warn`：打印警告信息的函数，是一个可选参数。

了解了 `addHandler` 函数所需的参数，再来看一下解析 `v-on` 指令时调用 `addHandler` 函数所传递的参数，如下代码所示：

```javascript
if (bindRE.test(name)) { // v-bind
  // 省略...
} else if (onRE.test(name)) { // v-on
  name = name.replace(onRE, '')
  addHandler(el, name, value, modifiers, false, warn)
} else { // normal directives
  // 省略...
}
```

如上代码中在调用 `addHandler` 函数时传递了全部六个参数。现在开始研究 `addHandler` 函数的实现，在 `addHandler` 函数的开头是这样一段代码：

```javascript
modifiers = modifiers || emptyObject
// warn prevent and passive modifier
/* istanbul ignore if */
if (
  process.env.NODE_ENV !== 'production' && warn &&
  modifiers.prevent && modifiers.passive
) {
  warn(
    'passive and prevent can\'t be used together. ' +
    'Passive handler can\'t prevent default event.'
  )
}
```

首先检测 `v-on` 指令的修饰符对象 `modifiers` 是否存在，如果在使用 `v-on` 指令时没有指定任何修饰符，则 `modifiers` 的值为 `undefined`，此时会使用冻结的空对象 `emptyObject` 作为替代。接着是一个 `if` 条件语句块，如果该 `if` 语句的判断条件成立，则说明开发同时使用了 `prevent` 修饰符和 `passive` 修饰符，此时如果是在非生产环境下并且 `addHandler` 函数的第六个参数 `warn` 存在，则使用 `warn` 函数打印警告信息，提示开发者 `passive` 修饰符不能和 `prevent` 修饰符一起使用，这是因为在事件监听中 `passive` 选项参数就是用来告诉浏览器该事件监听函数是不会阻止默认行为的。

再往下是这样一段代码：

```javascript
// check capture modifier
if (modifiers.capture) {
  delete modifiers.capture
  name = '!' + name // mark the event as captured
}
if (modifiers.once) {
  delete modifiers.once
  name = '~' + name // mark the event as once
}
/* istanbul ignore if */
if (modifiers.passive) {
  delete modifiers.passive
  name = '&' + name // mark the event as passive
}
```

这段代码由三个 `if` 条件语句块组成，如果事件指令中使用了 `capture` 修饰符，则第一个 `if` 语句块的内容将被执行，可以看到在第一个 `if` 语句块内首先将 `modifiers.capture` 选项移除，紧接着在原始事件名称之前添加一个字符 `!`。假设事件绑定代码如下：

```html
<div @click.capture="handleClick"></div>
```

如上代码中点击事件使用了 `capture` 修饰符，所以在 `addHandler` 函数内部，会把事件名称 `'click'` 修改为 `'!click'`。

和第一个 `if` 语句块类似，第二个和第三个 `if` 语句块分别用来处理当事件使用了 `once` 修饰符和 `passive` 修饰符的情况。可以看到如果事件使用了 `once` 修饰符，则会在事件名称的前面添加字符 `~`，如果事件使用了 `passive` 修饰符，则会在事件名称前面添加字符 `&`。也就是如下两段代码是等价的：

```html
<div @click.once="handleClick"></div>
```

等价于：

```html
<div @~click="handleClick"></div>
```

再往下是如下这段代码：

```javascript
// normalize click.right and click.middle since they don't actually fire
// this is technically browser-specific, but at least for now browsers are
// the only target envs that have right/middle clicks.
if (name === 'click') {
  if (modifiers.right) {
    name = 'contextmenu'
    delete modifiers.right
  } else if (modifiers.middle) {
    name = 'mouseup'
  }
}
```

这段代码用来规范化“右击”事件和点击鼠标中间按钮的事件，可以知道在浏览器总点击右键一般会出来一个菜单，这本质上触发了 `contextmenu` 事件。而 `Vue` 中定义“右击”事件的方式是为 `click` 事件添加 `right` 修饰符。所以如上代码中首先检查了事件名称是否是 `click`，如果事件名称是 `click` 并且使用了 `right` 修饰符，则会将事件名称重写为 `contextmenu`，同时使用 `delete` 操作符删除 `modifiers.right` 属性。类似地在 `Vue` 中定义点击滚轮事件的方式是为 `click` 事件指定 `middle` 修饰符，但可以知道鼠标本没有滚轮点击事件，一般区分用户点击的按钮是不是滚轮的方式是监听 `mouseup` 事件，然后通过事件对象的 `event.button` 属性值来判断，如果 `event.button === 1` 则说明用户点击的是滚轮按钮。

不过这里需要提醒一下，如果 `click` 事件使用了 `once` 修饰符，则事件的名字会被修改为 `~click`，所以当程序执行到如上这段时，事件名字是永远不会等于字符串 `'click'` 的，换句话说，如果同时使用 `once` 修饰符和 `right` 修饰符，则右击事件不会被触发，如下代码所示：

```html
<div @click.right.once="handleClickRightOnce"></div>
```

如上代码无效，作为变通方案可以直接监听 `contextmenu` 事件，如下：

```html
<div @contextmenu.once="handleClickRightOnce"></div>
```

但其实从源码角度也是很好解决的，只需要把规范化“右击”事件和点击鼠标中间按钮的事件的这段代码提前即可，实际上还有更好的解决方案，那就是从 `mouseup` 事件入手，将 `contextmenu` 事件和“右击”事件完全分离处理，这里就不展开讨论了。

回到 `addHandler` 函数继续看后面的代码，接下来要看的是如下这段代码：

```javascript
let events
if (modifiers.native) {
  delete modifiers.native
  events = el.nativeEvents || (el.nativeEvents = {})
} else {
  events = el.events || (el.events = {})
}
```

定义了 `events` 变量，然后判断是否存在 `native` 修饰符，如果 `native` 修饰符存在则会在元素描述对象上添加 `el.nativeEvents` 属性，初始值为一个空对象，并且 `events` 变量和 `el.nativeEvents` 属性具有相同的引用，另外可以注意如上代码中使用 `delete` 操作符删除了 `modifiers.native` 属性，到目前为止，在讲解 `addHandler` 函数时已经遇到了多次使用 `delete` 操作符删除修饰符对象的做法。这是因为在代码生成阶段会使用 `for...in` 语句遍历修饰符对象，然后做一些相关的事情，所以在生成 AST 阶段把那些不希望被遍历的属性删掉，更具体的内容在代码生成中详细讲解。回过头来，如果 `native` 属性不存在则会在元素描述对象上添加 `el.events` 属性，它的初始值也是一个空对象，此时 `events` 变量的引用将和 `el.events` 属性相同。 

再往下是这样一段代码：

```javascript
const newHandler: any = {
  value: value.trim()
}
if (modifiers !== emptyObject) {
  newHandler.modifiers = modifiers
}
```

定义了 `newHandler` 对象，该对象初始拥有一个 `value` 属性，该属性的值就是 `v-on` 指令的属性值。接着是一个 `if` 条件，该 `if` 语句的判断条件检测了修饰符对象 `modifiers` 是否不等于 `emptyObject`，可以知道一个事件没有使用任何修饰符时，修饰符对象 `modifiers` 会被初始化为 `emptyObject`，所以如果修饰符对象 `modifiers` 不等于 `emptyObject` 则说明事件使用了修饰符，此时会把修饰符对象赋值给 `newHandler.modifiers` 属性。

再往下是 `addHandler` 函数的最后一段代码：

```javascript
const handlers = events[name]
/* istanbul ignore if */
if (Array.isArray(handlers)) {
  important ? handlers.unshift(newHandler) : handlers.push(newHandler)
} else if (handlers) {
  events[name] = important ? [newHandler, handlers] : [handlers, newHandler]
} else {
  events[name] = newHandler
}

el.plain = false
```

首先定义了 `handlers` 常量，它的值是通过事件名称获取 `events` 对象下的对应的属性值的：`events[name]`，可以知道变量 `events` 要么是元素描述对象的 `el.nativeEvents` 属性的引用，要么就是元素描述对象 `el.events` 属性的引用。无论是谁的引用，在初始情况下 `events` 变量都是一个空对象，所以在第一次调用 `addHandler` 时 `handlers` 常量是 `undefined`，这就会导致接下来的代码中 `else` 语句块将被执行：

```javascript
if (Array.isArray(handlers)) {
  // 省略...
} else if (handlers) {
  // 省略...
} else {
  events[name] = newHandler
}
```

可以看到在 `else` 语句块内，为 `events` 对象定义了和事件名称相同的属性，并以 `newHandler` 对象作为属性值。举个例子，假设有如下模板代码：

```html
<div @click.once="handleClick"></div>
```

如上模板中监听了 `click` 事件，并绑定了名字为 `handleClick` 的事件监听函数，所以此时 `newHandler` 对象应该是：

```javascript
newHandler = {
  value: 'handleClick',
  modifiers: {} // 注意这里是空对象，因为 modifiers.once 修饰符被 delete 了
}
```

又因为使用了 `once` 修饰符，所以事件名称将变为字符串 `'~click'`，又因为在监听事件时没有使用 `native` 修饰符，所以 `events` 变量是元素描述对象的 `el.events` 属性的引用，所以调用 `addHandler` 函数的最终结果就是在元素描述对象的 `el.events` 对象是哪个添加相应的事件的处理结果：

```javascript
el.events = {
  '~click': {
    value: 'handleClick',
    modifiers: {}
  }
}
```

现在修改一下之前的模板，如下：

```html
<div @click.prevent="handleClick1" @click="handleClick2"></div>
```

如上模板所示，有两个事件的侦听，其中一个 `click` 事件使用了 `prevent` 修饰符，而另外一个 `click` 事件则没有使用修饰符，所以这两个 `click` 事件是不同的，但这两个事件的名称却是相同的，都是 `'click'`，所以这导致调用两次 `addHandler` 函数添加两次名称相同的事件，但是由于第一次调用 `addHandler` 函数添加 `click` 事件之后元素描述对象的 `el.events` 对象已经存在一个 `click` 属性，如下：

```javascript
el.events = {
  click: {
    value: 'handleClick1',
    modifiers: { prevent: true }
  }
}
```

所以当第二次调用 `addHandler` 函数时，如下 `else if` 语句块的代码将被执行：

```javascript
const handlers = events[name]
/* istanbul ignore if */
if (Array.isArray(handlers)) {
  important ? handlers.unshift(newHandler) : handlers.push(newHandler)
} else if (handlers) {
  events[name] = important ? [newHandler, handlers] : [handlers, newHandler]
} else {
  events[name] = newHandler
}
```

此时 `newHandler` 对象时第二个 `click` 事件侦听的信息对象，而 `handlers` 常量保存的则是第一次被添加的事件信息，可以看到上方的代码，`else if` 分支中的代码检测了参数 `important` 的真假，根据 `important` 参数的不同，会重新为 `events[name]` 赋值。可以看到 `important` 参数的真假所影响的仅仅是被添加的 `handlers` 对象的顺序。最终元素描述对象的 `el.events.click` 属性将变成一个数组，这个数组保存着前后两次添加的 `click` 事件的信息对象，如下：

```javascript
el.events = {
  click: [
    {
      value: 'handleClick1',
      modifiers: { prevent: true }
    },
    {
      value: 'handleClick2'
    }
  ]
}
```

再次修改模板：

```html
<div @click.prevent="handleClick1" @click="handleClick2" @click.self="handleClick3"></div>
```

在上一次修改的基础上添加了第三个 `click` 事件侦听，但是使用了 `self` 修饰符，所以这个 `click` 事件和前两个 `click` 事件也是不同的，此时如下 `if` 语句块的代码将被执行：

```javascript
const handlers = events[name]
/* istanbul ignore if */
if (Array.isArray(handlers)) {
  important ? handlers.unshift(newHandler) : handlers.push(newHandler)
} else if (handlers) {
  events[name] = important ? [newHandler, handlers] : [handlers, newHandler]
} else {
  events[name] = newHandler
}
```

由于此时 `el.events.click` 属性已经是一个数组，所以如上 `if` 语句的判断条件成立。在 `if` 语句块内执行了一行代码，这行代码是一个三元运算符，其作用很简单，可以知道 `important` 所影响的就是事件作用的顺序，所以根据 `important` 参数的不同，会选择使用数组的 `unshift` 方法将新添加的事件信息对象放到数组的头部，或者是选择数组的 `push` 方法将新添加的事件信息对象放到数组的尾部。这样无论有多少个同名的事件监听，都不会落下任何一个监听函数的执行。

接着注意到 `addHandler` 函数的最后一行代码，如下：

```javascript
el.plain = false
```

如果一个标签存在事件侦听，无论如何都不会认为这个元素是“纯”的，所以这里直接将 `el.plain` 设置为 `false`。`el.plain` 属性会影响代码生成阶段，并间接导致程序的执行行为，后面会总结关于 `el.plain` 的变更情况。

以上就是对于 `addHandler` 函数的讲解，可以发现 `addHandler` 函数对于元素描述对象的影响主要是在元素描述对象上添加了 `el.events` 属性和 `el.nativeEvents` 属性。对于 `el.events` 和 `el.nativeEvents` 属性的结构已经讲解得比较详细了。

最后回到 `src/compiler/parser/index.js` 文件中的 `processAttrs` 函数中，如下代码所示：

```javascript
if (bindRE.test(name)) { // v-bind
  // 省略...
} else if (onRE.test(name)) { // v-on
  name = name.replace(onRE, '')
  addHandler(el, name, value, modifiers, false, warn)
} else { // normal directives
  // 省略...
}
```

注意如上代码中调用 `addHandler` 函数时传递的第五个参数为 `false`，它实际上就是 `addHandler` 函数中名字为 `important` 的参数，它影响的是新添加的事件信息对象的顺序，由于上方代码中传递的 `important` 参数为 `false`，所以使用 `v-on` 添加的事件侦听函数将按照添加的顺序被先后执行。

以上就是对于 `processAttrs` 函数中对于 `v-on` 指令的解析。

### 解析其他指令

现在进入解析其他指令的部分，如下代码：

```javascript
if (bindRE.test(name)) { // v-bind
  // 省略...
} else if (onRE.test(name)) { // v-on
  // 省略...
} else { // normal directives
  name = name.replace(dirRE, '')
  // parse arg
  const argMatch = name.match(argRE)
  const arg = argMatch && argMatch[1]
  if (arg) {
    name = name.slice(0, -(arg.length + 1))
  }
  addDirective(el, name, rawName, value, arg, modifiers)
  if (process.env.NODE_ENV !== 'production' && name === 'model') {
    checkForAliasModel(el, value)
  }
}
```

如上代码所示，如果一个指令既不是 `v-bind` 也不是 `v-on`，则如上 `else` 语句块的代码将被执行。这段代码的作用是用来处理除 `v-bind` 和 `v-on` 指令之外的其他指令，但这些指令中不包含 `v-once` 指令，因为 `v-once` 指令已经在 `processOnce` 函数中被处理了，同样的 `v-if/v-else-if/v-else` 等指令也不会被如上这段代码处理，下面是一个表格，表格中列出了所有 Vue 内置提供的指令和已经处理过的指令和剩余未处理指令的对照表格：

| Vue 内置提供的所有指令 | 是否已经被解析 | 解析函数       |
| ---------------------- | -------------- | -------------- |
| `v-if`                 | 是             | `processIf`    |
| `v-else-if`            | 是             | `processIf`    |
| `v-else`               | 是             | `processIf`    |
| `v-for`                | 是             | `processFor`   |
| `v-on`                 | 是             | `processAttrs` |
| `v-bind`               | 是             | `processAttrs` |
| `v-pre`                | 是             | `processPre`   |
| `v-once`               | 是             | `processOnce`  |
| `v-text`               | 否             | 无             |
| `v-html`               | 否             | 无             |
| `v-show`               | 否             | 无             |
| `v-cloak`              | 否             | 无             |
| `v-model`              | 否             | 无             |

通过如上表格可以看到，到目前为止还有五个指令没有得到处理，分别是 `v-text`、`v-html`、`v-show` 以及 `v-model`，除了这个五个 Vue 内置提供的指令之外，开发者还可以自定义指令，所以上方的代码中 `else` 语句块内的代码就是用来处理剩余的这五个内置指令和其他自定义指令的。

回到 `else` 语句块内的代码，如下：

```javascript
if (bindRE.test(name)) { // v-bind
  // 省略...
} else if (onRE.test(name)) { // v-on
  // 省略...
} else { // normal directives
  name = name.replace(dirRE, '')
  // parse arg
  const argMatch = name.match(argRE)
  const arg = argMatch && argMatch[1]
  if (arg) {
    name = name.slice(0, -(arg.length + 1))
  }
  addDirective(el, name, rawName, value, arg, modifiers)
  if (process.env.NODE_ENV !== 'production' && name === 'model') {
    checkForAliasModel(el, value)
  }
}
```

在 `else` 语句块内，首先使用字符串的 `replace` 方法配合 `dirRE` 正则去掉属性名称中的 `'v-'` 或 `':'` 或 `'@'` 等字符，并重新赋值 `name` 变量，所以此时 `name` 变量应该只包含属性名字，假如在一个标签中使用 `v-show` 指令，则此时 `name` 变量的值为字符串 `'show'`。但是对于自定义指令，开发者很可能为该指令提供参数，假设有一个名为 `v-custom` 的指令，并且在使用该指令时为其指定了参数：`v-custom:arg`，这时重新赋值后的 `name` 变量应该是字符串 `'custom:arg'`。如果指令有修饰符那是不是 `name` 变量保存的字符串中也包含修饰符？不会的，在 `processAttrs` 函数中每解析一个指令时都优先使用 `parseModifiers` 函数将修饰符解析完毕了，并且修饰符相关的字符串已经被移除，所以如上代码中的 `name` 变量中将不会包含修饰符字符串。

重新赋值 `name` 变量之后，会执行如下这两行代码：

```javascript
const argMatch = name.match(argRE)
const arg = argMatch && argMatch[1]
```

第一行代码使用 `argRE` 正则匹配变量 `name`，并将匹配结果保存在 `argMatch` 常量中，由于使用的是 `match` 方法，所以如果匹配成功则会返回一个结果数组，匹配失败则会得到 `null`。`argRE` 正则在上一章讲解过，它用来匹配指令字符串中的参数部分，并且拥有一个捕获组用来捕获参数字符串，假设现在 `name` 变量的值为 `custom:arg`，则最终 `argMatch` 常量将是一个数组：

```javascript
const argMatch = [':arg', 'arg']
```

可以看到 `argMatch` 数组中索引为 `1` 的元素保存着参数字符串。有了 `argMatch` 数组后将会执行第二行代码，第二行代码首先检测了 `argMatch` 是否存在，如果存在则取 `argMatch` 数组中索引为 `1` 的元素作为常量 `arg` 的值，所以常量 `arg` 所保存的就是参数字符串。

再往下是一个 `if` 条件语句，如下：

```javascript
if (arg) {
  name = name.slice(0, -(arg.length + 1))
}
```

这个 `if` 语句检测了参数字符串 `arg` 是否存在，如果存在说明有参数传递给该指令，此时会执行 `if` 语句块的代码。可以发现 `if` 语句块内的这行代码的作用就是用来将参数字符串从 `name` 字符串中移除掉，由于参数字符串 `arg` 不包含冒号 `:` 字符，所以需要使用 `-(arg.length + 1)` 才能正确截取。举个例子，假设此时 `name` 字符串为 `'custom:arg'`，再经过如上代码处理之后，最终 `name` 字符串将变成 `'custom'`，可以看到此时的 `name` 变量已经变成了真正的指令名字了。

再往下，将执行如下这行代码：

```javascript
addDirective(el, name, rawName, value, arg, modifiers)
```

这行代码调用了 `addDirective` 函数，并传递给该函数六个参数，为了让大家有直观的感受，举个例子，假设有指令为：`v-custom:arg.modif="myMethod"`，最终调用 `addDirective` 函数时所传递的参数如下：

```javascript
addDirective(el, 'custom', 'v-custom:arg.modif', 'myMethod', 'arg', { modif: true })
```

实际上 `addDirective` 函数和 `addHandler` 函数类似，只不过 `addDirective` 函数的作用是用来在元素描述对象上添加 `el.directives` 属性的，如下是 `addDirective` 函数的源码，它来自 `src/compiler/helpers.js` 文件：

```javascript
export function addDirective (
  el: ASTElement,
  name: string,
  rawName: string,
  value: string,
  arg: ?string,
  modifiers: ?ASTModifiers
) {
  (el.directives || (el.directives = [])).push({ name, rawName, value, arg, modifiers })
  el.plain = false
}
```

可以看到 `addDirective` 函数接收六个参数，在 `addDirective` 函数体内，首先判断了元素描述对象的 `el.directives` 是否存在，如果不存在则先将其初始化一个空数组，然后再使用 `push` 方法添加一个指令信息对象到 `el.directives` 数组中，如果 `el.directives` 属性已经存在，则直接使用 `push` 方法将指令信息对象添加到 `el.directives` 数组中。我们一直说的 **指令信息对象** 实际上指的就是如上代码中传递给 `push` 方法的参数：

```javascript
{ name, rawName, value, arg, modifiers }
```

另外注意到在 `addDirective` 函数的最后，和 `addHandler` 函数类似，也有一行代码将元素描述对象的 `el.plain` 属性设置为 `false` 的代码。

回到 `processAttrs` 函数中，继续看代码，如下代码所示：

```javascript
if (bindRE.test(name)) { // v-bind
  // 省略...
} else if (onRE.test(name)) { // v-on
  // 省略...
} else { // normal directives
  name = name.replace(dirRE, '')
  // parse arg
  const argMatch = name.match(argRE)
  const arg = argMatch && argMatch[1]
  if (arg) {
    name = name.slice(0, -(arg.length + 1))
  }
  addDirective(el, name, rawName, value, arg, modifiers)
  if (process.env.NODE_ENV !== 'production' && name === 'model') {
    checkForAliasModel(el, value)
  }
}
```

这段代码是 `else` 语句块的最后一段代码，他是一个 `if` 条件语句块，在非生产环境下，如果指令的名字为 `model`，则会调用 `checkForAliasModel` 函数，并将元素描述对象和 `v-model` 属性值作为参数传递，找到 `checkForAliasModel` 函数，如下：

```javascript
function checkForAliasModel (el, value) {
  let _el = el
  while (_el) {
    if (_el.for && _el.alias === value) {
      warn(
        `<${el.tag} v-model="${value}">: ` +
        `You are binding v-model directly to a v-for iteration alias. ` +
        `This will not be able to modify the v-for source array because ` +
        `writing to the alias is like modifying a function local variable. ` +
        `Consider using an array of objects and use v-model on an object property instead.`
      )
    }
    _el = _el.parent
  }
}
```

`checkForAliasModel` 函数的作用就是从使用了 `v-model` 指令的标签开始，逐层向上遍历父级标签的元素描述对象，知道根元素为止。并且在遍历的过程中一旦发现这些标签的元素描述对象中存在满足条件：`_el.for && _el.alias === value` 的情况，就会打印警告信息。先来看如下条件：

```javascript
if (_el.for && _el.alias === value)
```

如果这个条件成立，则说明使用了 `v-model` 指令的标签或其父代标签使用了 `v-for` 指令，如下：

```html
<div v-for="item of list">
  <input v-model="item" />
</div>
```

假设如上代码中的 `list` 数组如下：

```javascript
[1, 2, 3]
```

此时将会渲染三个输入框，但是当修改输入框的值时，这个变更是不会体现到 `list` 数组的，换句话说如上代码中的 `v-model` 指令无效。这个问题和 `v-for` 指令的实现有关，如上代码中 `v-model` 指令所执行的修改操作等价于修改了函数的局部变量，这当然不会影响到真正的数据。为了解决这个问题，Vue 也提供了一个方案，那就是使用对象数组替代基本类型的数组，并在 `v-model` 指令中绑定对象的属性，修改一下上例并使其生效：

```html
<div v-for="obj of list">
  <input v-model="obj.item" />
</div>
```

此时再定义 `list` 数组时，应该将其定义为：

```javascript
[
  { item: 1 },
  { item: 2 },
  { item: 3 },
]
```

所以实际上 `checkForAliasModel` 函数的作用就是给开发者合适的提醒。

以上就是对自定义指令和剩余的五个未被解析的内置指令的处理，可以看到每当遇到一个这样的指令，都会在元素描述对象的 `el.directives` 数组中添加一个指令信息对象，如下：

```javascript
el.directives = [
  {
    name, // 指令名字
    rawName, // 指令原始名字
    value, // 指令的属性值
    arg, // 指令的参数
    modifiers // 指令的修饰符
  }
]
```

注意，如上注释中把指令信息对象中的 `value` 属性说成“指令的属性值”，在解析编译阶段一切都是字符串，并不是 Vue 中数据状态的值，千万不要搞混淆。

### 处理非指令属性

上一节中讲解了 `processAttrs` 函数对于指令的处理，紧接着讲解 `processAttrs` 函数对于那些非指令的属性是如何处理的，如下代码所示：

```javascript
function processAttrs (el) {
  const list = el.attrsList
  let i, l, name, rawName, value, modifiers, isProp
  for (i = 0, l = list.length; i < l; i++) {
    name = rawName = list[i].name
    value = list[i].value
    if (dirRE.test(name)) {
      // 省略...
    } else {
      // 省略...
    }
  }
}
```

如上代码所示，这个 `else` 语句块内代码的作用就是用来处理非指令属性的，如下列出的非指令属性是之前的讲解中已经讲过的属性：

- `key`
- `ref`
- `slot`、`slot-scope`、`scope`、`name`
- `is`、`inline-template`

这些非指令属性都已经被相应的处理函数解析过了，所以 `processAttrs` 函数是不负责处理如上这些非指令属性的。换句话说除了以上这些以外，其他的非指令属性基本都由 `processAttrs` 函数来处理，比如 `id`、`width` 等，如下：

```html
<div id="box" width="100px"></div>
```

如上 `div` 标签中的 `id` 属性和 `width` 属性都会被 `processAttrs` 函数处理，可能有的同学会有疑惑，`class` 属性是不是也被 `processAttrs` 函数处理了？并不是，在 `processElement` 函数中有这样一段代码：

```javascript
for (let i = 0; i < transforms.length; i++) {
  element = transforms[i](element, options) || element
}
```

这段代码是在 `processAttrs` 函数之前执行，并且这段代码的作用就是调用“中置处理”钩子，而 `class` 属性和 `style` 属性都会在“中置处理”钩子中被处理，而并非 `processAttrs` 函数。

接着查看这段用来处理非指令属性的代码，如下 `else` 语句块内的代码所示：

```javascript
if (dirRE.test(name)) {
  // 省略...
} else {
  // literal attribute
  if (process.env.NODE_ENV !== 'production') {
    const res = parseText(value, delimiters)
    if (res) {
      warn(
        `${name}="${value}": ` +
        'Interpolation inside attributes has been removed. ' +
        'Use v-bind or the colon shorthand instead. For example, ' +
        'instead of <div id="{{ val }}">, use <div :id="val">.'
      )
    }
  }
  addAttr(el, name, JSON.stringify(value))
  // #6887 firefox doesn't update muted state if set via attribute
  // even immediately after element creation
  if (!el.component &&
      name === 'muted' &&
      platformMustUseProp(el.tag, el.attrsMap.type, name)) {
    addProp(el, name, 'true')
  }
}
```

如上 `else` 语句块内的代码中，首先执行的是如下这段代码，它是一个 `if` 条件语句块：

```javascript
if (process.env.NODE_ENV !== 'production') {
  const res = parseText(value, delimiters)
  if (res) {
    warn(
      `${name}="${value}": ` +
      'Interpolation inside attributes has been removed. ' +
      'Use v-bind or the colon shorthand instead. For example, ' +
      'instead of <div id="{{ val }}">, use <div :id="val">.'
    )
  }
}
```

可以看到，在非生产环境下才会执行该 `if` 语句块内的代码，在该 `if` 语句块内首先调用了 `parseText` 函数，这个函数来自于 `src/compiler/parser/text-parser.js` 文件，`parseText` 函数的作用是用来解析字面量表达式的，如下模板代码所示：

```html
<div id="{{ isTrue ? 'a' : 'b' }}"></div>
```

其中字符串 `"b"` 就称为字面量表达式，此时就会使用 `parseText` 函数来解析这段字符串。至于 `parseText` 函数是如何对这段字符串进行解析的，后面讲解会详细说明。这里只需要知道，如果使用 `parseText` 函数能够成功解析某个非指令属性的属性值字符串，则说明该非指令属性的属性值使用了字面量表达式，就如同上方模板中的 `id` 属性一样。此时会打印警告信息，提示开发者使用绑定属性作为替代，如下：

```html
<div :id="isTrue ? 'a' : 'b'"></div>
```

这就是上方那段 `if` 语句块代码的作用，继续往下看代码，接下来将执行这行代码：

```javascript
addAttr(el, name, JSON.stringify(value))
```

可以看到，对于任何非指令属性，都会使用 `addAttr` 函数将该属性和该属性对应的字符串添加到元素描述对象的 `el.attrs` 数组中。这里需要注意的是，如上这行代码中使用 `JSON.stringify` 函数对属性值做了处理，这么做的目的就是让该属性的值当做一个纯字符串对待。

理论上，代码运行到这里已经足够了，该做的事情已经完成了，但是发现在 `else` 语句块的最后，还有如下这样一段代码：

```javascript
// #6887 firefox doesn't update muted state if set via attribute
// even immediately after element creation
if (!el.component &&
    name === 'muted' &&
    platformMustUseProp(el.tag, el.attrsMap.type, name)) {
  addProp(el, name, 'true')
}
```

实际上元素描述对象的 `el.attrs` 数组中所存储的任何属性都会在由虚拟 DOM 创建真实 DOM 的过程中使用 `setAttribute` 方法将属性添加到真实 DOM 元素上，而在火狐浏览器中存在无法通过 DOM 元素的 `setAttribute` 方法为 `video` 标签添加 `muted` 属性的问题，所以如上代码就是为了解决该问题的，其方案是如果一个属性的名字是 `muted` 并且该标签满足 `platformMustUseProp` 函数 (`video` 标签满足)，则会额外调用 `addProp` 函数将属性添加到元素描述对象的 `el.props` 数组中。之所以这样子做，是因为元素描述对象的 `el.props` 数组中所存储的任何属性都会在由虚拟 DOM 创建真实 DOM 的过程中直接使用真实 DOM 对象添加，也就是说对于 `<video>` 标签的 `muted` 属性的添加方式为：`videoEl.muted = true`。另外如上代码的注释中已经提供了相应的 `issue` 号：`#6887`。

## `preTransformNode` 前置处理

讲解完了 `processAttrs` 函数之后，所有的 `process*` 系列函数讲解完毕了。另外，目前所讲解的内容都是在 `parseHTML` 函数的 `start` 钩子中运行的代码，如下代码所示：

```javascript
parseHTML(template, {
  warn,
  expectHTML: options.expectHTML,
  isUnaryTag: options.isUnaryTag,
  canBeLeftOpenTag: options.canBeLeftOpenTag,
  shouldDecodeNewlines: options.shouldDecodeNewlines,
  shouldDecodeNewlinesForHref: options.shouldDecodeNewlinesForHref,
  shouldKeepComment: options.comments,
  start (tag, attrs, unary) {
    // 省略...
  },
  end () {
    // 省略...
  },
  chars (text: string) {
    // 省略...
  },
  comment (text: string) {
    // 省略...
  }
})
```

也就是说现在讲解的内容都是在当解析器遇到开始标签时所做的内容，接下来要讲解的内容就是 `start` 钩子函数中的如下这段代码：

```javascript
// apply pre-transforms
for (let i = 0; i < preTransforms.length; i++) {
  element = preTransforms[i](element, options) || element
}
```

这段代码是在应用前置转换 (或前置处理)，其中 `preTransforms` 变量是一个数组，这个数组中包含了所有前置处理的函数，如下代码所示：

```javascript
preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
```

由上代码可知 `preTransforms` 变量的值是使用 `pluckModuleFunction` 函数从 `options.modules` 编译器选项中读取 `preTransformNode` 字段筛选出来的。具体筛选过程在前面的章节中已经讲解过了。

这里说一下编译器选项中的 `modules`，在前边的章节当中，可以知道编译器的选项来自于两部分，一部分是创建编译器时传递的基本选项 `baseOptions`，另一部分则是在使用编译器编译模板时传递的选项参数。如下时创建编译器的基本选项：

```javascript
import { baseOptions } from './options'
import { createCompiler } from 'compiler/index'

const { compile, compileToFunctions } = createCompiler(baseOptions)
```

如上代码来自 `src/platforms/web/compiler/index.js` 文件，可以看到 `baseOptions` 导入自 `src/platforms/web/compiler/options.js` 文件。

最终了解到编译器选项的 `modules` 选项来自 `src/platforms/web/compiler/modules/index.js` 文件导出的一个数组，如下：

```javascript
import klass from './class'
import style from './style'
import model from './model'

export default [
  klass,
  style,
  model
]
```

如果把 `modules` 数组展开的话，它长成如下这个样子：

```javascript
[
  // klass
  {
    staticKeys: ['staticClass'],
    transformNode,
    genData
  },
  // style
  {
    staticKeys: ['staticStyle'],
    transformNode,
    genData
  },
  // model
  {
    preTransformNode
  }
]
```

根据如上数组可以发现 `modules` 数组中的每一个元素都是一个对象，并且 `klass` 对象和 `style` 对象都拥有 `transformNode` 属性，而 `model` 对象中则有一个 `preTransformNode` 属性。打开 `src/compiler/parser/index.js` 文件，找到如下代码：

```javascript
preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
```

这时应该知道 `preTransforms` 变量应该是一个数组：

```javascript
preTransforms = [
  preTransformNode
]
```

并且数组中只有一个元素 `preTransformNode`，而这里的 `preTransformNode` 就是来自于 `src/platforms/web/compiler/modules/model.js` 文件中的 `preTransformNode` 函数。接着重点讲解的是 `preTransformNode` 函数的作用，既然它是对元素描述对象做前置处理的，就看看它做了哪些处理。

> 为了方便描述，后续会把 `src/platforms/web/compiler/modules/model.js` 文件简称 `model.js`

如下是 `preTransformNode` 函数的签名以及函数体内一开始的一段代码：

```javascript
function preTransformNode (el: ASTElement, options: CompilerOptions) {
  if (el.tag === 'input') {
    const map = el.attrsMap
    if (!map['v-model']) {
      return
    }

    // 省略...
  }
}
```

`preTransformNode` 函数接收两个参数，第一个参数是预处理的元素描述对象，第二个参数则是透传过来的编译器的选项参数。在 `preTransformNode` 函数内部，所有的代码都被包含在一个 `if` 条件语句中，该 `if` 语句的条件是：

```javascript
if (el.tag === 'input')
```

也就是说只有当前解析的标签时 `input` 标签才会执行预处理工作，看来 `preTransformNode` 函数是用来预处理 `input` 标签的。如果当前解析的元素是 `input` 标签，则会继续判断该 `input` 标签是否使用了 `v-model` 属性：

```javascript
const map = el.attrsMap
if (!map['v-model']) {
  return
}
```

如果该 `input` 标签没有使用 `v-model` 属性，则函数直接返回，什么都不做。所以可以说 `preTransformNode` 函数要预处理的是 **使用了 `v-model` 属性的 `input` 标签**，不过还没有完，继续看如下代码：

```javascript
let typeBinding
if (map[':type'] || map['v-bind:type']) {
  typeBinding = getBindingAttr(el, 'type')
}
if (!map.type && !typeBinding && map['v-bind']) {
  typeBinding = `(${map['v-bind']}).type`
}

if (typeBinding) {
  // 省略...
}
```

上方这段代码是 `preTransformNode` 函数中剩余的所有代码，只不过省略了最后一个 `if` 语句块内的代码。注意如上代码的 `if` 条件语句，可以发现只有当 `typeBinding` 变量为 `true` 的情况下才会执行该 `if` 语句块内的代码，而该 `if` 语句块内的代码才是用来完成主要工作的代码。实际上 `typeBinding` 变量保存的是该 `input` 标签上绑定的 `type` 属性的值，举例说明，假如有如下模板：

```html
<input v-model="val" :type="inputType" />
```

则 `typeBinding` 变量的值为字符串 `'inputType'`。来看源码的实现，首先是如下这段代码：

```javascript
if (map[':type'] || map['v-bind:type']) {
  typeBinding = getBindingAttr(el, 'type')
}
```

由于开发者在绑定属性的时候可以选择 `v-bind:` 或者缩写 `:` 两种方式，所以如上代码中分别获取了通过这两种方式绑定的 `type` 属性，如果存在其一，则使用 `getBindingAttr` 函数获取绑定的 `type` 属性的值。如果开发者没有这两种方式绑定 `type` 属性，则代码会继续执行，来到如下这段 `if` 条件语句：

```javascript
if (!map.type && !typeBinding && map['v-bind']) {
  typeBinding = `(${map['v-bind']}).type`
}
```

如果该 `if` 条件语句的判断条件成立，则说明该 `input` 标签没有使用非绑定的 `type` 属性，并且没有使用 `v-bind:` 或 `:` 绑定 `type` 属性，并且开发者使用了 `v-bind`。这里需要注意，开发者即使没有使用 `v-bind:` 或 `:` 绑定 `type` 属性，但仍然可以通过如下方式绑定属性：

```html
<input v-model="val" v-bind="{ type: inputType }" />
```

此时就需要通过读取绑定对象的 `type` 属性来获取绑定的属性值，即：

```javascript
typeBinding = `(${map['v-bind']}).type`
```

如上这行代码相当于：

```javascript
typeBinding = `({ type: inputType }).type`
```

总之要想办法获取到绑定的 `type` 属性的值，如果获取不到则说明该 `input` 标签的类型是固定不变的，因为它是非绑定的。只有当一个 `input` 表单拥有绑定的 `type` 属性时才会执行真正的预处理代码，所以进一步总结：**`preTransformNode` 函数要预处理的是使用了 `v-model` 属性并且使用了绑定的 `type` 属性的 `input` 标签**。

紧接着看 `model.js` 文件开头的一段注释：

```javascript
/**
 * Expand input[v-model] with dyanmic type bindings into v-if-else chains
 * Turn this:
 *   <input v-model="data[type]" :type="type">
 * into this:
 *   <input v-if="type === 'checkbox'" type="checkbox" v-model="data[type]">
 *   <input v-else-if="type === 'radio'" type="radio" v-model="data[type]">
 *   <input v-else :type="type" v-model="data[type]">
 */
```

根据如上注释可知 `preTransformNode` 函数将形如：

```html
<input v-model="data[type]" :type="type">
```

这样的 `input` 标签扩展为如下三种 `input` 标签：

```html
<input v-if="type === 'checkbox'" type="checkbox" v-model="data[type]">
<input v-else-if="type === 'radio'" type="radio" v-model="data[type]">
<input v-else :type="type" v-model="data[type]">
```

可以知道在 AST 中一个标签对应一个元素描述对象，所以从结果来看，`preTransformNode` 函数将一个 `input` 元素描述对象扩展为三个 `input` 标签的元素描述对象。但是由于扩展后的标签由 `v-if`、`v-else-if`、`v-else` 三个条件指令组成，在前面的分析可以得知，对于使用了 `v-else-if`、`v-else` 指令的标签，其元素描述对象是会被添加到那个使用 `v-if` 指令的元素描述对象的 `el.ifConditions` 数组中的。所以虽然把一个 `input` 标签扩展成了三个，但实际上并不会影响到 AST 的结构，并且从渲染结果上来看，也是一致的。

之所以要将一个 `input` 标签扩展为三个，是因为由于使用了绑定的 `type` 属性，所以该 `input` 标签的类型是不确定的，并且同样是 `input` 标签，但是类型为 `checkbox` 的 `input` 标签和类型为 `radio` 的 `input` 标签的行为是不一致的。到代码生成阶段会看到，正是因为这里将 `input` 标签类型做了区分，才使得代码生成时能根据三种不同情况生成三种对应的代码，从而实现三种不同的功能。但是这里不做区分也是可以的，假设这里不做区分，那么在代码生成时不可以知道目标 `input` 元素的类型，为了保证实现所有类型 `input` 标签的功能可用，必须保证生成的代码能够能完成所有类型标签的工作。所以需要选择再编译阶段区分类型，或者运行阶段区分类型，Vue 选择了在编译阶段将类型区分开来，好处是运行时的代码在针对某种特定类型的 `input` 标签时所执行的代码是单一职责的。后面代码生成时，在编译阶段区分类型使得代码编写更加容易。从另外一个角度来说，由于不同类型的 `input` 标签所绑定的事件未必相同，这也是在编译阶段区分 `input` 标签类型的一个重要因素。

看具体实现，首先是如下这段代码：

```javascript
if (typeBinding) {
  const ifCondition = getAndRemoveAttr(el, 'v-if', true)
  const ifConditionExtra = ifCondition ? `&&(${ifCondition})` : ``
  const hasElse = getAndRemoveAttr(el, 'v-else', true) != null
  const elseIfCondition = getAndRemoveAttr(el, 'v-else-if', true)
  // 省略...
}
```

这段代码定义了四个常量，分别是 `ifCondition`、`ifConditionExtra`、`hasElse` 以及 `elseIfCondition`，其中 `ifCondition` 常量保存的值是通过 `getAndRemoveAttr` 函数取得的 `v-if` 指令的值，注意如上代码中调用 `getAndRemoveAttr` 函数时传递的第三个参数为 `true`，所以在获取到属性值之后，会将属性从元素描述对象的 `el.attrsMap` 中移除。

假设有如下模板：

```html
<input v-model="val" :type="inputType" v-if="display" />
```

则 `ifCondition` 常量的值为字符串 `'display'`。

第二个常量 `ifConditionExtra` 同样是一个字符串，还是以如上模板为例，由于 `ifCondition` 常量存在，所以 `ifConditionExtra` 常量的值为字符串 `'&&(display)'`，假若 `ifCondition` 常量不存在，则 `ifConditionExtra` 常量的值将是一个空字符串。

第三个常量 `hasElse` 是一个布尔值，代表着 `input` 指令标签是否使用了 `v-else` 指令。其实现方式也是通过 `getAndRemoveAttr` 函数获取 `v-else` 指令的属性值，然后将值和 `null` 作比较。如果 `input` 标签使用 `v-else` 指令，则 `hasElse` 常量的值为 `true`，反之为 `false`。

第四个常量 `elseIfCondition` 和 `ifCondition` 类似，只不过 `elseIfCondition` 所存储的是 `v-else-if` 指令的属性值。

往下代码如下：

```javascript
// 1. checkbox
const branch0 = cloneASTElement(el)
// process for on the main node
processFor(branch0)
addRawAttr(branch0, 'type', 'checkbox')
processElement(branch0, options)
branch0.processed = true // prevent it from double-processed
branch0.if = `(${typeBinding})==='checkbox'` + ifConditionExtra
addIfCondition(branch0, {
  exp: branch0.if,
  block: branch0
})
```

前面所讲，`preTransformNode` 函数的作用就是将一个拥有绑定类型和 `v-model` 指令的 `input` 标签扩展为三个 `input` 标签，这三个 `input` 标签分别是复选按钮 `checkbox`、单选按钮 `radio` 和其他 `input` 标签。而如上这段代码的作用就是创建复选按钮的，首先调用 `cloneASTElement` 函数克隆出一个和原始标签的元素描述对象一模一样的元素描述对象出来，并将新克隆出的元素描述对象赋值给 `branch0` 常量。看 `cloneASTElement` 函数的实现，如下：

```javascript
function cloneASTElement (el) {
  return createASTElement(el.tag, el.attrsList.slice(), el.parent)
}
```

就是通过 `createASTElement` 函数再创建出一个元素描述对象，但是由于 `el.attrsList` 数组是引用类型，所以为了避免克隆的元素描述对象和原始描述对象互相干扰，所以需要使用数组的 `slice` 方法克隆出一个新的 `el.attrsList` 数组。

获取到新的元素描述对象参考 `src/compiler/parser/index.js` 文件，在解析开始标签的 `start` 钩子函数中有如下代码：

```javascript
if (inVPre) {
  processRawAttrs(element)
} else if (!element.processed) {
  // structural directives
  processFor(element)
  processIf(element)
  processOnce(element)
  // element-scope stuff
  processElement(element, options)
}
```

如上代码中的 `else if` 分支中，对于一个不在 `v-pre` 指令内的标签，会使用四个 `process*` 函数处理，所以在 `preTransformNode` 函数中同样需要这四个 `process*` 函数对标签的元素描述对象做处理，如下代码所示：

```javascript
// 1. checkbox
const branch0 = cloneASTElement(el)
// process for on the main node
processFor(branch0)
addRawAttr(branch0, 'type', 'checkbox')
processElement(branch0, options)
branch0.processed = true // prevent it from double-processed
branch0.if = `(${typeBinding})==='checkbox'` + ifConditionExtra
addIfCondition(branch0, {
  exp: branch0.if,
  block: branch0
})s
```

上方代码中分别调用了 `processFor` 函数和 `processElement` 函数，这里并没有调用 `processOnce` 函数以及 `processIf` 函数，对于 `processOnce` 函数，没有使用说明一个问题，即如下代码中的 `v-once` 指令无效：

```html
<input v-model="val" :type="inputType" v-once />
```

对于一个使用了 `v-model` 指令又使用了绑定 `type` 属性的 `input` 标签而言，不存在静态的意义。

没有调用 `processIf` 函数，因为对于条件指令在前边已经处理完了，如下所示：

```javascript
const ifCondition = getAndRemoveAttr(el, 'v-if', true)
const ifConditionExtra = ifCondition ? `&&(${ifCondition})` : ``
const hasElse = getAndRemoveAttr(el, 'v-else', true) != null
const elseIfCondition = getAndRemoveAttr(el, 'v-else-if', true)
```

实际上 `preTransformNode` 函数的处理逻辑就是把一个 `input` 标签扩展为多个标签，并且这些扩展出来的标签彼此之间是互斥的，这些扩展出来的标签都存在于元素描述对象的 `el.ifConditions` 数组中。

接着看代码，如下代码所示：

```javascript
// 1. checkbox
const branch0 = cloneASTElement(el)
// process for on the main node
processFor(branch0)
addRawAttr(branch0, 'type', 'checkbox')
processElement(branch0, options)
branch0.processed = true // prevent it from double-processed
branch0.if = `(${typeBinding})==='checkbox'` + ifConditionExtra
addIfCondition(branch0, {
  exp: branch0.if,
  block: branch0
})
```

在 `processFor` 函数和 `processElement` 函数中调用了 `addRawAttr` 函数，该函数来自于 `src/compiler/helpers.js` 文件，源码如下：

```javascript
export function addRawAttr (el: ASTElement, name: string, value: any) {
  el.attrsMap[name] = value
  el.attrsList.push({ name, value })
}
```

代码很容易理解，`addRawAttr` 函数的作用就是将属性的名和值分别添加到元素描述对象的 `el.attrsMap` 对象和 `el.attrsList` 数组中，以如下为例：

```javascript
addRawAttr(branch0, 'type', 'checkbox')
```

这样相当于把新克隆的标签视为：

```html
<input type="checkbox" />
```

继续看下方代码：

```javascript
// 1. checkbox
const branch0 = cloneASTElement(el)
// process for on the main node
processFor(branch0)
addRawAttr(branch0, 'type', 'checkbox')
processElement(branch0, options)
branch0.processed = true // prevent it from double-processed
branch0.if = `(${typeBinding})==='checkbox'` + ifConditionExtra
addIfCondition(branch0, {
  exp: branch0.if,
  block: branch0
})
```

如上代码中 `branch0.processed = true` 将元素描述对象的 `el.processed` 属性设置为 `true`，标识着当前元素描述对象已经被处理过了，回到 `src/compiler/parser/index.js` 文件中 `start` 钩子函数的如下这段代码：

```javascript
if (inVPre) {
  processRawAttrs(element)
} else if (!element.processed) {
  // structural directives
  processFor(element)
  processIf(element)
  processOnce(element)
  // element-scope stuff
  processElement(element, options)
}
```

如上的 `else if` 中的条件判断语句 `!el.processed` 属性值已经为 `true`，所以判断条件将会为 `false`，即 `else if` 语句块内的代码将不会被执行，这样的目的是为了避免重复的解析。

对于第一个克隆的元素描述对象来说，最后执行的将是如下代码：

```javascript
branch0.if = `(${typeBinding})==='checkbox'` + ifConditionExtra
addIfCondition(branch0, {
  exp: branch0.if,
  block: branch0
})
```

这段代码为元素描述对象添加了 `el.if` 属性，其 `if` 属性值为：

```javascript
`(${typeBinding})==='checkbox'` + ifConditionExtra
```

假设有如下模板：

```html
<input v-model="val" :type="inputType" v-if="display" />
```

则 `el.if` 属性的值将为：`'(${inputType}) === "checkbox" && display`，可以看到只有当本地状态 `inputType` 的值为字符串 `'checkbox'` 并且本地状态 `display` 为 `true` 才会渲染该复选按钮。

另外如果一个标签使用了 `v-if` 指令，则该标签的元素描述对象被添加到其自身的 `el.ifConditions` 数组中，所以需要执行如下代码：

```javascript
addIfCondition(branch0, {
  exp: branch0.if,
  block: branch0
})
```

至此，对于第一个扩展出来的复选按钮就完成了，紧接着是后面的代码，如下：

```javascript
// 2. add radio else-if condition
const branch1 = cloneASTElement(el)
getAndRemoveAttr(branch1, 'v-for', true)
addRawAttr(branch1, 'type', 'radio')
processElement(branch1, options)
addIfCondition(branch0, {
  exp: `(${typeBinding})==='radio'` + ifConditionExtra,
  block: branch1
})
// 3. other
const branch2 = cloneASTElement(el)
getAndRemoveAttr(branch2, 'v-for', true)
addRawAttr(branch2, ':type', typeBinding)
processElement(branch2, options)
addIfCondition(branch0, {
  exp: ifCondition,
  block: branch2
})
```

这段代码和扩展复选按钮一样，分为两部分，第一部分用来扩展单选按钮，而第二部分用来扩展其他类型的 `input` 标签。需要注意的有两点，第一点是如上代码中无论是扩展单选按钮还是扩展其他类型的 `input` 标签，它们都重新使用 `cloneASTElement` 函数克隆出了新的元素描述对象并且这两个元素描述对象都会被添加到复选按钮元素描述对象的 `el.ifConditions` 数组中。第二点需要注意的是无论是扩展单选按钮还是扩展其他类型的 `input` 标签，都执行了如下这行代码：

```javascript
getAndRemoveAttr(branch2, 'v-for', true)
```

这里将克隆出来的元素描述对象中的 `v-for` 属性移除掉，因为在复选按钮中已经使用 `processFor` 处理了 `v-for` 指令，由于它们本身是互斥的，本质上等价于是同一个元素，只是根据不同的条件渲染不同的标签而已，因此 `v-for` 指令处理一次就够了。

再往下执行时如下这段代码：

```javascript
if (hasElse) {
  branch0.else = true
} else if (elseIfCondition) {
  branch0.elseif = elseIfCondition
}
```

前面的讲解中，所举的例子都是使用 `v-if` 指令的 `input` 标签，但是 `input` 标签也可能使用 `v-else-if` 或 `v-else`，如下：

```html
<div v-if="num === 1"></div>
<input v-model="val" :type="inputType" v-else />
```

最后 `preTransformNode` 函数将返回一个全新的元素描述对象：

```javascript
function preTransformNode (el: ASTElement, options: CompilerOptions) {
  if (el.tag === 'input') {
    // 省略...
    if (typeBinding) {
      // 省略...
      return branch0
    }
  }
}
```

回到 `src/compiler/parser/index.js` 文件找到应用预处理钩子的代码，如下：

```javascript
// apply pre-transforms
for (let i = 0; i < preTransforms.length; i++) {
  element = preTransforms[i](element, options) || element
}
```

通过预处理函数之后得到了新的元素描述对象，则使用新的元素描述对象替换当前元素描述对象 (`element`)，否则仍然使用 `element` 作为元素描述对象。

## `transformNode` 中置处理

在前置处理中，目前只有一个用来处理使用了 `v-model` 指令并且使用绑定的 `type` 属性的 `input` 标签的前置处理函数。与之不同，中置处理函数 `transformNode` 则有两个分别对 `class` 属性和 `style` 属性进行扩展，打开 `src/compiler/parser/index.js` 函数找到 `processElement` 函数，如下代码所示：

```javascript
export function processElement (element: ASTElement, options: CompilerOptions) {
  processKey(element)

  // determine whether this is a plain element after
  // removing structural attributes
  element.plain = !element.key && !element.attrsList.length

  processRef(element)
  processSlot(element)
  processComponent(element)
  for (let i = 0; i < transforms.length; i++) {
    element = transforms[i](element, options) || element
  }
  processAttrs(element)
}
```

可以看到中置处理函数的应用时机是在 `processAttrs` 函数之前，使用 `for` 循环遍历了 `transforms` 数组，`transform` 数组中包含了两个 `transformNode` 函数，分别来自 `src/platforms/web/compiler/modules/class.js` 文件和 `src/platforms/web/compiler/modules/style.js` 文件。先看 `class.js` 文件，在该文件下找到 `transformNode` 函数，如下：

```javascript
function transformNode (el: ASTElement, options: CompilerOptions) {
  const warn = options.warn || baseWarn
  const staticClass = getAndRemoveAttr(el, 'class')
  if (process.env.NODE_ENV !== 'production' && staticClass) {
    const res = parseText(staticClass, options.delimiters)
    if (res) {
      warn(
        `class="${staticClass}": ` +
        'Interpolation inside attributes has been removed. ' +
        'Use v-bind or the colon shorthand instead. For example, ' +
        'instead of <div class="{{ val }}">, use <div :class="val">.'
      )
    }
  }
  if (staticClass) {
    el.staticClass = JSON.stringify(staticClass)
  }
  const classBinding = getBindingAttr(el, 'class', false /* getStatic */)
  if (classBinding) {
    el.classBinding = classBinding
  }
}
```

在该 `transformNode` 函数内，首先执行的是如下两句代码：

```javascript
const warn = options.warn || baseWarn;
const staticClass = getAndRemoveAttr(el, 'class');
```

定义 `warn` 常量，它是一个函数，用来打印警告信息。接着使用 `getAndRemoveAttr` 函数从元素描述对象上获取非绑定的 `class` 属性的值，并将其保存在 `staticClass` 常量中。接着进入一条 `if` 条件语句：

```javascript
if (process.env.NODE_ENV !== 'production' && staticClass) {
  const res = parseText(staticClass, options.delimiters)
  if (res) {
    warn(
      `class="${staticClass}": ` +
      'Interpolation inside attributes has been removed. ' +
      'Use v-bind or the colon shorthand instead. For example, ' +
      'instead of <div class="{{ val }}">, use <div :class="val">.'
    )
  }
}
```

在非生产环境下，并且非绑定的 `class` 属性值存在，则会使用 `parseText` 函数解析该值，如果解析成功则说明在非绑定的 `class` 属性中使用了字面量表达式，例如：

```html
<div class="{{ isActive ? 'active' : '' }}"></div>
```

这时 Vue 会打印警告信息，提示你使用如下这种方式替代：

```html
<div :class="{ 'active': isActive }"></div>
```

再往下是这段代码：

```javascript
if (staticClass) {
  el.staticClass = JSON.stringify(staticClass);
}
```

如果非绑定的 `class` 属性值存在，则将该值保存在元素描述对象的 `el.staticClass` 属性中，注意这里使用了 `JSON.stringify` 函数对值做了处理。再往下是 `transformNode` 函数的最后一段代码：

```javascript
const classBinding = getBindingAttr(el, 'class', false /* getStatic */);
if (classBinding) {
  el.classBinding = classBinding;
}
```

这段代码使用了 `getBindingAttr` 函数获取绑定的 `class` 属性，如果绑定的 `class` 属性的值存在，则将该值保存在 `el.classBinding` 属性中。

以上是中置处理对于 `class` 属性的处理方式，总结一下：

- 非绑定的 `class` 属性保存在元素描述对象中的 `staticClass` 属性中，有如下模板：

  ```html
  <div class="a b c"></div>
  ```

  则该标签元素描述对象的 `el.staticClass` 属性值为：

  ```javascript
  el.staticClass = JSON.stringify('a b c`);
  ```

- 绑定的 `class` 属性值保存在元素描述对象的 `el.classBinding` 属性中，有如下模板：

  ```html
  <div :class="{{ 'active': isActive }}"></div>
  ```

  则该标签元素描述对象的 `el.classBinding` 属性值为：

  ```javascript
  el.classBinding = "{ 'active': isActive}";
  ```

对于 `style` 属性的处理和 `class` 属性的处理类似，用于处理 `style` 属性的中置处理函数位于 `src/platforms/web/compiler/modules/style.js` 文件，如下：

```javascript
function transformNode (el: ASTElement, options: CompilerOptions) {
  const warn = options.warn || baseWarn
  const staticStyle = getAndRemoveAttr(el, 'style')
  if (staticStyle) {
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production') {
      const res = parseText(staticStyle, options.delimiters)
      if (res) {
        warn(
          `style="${staticStyle}": ` +
          'Interpolation inside attributes has been removed. ' +
          'Use v-bind or the colon shorthand instead. For example, ' +
          'instead of <div style="{{ val }}">, use <div :style="val">.'
        )
      }
    }
    el.staticStyle = JSON.stringify(parseStyleText(staticStyle))
  }

  const styleBinding = getBindingAttr(el, 'style', false /* getStatic */)
  if (styleBinding) {
    el.styleBinding = styleBinding
  }
}
```

可以看到，用来处理 `style` 属性的 `transformNode` 函数基本和用来处理 `class` 属性的 `transformNode` 函数相同，不过下方这行代码需要格外注意，留个心：

```javascript
el.staticClass = JSON.stringify(parseStyleText(staticClass));
```

和 `class` 属性不同，如果一个标签使用了非绑定的 `style` 属性，则会使用 `parseStyleText` 函数对属性值进行处理，`parseStyleText` 函数来自 `src/platforms/web/util/style.js` 文件，举个例子说明 `parseStyleText` 函数处理非绑定 `style` 属性值，如下所示：

```html
<div style="color: red; background: green;"></div>
```

如上模板中使用了非绑定的 `style` 属性，属性值为字符串 `'color: red; background: green;'`，`parseStyleText` 函数会把这个字符串解析为对象，如下：

```javascript
{
  color: 'red';
  background: 'green';
}
```

最后再使用 `JSON.stringify` 函数将如上对象变为字符串后赋值给元素描述对象的 `el.staticClass` 属性。

接着细看 `parseStyleText` 函数的源码：

```javascript
export const parseStyleText = cached(function (cssText) {
  const res = {}
  const listDelimiter = /;(?![^(]*\))/g
  const propertyDelimiter = /:(.+)/
  cssText.split(listDelimiter).forEach(function (item) {
    if (item) {
      var tmp = item.split(propertyDelimiter)
      tmp.length > 1 && (res[tmp[0].trim()] = tmp[1].trim())
    }
  })
  return res
})
```

由以上代码可知 `parseStyleText` 函数是由 `cached` 函数创建的高阶函数，`parseStyleText` 接收内联样式字符串作为参数并返回解析后的对象。在 `parseStyleText` 函数内部定义了 `res` 常量，该常量作为 `parseStyleText` 函数的返回值，其初始值是一个空对象，接着定义了两个正则常量 `listDelimiter` 和 `propertyDelimiter`，其实把一个内联样式字符串解析为对象的思路很简单，首先找到样式字符串的规则，如下：

```html
<div style="color: red; background: green;"></div>
```

可以知道，样式字符串中的 `;` 作为样式规则的分隔，而 `:` 作为样式规则中的属性名和值的分隔，所以就了如下的思路：

- 使用 `;` 把样式字符串分割成一个数组，数组中的每个元素都是一条样式规则，以如上模板为例，分隔后的数组应该是：
  
  ```javascript
  [
    'color: red',
    'background: green'
  ]
  ```

  接着遍历该数组，对于每一条样式规则使用 `:` 将其属性名和值再次进行分割，这样就可以得到想要的结果。明白这个思路就很容易理解 `parseStyleText` 函数的代码。

对于 `parseStyleText` 函数的逻辑就不做过多的解释了，这里重点解析一下 `listDelimiter` 正则，如下：

```javascript
const listDelimiter = /;(?![^(]*\))/g;
```

该正则表达式使用了 **正向否定查询 (`(?!`)**，举个例子说明正向否定查询，正则表达式 `/a(?!b)/` 用来匹配后面没有跟字符 `'b'` 的字符 `'a'`。所以如上的正则表达式用来全局匹配字符串中的 `;`，但是该分号必须满足一个条件：**该分号的后面不能跟右圆括号 (`)`)，除非有一个相应的右圆括号 (`)`) 存在**，说起来抽象，举例说明，如下模板所示：

```html
<div style="color: red; background: url(www.xxx.com?a=1&amp;copy=3);"></div>
```

仔细观察如上 `div` 标签的 `style` 属性值中存在三个分号，但只有其中两个分号才是真正的样式规则分隔符，而字符串 `'url(www.xxx.com?a=1&amp;copy=3)'` 中的分号则是不能作为规则分割符的，正则常量 `listDelimiter` 正是为了实现这个功能而设计的。实际上正如上方例子所示，可以知道内联样式是写在 `html` 文件中的，而在 `html` 规则中存在一个叫做 `html` 实体的概念，看如下这段 `html` 模板：

```html
<a href="foo.cgi?chapter=1&copy=3">link</a>
```

这段 `html` 模板在一些浏览器中不能正常工作，这是因为有些浏览器会把 `&copy` 当做 `html` 实体从而把其解析为字符串 `©`，这导致打开链接的时候，变成了访问：`foo.cgi?chapter=1©=3`。

总之，对于非绑定的 `style` 属性，会在元素描述对象上添加 `el.staticStyle` 属性，该属性的值是一个字符串化后的对象。接着对于绑定的 `style` 属性，则会使用如下这段代码来处理：

```javascript
const styleBinding = getBindingAttr(el, 'style', false /* getStatic */)
if (styleBinding) {
  el.styleBinding = styleBinding
}
```

和处理绑定的 `class` 属性类似，使用 `getBindingAttr` 函数获取到绑定的 `style` 属性值后，如果值存在则直接将其赋值给元素描述对象的 `el.styleBinding` 属性。

以上就是中置中置处理对于 `style` 属性的处理方式，总结一下：

- 非绑定的 `style` 属性值保存在元素描述对象的 `el.statciStyle` 属性中，假设有如下模板：
  
  ```html
  <div style="color: red; background: green;"></div>
  ```

  则该标签元素描述对象的 `el.staticStyle` 属性值为：

  ```javascript
  el.staticStyle = JSON.stringify({
    color: 'red',
    background: 'green'
  })
  ```

- 绑定的 `style` 属性值保存在元素描述对象的 `el.styleBinding` 属性中，假设有如下模板：

  ```html
  <div :style="{ fontSize: fontSize + 'px' }"></div>
  ```

  则该标签元素描述对象的 `el.styleBinding` 属性值为：

  ```javascript
  el.styleBinding = "{ fontSize: fontSize + 'px' }"
  ```

现在前置处理 `preTransformNode` 和中置处理 `transformNode` 都讲解完了，还剩下后置处理 `postTransformsNode` 没有讲，每当遇到非一元标签的结束标签或遇到一元标签时都会应用后置处理器，回到 `src/compiler/parser/index.js` 文件，如下代码所示：

```javascript
function closeElement (element) {
  // check pre state
  if (element.pre) {
    inVPre = false
  }
  if (platformIsPreTag(element.tag)) {
    inPre = false
  }
  // apply post-transforms
  for (let i = 0; i < postTransforms.length; i++) {
    postTransforms[i](element, options)
  }
}
```

该 `for` 循环遍历了 `postTransforms` 数组，但实际上 `postTransforms` 是一个空数组，因为当前还没有任何后置处理的钩子函数。这里只是暂时提供一个用于后置处理的出口，当有需要的时候就可以使用。

## 文本节点的元素描述对象

接着主要讲解每当解析器遇到一个文本节点时，会如何为文本节点创建元素描述对象，会对文本节点做哪些特殊的处理。打开文件 `src/compiler/parser/index.js` 文件中找到 `parserHTML` 函数的 `chars` 钩子函数选项，如下代码所示：

```javascript
parseHTML(template, {
  // 省略...
  chars (text: string) {
    // 省略...
  },
  // 省略...
})
```

当解析器遇到文本节点时，如上代码中的 `chars` 钩子函数就会被调用，并且接收该文本节点的文本内容作为参数，来看 `chars` 钩子函数最开始的这段代码：

```javascript
if (!currentParent) {
  if (process.env.NODE_ENV !== 'production') {
    if (text === template) {
      warnOnce(
        'Component template requires a root element, rather than just text.'
      )
    } else if ((text = text.trim())) {
      warnOnce(
        `text "${text}" outside root element will be ignored.`
      )
    }
  }
  return
}
```

这段代码是连续的几个 `if` 条件语句，首先判断了 `currentParent` 变量是否存在，可以知道 `currentParent` 变量指向的是当前节点的父节点，如果父节点不存在才会执行该 `if` 条件语句里面的代码。思考一下，如果 `currentParent` 变量不存在说明什么问题？当代码执行到了这里，那么当前的节点必然是文本节点，并且该文本节点没有父级节点。而一个文本节点没有父节点有两种情况：

- 一：模板中只有文本节点
  
  ```html
  <template>
    我是文本节点
  </template>
  ```

  如上模板中没有根元素，只有一个文本节点。由于没有元素节点，所以 `currentParent` 变量是肯定不存在值的。而 `Vue` 的模板要求必须有一个根元素节点才行。当解析器解析如上模板时，由于模板只有一个文本节点，所以在解析过程中只会调用一次 `chars` 钩子函数，同时将文本节点的内容作为参数传递，此时就会出现一种情况，即：**整个模板的内容和文本节点的内容完全一致**，换句话说 `text === template` 条件成立，这时解析器会打印警告信息提示模板不能只是文本，必须有一个元素节点才行。

- 二：文本节点在根元素的外面

  ```html
  <template>
    <div>根元素内的文本节点</div>根元素外的文本节点
  </template>
  ```

  


## `parseText` 函数解析字面量表达式



## 对结束标签的处理



## 注释节点的元素描述对象