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

## `processAttrs` 处理剩余属性值的方式

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