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



### 解析其他指令



### 处理非指令属性



## `preTransformNode` 前置处理



## `transformNode` 中置处理



## 文本节点的元素描述对象



## `parseText` 函数解析字面量表达式



## 对结束标签的处理



## 注释节点的元素描述对象