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





### 解析 `v-bind` 指令



### 解析 `v-on` 指令



### 解析其他指令



### 处理非指令属性



## `preTransformNode` 前置处理



## `transformNode` 中置处理



## 文本节点的元素描述对象



## `parseText` 函数解析字面量表达式



## 对结束标签的处理



## 注释节点的元素描述对象