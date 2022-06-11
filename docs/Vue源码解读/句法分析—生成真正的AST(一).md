### 句法分析—生成真正的AST(一)

在上一章节中，讲解了解析 `html` 字符串时词法分析的方式，本章节将再进一步，讲解 `Vue` 是如何在词法分析的基础上构建抽象语法树 (AST) 的，即句法分析。

`src/compiler/index.js` 文件中，代码如下：

```javascript
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})
```

可以看到 `parse` 函数的返回值就是抽象语法树 AST，根据文件头部的引用关系可以知道 `parse` 函数来自于 `src/compiler/parser/index.js` 文件，实际上该文件所有内容都在做一件事，即创建 AST。

本章节讲解的目标就是 `src/compiler/parser/index.js` 文件，不过在具体到源码之前，有必要先独立思考一下如何根据词法分析创建一棵抽象语法树。

## 根据令牌生成 AST 的思路

在上一节的末尾讲解了 `parseHTML` 函数的使用，该函数接收一些选项参数，其中包括几个重要的钩子函数，如每当遇到一个开始标签时会调用 `options.start` 钩子函数，每当遇到一个结束标签时会调用的 `options.end` 钩子函数等等。实际上一棵抽象语法树的构建最关键的就是这两个钩子函数，接下来，简单讲解一下构建抽象语法树的思路。

假设有一段 `html` 字符串，如下：

```html
<ul>
  <li>
    <span>文本</span>
  </li>
</ul>
```

那么最终生成的这棵树应该是和如上 `html` 字符串的结构一一对应：

```javascript
├── ul
│   ├── li
│   │   ├── span
│   │   │   ├── 文本
```

如果每一个节点都用一个 JS 对象来表示的话，那么 `ul` 标点可以表示为如下对象：

```json
{
  type: 1,
  tag: 'ul'
}
```

由于每个节点都存在一个父节点和若干个子节点，所以为如上对象添加两个属性：`parent` 和 `children`，分别用来表示当前节点的父节点和它所包含的子节点：

```json
{
  type: 1,
  tag: 'ul',
  parent: null,
  children: []
}
```

同时每个元素节点还可能包含很多属性 (`attributes`)，所以可以为每个节点添加 `attrsList` 属性，用来存储当前节点所拥有的属性：

```javascript
{
  type: 1,
  tag: 'ul',
  parent: null,
  children: [],
  attrsList: []
}
```

按照以上思路，实际上可以为节点的描述对象添加任何需要的属性，从而进一步描述该节点的特征。如果使用如上对象描述之前定义的 `html` 字符串，那么这棵抽象语法树应该长成如下这个样子：

```javascript
{
  type: 1,
  tag: 'ul',
  parent: null,
  attrsList: [],
  children: [
    {
      type: 1,
      tag: 'li',
      parent: ul,
      attrsList: [],
      children: [
        {
          type: 1,
          tag: 'span',
          parent: li,
          attrsList: [],
          children: [
            {
              type: 2,
              tag: '',
              parent: span,
              attrsList: [],
              text: '文本'
            }
          ]
        }
      ]
    }
  ]
}
```

实际上构建抽象语法树的工作就是创建一个类似如上所示的一个能够描述节点关系的对象树，节点和节点之间通过 `parent` 和 `children` 建立关系，每个节点的 `type` 属性用来标识该节点的类别，比如 `type` 为 `1` 代表该节点为元素节点，`type` 为 `2` 代表该节点为文本节点，这只是人为的一个规定，可以用任何方便的形式加以区分。

回到 `parseHTML` 函数，因为目前为止所拥有的只有这一个函数，需要使用该函数构建出一棵如上所述的描述对象。

首先需要定义一个 `parse` 函数，假设该函数就是用来把 `html` 字符串生成 AST 的，如下：

```javascript
function parse (html) {
  let root
  //...
  return root
}
```

如上代码所示，在 `parse` 函数内定义了变量 `root` 并将其返回，其中 `root` 所代表的就是整棵 AST，`parse` 函数体中间的所有代码都是为了充实 `root` 变量。这时就需要借助 `parseHTML` 函数帮助解析 `html` 字符串，如下：

```javascript
function parse (html) {
  let root
  
  parseHTML(html, {
    start (tag, attrs, unary) {
      // 省略...
    },
    end () {
      // 省略...
    }
  })
  
  return root
}
```

从简出发，假设需要解析的 `html` 字符串如下：

```html
<div></div>
```

这段 `html` 字符串仅仅是一个简单的 `div` 标签，甚至没有任何子节点。若要解析如上标签，可以编写如下代码：

```javascript
function parse (html) {
  let root
  
  parseHTML(html, {
    start (tag, attrs, unary) {
      const element = {
        type: 1,
        tag: tag,
        parent: null,
        attrsList: attrs,
        children: []
      }
      if (!root) root = element
    },
    end () {
      // 省略...
    }
  })
  
  return root
}
```

如上代码所示，在 `start` 钩子函数中首先定义了 `element` 常量，它是元素节点的描述对象，接着判断 `root` 是否存在，如果不存在则直接将 `element` 赋值给 `root`。这段代码对于解析 `<div></div>` 这段 `html` 字符串来说已经足够了，当解析这段 `html` 字符串时首先会遇到 `div` 元素的开始标签，此时 `start` 钩子函数将被调用，最终 `root` 变量将被设置为：

```javascript
root = {
  type: 1,
  tag: 'div',
  parent: null,
  attrsList: [],
  children: []
}
```

但是当解析的 `html` 字符串稍微复杂一点的时候，这段用来解析的代码就不能正常了，比如对于如下这段 `html` 字符串：

```html
<div>
  <span></span>
</div>
```

这段 `html` 字符串与之前的 `html` 字符串的不同之处在于 `div` 标签多了一个子节点，即多了一个 `span` 标签。如果继续沿用之前的解析代码，当解析如上 `html` 字符串时首先会遇到 `div` 元素的开始标签，此时 `start` 钩子函数被调用，`root` 变量被设置为：

```javascript
root = {
  type: 1,
  tag: 'div',
  parent: null,
  attrsList: [],
  children: []
}
```

接着会遇到 `span` 元素的开始标签，会再次调用 `start` 钩子函数，由于此时 `root` 变量已经存在，所以不会再次设置 `root` 变量。为了能够更好的解析 `span` 标签，需要对之前的解析代码做一些改变，如下：

```javascript
function parse (html) {
  let root
  let currentParent
  
  parseHTML(html, {
    start (tag, attrs, unary) {
      const element = {
        type: 1,
        tag: tag,
        parent: currentParent,
        attrsList: attrs,
        children: []
      }

      if (!root) {
        root = element
      } else if (currentParent) {
        currentParent.children.push(element)
      }
      if (!unary) currentParent = element
    },
    end () {
      // 省略...
    }
  })
  
  return root
}
```

如上代码所示，首先需要定义 `currentParent` 变量。它的作用是每遇到一个非一元标签，都会将该标签的描述对象作为 `currentParent` 的值，这样当解析该非一元标签的子节点时，子节点的父级就是 `currentParent` 变量。另外在 `start` 钩子函数内部我们在创建 `element` 描述对象时我们使用 `currentParent` 的值作为每个元素描述对象的 `parent` 属性的值。

如果使用以上代码解析如下 `html` 字符串：

```html
<div>
  <span></span>
</div>
```

那么其过程大概是这样的：首先会遇到 `div` 元素的开始标签，此时由于 `root` 不存在，并且 `currentParent` 也不存在，所以会创建一个用于描述该 `div` 元素的对象，并设置 `root` 的值如下：

```javascript
root = {
  type: 1,
  tag: 'div',
  parent: undefined,
  attrsList: [],
  children: []
}
```

还没完，由于 `div` 元素是非一元标签，可以看到在 `start` 钩子函数的末尾有一个 `if` 条件语句，当一个元素为非一元标签时，会设置 `currrentParent` 为该元素的描述对象，所以此时 `currentParent` 也是：

```javascript
currentParent = {
  type: 1,
  tag: 'div',
  parent: undefined,
  attrsList: [],
  children: []
}
```

接着解析这段 `html` 字符串，会遇到 `span` 元素的开始标签，由于此时 `root` 已经存在，所以 `start` 钩子函数会执行 `else...if` 条件的判断，检查 `currentParent` 是否存在，由于 `currentParent` 存在，所以会将 `span` 元素的描述对象添加到 `currentParent` 的 `children` 数组中作为子节点，所以最终生成的 `root` 描述对象为：

```javascript
root = {
  type: 1,
  tag: 'div',
  parent: undefined,
  attrsList: [],
  children: [
    {
      type: 1,
      tag: 'span',
      parent: div,
      attrsList: [],
      children: []
    }
  ]
}
```

到现在为止，解析逻辑看上去可以用了，但实际上还是存在问题的，假设要解析 `html` 字符串再稍微复杂一点，如下：

```html
<div>
  <span></span>
  <p></p>
</div>
```

在之前的基础上 `div` 元素的子节点多了一个 `p` 标签，按照现有的解析逻辑在解析这段 `html` 字符串时，首先会遇到 `div` 元素的开始标签，此时 `root` 和 `currentParent` 将被设置为 `div` 标签的描述对象。接着会遇到 `span` 元素的开始标签，此时 `span` 标签的描述对象将被添加到 `div` 标签描述对象的 `children` 数组中，同时别忘了 `span` 标签也是非一元标签，所以 `currentParent` 变量会被设置为 `span` 标签的描述对象。接着继续解析，会遇到 `span` 元素的结束标签，由于 `end` 钩子函数什么都没有做，直接跳过。再继续解析将遇到 `p` 元素的开始标签，需要注意，**在解析 `p` 元素的开始标签时，由于 `currentParent` 变量引用的是 `span` 元素的描述对象，所以 `p` 元素的描述对象将被添加到 `span` 元素描述对象的 `children` 数组中，被误认为是 `span` 元素的子节点。**而事实上 `p` 标签是 `div` 元素的子节点，这就是问题的所在。

为了解决这个问题，需要每当遇到一个非一元标签的结束标签时，都将 `currentParent` 变量的值回退到之前的元素描述对象，这样就能够保证当前正在解析的标签拥有正确的父级。若要回退之前的值，那么必然需要一个变量保存之前的值，所以需要一个数组 `stack`，如下代码所示：

```javascript
function parse (html) {
  let root
  let currentParent
  const stack = []
  
  parseHTML(html, {
    start (tag, attrs, unary) {
      const element = {
        type: 1,
        tag: tag,
        parent: currentParent,
        attrsList: attrs,
        children: []
      }

      if (!root) {
        root = element
      } else if (currentParent) {
        currentParent.children.push(element)
      }
      if (!unary) {
        currentParent = element
        stack.push(currentParent)
      }
    },
    end () {
      stack.pop()
      currentParent = stack[stack.length - 1]
    }
  })
  
  return root
}
```

如上代码所示，首先定义了 `stack` 常量，是一个数组，接着做了一些修改，每次修改遇到非一元开始标签，除了设置 `currentParent` 的值之外，还会将 `currentParent` 添加到 `stack` 数组。接着在 `end` 钩子函数中添加了一行代码，也就是说每当遇到一个非一元标签的结束标签时，都会回退 `currentParent` 变量的值为之前的值，这样就修正了当前正在解析的元素的父级元素。

以上就是根据 `parseHTML` 函数生成 AST 的基本方式，实际上，考虑的还不够周全，比如上面讲解中没有处理一元标签，另外还需要处理文本节点和注释节点等等。不过上面的讲解为后续对源码的解析做了铺垫，更详细的内容在接下来的源码解析阶段进行讲解。

## 解析前的准备工作

前面说过，整个 `src/compiler/parser/index.js` 文件所做的工作都是在创建 AST，所以应该先了解一下这个文件的结构，以方便后续的理解。在该文件的开头定义了一些常量和变量，其中包括一些正则常量，后续会详细讲解。

接着定义了 `createASTElement` 函数，如下：

```javascript
export function createASTElement (
  tag: string,
  attrs: Array<Attr>,
  parent: ASTElement | void
): ASTElement {
  return {
    type: 1,
    tag,
    attrsList: attrs,
    attrsMap: makeAttrsMap(attrs),
    parent,
    children: []
  }
}
```

`createASTElement` 函数用来创建一个元素的描述对象，这样在创建元素描述对象时就不需要手动编写对象字面量了，方便的同时还能提高代码的整洁性。

再往下定义了整个文件最重要的一个函数，即 `parse` 函数，结构如下：

```javascript
export function parse (
  template: string,
  options: CompilerOptions
): ASTElement | void {
  /*
   * 省略...
   * 省略的代码用来初始化一些变量的值，以及创建一些新的变量，其中包括 root 变量，该变量为 parse 函数的返回值，即 AST
   */
  
  function warnOnce (msg) {
    // 省略...
  }

  function closeElement (element) {
    // 省略...
  }

  parseHTML(template, {
    // 其他选项...
    start (tag, attrs, unary, start, end) {
      // 省略...
    },

    end (tag, start, end) {
      // 省略...
    },

    chars (text: string) {
      // 省略...
    },
    comment (text: string) {
      // 省略...
    }
  })
  return root
}
```

通过如上代码的简化，可以清晰看到 `parse` 函数的结构，在 `parse` 函数开头的代码用来初始化一些变量的值，以及创建一些新的变量，其中包括 `root` 变量，该变量为 `parse` 函数的返回值，即最终的 AST。然后定义了两个函数 `warnOnce` 和 `closeElement`。接着调用了 `parseHTML` 函数，通过上一小节的铺垫，应该都知道了 `parse` 函数是如何创建 AST 的了。另外注意在调用 `parseHTML` 函数时传递了很多选项，其中包括了四个重要的钩子函数选项：`start`、`end`、`chars`、`comment`。最后 `parse` 函数将 `root` 变量返回，也就是最终生成的 AST。

在 `parse` 函数的后面，定义了许多的函数，如下：

```javascript
function processPre (el) {/* 省略...*/}
function processRawAttrs (el) {/* 省略...*/}
export function processElement (element: ASTElement, options: CompilerOptions) {/* 省略...*/}
function processKey (el) {/* 省略...*/}
function processRef (el) {/* 省略...*/}
export function processFor (el: ASTElement) {/* 省略...*/}
export function parseFor (exp: string): ?ForParseResult {/* 省略...*/}
function processIf (el) {/* 省略...*/}
function processIfConditions (el, parent) {/* 省略...*/}
function findPrevElement (children: Array<any>): ASTElement | void {/* 省略...*/}
export function addIfCondition (el: ASTElement, condition: ASTIfCondition) {/* 省略...*/}
function processOnce (el) {/* 省略...*/}
function processSlot (el) {/* 省略...*/}
function processComponent (el) {/* 省略...*/}
function processAttrs (el) {/* 省略...*/}
function checkInFor (el: ASTElement): boolean {/* 省略...*/}
function parseModifiers (name: string): Object | void {/* 省略...*/}
function makeAttrsMap (attrs: Array<Object>): Object {/* 省略...*/}
function isTextTag (el): boolean {/* 省略...*/}
function isForbiddenTag (el): boolean {/* 省略...*/}
function guardIESVGBug (attrs) {/* 省略...*/}
function checkForAliasModel (el, value) {/* 省略...*/}
```

可以发现，这些函数的名字大部分都以 `process` 开头，并且接收的参数中基本上都包含了 `el`，实际上 `el` 就是元素的描述对象，如下：

```javascript
el = {
  type: 1,
  tag,
  attrsList: attrs,
  attrsMap: makeAttrsMap(attrs),
  parent,
  children: []
}
```

`process*` 系列的函数的作用就是对元素描述对象做进一步处理，比如其中一个函数叫做 `processPre`，这个函数的作用就是用来检测 `el` 元素是否拥有 `v-pre` 属性，如果有 `v-pre` 属性则会在 `el` 描述对象上添加一个 `pre` 属性，如下：

```javascript
el = {
  type: 1,
  tag,
  attrsList: attrs,
  attrsMap: makeAttrsMap(attrs),
  parent,
  children: [],
  pre: true
}
```

类似的，所有 `process*` 系列函数的作用都是为了让一个元素的描述对象更加充实，使这个对象能更加详细地描述一个元素，并且这些函数都会用在 `parseHTML` 函数的钩子选项函数中。

另外，还有很多非 `process*` 系列的函数，例如：`findPreElement`、`makeAttrsMap` 等等，这些函数实际上就是工具函数。

以上就是 `src/compiler/parser/index.js` 文件的整体结构。紧接着将回到该文件的开头部分，看看定义了哪些常量或变量。

### 正则常量分析

#### 正则常量 `onRE`

首先讲解的是 `onRE` 正则常量，其源码如下：

```javascript
export const onRE = /^@|^v-on:/
```

这个常量用来匹配以字符 `@` 或 `v-on:` 开头的字符串，主要用来检测标签属性名是否是监听事件的指令。

#### 正则常量 `dirRE`

正则常量 `dirRE` 源码如下：

```javascript
export const dirRE = /^v-|^@|^:/
```

它用来匹配以字符 `v-` 或 `@` 或 `:` 开头的字符串，主要作用是检测标签属性名是否是指令。通过这个正则可以知道，在 `vue` 中所有以 `v-` 开头的属性都被认为是指令，另外 `@` 字符是 `v-on` 的缩写，`:` 字符是 `v-bind` 的缩写。

#### 正则常量 `forAliasRE`

正则 `forAliasRE` 源码如下：

```javascript
export const forAliasRE = /([^]*?)\s+(?:in|of)\s+([^]*)/
```

该正则包含三个分组，第一个分组为 `([^]*?)`，该分组是一个惰性匹配的分组，它匹配的内容为任何字符，包括换行符等等。第二个分组为 `(?:in|of)`，该分组用来匹配字符串 `in` 或者 `of`，并且该分组是非捕获的分组。第三个分组为 `([^]*)`，与第一个分组类似，不同的是第三个分组是非惰性匹配。同时每个分组之间都会匹配至少一个空白符 `\s+`。通过以上说明可知，正则 `forAliasRE` 用来匹配 `v-for` 属性的值，并捕获 `in` 或 `of` 前后的字符串。假设像如下这样使用 `v-for`：

```html
<div v-for="obj of list"></div>
```

那么正则 `forAliasRE` 用来匹配字符串 `'obj of list'`，并捕获到两个字符串 `'obj'` 和 `'list'`。

#### 正则常量 `forIteratorRE`

正则 `forIteratorRE` 如下：

```javascript
export const forIteratorRE = /,([^,\}\]]*)(?:,([^,\}\]]*))?$/
```

该正则用来匹配 `forAliasRE` 第一个捕获组所捕获到的字符串，可以看到如上正则中拥有三个分组，有两个捕获组，第一个捕获组用来捕获一个不包含 `,`、`}`、`]` 的字符串，且该字符串前面有一个字符 `,`，如：`', index'`。第二个分组为非捕获组的分组，第三个分组为捕获的分组，其捕获的内容和第一个捕获组相同。

举几个例子，可以知道 `v-for` 有几种不同的写法，其中一种使用 `v-for` 的方式是：

```html
<div v-for="obj of list"></div>
```

如果像如上这样使用 `v-for`，那么 `forAliasRE` 正则的第一个捕获组的内容为字符串 `'obj'`，此时使用 `forIteratorRE` 正则去匹配字符串 `'obj'` 将得不到任何内容。

第二种使用 `v-for` 的方式为：

```html
<div v-for="(obj, index) of list"></div>
```

此时 `forAliasRE` 正则的第一个捕获组的内容为字符串 `'(obj, index)'`，如果去掉左右括号则该字符串为 `'obj, index'`，如果使用 `forIteratorRE` 正则去匹配字符串 `'obj, index'` 则会匹配成功，并且 `forIteratorRE` 正则的第一个捕获组将捕获到字符串 `'index'`，但第二个捕获组捕获不到任何内容。

第三种使用 `v-for` 的方式为：

```html
<div v-for="(value, key, index) in object"></div>
```

以上方式主要用于遍历对象而非数组，此时 `forAliasRE` 正则的第一个捕获组的内容为字符串 `'(value, key, index)'`，如果去掉左右括号则该字符串为 `'value, key, index'`，如果使用 `forIteratorRE` 正则去匹配字符串 `'value, key, index'` 则会匹配成功，并且 `forIteratorRE` 正则的第一个捕获组将捕获到字符串 `'key'`，但第二个捕获组将捕获到字符串 `'index'`。

#### 正则常量 `stripParensRE`

源码如下：

```javascript
const stripParensRE = /^\(|\)$/g
```

这个捕获组用来捕获要么以 `(` 开头，要么以 `)` 结尾的字符串，或者两者都满足。在讲解正则 `forIteratorRE` 时有个细节，就是 `forIteratorRE` 正则所匹配的字符串是 `'obj, index'`，而不是 `'(obj, index)'`，这两个字符串的区别就在于第二个字符串拥有左右括号，所以在使用 `forIteratorRE` 正则之前，需要使用 `stripParensRE` 正则去掉字符串 `'(obj, index)'` 中的左右括号，实现方式很简单：

```javascript
'(obj, index)'.replace(stripParensRE, '')
```

#### 正则常量 `argRE`

源码如下：

```javascript
const argRE = /:(.*)$/
```

正则 `argRE` 用来匹配指令中的参数，如下：

```html
<div v-on:click.stop="handleClick"></div>
```

其中 `v-on` 为指令，`click` 为传递给 `v-on` 指令参数，`stop` 为修饰符，所以 `argRE` 正则用来匹配指令编写中的参数，并且拥有一个捕获组，用来捕获参数的名字。

#### 正则常量 `bindRE`

源码如下：

```javascript
export const bindRE = /^:|^v-bind:/
```

该正则用来匹配以 `:` 或字符串 `v-bind:` 开头的字符串，主要用来检测一个标签的属性是否绑定 (`v-bind`)。

#### 正则常量 `modifierRE`

源码如下：

```javascript
const modifierRE = /\.[^.]+/g
```

该正则用来匹配修饰符，但是没有捕获任何东西，举例来说：

```javascript
const matchs = 'v-on:click.stop'.match(modifierRE)
```

那么 `matchs` 数组第一个元素为字符串 `'.stop'`，所以指令修饰符应该是：

```javascript
matchs[0].slice(1)  // 'stop'
```

### HTML 实体解码函数 `decodeHTMLCached`

源码如下：

```javascript
const decodeHTMLCached = cached(he.decode)
```

`cached` 函数前边已经遇到过，它的作用是接收一个函数作为参数并返回一个新的函数，新函数的功能和参数传递的函数功能相同，唯一不同的是新函数具有缓存值的功能，如果一个函数在接收相同参数的情况下总是相同的，那么 `cached` 函数将会为该函数提供性能提升的优势。

可以看到传递给 `cached` 函数的参数是 `he.decode` 函数，其中 `he` 为第三方的库，`he.decode` 函数用于 `HTML` 字符实体的解码工作，如：

```javascript
console.log(he.decode('&#x26;'))  // &#x26; -> '&'
```

由于字符实体 `&#x26;` 代表的字符为 `&`。所以字符串 `&#x26` 经过解码后将变成字符 `&`。`decodeHTMLCached` 函数在后面将被用于对纯文本的解码，如果不解码，那么用户将无法使用字符实体编写字符。

### 定义平台化选项变量

再往下，定义了一些平台化的选项变量，如下：

```javascript
export let warn: any
let delimiters
let transforms
let preTransforms
let postTransforms
let platformIsPreTag
let platformMustUseProp
let platformGetTagNamespace
```

上面的代码中定义了 `8` 个平台化的变量，后面讲解 `parse` 函数时，能够看到这些变量将被初始化一个值，这些值都是平台化的编译器选项参数，不同平台这些变量将被初始化的值是不同的，可以找到 `parse` 函数看一下：

```javascript
export function parse (
  template: string,
  options: CompilerOptions
): ASTElement | void {
  warn = options.warn || baseWarn

  platformIsPreTag = options.isPreTag || no
  platformMustUseProp = options.mustUseProp || no
  platformGetTagNamespace = options.getTagNamespace || no

  transforms = pluckModuleFunction(options.modules, 'transformNode')
  preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
  postTransforms = pluckModuleFunction(options.modules, 'postTransformNode')

  delimiters = options.delimiters

  // 省略...
}
```

如上代码所示，可以清晰看到在 `parse` 函数的一开始为这 `8` 个平台化的变量进行了初始化，初始化的值都是曾经讲过的编译器的选项参数，由于前面讲解的都是 `web` 平台下的编译器选项，所以这里初始化的值只用于 `web` 平台。

### `createASTElement` 函数

在平台化变量的后面，定义了 `createASTElement` 函数，这个函数的作用就是方便创建一个节点，或者方便创建一个元素的描述对象，如下：

```javascript
export function createASTElement (
  tag: string,
  attrs: Array<Attr>,
  parent: ASTElement | void
): ASTElement {
  return {
    type: 1,
    tag,
    attrsList: attrs,
    attrsMap: makeAttrsMap(attrs),
    parent,
    children: []
  }
}
```

它接收三个参数，分别是标签名字 `tag`，该标签拥有的属性数组 `attrs` 以及该标签的父标签描述对象的引用，比如使用 `parseHTML` 解析如下标签时：

```html
<div v-for="obj of list" class="box"></div>
```

当遇到 `div` 的开始标签时 `parseHTML` 函数的 `start` 钩子函数的前两个参数分别是：

```javascript
const html = '<div v-for="obj of list" class="box"></div>'
parseHTML(html, {
  start (tag, attrs) {
    console.log(tag)    // 'div'
    console.log(attrs)  // [ { name: 'v-for', value: 'obj of list' }, { name: 'class', value: 'box' } ]
  }
})
```

此时只需要调用 `createASTElement` 函数并将这两个参数传递过去，即可创建该 `div` 标签的描述对象：

```javascript
const html = '<div v-for="obj of list" class="box"></div>'
parseHTML(html, {
  start (tag, attrs) {
    console.log(tag)    // 'div'
    console.log(attrs)  // [ { name: 'v-for', value: 'obj of list' }, { name: 'class', value: 'box' } ]
    const element = createASTElement(tag, attrs)
  }
})
```

最终创建出来的元素描述对象如下：

```javascript
element = {
  type: 1,
  tag: 'div',
  attrsList: [
    {
      name: 'v-for',
      value: 'obj of list'
    },
    {
      name: 'class',
      value: 'box'
    }
  ],
  attrsMap: makeAttrsMap(attrs),
  parent,
  children: []
}
```

上面的描述对象中的 `parent` 属性没有细说，在上一小节讲解思路的时候已经结束过 `currentParent` 变量的作用，实际上元素描述对象间的引用关系就是通过 `currentParent` 完成的，后面会仔细讲解，另外注意到描述对象中除了 `attrsList` 属性是原始的标签属性数组之外，还有一个叫做 `attrsMap` 的属性：

```javascript
{
  // 省略...
  attrsMap: makeAttrsMap(attrs),
  // 省略...
}
```

可以看到它的值是 `makeAttrsMap` 函数的返回值，并且 `makeAttrsMap` 函数接收一个参数，该参数恰好是标签的属性数组 `attrs`，此时需要查看 `makeAttrsMap` 的代码，如下：

```javascript
function makeAttrsMap (attrs: Array<Object>): Object {
  const map = {}
  for (let i = 0, l = attrs.length; i < l; i++) {
    if (
      process.env.NODE_ENV !== 'production' &&
      map[attrs[i].name] && !isIE && !isEdge
    ) {
      warn('duplicate attribute: ' + attrs[i].name)
    }
    map[attrs[i].name] = attrs[i].value
  }
  return map
}
```

首先注意 `makeAttrsMap` 函数的第一行代码和最后一行代码，第一行定义了 `map` 常量并在最后一行代码中将其返回，这两行代码中间是一个 `for` 循环，用于遍历 `attrs` 数组，注意 `for` 循环中有这样一行代码：

```javascript
map[attrs[i].name] = attrs[i].value
```

也就是说，如果标签的属性数组 `attrs` 为：

```javascript
attrs = [
  {
    name: 'v-for',
    value: 'obj of list'
  },
  {
    name: 'class',
    value: 'box'
  }
]
```

那么最终生成的 `map` 对象则是：

```javascript
map = {
  'v-for': 'obj of list',
  'class': 'box'
}
```

所以 `makeAttrsMap` 函数的作用就是将标签的属性数组转换成名值对一一对应的对象。这么做的目的之后遇到会讲解，最终生成的对象描述对象如下：

```javascript
element = {
  type: 1,
  tag: 'div',
  attrsList: [
    {
      name: 'v-for',
      value: 'obj of list'
    },
    {
      name: 'class',
      value: 'box'
    }
  ],
  attrsMap: {
    'v-for': 'obj of list',
    'class': 'box'
  },
  parent,
  children: []
}
```

以上就是 `parse` 函数之前定义的所有常量、变量以及函数的讲解，接下来正是进入到 `parse` 函数的实现讲解。

## `parse` 函数创建 AST 前的准备工作

本节主要讲解 `parse` 函数的结构以及真正开始解析之前的准备工作，可以知道 `parse` 函数中主要是通过调用 `parseHTML` 函数来辅助完成 AST 构建的，但是在调用 `parseHTML` 函数之前还需要做一些准备工作，比如前边提到的在 `parse` 函数的开头为平台化变量赋了值，如下是 `parse` 函数的整体结构：

```javascript
export function parse (
  template: string,
  options: CompilerOptions
): ASTElement | void {
  warn = options.warn || baseWarn

  platformIsPreTag = options.isPreTag || no
  platformMustUseProp = options.mustUseProp || no
  platformGetTagNamespace = options.getTagNamespace || no

  transforms = pluckModuleFunction(options.modules, 'transformNode')
  preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
  postTransforms = pluckModuleFunction(options.modules, 'postTransformNode')

  delimiters = options.delimiters

  const stack = []
  const preserveWhitespace = options.preserveWhitespace !== false
  let root
  let currentParent
  let inVPre = false
  let inPre = false
  let warned = false

  function warnOnce (msg) {
    // 省略...
  }

  function closeElement (element) {
    // 省略...
  }

  parseHTML(template, {
    // 省略...
  })
  return root
}
```

从上至下一点点来看，首先是如下这段代码：

```javascript
warn = options.warn || baseWarn

platformIsPreTag = options.isPreTag || no
platformMustUseProp = options.mustUseProp || no
platformGetTagNamespace = options.getTagNamespace || no

transforms = pluckModuleFunction(options.modules, 'transformNode')
preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
postTransforms = pluckModuleFunction(options.modules, 'postTransformNode')
```

前边讲过，这段代码为 `8` 个平台化的变量初始化了值，并且这些变量的值基本上都来自于编译器选项参数。在编译器初探一节中对一部分参数进行过讲解。

回过头来继续分析这些平台化的变量，首先是 `warn` 变量的值为 `options.warn` 函数，如果 `options.warn` 选项参数不存在，则会降级使用 `baseWarn` 函数，所以 `warn` 函数作用是用来打印警告信息的，另外 `baseWarn` 函数来自于 `src/compiler/helpers.js` 文件，该文件用来存放一些助手工具函数，后面分析 `parse` 函数源码时将会经常看到调用来自该文件的函数。其中 `baseWarn` 函数源码如下：

```javascript
export function baseWarn (msg: string) {
  console.error(`[Vue compiler]: ${msg}`)
}
```

可以看到 `baseWarn` 函数的作用无非就是通过 `console.error` 函数打印错误信息。

第二个赋值的是 `platformPreTag` 变量，如下时它的赋值语句：

```javascript
platformIsPreTag = options.isPreTag || no
```

可知 `platformIsPreTag` 变量的值为 `options.isPreTag` 函数，该函数是一个编译器选项，其作用是通过给定的标签名字判断该标签是否是 `pre` 标签。另外如上代码所示如果编译器选项中不包含 `options.isPreTag` 函数则会降级使用 `no` 函数，该函数是一个空函数，即什么都不做。

第三个赋值的是 `platformMustUseProp` 变量，如下是它的赋值语句：

```javascript
platformMustUseProp = options.mustUseProp || no
```

可知 `platformMustUseProp` 变量的值为 `options.mustUseProp` 函数，该函数也是一个编译器选项，其作用是用来检测一个属性在标签中是否要使用元素对象原生的 `prop` 进行绑定，注意：**这里的 `prop` 指的是元素对象的属性，而非 `Vue` 中的 `props` 概念**。同样的如果选项参数中不包含 `options.mustUseProp` 函数则会降级为 `no` 函数。

第四个赋值的是 `platformGetTagNamespace` 变量，如下时它的赋值语句：

```javascript
platformGetTagNamespace = options.getTagNamespace || no
```

可知 `platformGetTagNamespace` 变量的值为 `options.getTagNamespace` 函数，该函数是一个编译器选项，其作用是用来获取元素 (标签) 的命名空间。如果选项参数中不包含 `options.getTagNamespace` 函数则会降级为 `no` 函数。

第五个赋值的变量是 `transforms`，如下是它的赋值语句：

```javascript
transforms = pluckModuleFunction(options.modules, 'transformNode')
```

可以看到 `transforms` 变量的值与前面讲解的变量不同，它的值是 `pluckModuleFunction` 函数的返回值，并以 `options.modules` 选项以及一个字符串 `'transformNode'` 作为参数。

通过前面的分析可以知道 `options.modules` 的值，它是一个数组，如下：

```javascript
options.modules = [
  {
    staticKeys: ['staticClass'],
    transformNode,
    genData
  },
  {
    staticKeys: ['staticStyle'],
    transformNode,
    genData
  },
  {
    preTransformNode
  }
]
```

`options.modules` 既然是 `web` 平台下的编译器选项参数，它们必然来自 `src/platforms/web/compier/options.js` 文件，其中 `options.modules` 选项参数的值为来自 `src/platforms/web/compiler/modules/` 目录下几个文件组合而成的。

知道了这些，可以具体查看 `pluckModuleFunction` 函数的代码，`pluckModuleFunction` 函数来自 `src/compiler/helper.js` 文件，如下是源码：

```javascript
export function pluckModuleFunction<F: Function> (
  modules: ?Array<Object>,
  key: string
): Array<F> {
  return modules
    ? modules.map(m => m[key]).filter(_ => _)
    : []
}
```

`pluckModuleFunction` 函数的作用是从第一个参数“采摘”出函数名字与第二个参数所指定字符串相同的函数，并将它们组成一个数组。拿如下这段代码说明：

```javascript
transforms = pluckModuleFunction(options.modules, 'transformNode')
```

可知传递给 `pluckModuleFunction` 函数的第二个参数的字符串为 `'transformNode'`，同时观察 `options.modules` 数组：

```javascript
options.modules = [
  {
    staticKeys: ['staticClass'],
    transformNode,
    genData
  },
  {
    staticKeys: ['staticStyle'],
    transformNode,
    genData
  },
  {
    preTransformNode
  }
]
```

如上代码所示 `options.modules` 数组的第一个元素和第二个元素都是一个对象，且这两个对象中都包含了 `transformNode` 函数，但是第三个元素对象中没有 `transformNode` 函数。此时按照 `pluckModuleFunction` 函数的逻辑：

```javascript
return modules
  ? modules.map(m => m[key]).filter(_ => _)
  : []
```

如上代码等价于：

```javascript
return options.modules
  ? options.modules.map(m => m['transformNode']).filter(_ => _)
  : []
```

先看这行代码：

```javascript
options.modules.map(m => m['transformNode'])
```

这行代码会创建一个新的数组：

```javascript
[
  transformNode,
  transformNode,
  undefined
]
```

由于 `options.modules` 数组第三个元素对象不包含 `transformNode` 函数，所以生成的数组中第三个元素的值为 `undefined`。这还没完，可以看到紧接着又在新生成的数组之上调用了 `filter` 函数，即：

```javascript
[
  transformNode,
  transformNode,
  undefined
].filter(_ => _)
```

这么做的结果是把值为 `undefined` 的元素过滤掉，所以最终生成的数组如下：

```javascript
[
  transformNode,
  transformNode
]
```

而这个数组就是变量 `transforms` 的值。

第六个赋值的变量是 `preTransforms`，如下是它的赋值语句：

```javascript
preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
```

和 `transforms` 变量的赋值语句如出一辙，同样是使用 `pluckModuleFunction` 函数，第一个参数同样是 `options.modules`，不同的是第二个参数为字符串 `'preTransformNode'`。也就是此时“采摘”的应该是名字为 `preTransformNode` 的函数，并将它们组装一个数组。由于 `options.modules` 数组中只有第三个元素对象包含 `preTransformNode` 函数，所以最终 `preTransforms` 变量的值为：

```javascript
preTransforms = [preTransformNode]
```

第七个赋值的变量是 `postTransforms`，如下是它的赋值语句：

```javascript
postTransforms = pluckModuleFunction(options.modules, 'postTransformNode')
```

可以知道此时“采摘”的应该是名字为 `postTransformNode` 的函数，并将它们组装成一个数组。由于 `options.modules` 数组中的三个元素对象都不包含 `postTransformNode` 函数，所以最终 `postTransforms` 变量的值将是一个空数组：

```javascript
postTransforms = []
```

最后一个赋值的变量为 `delimiters`，如下：

```javascript
delimiters = options.delimiters
```

它的值为 `options.delimiters` 属性，它的值就是在创建 `Vue` 实例对象时传递的 `delimiters` 选项，它是一个数组。

在如上讲解的八个平台化变量的下面，又定义了一些常量和变量，如下：

```javascript
const stack = []
const preserveWhitespace = options.preserveWhitespace !== false
let root
let currentParent
let inVPre = false
let inPre = false
let warned = false
```

首先定义的是 `stack` 常量，它的初始值是一个空数组。在讲解创建 AST 思路的时候也使用了 `stack` 数组，当时讲到了它的作用是用来修正当前正在解析元素的父级。在 `stack` 常量之后定义了 `preserveWhitespace` 常量，它是一个布尔值并且它的值和编译器选项中的 `options.preserveWhitespace` 常量，它是一个布尔值并且它的值和编译器选项中的 `options.preserveWhitespace` 选项有关，只要 `options.preserveWhitespace` 的值不为 `false`，那么 `preserveWhitespace` 的值就为 `true`。其中 `options.preserveWhitespace` 选项用来告诉编译器在编译 `html` 字符串时是否放弃标签之间的空格，如果为 `true` 则代表放弃。

接着定义了 `root` 变量，可以知道 `parse` 函数的返回值就是 `root` 变量，所以 `root` 变量就是最终的 AST。在 `root` 变量之后定义了 `currentParent` 变量，在讲解创建 AST 思路也定义了一个 `currentParent`，可以知道元素描述对象之间的父子关系就是靠着该变量进行联系的。

接着又定义了三个变量，分别是 `inVPre`、`inPre`、`warned`，并且它们的初始值都为 `false`。其中 `inVPre` 变量用来标识当前解析的标签是否在拥有 `v-pre` 的标签之内，`inPre` 变量用来标识当前正在解析的标签是否在 `<pre></pre>` 标签之内。而 `warned` 变量则用于接下来定义的 `warnOnce` 函数：

```javascript
function warnOnce (msg) {
  if (!warned) {
    warned = true
    warn(msg)
  }
}
```

`warned` 变量的初始值为 `false`，通过如上代码可以看到 `warnOnce` 函数同样是用来打印警告信息的函数，只不过 `warnOnce` 函数就如其名字一般，只会打印一次警告信息，并且 `warnOnce` 函数也是通过调用 `warn` 函数来实现的。

在 `warnOnce` 函数的下面定义了 `closeElement` 函数，如下：

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

每当遇到一个标签的结束标签时，或遇到一元标签时都会调用该方法“闭合”标签，具体功能在后面的内容中详细讲解。

经过了一系列的准备，到了关键的一步，即调用 `parseHTML` 函数解析模板字符串并借助它来构建一棵 AST，如下时调用 `parseHTML` 函数时所传递的选项参数：

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

在 词法分析—为生成 AST 做准备 中讲解 `parseHTML` 函数时已经顺带分析了所有选项的作用。其中对于构建 AST 来说最关键的选项就是四个钩子函数选项：

- `start` 钩子函数，在解析 `html` 字符串时每次遇到**开始标签**时就会调用该函数。
- `end` 钩子函数，在解析 `html` 字符串时每次遇到**结束标签**时就会调用该函数。
- `char` 钩子函数，在解析 `html` 字符串时每次遇到**纯文本**时就会调用该函数。
- `comment` 钩子函数，在解析 `html` 字符串每次遇到**注释节点**时就会调用该函数。

下面就从 `start` 钩子函数开始讲起，因为正常情况下，解析一段 `html` 字符串时必然最先遇到的就是开始标签。所以从 `start` 钩子函数开始讲起，在讲解的过程中为了说明某些问题会逐个举例。

## 解析一个开始标签需要做的事情

接下来从 `start` 钩子函数开始，研究一下解析一个开始标签需要做的事情,如下是在 `parse` 函数中，调用 `parseHTML` 函数时传递的 `start` 钩子函数：

```javascript
start (tag, attrs, unary) {
  // 省略...
}
```

可以知道 `start` 钩子函数是接收五个参数的，但是如上代码中只使用了 `start` 钩子函数的前三个参数，也就是说只需要这三个参数就足够完成任务了，这三个参数分别是标签名字 `tag`，该标签的属性数组 `attrs`，以及代表着该标签是否是一元标签的标识 `unary`。

在 `start` 钩子函数的内部首先执行的是如下这行代码：

```javascript
const ns = (currentParent && currentParent.ns) || platformGetTagNamespace(tag)
```

为了让大家更好理解，这里规定一些事情，比如既然讲解的是 `start` 钩子函数，那么当前的解析必然处于遇到一个开始标签的阶段，把当前解析所遇到的开始标签称为：**当前元素**，另外把当前 **当前元素** 的父标签称为：**父级元素**。

如上这行代码定义了 `ns` 常量，它的值为标签的命名空间，获取当前元素的命名空间，首先是通过检测 `currentParent` 变量是否存在，`currentParent` 变量为当前元素的父级描述对象，如果当前元素存在父级并且父级元素存在命名空间，则使用父级的命名空间作为当前元素的命名空间。如果父级元素不存在或父级元素没有命名空间，那么会调用 `platformGetTagNamespace(tag)` 函数获取当前元素的命名空间。举例说明，假设解析的模板字符串为：

```html
<svg width="100%" height="100%" version="1.1" xmlns="http://www.w3.org/2000/svg">
  <rect x="20" y="20" width="250" height="250" style="fill:blue;"/>
</svg>
```

如上是用来画一个蓝色矩形的 `svg` 代码，其中使用了两个标签：`svg` 标签和 `rect` 标签，当解析如上代码时首先会遇到 `svg` 标签的开始标签，由于 `svg` 标签没有父级元素，所以会通过 `platformGetTagNamespace(tag)` 获取 `svg` 标签的命名空间，最终得到 `svg` 字符串：

```javascript
platformGetTagNamespace('svg')  // 'svg'
```

下一个遇到的开始标签则是 `rect` 标签的开始标签，由于 `rect` 标签存在父级元素 (`svg` 标签)，所以此时 `rect` 标签会使用它父级元素的命名空间作为自己的命名空间。

`platformGetTagNamespace` 函数只会获取 `svg` 和 `match` 这两个标签的命名空间，但这两个标签的所有子标签都会继承它们两个的命名空间。对于其他标签则不存在命名空间。

总之在 `start` 钩子函数内部首先会尝试获取一个元素的命名空间，并将获取到的命名空间的名字赋给 `ns` 常量，这个常量在后面会用到。

在获取命名空间之后，执行的是如下这段 `if` 条件语句块：

```javascript
if (isIE && ns === 'svg') {
  attrs = guardIESVGBug(attrs)
}
```

`isIE` 函数用来判断当前宿主环境是否是 `IE` 浏览器，如果是 `IE` 浏览器并且当前元素的命名空间为 `svg`，则会调用 `guardIESVGBug` 函数处理当前元素的属性数组 `attrs`，并使用处理后的结果重新赋值给 `attrs` 变量。这看上去在处理 `IE` 浏览器中关于 `svg` 标签的 `bug`，实际上确实是这样，`IE` 的 Bug 是：`svg` 标签中渲染多余的属性，如下时 `svg` 标签：

```html
<svg xmlns:feature="http://www.openplans.org/topp"></svg>
```

被渲染成了：

```html
<svg xmlns:NS1="" NS1:xmlns:feature="http://www.openplans.org/topp"></svg>
```

标签中多了 `'xmlns:NS1="" NS1:'` 这段字符串，解决的方法很简单，将整个多余的字符串去掉即可。而 `guardIESVGBug` 函数就是用来修改 `NS1:xmlns:feature` 属性并移除 `xmlns:NS1=""` 属性的，如下是 `guardIESVGBug` 函数的源码以及它需要的两个正则：

```javascript
const ieNSBug = /^xmlns:NS\d+/
const ieNSPrefix = /^NS\d+:/

/* istanbul ignore next */
function guardIESVGBug (attrs) {
  const res = []
  for (let i = 0; i < attrs.length; i++) {
    const attr = attrs[i]
    if (!ieNSBug.test(attr.name)) {
      attr.name = attr.name.replace(ieNSPrefix, '')
      res.push(attr)
    }
  }
  return res
}
```

在 `guardIESVGBug` 函数之前定义了两个正则常量，其中 `ieNSBug` 正则用来匹配那些以字符串 `xmlns:NS` 再加一个或多个数字组成的字符串开头的属性名，如：

```html
<svg xmlns:NS1=""></svg>
```

如上标签的 `xmlns:NS1` 属性将会被 `ieNSBug` 正则匹配成功。另外一个正则常量是 `ieNSPrefix`，它用来匹配那些以字符串 `NS` 再加一个或多个数字以及字符 `:` 所组成的字符串开头的属性名，如：

```html
<svg NS1:xmlns:feature="http://www.openplans.org/topp"></svg>
```

如上标签的 `NS1:xmlns:feature` 属性将被 `isNSPrefix` 正则匹配成功。

`guardIESVGBug` 函数接收元素的属性数组作为参数，并返回一个新的数组，新数组和原数组结构相同。可以看到 `guardIESVGBug` 函数内部通过 `for` 循环遍历了元素的属性数组，接着使用正则 `ieNSBug` 去匹配属性名字，可以发现只要不满足 `ieNSBug` 正则的属性名，都会尝试使用 `ieNSPrefix` 正则去匹配该属性名并将匹配到的字符替换为空字符串。如下是渲染产生 `bug` 后的代码：

```html
<svg xmlns:NS1="" NS1:xmlns:feature="http://www.openplans.org/topp"></svg>
```

在解析如上标签时，传递给 `start` 钩子函数的标签属性数组 `attrs` 为：

```javascript
attrs = [
  {
    name: 'xmlns:NS1',
    value: ''
  },
  {
    name: 'NS1:xmlns:feature',
    value: 'http://www.openplans.org/topp'
  }
]
```

在经过 `guardIESVGBug` 函数处理之后，属性数组中的第一项因为属性名满足 `ieNSBug` 正则被剔除，第二项属性名字 `NS1:xmlns:feature` 将变成 `xmlns:feature`，所以 `guardIESVGBug` 返回属性数组为：

```javascript
attrs = [
  {
    name: 'xmlns:feature',
    value: 'http://www.openplans.org/topp'
  }
]
```

以上就是 `guardIESVGBug` 函数的作用。

处理完 `IE` 的 `SVG` 问题之后，执行的是如下代码：

```javascript
let element: ASTElement = createASTElement(tag, attrs, currentParent)
if (ns) {
  element.ns = ns
}
```

这段代码是很关键的一段代码，如上代码所示，这行代码的执行为当前元素创建了描述对象，并且元素描述对象的创建是通过前面所讲的 `createASTElement` 完成的，并将当前标签的元素描述对象赋值给 `element` 变量。紧接着检查当前元素是否存在命名空间 `ns`，如果存在则在元素对象上添加 `ns` 属性，其值为命名空间的值。

通过如上代码可知，如果当前解析的开始标签为 `svg` 标签或者 `math` 标签或者它们两个的子节点标签都将会比其他 `html` 标签描述对象多出一个 `ns` 属性，且该属性标识了该标签的命名空间。

再往下是这样一段代码：

```javascript
if (isForbiddenTag(element) && !isServerRendering()) {
  element.forbidden = true
  process.env.NODE_ENV !== 'production' && warn(
    'Templates should only be responsible for mapping the state to the ' +
    'UI. Avoid placing tags with side-effects in your templates, such as ' +
    `<${tag}>` + ', as they will not be parsed.'
  )
}
```

这段代码是一段 `if` 条件语句块，根据判断条件可知，该 `if` 语句用来判断非服务端渲染情况下，当前元素是否是禁止在模板中使用的标签。其中使用 `isForbiddenTag(element)` 函数检查当前元素是否为被禁止的标签，`<style>` 和 `<script>` 都被认为是被禁止的标签，因为 Vue 认为模板应该只负责做数据状态到 UI 的映射，而不应该存在引起副作用的代码，如果模板中存在 `<script>` 标签，那么该标签内的代码很容易引起副作用。但是有一种情况例外，比如其中一种定义模板的方式为：

```html
<script type="text/x-template" id="hello-world-template">
  <p>Hello hello hello</p>
</script>
```

把模板放到 `<script>` 元素中，并在 `<script>` 元素中添加 `type="text/x-template"` 属性。可以看到 Vue 并非禁止了所有的 `<script>` 元素，这在 `isForbiddenTag` 函数中是有体现的，如下是 `isForbiddenTag` 函数的代码：

```javascript
function isForbiddenTag (el): boolean {
  return (
    el.tag === 'style' ||
    (el.tag === 'script' && (
      !el.attrsMap.type ||
      el.attrsMap.type === 'text/javascript'
    ))
  )
}
```

`isForbiddenTag` 函数接收一个元素描述对象作为参数，并返回布尔值，如果返回值为 `true` 则代表该标签是被禁止的，否则为非禁止。根据源码可知如下标签时为被禁止的标签：

- `<style>` 标签为被禁止标签。
- 没有指定 `type` 属性或虽然指定了 `type` 属性但其值为 `text/javascript` 的 `<script>` 标签被认为是被禁止的。

如果当前标签是被禁止的，并且在非服务端渲染的情况下，会打印警告信息，同时还会在当前元素的描述对象上添加 `el.forbidden` 属性，并将其值设置为 `true`。

接着看下面的代码，接下来要执行的是如下这段代码：

```javascript
for (let i = 0; i < preTransforms.length; i++) {
  element = preTransforms[i](element, options) || element
}
```

如上代码中使用了 `for` 循环遍历了 `preTransforms` 数组，可以知道 `preTransforms` 是通过 `pluckModuleFunction` 函数从 `options.module` 选项中筛选出名字为 `preTransformNode` 函数所组成的数组。该数组中每个元素都是一个函数，所以以上代码的 `for` 循环内部直接调用了 `preTransforms` 数组中的每一个函数并为这些函数传递了两个参数，分别是当前元素描述对象 `element` 以及编译器选项 `options`。

这里简单说一下 `preTransforms` 数组中的函数的作用，本质上这些函数的作用和之前见到的 `process*` 系列的函数没什么区别，都是用来对当前描述对象做进一步处理。不仅是 `preTransforms` 数组，对于 `transforms` 数组和 `postTransforms` 数组也是一样的，它们之间的区别就像它们的名字一样，根据不同的调用时机为它们定义了相应的名字。这三个数组中的处理函数与当前文件中 `process*` 系列函数区分开来是出于平台化的考虑，通过前面的分析可以知道 `preTransforms` 数组中的那些 `preTransformNode` 函数是 `src/platforms/web/compiler/modules` 目录下定义的一些文件定义的，根据目录路径可知这些代码应该是用来处理 `web` 平台相关逻辑的，除了 `web` 平台之外也可以看到 `weex` 平台下相应的代码，可以在源码中找到这个目录：`src/platforms/weex/compiler/modules`。

总之需要知道 `preTransforms` 数组中的那些函数和 `process*` 系列函数唯一的区别就是平台化的区别即可。

根据前边的分析，实际上 `preTransforms` 数组中只有一个函数，这个函数就是由 `src/platforms/web/compiler/modules/model.js` 文件导出的 `preTransformNode` 函数。打开文件查看 `preTransformNode` 函数，可以发现该函数内部大量使用了 `process*` 系列的函数，并且该函数只用来处理 `input` 标签，正是由于这一点，所以暂时不对其进行讲解，因为这会让我们脱离主线，在接下来的讲解中会逐个击破 `process*` 系列函数的作用，等大家了解了 `process*` 系列函数所做的事情之后再回头来看 `preTransformNode` 函数的代码会更加容易理解。

回过头来看后面的代码，接下来执行的将是如下代码：

```javascript
if (!inVPre) {
  processPre(element)
  if (element.pre) {
    inVPre = true
  }
}
if (platformIsPreTag(element.tag)) {
  inPre = true
}
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

可以看到这段代码中开始大量调用 `process*` 系列的函数，前面说过了，这其实就是在对当前元素描述对象做额外的处理，使得该元素描述对象能更好的描述一个标签。简单点说就是在元素描述对象上添加各种各样的具有标识作用的属性，就比如之前遇到的 `ns` 属性和 `forbidden` 属性，它们都能够对标签起到描述作用。

不过本节主要总结 **解析一个开始标签需要做的事情**，所以暂时不具体看上面这些代码的实现，继续往下走，接下来定义了一个叫做 `checkRootConstraints` 的函数：

```javascript
function checkRootConstraints (el) {
  if (process.env.NODE_ENV !== 'production') {
    if (el.tag === 'slot' || el.tag === 'template') {
      warnOnce(
        `Cannot use <${el.tag}> as component root element because it may ` +
        'contain multiple nodes.'
      )
    }
    if (el.attrsMap.hasOwnProperty('v-for')) {
      warnOnce(
        'Cannot use v-for on stateful component root element because ' +
        'it renders multiple elements.'
      )
    }
  }
}
```

该函数的作用是用来检测模板根元素是否符合要求，在编写 Vue 模板的时候会受到两种约束，首先模板必须有且仅有一个被渲染的根元素，第二不能使用 `slot` 标签和 `template` 标签作为模板的根元素，对于第一点为什么模板必须有且仅有一个被渲染的根元素，在代码生成的部分会讲解，对于第二点为什么不能使用 `slot` 和 `template` 标签作为模板根元素，这是因为 `slot` 作为插槽，它的内容是由外界决定的，而插槽的内容很有可能渲染多个节点，`template` 元素的内容虽然不是由外界决定的，但它本身作为抽象组件是不会渲染任何内容到页面的，而其又可能包含多个子节点，所以也不允许使用 `template` 标签作为根节点。总之这些限制都是出于 **必须有且仅有一个根元素** 考虑的。

在 `checkRootConstraints` 函数的下面是一段 `if...else if` 语句块，首先从查看 `if` 语句块：

```javascript
if (!root) {
  root = element
  checkRootConstraints(root)
} else if (!stack.length) {
  // 省略...
}
```

`if` 语句块的判断条件是如果 `root` 不存在则执行语句块内的代码，可以知道 `root` 变量在一开始是不存在的，如果 `root` 不存在那说明当前元素应该就是根元素了，所以在 `if` 语句块内直接将当前元素的描述对象 `element` 赋值给 `root` 变量，同时会调用上面刚刚讲过的 `checkRootConstraint` 函数检查根元素是否符合要求。

再看 `else if` 语句的条件，它检测了 `stack.length` 是否为 `0`，也就是说 `stack` 数组为空的情况下会执行 `else if` 语句块内的代码。试想一下如果 `stack` 数组为空并且当前正在解析开始标签，这说明了什么问题？要想知道这个问题首先要知道 `stack` 数组的作用，前面已经多次提到过了每当遇到一个非一元标签时就会将该标签的描述对象放进数组，并且每当遇到一个结束标签时都会将该标签的描述对象从 `stack` 数组中拿掉，也就是说在只有一个根元素的情况下，正常解析完成一段 `html` 代码后 `stack` 数组应该为空，或者换个说法，即当 `stack` 数组被清空后则说明整个模板字符串已经解析完毕了，但此时 `start` 钩子函数仍然被调用了，这说明模板中存在多个根元素，这时 `else if` 语句块内的代码将被执行，如下：

```javascript
if (root.if && (element.elseif || element.else)) {
  // 省略...
} else if (process.env.NODE_ENV !== 'production') {
  warnOnce(
    `Component template should contain exactly one root element. ` +
    `If you are using v-if on multiple elements, ` +
    `use v-else-if to chain them instead.`
  )
}
```

在编写 Vue 模板时的约束必须有且仅有一个被渲染的根元素，但你可以定义多个根元素，只要能够保证最终只渲染其中一个元素即可，能够达到这个目的方式只有一种，那就是在多个根元素之间使用 `v-if` 或 `v-else-if` 或 `v-else`。看如上代码实现，首先观察如上代码中 `if` 条件语句的判断条件：

```javascript
if (root.if && (element.elseif || element.else))
```

这里解释一下元素对象中的 `.if` 属性、`.elseif` 属性以及 `.else` 属性都是哪里来得，它们是在通过 `processIf` 函数处理元素描述对象时，如果发现元素的属性中有 `v-if` 或 `v-else-if` 或 `v-else`，则会在元素描述对象上添加相应的属性作为标识。

回到上面的 `if` 判断条件，首先 `root.if` 必须为真，要知道一点，即无论定义多少个根元素，`root` 变量始终存储的是第一个根元素描述对象，所以 `root.if` 为真就保证了第一个定义的根元素使用了 `v-if` 指令的。同时条件 `(element.elseif || element.else)` 也必须为真，注意这里是 `element.elseif` 或 `element.else`，而不是 `root.elseif` 或 `root.else`。`root` 为第一个根元素的描述对象，`element` 为当前元素描述对象，即非第一个根元素的描述对象。如果以上条件成立就能保证所有根元素都是由 `v-if`、`v-else-if`、`v-else` 等指令控制的，这就间接保证了被渲染的根元素只有一个，此时 `if` 语句块内的代码将被执行，如下：

```javascript
if (root.if && (element.elseif || element.else)) {
  checkRootConstraints(element)
  addIfCondition(root, {
    exp: element.elseif,
    block: element
  })
} else if (process.env.NODE_ENV !== 'production') {
  // 省略...
}
```

在 `if` 语句块内首先使用了 `checkRootConstraint` 函数检查当前元素是否符合作为根元素的要求，接着调用了 `addIfCondition` 函数，该源码如下：

```javascript
export function addIfCondition (el: ASTElement, condition: ASTIfCondition) {
  if (!el.ifConditions) {
    el.ifConditions = []
  }
  el.ifConditions.push(condition)
}
```

`addIfCondition` 函数接收两个参数，第一个参数为元素的描述对象，第二个参数的类型为 `ASTIfCondition`，说白了也是一个对象，该对象包含两个属性：

```javascript
declare type ASTIfCondition = { exp: ?string; block: ASTElement };
```

分别是 `exp` 属性和 `block` 属性，我们根据调用 `addIfCondition` 函数时传递的参数：

```javascript
if (root.if && (element.elseif || element.else)) {
  checkRootConstraints(element)
  addIfCondition(root, {
    exp: element.elseif,
    block: element
  })
} else if (process.env.NODE_ENV !== 'production') {
  // 省略...
}
```

可知 `exp` 为当前元素描述对象的 `element.elseif` 的值，而 `block` 就是当前元素描述对象。并且第一个参数为 `root`，就是第一个根元素描述对象。此时再看 `addIfCondition` 函数的代码：

```javascript
export function addIfCondition (el: ASTElement, condition: ASTIfCondition) {
  if (!el.ifConditions) {
    el.ifConditions = []
  }
  el.ifConditions.push(condition)
}
```

在 `addIfCondition` 函数内首先检查根元素描述对象是否有 `el.ifConditions` 属性，如果没有则创建该属性并初始化为空数组，接着将 `ASTIfCondition` 类型的对象推进到该数组中，实际上该函数时一个通用的函数，不仅仅用在根元素中，它用在任何由 `v-if`、`v-else-if`、`v-else` 组成的条件渲染的模板中。

通过如上分析可以发现，具有 `v-else-if` 或 `v-else` 属性的元素的描述对象会被添加到具有 `v-if` 属性的元素描述对象的 `.ifConditions` 数组中。

举个例子，如下模板：

```html
<div v-if="a"></div>
<p v-else-if="b"></p>
<span v-else></span>
```

解析后生成的 AST 如下 (简化版)：

```javascript
{
  type: 1,
  tag: 'div',
  ifConditions: [
    {
      exp: 'b',
      block: { type: 1, tag: 'p' /* 省略其他属性 */ }
    },
    {
      exp: undefined,
      block: { type: 1, tag: 'span' /* 省略其他属性 */ }
    }
  ]
  // 省略其他属性...
}
```

可以看到代码 `v-else-if` 和 `v-else` 属性的元素描述对象都被添加到了带有 `v-if` 属性的元素描述对象的 `.ifConditions` 数组中，其实如上描述其实是不正确的，后面会发现带有 `v-if` 属性的元素也会将自身的元素描述对象添加到自身的 `.ifConditions` 数组中，即：

```javascript
{
  type: 1,
  tag: 'div',
  ifConditions: [
    {
      exp: 'a',
      block: { type: 1, tag: 'div' /* 省略其他属性 */ }
    },
    {
      exp: 'b',
      block: { type: 1, tag: 'p' /* 省略其他属性 */ }
    },
    {
      exp: undefined,
      block: { type: 1, tag: 'span' /* 省略其他属性 */ }
    }
  ]
  // 省略其他属性...
}
```

以上就是实现允许使用 `v-if`、`v-else-if` 和 `v-else` 定义多个根元素的方式，顺带着讲解了一个重要的函数 `addIfCondition` 的实现和使用。

话说回来，假如当前元素不满足条件：`root.if && (element.elseif || element.else)`，那么在非生产环境下 `elseif` 语句块的代码将会被执行：

```javascript
if (root.if && (element.elseif || element.else)) {
  // 省略...
} else if (process.env.NODE_ENV !== 'production') {
  warnOnce(
    `Component template should contain exactly one root element. ` +
    `If you are using v-if on multiple elements, ` +
    `use v-else-if to chain them instead.`
  )
}
```

可以看到，在 `elseif` 语句块内通过 `warnOnce` 函数打印了警告信息给开发者友好的提示。

再往下就是如下这段 `if` 条件语句块：

```javascript
if (currentParent && !element.forbidden) {
  // 省略...
}
```

不过暂时跳过它，优先看一下 `start` 钩子函数的最后一段代码，如下：

```javascript
if (!unary) {
  currentParent = element
  stack.push(element)
} else {
  closeElement(element)
}
```

如上这段代码是一个 `if...else` 条件分支语句块，首先看 `if` 语句的条件，它检测了当前元素是否是非一元标签，前面说过了如果一个元素是非一元标签，那么应该将该元素的描述对象添加到 `stack` 栈中，并且将 `currentParent` 变量的值更新为当前元素的描述对象，如上代码中 `if` 语句块内的代码说明了一切。

反之，如果一个元素是一元标签，那么应该调用 `closeElement` 函数闭合该元素。对于 `closeElement` 函数后面再详细说，现在重点关注 `if` 语句块内的两行代码，通过这两行代码至少可以得出一个总结：**每当遇到一个非一元标签都会将该元素的描述对象添加到 `stack` 数组，并且 `currentParent` 始终存储的是 `stack` 栈顶的元素，即当前解析元素的父级**。

知道了这些再回过头来看如下代码：

```javascript
if (currentParent && !element.forbidden) {
  // 省略...
}
```

首先观察 `if` 条件语句的判断条件：

```javascript
currentParent && !element.forbidden
```

如果这个条件成立，则说明当前元素存在父级 `currentParent`，并且当前元素不是被禁止的元素。只有在这种情况下才会执行该 `if` 条件语句块内的代码。在 `if` 语句块内是如下代码：

```javascript
if (element.elseif || element.else) {
  processIfConditions(element, currentParent)
} else if (element.slotScope) { // scoped slot
  currentParent.plain = false
  const name = element.slotTarget || '"default"';
  (currentParent.scopedSlots || (currentParent.scopedSlots = {}))[name] = element
} else {
  currentParent.children.push(element)
  element.parent = currentParent
}
```

在如上这段代码中，最关键的代码应该是 `else` 语句块内的代码：

```javascript
if (element.elseif || element.else) {
  // 省略...
} else if (element.slotScope) { // scoped slot
  // 省略...
} else {
  currentParent.children.push(element)
  element.parent = currentParent
}
```

在 `else` 语句块内，会把当前元素描述对象添加到父级描述对象 `currentParent` 的 `children` 数组中，同时将当前元素对象的 `parent` 属性指向父级元素对象，这样就建立了元素描述对象间的父子级的关系。

但是就像前面讲过的，如果一个标签使用 `v-else-if` 或 `v-else` 指令，那么该元素的描述对象实际上会被添加到对应的 `v-if` 元素描述对象的 `ifConditions` 数组中，而非作为一个独立的子节点，这个工作就是由如上代码中 `if` 语句块的代码完成的：

```javascript
if (element.elseif || element.else) {
  processIfConditions(element, currentParent)
} else if (element.slotScope) { // scoped slot
  // 省略...
} else {
  // 省略...
}
```

由如上代码所示的 `if` 语句的条件可知，如果当前元素使用了 `v-else-if` 或 `v-else` 指令，则会调用 `processIfConditions` 函数，同时将当前元素描述对象 `element` 和父级元素的描述对象 `currentParent` 作为参数传递，如下是 `processIfConditions` 函数的源码：

```javascript
function processIfConditions (el, parent) {
  const prev = findPrevElement(parent.children)
  if (prev && prev.if) {
    addIfCondition(prev, {
      exp: el.elseif,
      block: el
    })
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `v-${el.elseif ? ('else-if="' + el.elseif + '"') : 'else'} ` +
      `used on element <${el.tag}> without corresponding v-if.`
    )
  }
}
```

在 `processIfConditions` 函数内部，首先通过 `findPrevElement` 函数找到当前元素的前一个元素描述对象，并将其赋值给 `prev` 常量，接着进入 `if` 条件语句，判断当前元素的前一个元素是否使用了 `v-if` 指令，可以知道对于使用了 `v-else-if` 或 `v-else` 指令的元素来讲，它们前一个元素必须是使用了 `v-if` 指令才行。如果前一个元素确实使用了 `v-if` 指令，那么则会调用 `addIfCondition` 函数将当前元素描述对象添加到前一个元素的 `ifConditions` 数组中。如果前一个元素没有使用 `v-if` 指令，那么此时将会进入 `else...if` 条件语句的判断，即如果是非生产环境下，会打印警告信息提示开发者没有相符的使用了 `v-if` 指令的元素。

以上是当前元素使用了 `v-else-if` 或 `v-else` 指令时的特殊处理，由此可知 **当一个元素使用了 `v-else-if` 或 `v-else` 指令时，它们是不会作为父级元素子节点的**，而是会被添加到相符的使用了 `v-if` 指令的元素描述对象的 `ifConditions` 数组中。

如果当前元素没有使用 `v-else-if` 或 `v-else` 指令，那么还会判断当前元素是否使用了 `slot-scope` 特性，如下：

```javascript
if (element.elseif || element.else) {
  // 省略...
} else if (element.slotScope) { // scoped slot
  currentParent.plain = false
  const name = element.slotTarget || '"default"'
  ;(currentParent.scopedSlots || (currentParent.scopedSlots = {}))[name] = element
} else {
  // 省略...
}
```

如上代码所示，如果一个元素使用了 `slot-scope` 特性，那么该元素的描述对象会被添加到父级元素的 `scopedSlots` 对象下，也就是说使用了 `slot-scope` 特性的元素和使用了 `v-else-if` 或 `v-else` 指令的元素一样，它们都不会作为父元素的子节点，对于使用了 `slot-scope` 特性的元素来讲，它们会被添加到父级元素描述对象的 `scopedSlots` 对象下。另外由于如上代码中 `elseif` 语句块涉及 `slot-scope` 相关的处理，打算放到后面统一讲解。

接着对 `findPrevElement` 函数做一个补充讲解，`findPrevElement` 函数的作用是寻找当前元素的前一个元素节点，如下是其源码：

```javascript
function findPrevElement (children: Array<any>): ASTElement | void {
  let i = children.length
  while (i--) {
    if (children[i].type === 1) {
      return children[i]
    } else {
      if (process.env.NODE_ENV !== 'production' && children[i].text !== ' ') {
        warn(
          `text "${children[i].text.trim()}" between v-if and v-else(-if) ` +
          `will be ignored.`
        )
      }
      children.pop()
    }
  }
}
```

首先 `findPrevElement` 函数只用在了 `processIfConditions` 函数中，它的作用就是当解析器遇到一个带有 `v-else-if` 或 `v-else` 指令的元素时，找到该元素的前一个元素节点，假设解析如下的 `html` 字符串：

```html
<div>
  <div v-if="a"></div>
  <p v-else-if="b"></p>
  <span v-else="c"></span>
</div>
```

当解析器遇到带有 `v-else-if` 指令的 `p` 标签时，那么此时它的前一个元素接单应该是带有 `v-if` 指令的 `div` 标签，由于当前正在解析的标签为 `p`，此时 `p` 标签的元素描述对象还没有被添加到父级元素描述对象的 `children` 数组中，所以此时父级元素描述对象的 `children` 数组中的最后一个元素节点就应该是 `div` 元素。我们所说的是 **最后一个元素节点，而不是最后一个节点**。所以要想得到 `div` 标签，只要找到父级元素描述对象的 `children` 数组最后一个元素节点即可。

当解析器遇到带有 `v-else` 指令的 `span` 标签时，此时 `span` 标签的前一个元素节点还是 `div` 标签，而不是 `p` 标签，这是因为 `p` 标签的元素描述对象没有被添加到父级元素描述对象的 `children` 数组中，而是被添加到 `div` 标签元素描述对象的 `ifConditions` 数组中了。所以对于 `span` 标签来讲，它的前一个元素节点仍然是 `div` 标签。

总之发现 `findPrevElement` 函数只需要找到父级元素描述对象的最后一个元素节点即可，如下：

```javascript
function findPrevElement (children: Array<any>): ASTElement | void {
  let i = children.length
  while (i--) {
    if (children[i].type === 1) {
      return children[i]
    } else {
      if (process.env.NODE_ENV !== 'production' && children[i].text !== ' ') {
        warn(
          `text "${children[i].text.trim()}" between v-if and v-else(-if) ` +
          `will be ignored.`
        )
      }
      children.pop()
    }
  }
}
```

`findPrevElement` 函数通过 `while` 循环从后向前遍历父级的子节点，并找到最后一个元素节点。理论上该节点就应该带有 `v-if` 指令的元素，如果该元素节点没有 `v-if` 指令，会在 `processIfConditions` 函数中打印警告信息。大家注意 `while` 循环内的代码，使用 `if` 语句检测了子节点的类型是否为 `1`，即是否为元素节点，只有是元素节点时才会将该节点的描述对象作为返回值返回。如果在找到元素节点之前遇到了非元素节点，那么 `else` 分支的代码将会被执行：

```javascript
if (children[i].type === 1) {
  // 省略...
} else {
  if (process.env.NODE_ENV !== 'production' && children[i].text !== ' ') {
    warn(
      `text "${children[i].text.trim()}" between v-if and v-else(-if) ` +
      `will be ignored.`
    )
  }
  children.pop()
}
```

如上代码所示，可以看到非元素节点被从 `children` 数组中 `pop` 出去，所以在非生产环境下如果该非元素节点 `.text` 属性如果不为空，则打印警告信息提示开发者这部分存在于 `v-if` 指令和 `v-else(-if)` 指令之间的内容将被忽略。举个例子：

```html
<div>
  <div v-if="a"></div>
  aaaaa
  <p v-else-if="b"></p>
  bbbbb
  <span v-else="c"></span>
</div>
```

如上代码中的文本 `aaaaa` 和 `bbbbb` 都将被忽略。

到目前为止，大概粗略地过了一遍 `start` 钩子函数的内容，接下来会做一些总结，以使得思路更加清晰：

- `start` 钩子函数是当解析 `html` 字符串遇到开始标签时被调用的。
- 模板中禁止使用 `<style>` 标签和那些没有指定 `type` 属性或 `type` 属性值为 `text/javascript` 的 `<script>` 标签。
- 在 `start` 钩子函数中会调用前置处理函数，这些前置处理函数都放在 `preTransforms` 数组中，这么做的目的是为了不同平台提供对应平台下提供对应平台下的解析工作。
- 前置处理函数执行完之后会调用一些列 `process*` 函数继续对元素描述对象进行加工。
- 通过判断 `root` 是否存在来判断当前解析的元素是否为根元素。
- `slot` 标签和 `template` 标签不能作为根元素，并且根元素不能使用 `v-for` 指令。
- 可以定义多个根元素，但必须使用 `v-if`、`v-else-if` 以及 `v-else` 保证有且仅有一个根元素被渲染。
- 构建 AST 并构建父子级关系是在 `start` 钩子函数中完成的，每当遇到非一元标签，会把它存到 `currentParent` 变量中，当解析该标签的子节点时通过访问 `currentParent` 变量获取父级元素。
- 如果一个元素使用了 `v-else-if` 或 `v-else` 指令，则该元素不会作为子节点，而是会被添加到相符的使用了 `v-if` 指令的元素描述对象的 `ifConditions` 数组中。
- 如果一个元素使用了 `slot-scope` 特性，则该元素也不会作为子节点，它会被添加到父级元素描述对象的 `scopedSlots` 属性中。
- 对于没有使用条件指令或 `slot-scope` 特性的元素，会正常建立父子级关系。

以上的总结就是 `start` 钩子函数在处理开始标签时所做的事情，实际上由于开始标签中包含了大量指令信息 (如 `v-if` 等) 或特性 (如 `slot-scope` 等)，所以在生产 AST 过程中，大部分工作都是由 `start` 函数完成的，接下来将更加细致的去讲解解析过程中的每一个细节。

## 处理使用了 `v-pre` 指令的元素及其子元素

回到 `start` 钩子函数中，开始对 `start` 钩子函数内的代码做细致的分析，首先找到如下这段代码：

```javascript
if (!inVPre) {
  processPre(element)
  if (element.pre) {
    inVPre = true
  }
}
```

为了讲解的流畅性，同时也为了大家容易理解，想要明白如上代码的作用，首先需要了解 `processPre` 函数的作用，如下时 `processPre` 函数的源码：

```javascript
function processPre (el) {
  if (getAndRemoveAttr(el, 'v-pre') != null) {
    el.pre = true
  }
}
```

`processPre` 函数接收元素描述对象作为参数，在 `processPre` 函数内部首先通过 `getAndRemoveAttr` 函数并使用其返回值和 `null` 做比较，如果 `getAndRemoveAttr` 函数的返回值不等于 `null` 则执行 `if` 语句块内的代码，即在元素描述对象上添加 `.pre` 属性并将其值设置为 `true`。

根据传递给该函数的两个参数：第一个参数是元素描述对象，第二个参数是一个字符串 `'v-pre'`。大概可以想到 `getAndRemoveAttr` 函数应该能够获取给定元素的某个属性的值，那么如上代码就应该是获取给定元素的 `'v-pre'` 属性的值。实际上这个猜测只正确了一部分，实际上 `getAndRemoveAttr` 函数还会做更多事情，`getAndRemoveAttr` 函数来自于 `src/compiler/helpers.js` 文件，如下是其代码：

```javascript
export function getAndRemoveAttr (
  el: ASTElement,
  name: string,
  removeFromMap?: boolean
): ?string {
  let val
  if ((val = el.attrsMap[name]) != null) {
    const list = el.attrsList
    for (let i = 0, l = list.length; i < l; i++) {
      if (list[i].name === name) {
        list.splice(i, 1)
        break
      }
    }
  }
  if (removeFromMap) {
    delete el.attrsMap[name]
  }
  return val
}
```

`getAndRemoveAttr` 接收三个参数，其中第三个参数 `removeFromMap` 是一个可选参数，并且它应该是一个布尔值，第一个参数 `el` 为元素描述对象，第二个参数为要获取属性的名字。在 `getAndRemoveAttr` 函数内部首先定义了 `val` 变量，紧接着是一个 `if` 条件语句块，其判断条件为：

```javascript
if ((val = el.attrsMap[name]) != null)
```

由此可知变量 `val` 保存的是要获取属性的值，并且获取属性的值得方式是通过读取元素描述对象的 `.attrsMap` 属性中与给定属性名字 (`name`) 同名的属性值来实现，可以知道元素描述对象的 `.attrsMap` 对象时该元素所有属性的名值对应表。获取到属性值并赋值给 `val` 变量后，会使用该属性值和 `null` 做比较，如果不相等则说明属性值存在，此时会执行 `if` 语句块的代码，如下：

```javascript
const list = el.attrsList
for (let i = 0, l = list.length; i < l; i++) {
  if (list[i].name === name) {
    list.splice(i, 1)
    break
  }
}
```

在 `if` 语句块内遍历了元素描述对象的 `el.attrsList` 数组，并通过属性名 (`name`) 找到相应的数组元素，目的是使用数组的 `splice` 方法将该数组元素从元素描述对象的 `attrsList` 数组中移除。

接着 `getAndRemoveAttr` 函数还会做一个事情，如下：

```javascript
if (removeFromMap) {
  delete el.attrsMap[name]
}
```

如果第三个参数为真，那么还会将该属性从属性名值表 (`attrsMap`) 中移除。最后 `getAndRemoveAttr` 函数会将属性值 `val` 返回，当然如果属性不存在的话，则变量 `val` 的值为 `undefined`。

举个例子直观感受一下 `getAndRemoveAttr` 的作用，假设有如下模板：

```html
<div v-if="display" ></div>
```

如上 `div` 标签的元素描述对象为：

```javascript
element = {
  // 省略其他属性
  type: 1,
  tag: 'div',
  attrsList: [
    {
      name: 'v-if',
      value: 'display'
    }
  ],
  attrsMap: {
    'v-if': 'display'
  }
}
```

假设现在使用 `getAndRemoveAttr` 函数获取该元素的 `v-if` 属性的值：

```javascript
getAndRemoveAttr(element, 'v-if')
```

则该函数的返回值为字符串 `'display'`，同时会将 `v-if` 属性从 `attrsList` 数组中移除，所以经过 `getAndRemoveAttr` 函数处理之后元素的描述对象将变为：

```javascript
element = {
  // 省略其他属性
  type: 1,
  tag: 'div',
  attrsList: [],
  attrsMap: {
    'v-if': 'display'
  }
}
```

可以看到 `attrsList` 属性变为一个空数组，如果传递给 `getAndRemoveAttr` 函数的第三个参数为真：

```javascript
getAndRemoveAttr(element, 'v-if', true)
```

那么除了将 `v-if` 属性从 `attrsList` 数组中移除之外，也会将其从 `attrsMap` 中移除，此时元素描述对象将变为：

```javascript
element = {
  // 省略其他属性
  type: 1,
  tag: 'div',
  attrsList: [],
  attrsMap: {}
}
```

以上就是 `getAndRemoveAttr` 函数的作用，除了获取给定属性的值之外，还会将该属性从 `attrsList` 数组中移除，并可以选择性地将该属性从 `attrsMap` 对象找那个移除。

回到 `processPre` 函数中：

```javascript
function processPre (el) {
  if (getAndRemoveAttr(el, 'v-pre') != null) {
    el.pre = true
  }
}
```

现在来看 `processPre` 函数的逻辑就很容易理解了，可以知道 `processPre` 函数获取给定元素 `v-pre` 属性的值，如果 `v-pre` 属性的值不等于 `null` 则会在元素描述对象上添加 `.pre` 属性，并将其设置为 `true`。这里简单提一下，由于使用 `v-pre` 指令时不需要指定属性值，所以使用 `getAndRemoveAttr` 函数获取到的属性值为空字符串，由于 `'' != null` 成立，所以以上判断条件成立。

了解了 `processPre` 函数的作用之后，再回到 `start` 钩子函数中，如下代码所示：

```javascript
if (!inVPre) {
  processPre(element)
  if (element.pre) {
    inVPre = true
  }
}
```

代码判断了元素对象的 `.pre` 属性是否为真，可以知道假如一个标签使用了 `v-pre` 指令，经过了 `processPre` 函数处理之后，该元素描述对象的 `.pre` 属性值为 `true`，这时会将 `inVPre` 变量的值也设置为 `true`。当 `inVPre` 变量为 `true` 时，意味着 **后续的所有解析工作都处于 `v-pre` 环境下**，编译器会跳过拥有 `v-pre` 指令元素以及其子元素的编译过程，所以后续的编译逻辑需要 `inVPre` 变量作为标识才可以。

另外在如上的代码中需要注意判断条件：`if(!inVPre)`，该条件保证了如果当前解析工作已经处于 `v-pre` 环境下了，则不需要再次执行该 `if` 语句块内的代码。

再往下需要讲的是 `start` 钩子函数中的如下代码：

```javascript
if (platformIsPreTag(element.tag)) {
  inPre = true
}
```

这段代码相对简单一些，使用 `platformIsPreTag` 函数判断当前元素是否是 `<pre>` 标签，如果是 `<pre>` 标签则将 `inPre` 变量设置为 `true`。实际上 `inPre` 变量与 `inVPre` 变量的作用相同，都是用来作为一个标识，只不过 `inPre` 变量标识当前解析环境是否在 `<pre>` 标签内，因为 `<pre>` 标签内的解析行为和其他 `html` 标签是不同。具体不同体现在：

- `<pre>` 标签会对其所包含的 `html` 字符实体进行解码
- `<pre>` 标签会保留 `html` 字符串编写时的空白

更具体的实现会在后面的分析中会讲到，再往下需要看的是如下这段代码：

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

这段代码是一个 `if...else if` 语句块，其中 `if` 语句块内的代码会在判断条件 `inVPre` 为 `true` 的情况下执行，`inVPre` 为 `true` 说明当前解析环境是在 `v-pre` 环境下。可以知道使用 `v-pre` 指令的标签及其子标签的解析行为是不一致的，编译器会跳过使用了 `v-pre` 指令元素及其子元素的编译工作。通过如上代码可以知道当前元素的解析处于 `v-pre` 环境，则直接使用 `processRawAttrs` 函数对元素描述对象进行加工。同时注意 `else if` 分支内的代码，可以看到如果当前元素的解析没有处于 `v-pre` 环境，那么会调用一系列 `process*` 函数来处理该元素的描述对象。

现在假设要解析的标签使用了 `v-pre` 指令，如下：

```html
<div v-pre v-on:click="handleClick"></div>
```

当解析如上 `html` 字符串时首先会遇到 `div` 开始标签，由于该 `div` 开始标签使用了 `v-pre` 指令，所以此时 `inVPre` 的值为 `true`，所以 `processRawAttrs` 函数将被执行，如下是 `processRawAttrs` 函数的源码：

```javascript
function processRawAttrs (el) {
  const l = el.attrsList.length
  if (l) {
    const attrs = el.attrs = new Array(l)
    for (let i = 0; i < l; i++) {
      attrs[i] = {
        name: el.attrsList[i].name,
        value: JSON.stringify(el.attrsList[i].value)
      }
    }
  } else if (!el.pre) {
    // non root node in pre blocks with no attributes
    el.plain = true
  }
}
```

`processRawAttrs` 函数接收元素描述对象作为参数，其作用是将该元素所有属性全部作为原生的属性 (`attr`) 处理。在 `processRawAttrs` 函数内部首先定义了 `l` 常量，它是元素描述对象属性数组 `el.attrsList` 的长度，接着使用一个 `if` 语句判断 `l` 是否为 `true`，如果为 `true` 说明该元素的开始标签上有属性，此时会执行 `if` 语句块内的代码，在 `if` 语句块内首先定义了 `attrs` 常量，它与 `el.attrs` 属性有着相同的引用，初始值是长度为 `l` 的数组。接着使用 `for` 循环遍历 `el.attrsList` 数组中的每一个属性，并将这些属性挪到 `attrs` 数组中：

```javascript
for (let i = 0; i < l; i++) {
  attrs[i] = {
    name: el.attrsList[i].name,
    value: JSON.stringify(el.attrsList[i].value)
  }
}
```

可以看到 `attrs` 数组中的每个元素都与 `el.attrsList` 数组中的元素相同，都是一个带有 `name` 属性和 `value` 属性的对象，其中 `name` 属性存储着属性的名字，`value` 属性存储着属性的值，这里大家注意 `value` 的值：

```javascript
JSON.stringify(el.attrsList[i].value)
```

这里的 `JSON.stringify` 函数很重要，实际上 `el.attrsList[i].value` 本身就已经是一个字符串了，在字符串的基础上继续使用 `JSON.stringify` 的意义在下方的例子可窥一二，如下时两个使用了 `new Function()` 创建函数的例子：

```javascript
const fn1 = new Function('console.log(1)')
const fn2 = new Function(JSON.stringify('console.log(1)'))
```

上方代码中定义了两个函数 `fn1` 和 `fn2`，它们的区别在于 `fn2` 的参数使用了 `JSON.stringify`，实际上，上方代码等价于：

```javascript
const fn1 = function () {
  console.log(1)
}
const fn2 = function () {
  'console.log(1)'
}
```

可以看到 `fn1` 函数的执行能够通过 `console.log` 语句打印数字 `1`，而 `fn2` 函数体内的 `console.log` 语句时一个字符串。

回到这段代码中：

```javascript
attrs[i] = {
  name: el.attrsList[i].name,
  value: JSON.stringify(el.attrsList[i].value)
}
```

同样的，这里使用 `JSON.stringify` 实际上就是保证了最终生成的代码中 `el.attrsList[i].value` 属性始终都被作为普通的字符串处理。通过以上代码的讲解可以知道，如果一个标签的解析处于 `v-pre` 环境，则会将该标签的属性全部添加到元素描述对象的 `.attrs` 数组中，并且 `.attshenrs` 数组和 `.attrsList` 数组几乎相同，唯一不同的是在 `.attrs` 数组中每个对象中的 `value` 属性值都是通过 `JSON.stringify` 处理过的。

注意 `processRawAttrs` 函数还没完，如下：

```javascript
function processRawAttrs (el) {
  const l = el.attrsList.length
  if (l) {
    // 省略...
  } else if (!el.pre) {
    // non root node in pre blocks with no attributes
    el.plain = true
  }
}
```

假如 `el.attrsList` 数组的长度为 `0`，则会进入 `else...if` 分支的判断，检查该元素是否使用了 `v-pre` 指令，如果没有使用 `v-pre` 指令才会执行 `else...if` 语句块的代码，思考一下，首先具备一个大前提，即 `processRawAttrs` 函数的执行说明当前解析必然处于 `v-pre` 环境，要么是使用 `v-pre` 指令的标签本身，要么就是其子节点。同时 `el.attrsList` 数组的长度为 `0` 说明该元素没有任何属性，而且 `else...if` 条件的成立也说明该元素没有使用 `v-pre` 指令，这说明该元素一定是使用了 `v-pre` 指令的标签的子标签，如下：

```html
<div v-pre>
  <span></span>
</div>
```

如上 `html` 字符串所示，当解析 `span` 标签时，由于 `span` 标签没有任何属性，并且 `span` 标签也没有使用 `v-pre` 指令，所以此时会在 `span` 标签的元素描述对象上添加 `.plain` 属性并将其设置为 `true`，用来标识哪些元素是纯的，在代码生成的部分将看到一个被标识为 `plain` 的元素将有哪些不同。

最后对使用了 `v-pre` 指令的标签所生成的元素描述对象做一个总结：

- 如果标签使用了 `v-pre` 指令，则该标签的元素描述对象的 `element.pre` 属性将为 `true`。
- 对于使用了 `v-pre` 指令的标签以及其子代标签，它们的任何属性都将会被作为原始属性处理，即使用 `processRawAttrs` 函数处理。
- 经过 `processRawAttrs` 函数的处理，会在元素的描述对象上添加 `element.attrs` 属性，它与 `element.attrsList` 数组结构相同，不同的是 `element.attrs` 数组中每个对象的 `value` 值都会经过 `JSON.stringify` 函数处理。
- 如果一个标签没有任何属性，并且该标签时使用了 `v-pre` 指令标签的子代标签，那么该标签的元素描述对象将被添加 `element.plain` 属性，并且其值为 `true`。

以上就是生成 AST 过程中对于使用了 `v-pre` 指令标签的元素描述对象的处理。

## 处理使用了 `v-for` 指令的元素

接下来回到如下这段代码：

```javascript
if (inVPre) {
  // 省略...
} else if (!element.processed) {
  // 省略...
}
```

如果一个标签使用了 `v-pre` 指令，那么该标签及其子标签的解析都会由 `if` 语句块内的 `processRawAttrs` 函数来完成，反之将会执行 `else...if` 条件语句的判断，可以看到其判断条件为 `!element.processed`，这里要补充一下元素描述对象的 `element.processed` 属性是一个布尔值，它标识着当前元素是否已经被解析过了，或许大家会对 `element.processed` 属性有疑问，实际上 `element.processed` 属性是在元素描述对象应用 `preTransforms` 数组中的处理函数时被添加的，可以打开 `src/platforms/web/compiler/modules/model.js` 文件找到 `preTransformNode` 函数，该函数中有这样一段代码，如下：

```javascript
processFor(branch0)
addRawAttr(branch0, 'type', 'checkbox')
processElement(branch0, options)
branch0.processed = true // prevent it from double-processed
```

由于还没有对 `preTransforms` 前置处理函数进行讲解，所以大家看不明白上方代码是没有关系的，只需要知道如上代码的处理之后，由于元素已经被处理过了，所以这里会通过 `.processed` 做一个标识，以防止被重复处理。再回到如下这段代码：

```javascript
if (inVPre) {
  // 省略...
} else if (!element.processed) {
  // structural directives
  processFor(element)
  // 省略...
}
```

如果元素没有被处理过，那么 `else...if` 语句块内的代码将会被执行，可以看到对元素描述对象应用的第一个处理函数是 `processFor` 函数，接下来的目标就是研究 `processFor` 函数对元素描述对象做了怎么样的处理。

找到 `processFor` 函数，如下是其源码：

```javascript
export function processFor (el: ASTElement) {
  let exp
  if ((exp = getAndRemoveAttr(el, 'v-for'))) {
    const res = parseFor(exp)
    if (res) {
      extend(el, res)
    } else if (process.env.NODE_ENV !== 'production') {
      warn(
        `Invalid v-for expression: ${exp}`
      )
    }
  }
}
```

`processFor` 函数接收元素描述对象作为参数，在 `processFor` 函数内部首先定义了 `exp` 变量，接着是一个 `if` 条件语句块。在判断条件中首先通过 `getAndRemoveAttr` 函数从元素描述对象中获取 `v-for` 属性对应的属性值，并将值赋值给 `exp` 变量，如果标签的 `v-for` 属性值存在则会执行 `if` 语句块内的代码，否则什么都不会做。

对于 `getAndRemoveAttr` 函数前面已经讲过了这里就不再补充了。现在假设当前元素是一个使用了 `v-for` 指令的 `div` 标签，如下：

```html
<div v-for="obj in list"></div>
```

那么 `exp` 变量的值将是字符串 `'obj in list'`，此时 `if` 语句块内的代码将会执行，在 `if` 语句块内一上来就通过 `parseFor` 函数对 `v-for` 属性的值做解析，把目光转移到 `parseFor` 函数上，细看 `parseFor` 函数是如何解析字符串 `'obj in list'` 的。

`parseFor` 函数的源码如下：

```javascript
export function parseFor (exp: string): ?ForParseResult {
  const inMatch = exp.match(forAliasRE)
  if (!inMatch) return
  const res = {}
  res.for = inMatch[2].trim()
  const alias = inMatch[1].trim().replace(stripParensRE, '')
  const iteratorMatch = alias.match(forIteratorRE)
  if (iteratorMatch) {
    res.alias = alias.replace(forIteratorRE, '')
    res.iterator1 = iteratorMatch[1].trim()
    if (iteratorMatch[2]) {
      res.iterator2 = iteratorMatch[2].trim()
    }
  } else {
    res.alias = alias
  }
  return res
}
```

`parseFor` 函数接收 `v-for` 指令的值作为参数，现在假设参数 `exp` 的值为字符串 `'obj in list'`。在 `parseFor` 函数开头首先使用字符串 `exp` 去匹配正则 `forAliasRE`，并将匹配的结果保存在 `inMatch` 常量中，该正则的作用在本章开头讲过，所以这里不做过多讲解，如果 `exp` 字符串为 `'obj in list'`，那么最终 `inMatch` 常量则是一个数组，如下：

```javascript
const inMatch = [
  'obj in list',
  'obj',
  'list'
]
```

如果匹配失败则 `inMatch` 常量的值将为 `null`。可以看到在 `parseFor` 函数内部如果匹配失败则函数直接返回 `undefined`：

```javascript
export function parseFor (exp: string): ?ForParseResult {
  const inMatch = exp.match(forAliasRE)
  if (!inMatch) return
  // 省略...
}
```

可以回到 `processFor` 函数，注意以下代码：

```javascript
export function processFor (el: ASTElement) {
  let exp
  if ((exp = getAndRemoveAttr(el, 'v-for'))) {
    const res = parseFor(exp)
    if (res) {
      extend(el, res)
    } else if (process.env.NODE_ENV !== 'production') {
      warn(
        `Invalid v-for expression: ${exp}`
      )
    }
  }
}
```

可以看到在 `processFor` 函数内部定义了 `res` 常量接收 `parseFor` 函数对 `exp` 字符串的解析结果，如果解析失败则 `res` 常量的值将为 `undefined`，所以在非生产环境下会打印警告信息提示开发者所编写的 `v-for` 指令的值为无效的。

再回到 `parseFor` 函数中，如果对 `exp` 字符串解析成功，则如下的第三、第四两行代码将被执行：

```javascript
export function parseFor (exp: string): ?ForParseResult {
  const inMatch = exp.match(forAliasRE)
  if (!inMatch) return
  const res = {}
  res.for = inMatch[2].trim()
  // 省略...
  return res
}
```

定义了 `res` 常量，它的初始值是一个空对象，可以看到最后 `parseFor` 函数会将 `res` 对象作为返回值返回。接着在 `res` 对象上添加 `res.for` 属性，它的值为 `inMatch` 数组的第三个元素，假如 `exp` 字符串的值为 `'obj in list'`，则 `res.for` 属性的值将是字符串 `'list'`，所以大家应该能够猜测到 `res.for` 属性所存储的值应该是被遍历的目标变量的名字。

再往下将会执行如下第二第三两行代码：

```javascript
export function parseFor (exp: string): ?ForParseResult {
  // 省略...
  res.for = inMatch[2].trim()
  const alias = inMatch[1].trim().replace(stripParensRE, '')
  const iteratorMatch = alias.match(forIteratorRE)
  // 省略...
  return res
}
```

定义了 `alias` 常量，它的值比较复杂，一点点来看，假设字符串 `exp` 的值为 `'obj in list'`，则 `inMatch[1]` 的值应该是字符串 `'obj'`，如果 `exp` 字符串的值 `'(obj, index) in list'`，那么 `inMatch[1]` 的值应该是字符串 `'(obj, index)'`，当然了，如果在编写 `v-for` 指令时存在多余空格，比如：

```html
<div v-for="  obj in list"></div>
```

则 `exp` 字符串也会有多余的空格：`'  obj in list'`，这时就会导致 `inMatch[1]` 的值中也会包含多余的空格：`'  obj'`。理想的做法是此时将多余的空格去掉，然后再做下一步处理，这就是 `parseFor` 函数中对 `inMatch[1]` 字符串使用 `trim()` 函数的原因。去掉空格之后，可以看到紧接着使用该字符串的 `replace` 方法匹配正则 `stripParenRE`，并将匹配的内容替换成空字符串，最终的结果是将 `inMatch[1]` 中的左右圆括号移除，本章开头讲解了正则 `stripParensRE` 的作用，它用来匹配字符串中的左右圆括号。

如下是 `v-for` 指令的值和 `alias` 常量值的对应关系：

- 如果 `v-for` 指令的值为 `'obj in list'`，则 `alias` 的值为字符串 `'obj'`
- 如果 `v-for` 指令的值为 `(obj, index) in list`，则 `alias` 的值为字符串 `obj, index`
- 如果 `v-for` 指令的值为 `(obj, key, index) in list`，则 `alias` 的值为字符串 `'obj, key, index'`

了解了 `alias` 常量的值之后，再来看如下这行代码：

```javascript
const iteratorMatch = alias.match(forIteratorRE)
```

这里定义了 `iteratorMatch` 常量，它的值是使用 `alias` 字符串的 `match` 方法匹配正则 `forIteratorRE` 得到的，其中正则 `forIteratorRE` 在前面的章节中讲过了，这里总结一下对于不同的 `alias` 字符串其对应的匹配结果：

- 如果 `alias` 字符串的值为 `'obj'`，则匹配结果 `iteratorMatch` 常量的值为 `null`
- 如果 `alias` 字符串的值为 `'obj, index'`，则匹配结果 `iteratorMatch` 常量的值是一个包含两个元素的数组 `[', index', 'index']`
- 如果 `alias` 字符串的值为 `obj, key, value`，则匹配结果 `iteratorMatch` 常量的值是一个包含三个元素的数组：`[', key, index', 'key', 'index']`

明白了这些，继续看 `parseFor` 函数的代码，接下来要看的是如下这段代码：

```javascript
export function parseFor (exp: string): ?ForParseResult {
  // 省略...
  const iteratorMatch = alias.match(forIteratorRE)
  if (iteratorMatch) {
    res.alias = alias.replace(forIteratorRE, '')
    res.iterator1 = iteratorMatch[1].trim()
    if (iteratorMatch[2]) {
      res.iterator2 = iteratorMatch[2].trim()
    }
  } else {
    res.alias = alias
  }
  return res
}
```

如上代码所示，可以知道如果 `alias` 常量的值为字符串 `'obj'` 时，则匹配结果 `iteratorMatch` 常量的值会是 `null`，所以此时 `if` 条件语句判断失败，`else` 语句块的代码将被执行，即在 `res` 对象上添加 `res.alias` 属性，其值就是 `alias` 常量的值，也就是字符串 `'obj'`。

如果 `alias` 常量的值为字符串 `'obj, index'`，则匹配结果 `iteratorMatch` 常量将会是一个拥有两个元素的数组，此时 `if` 语句块内的代码将被执行，在 `if` 语句块内首先执行的是如下代码：

```javascript
res.alias = alias.replace(forIteratorRE, '')
```

使用 `alias` 字符串的 `replace` 方法去匹配正则 `forIteratorRE`，并将匹配到的内容替换成空字符串，最后将结果赋给 `res.alias` 属性。如果字符串 `alias` 的值为 `'obj, index'`，则替换后的结果应该为字符串 `'obj'`。所以 `res.alias` 属性的值就是字符串 `'obj'`。

接着执行的是这行代码：

```javascript
res.iterator1 = iteratorMatch[1].trim()
```

在 `res` 对象上定义 `res.iterator1` 属性，它的值是匹配结果 `iteratorMatch` 数组第二个元素去除前后空白之后的值。假设 `alias` 字符串为 `'obj, index'`，则 `res.iterator1` 的值应该为字符串 `'index'`。

再往下会进入另外一个 `if` 条件语句：

```javascript
if (iteratorMatch[2]) {
  res.iterator2 = iteratorMatch[2].trim()
}
```

由于 `alias` 字符串的值为 `'obj, index'`，对应的匹配结果 `iteratorMatch` 数组只有两个元素，所以 `iteratorMatch[2]` 的值为 `undefined`，此时如上 `if` 语句块内的代码不会被执行，但是如果 `alias` 字符串的值为 `obj, key, index`，则匹配结果 `iteratorMatch[2]` 的值将会是字符串 `'index'`，此时 `if` 语句块内的代码将会被执行，可以看到 `res` 对象上定义了 `res.iterator2` 属性，其值就是字符串 `iteratorMatch[2]` 去掉前后空白后的结果。

以上就是 `parseFor` 函数的全部实现，它的作用是解析 `v-for` 指令的值，并创建一个包含解析结果的对象，最后将该对象返回。来做一个简洁的总结：

- 如果 `v-for` 指令的值为字符串 `'obj in list'`，则 `parseFor` 函数的返回值为：

  ```javascript
  {
    for: 'list',
    alias: 'obj'
  }
  ```

- 如果 `v-for` 指令的值为字符串 `'(obj, index) in list'`，则 `parseFor` 函数的返回值为：

  ```javascript
  {
    for: 'list',
    alias: 'obj',
    iterator1: 'index'
  }
  ```

- 如果 `v-for` 指令的值为字符串 `'(obj, key, index) in list'`，则 `parseFor` 函数的返回值为：

  ```javascript
  {
    for: 'list',
    alias: 'obj',
    iterator1: 'key',
    iterator2: 'index'
  }
  ```

最后再回到 `processFor` 函数，看如下这段代码：

```javascript
export function processFor (el: ASTElement) {
  let exp
  if ((exp = getAndRemoveAttr(el, 'v-for'))) {
    const res = parseFor(exp)
    if (res) {
      extend(el, res)
    } else if (process.env.NODE_ENV !== 'production') {
      warn(
        `Invalid v-for expression: ${exp}`
      )
    }
  }
}
```

可以看到如果 `parseFor` 函数对 `v-for` 指令的值解析成功，则会将解析结果保存在 `res` 常量中，并使用 `extend` 函数将 `res` 常量中的属性混入当前元素的描述对象中。

以上就是解析器对于使用了 `v-for` 指令的标签的解析过程，以及对该元素描述对象的补充。

## 处理使用条件指令和 `v-once` 指令的元素

在使用 `processFor` 函数处理完元素描述对象之后，紧接着使用了 `processIf` 函数继续对元素的描述对象进行处理，如下代码所示：

```javascript
if (inVPre) {
  processRawAttrs(element)
} else if (!element.processed) {
  // structural directives
  processFor(element)
  processIf(element)
  // 省略...
}
```

`processIf` 函数用来处理那些使用了条件指令的标签的元素描述对象，所谓条件指令指的是 `v-if`、`v-else-if`、`v-else` 这三个指令。找到 `processIf` 函数，源码如下：

```javascript
function processIf (el) {
  const exp = getAndRemoveAttr(el, 'v-if')
  if (exp) {
    el.if = exp
    addIfCondition(el, {
      exp: exp,
      block: el
    })
  } else {
    if (getAndRemoveAttr(el, 'v-else') != null) {
      el.else = true
    }
    const elseif = getAndRemoveAttr(el, 'v-else-if')
    if (elseif) {
      el.elseif = elseif
    }
  }
}
```

`processIf` 函数接收元素描述对象作为参数，在 `processIf` 函数内部首先通过 `getAndRemoveAttr` 函数从该元素描述对象的 `attrsList` 属性中获取并移除 `v-if` 指令的值，并将属性赋值给 `exp` 常量，这里需要注意的是如何判断是否使用了 `v-if` 指令，如上代码是这样判断的：

```javascript
function processIf (el) {
  const exp = getAndRemoveAttr(el, 'v-if')
  if (exp) {
    // 省略...
  } else {
    // 省略...
  }
}
```

能不能改成如下这种判断方式呢？实际上下边的这种判断方式已经见过很多次了：

```javascript
function processIf (el) {
  if (getAndRemoveAttr(el, 'v-if') != null) {
    // 省略...
  } else {
    // 省略...
  }
}
```

如上这种比较方式实际上是把 `v-if` 指令的值和 `null` 做对比，只要不等于 `null` 则条件就会成立，所以如果在编写 `v-if` 指令时没有写属性值，则通过 `getAndRemoveAttr` 函数获取到的 `v-if` 属性值将是一个空字符串，由于空字符串不等于 `null`，所以如上条件会成立。但是源码中的比较方式不会这样，如果在编写 `v-if` 指令时没有写属性，则 `exp` 常量就是空字符串，所以 `if` 条件语句不会执行。实际上源码的实现方式更合理，源码的逻辑只要没有写 `v-if` 指令的属性，那么就当做你根本没有使用 `v-if` 指令，不然的话元素永远不会渲染。

假设读取到了 `v-if` 指令的值，此时 `if` 语句块内的代码将被执行，如下：

```javascript
function processIf (el) {
  const exp = getAndRemoveAttr(el, 'v-if')
  if (exp) {
    el.if = exp
    addIfCondition(el, {
      exp: exp,
      block: el
    })
  } else {
    // 省略...
  }
}
```

在 `if` 语句块内首先在元素描述对象上定义了 `el.if` 属性，并且该属性的值就是 `v-if` 指令的属性值，注意目前所说的属性值都指的是字符串，比如 `html` 字符串如下：

```html
<div v-if="a && b"></div>
```

则该元素描述对象的 `el.if` 的值为字符串 `'a && b'`。在设置完 `el.if` 属性之后，紧接着调用了 `addIfCondition` 函数，可以看到第一个参数就是当前元素描述对象本身，所以如果一个元素使用了 `v-if` 指令，那么它会把自身作为一个条件对象添加到自身元素描述的 `ifConditions` 数组中，补充一下这里所说的条件对象指的是形如 `addIfCondition` 函数第二个参数的对象结构：

```javascript
{
  exp: exp,
  block: el
}
```

这一点在前边分析 `processIfConditions` 函数时有提到过。

再回到 `processIf` 函数中，如下：

```javascript
function processIf (el) {
  const exp = getAndRemoveAttr(el, 'v-if')
  if (exp) {
    // 省略...
  } else {
    if (getAndRemoveAttr(el, 'v-else') != null) {
      el.else = true
    }
    const elseif = getAndRemoveAttr(el, 'v-else-if')
    if (elseif) {
      el.elseif = elseif
    }
  }
}
```

如果没有获取到 `v-if` 指令的属性值，则 `else` 语句块的代码将被执行，可以看到在 `else` 语句块内分别处理了 `v-else` 指令以及 `v-else-if` 指令。首先先来看对于 `v-else` 指令的处理，如下：

```javascript
if (getAndRemoveAttr(el, 'v-else') != null) {
  el.else = true
}
```

通过 `getAndRemoveAttr` 函数获取并移除元素描述对象的 `attrsList` 数组中名字为 `v-else` 的属性值，可以看到和 `v-if` 指令的判断条件不同，这里是将属性值和 `null` 做比较，这说明 `v-else` 指令时即使不写属性值也会当做使用了 `v-else` 指令，很显然 `v-else` 指令根本就不需要属性值。如果该元素使用了 `v-else` 指令则会在该元素的描述对象上添加 `el.else` 属性，并将其设置为 `true`。

接着还要处理使用了 `v-else-if` 指令的标签，如下：

```javascript
const elseif = getAndRemoveAttr(el, 'v-else-if')
if (elseif) {
  el.elseif = elseif
}
```

很简单，和处理 `v-if` 指令的方式相同，唯一不同的就是此时会在元素描述对象上添加 `el.elseif` 属性，并且它的值为 `v-else-if` 的属性值。

最后大家注意一件事，就是对于使用了 `v-else` 和 `v-else-if` 这两个条件指令的标签，经过 `processIf` 函数的处理之后仅仅是在元素描述对象上添加了 `el.else` 属性和 `el.elseif` 属性，并没有做额外的工作。但是在前面分析 `processIfConditions` 函数时能够知道，当一个元素描述对象存在 `el.else` 属性或 `el.elseif` 属性时，该元素描述对象不会作为 AST 中的一个普通节点，而是会被添加到与之相符带有 `v-if` 指令的元素描述对象的 `ifConditions` 数组中。

按照惯例做一个简短的总结：

- 如果标签使用了 `v-if` 指令，则该标签的元素描述对象的 `el.if` 属性存储着 `v-if` 指令的属性值
- 如果标签使用了 `v-else` 指令，则该标签的元素描述对象的 `el.else` 属性值为 `true`
- 如果标签使用了 `v-else-if` 指令，则该标签的元素描述对象的 `el.elseif` 属性值存储着 `v-else-if` 指令的属性值
- 如果标签使用了 `v-if` 指令，则该标签的元素描述对象的 `ifConditions` 数组中包含自己
- 如果标签使用了 `v-else` 或 `v-else-if` 指令，则该标签的元素描述对象会被添加到与之相符的带有 `v-if` 指令的元素描述对象的 `ifConditions` 数组中。

讲解完 `processIf` 函数之后，再来看一下在 `processIf` 函数之后执行的 `processOnce` 函数：

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

`processOnce` 函数用来处理使用了 `v-once` 指令的标签，处理方式很简单，如下：

```javascript
function processOnce (el) {
  const once = getAndRemoveAttr(el, 'v-once')
  if (once != null) {
    el.once = true
  }
}
```

首先通过 `getAndRemoveAttr` 函数获取并移除描述对象的 `attrsList` 数组中名字为 `v-once` 的属性值，并将获取到的属性值赋给 `once` 常量，接着使用 `if` 条件语句，如果 `once` 常量不等于 `null`，则说明使用了 `v-once` 指令，此时会在元素描述对象上添加 `el.once` 属性并将其设置为 `true`。

## 处理使用了 `key` 属性的元素

再往下是 `processElement` 函数了，如下：

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

实际上 `processElement` 函数是其他一系列 `process*` 函数的集合，如下：

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

如上是 `processElement` 函数的全部代码，可以看到在 `processElement` 函数内确实调用了很多其他的 `process*` 函数，除此之外在 `processComponent` 函数与 `processAttrs` 函数之间应用了 `transforms` 数组中的转换函数，一点点来分析，首先看 `processElement` 函数内执行的第一个函数，即 `processKey` 函数，如下是 `processKey` 函数的源码：

```javascript
function processKey (el) {
  const exp = getBindingAttr(el, 'key')
  if (exp) {
    if (process.env.NODE_ENV !== 'production' && el.tag === 'template') {
      warn(`<template> cannot be keyed. Place the key on real elements instead.`)
    }
    el.key = exp
  }
}
```

`processKey` 函数接收元素的描述对象作为参数，在 `processKey` 函数内部首先调用了 `getBindingAttr` 函数，这个函数目前还是第一次遇到，尽管将它当做和 `getAndRemoveAttr` 函数的作用相同即可，后面会详细讲解。`getBindingAttr` 函数和 `getAndRemoveAttr` 函数接收的前两个参数是一样的，并且也会返回第二个参数指定的属性值，所以如上代码中通过 `getBindingAttr` 函数从元素描述对象的 `attrsList` 数组中获取到属性名字为 `key` 的属性值，并将值赋值给 `exp` 常量。接着用一个 `if` 条件语句检查 `exp` 是否存在，如果不存在则说明没有为该标签的 `key` 属性指定属性值，如果属性值存在则会为元素描述对象添加 `el.key` 属性，且它的值就是 `key` 属性的值。另外能够看到在为 `el.key` 属性赋值之前还有一个 `if` 条件语句，如下：

```javascript
if (process.env.NODE_ENV !== 'production' && el.tag === 'template') {
  warn(`<template> cannot be keyed. Place the key on real elements instead.`)
}
```

在非生产环境下会检测该标签是否是 `<template>` 标签，如果是 `template` 标签则会提示开发者不要在 `<template>` 标签上使用 `key` 属性。

以下是对使用了 `key` 属性的标签的解析总结：

- `key` 属性不能被应用到 `<template>` 标签
- 使用了 `key` 属性的标签，其元素描述对象的 `el.key` 属性保存着 `key` 属性的值

## 获取绑定的属性值以及过滤器的解析

在讲解 `processKey` 函数时遇到了 `getBindingAttr` 函数，当时没有仔细讲解，并且让大家理解为它的作用和 `getAndRemoveAttr` 函数的作用相同。接下来仔细研究 `getBindingAttr` 函数，如下是其源码：

```javascript
export function getBindingAttr (
  el: ASTElement,
  name: string,
  getStatic?: boolean
): ?string {
  const dynamicValue =
    getAndRemoveAttr(el, ':' + name) ||
    getAndRemoveAttr(el, 'v-bind:' + name)
  if (dynamicValue != null) {
    return parseFilters(dynamicValue)
  } else if (getStatic !== false) {
    const staticValue = getAndRemoveAttr(el, name)
    if (staticValue != null) {
      return JSON.stringify(staticValue)
    }
  }
}
```

如上代码，可以发现在 `getBindingAttr` 函数内部多次调用了 `getAndRemoveAttr` 函数。实际上 `getBindingAttr` 函数的作用就像它的名字一样，用来获取绑定属性的值。绑定属性就是通过 `v-bind:` 或其缩写 `:` 所定义的属性。`getBindingAttr` 函数接收三个参数，前两个参数和 `getAndRemoveAttr` 函数相同，分别是元素的描述对象和要获取的属性的名字。在 `getBindingAttr` 函数内部首先执行的是如下这段代码：

```javascript
const dynamicValue =
  getAndRemoveAttr(el, ':' + name) ||
  getAndRemoveAttr(el, 'v-bind:' + name)
```

可以看到这段代码首先通过 `getAndRemoveAttr` 函数获取名字为 `':' + name` 的属性值，如果传递给 `getBindingAttr` 函数的第二个参数为字符串 `'key'`，则表达式 `':' + name` 的值就是 `':key'`，如果获取不到属性名为 `:key` 的属性的值，则会继续使用 `getAndRemoveAttr` 获取 `v-bind:key` 属性的值，这是因为没有办法保证开发者到底通过 `v-bind:` 还是通过其缩写 `:` 来绑定属性，所以两种方式都要尝试。最后将获取到的属性值赋值给 `dynamicValue` 常量。

获取到了绑定的属性值之后，将会执行如下代码：

```javascript
if (dynamicValue != null) {
  return parseFilters(dynamicValue)
} else if (getStatic !== false) {
  const staticValue = getAndRemoveAttr(el, name)
  if (staticValue != null) {
    return JSON.stringify(staticValue)
  }
}
```

这段代码是一段 `if...else` 添加语句块，这里再次强调 `if` 语句的条件是在判断绑定的属性是否存在，而非判断属性值 `dynamicValue` 是否存在，因为即使获取到的属性值为空字符串，但由于空字符串不与 `null` 相等，所以 `if` 条件语句成立。只有当绑定属性本身不存在时，此时获取到的属性值为 `undefined`，与 `null` 相等，这时才会执行 `elseif` 分支的判断。

假设成功得到了获取绑定的属性值，那么 `if` 语句块内的代码将被执行，可以看到在 `if` 语句块内直接调用了 `parseFilters` 函数并将该函数的返回值作为 `getBindingAttr` 函数的返回值。其中 `parseFilters` 函数是接下来需要重点讲解的函数，不过现在仍需要将目光聚焦在 `getBindingAttr` 函数上。

如果获取绑定的值失败，则会执行 `else if` 分支的判断，可以看到 `else if` 分支检测了 `getBindingAttr` 函数的第三个参数 `getStatic` 是否和 `false` 不全等，这里的关键是一定是不全等才行，也就是说如果调用 `getBindingAttr` 函数时不传递第三个参数，则参数 `getStatic` 的值为 `undefined`，它不全等于 `false`，所以可以理解为当不传递第三个参数时 `elseif` 分支的条件默认成立。`else if` 语句块内的代码的作用是用来获取非绑定的属性值，因为代码既然执行到了 `else if` 分支，则说明此时获取绑定的属性值失败，可以知道当为元素或组件添加属性时，这个属性可以是绑定的也可以是非绑定的，所以当获取绑定的属性失败时，不能够武断地认为开发者没有编写该属性，而是应该继续尝试获取非绑定的属性值，如下代码所示：

```javascript
if (dynamicValue != null) {
  return parseFilters(dynamicValue)
} else if (getStatic !== false) {
  const staticValue = getAndRemoveAttr(el, name)
  if (staticValue != null) {
    return JSON.stringify(staticValue)
  }
}
```

非绑定属性值的获取方式同样是使用 `getAndRemoveAttr` 函数，只不过此时传递给该函数的第二个参数是原始的属性名字，不带有 `v-bind` 或 `:`。同时将获取结果保存在 `staticValue` 常量中，接着进入一个条件判断，如果属性值存在则使用 `JSON.stringify` 函数对属性值进行处理后将其返回。

`JSON.stringify` 函数对属性值的处理至关重要，这么做能够保证对于非绑定的属性来讲，总是会将该属性的值作为字符串处理，为了更好理解，举个例子，编译器生成的渲染函数其实是字符串形式的渲染函数，该字符串要通过 `new Function(str)` 之后才能变成真正的函数，对比如下代码：

```javascript
// 代码一
const fn1 = new Function('console.log(1)')

// 代码二
const fn2 = new Function(JSON.stringify('console.log(1)'))
```

当执行 `fn1` 函数时，在控制台中会得到输出数字 `1`，而当执行 `fn2` 函数时则不会得到任何输出，实际上下面的代码和如上代码等价：

```javascript
// 代码一
const fn1 = function () {
  console.log(1)
}

// 代码二
const fn2 = function () {
  'console.log(1)'
}
```

实际上 `JSON.stringify('console.log(1)')` 的结果等价于 `"'console.log(1)'"`。

现在应该明白为什么对于非绑定的属性，需要使用 `JSON.stringify` 函数处理其属性值的原因，目的就是确保非绑定的属性值作为字符串处理，而不是变量或表达式。

讲完了非绑定属性值的获取及处理方式，回过头来看对于绑定的属性值应该如何处理，可以知道非绑定的属性值始终会被作为字符串对待，但是对于绑定的值则需要将其作为一个表达式对待才可以，如下代码所示：

```javascript
if (dynamicValue != null) {
  return parseFilters(dynamicValue)
} else if (getStatic !== false) {
  const staticValue = getAndRemoveAttr(el, name)
  if (staticValue != null) {
    return JSON.stringify(staticValue)
  }
}
```

可见，对于绑定的属性值需要通过 `parseFilters` 函数处理，并将处理后的值作为最终的返回结果。`parseFilters` 函数的作用就像它的名字一样，是用来解析过滤器的，换句话来说在编写绑定的属性可以使用过滤器，也许大家在平时开发中使用过滤器更多的场景是如下这种方式：

```html
<div>{{ date | format('yy-mm-dd') }}</div>
```

实际上对于绑定的属性值同样可以使用过滤器，如下：

```html
<div :key="id | featId"></div>
```

不过这只是从技术上讲，实际开发中更合适的方案是使用计算属性。总之对于绑定的属性值，为了让其拥有使用过滤器的能力，就需要使用 `parseFilters` 函数处理。`parseFilters`  函数来自于 `src/compiler/parser/filter-parser.js` 文件，它的作用简单来说就是将绑定的值分为两部分，一部分称之为表达式，另一部分是过滤器函数，然后将这两部分结合在一起，举个例子，如下代码所示：

```html
<div :key="id | featId"></div>
```

如上 `div` 标签拥有一个绑定的属性 `key`，它的值为 `id | featId`。对于这个值可以把它分为两部分：

- 第一部分，表达式：`id`
- 第二部分，过滤器：`featId`

现在假如给一个字符串 `'id | featId'`，如何将这个字符串分为上述的两部分，有人说可以以管道符 `|` 为分界，左边是表达式，右边就是过滤函数，接着看如下代码：

```html
<div :key="'id | featId'"></div>
```

如上代码和之前相比，不同的地方在于绑定属性 `key` 的值就是一个单纯的字符串，它没有过滤器，因为管道符是在单引号 `''` 之内的。不仅仅是单引号，以下代码中出现的管道符都不应该被作为过滤器的分界线：

```html
<div :key="'id | featId'"></div>  <!-- 单引号内的管道符 -->
<div :key='"id | featId"'></div>  <!-- 双引号内的管道符 -->
<div :key="`id | featId`"></div>  <!-- 模板字符串内的管道符 -->
```

这三种情况之外还有一种比较特殊的情况，就是正则表达式的管道符，如下：

```html
<div :key="/id|featId/.test(id).toString()"></div>  <!-- 正则表达式内的管道符 -->
```

以上代码中绑定属性 `key` 的属性值是一个表达式：`/id|featId/.test(id).toString()`，该表达式存在一个正则，可以知道正则表达式中管道符是由特殊用途的，所以在解析字符串 `'/id|featId/.test(id).toString()'` 时不能单纯的认为管道符为表达式与过滤器的而分界线。

还有一种最常见的情况，如下：

```html
<div :key="id || featId"></div>  <!-- 逻辑或运算符内的管道符 -->
```

如上代码所示，绑定属性 `key` 的属性值是一个表达式，该表达式里的 `||` 符号代表的是逻辑或运算符，而逻辑或运算符是由两个管道符 `|` 组成的，所以不能把这两个管道符中的任何一个作为过滤器的分界线。

实际上除了以上五种情况之外，管道符存在歧义的地方还有 **按位或** 运算符，它是位运算符中的一个运算符，该运算符就是由一个管道符组成的，所以它与过滤器的分界线完全一样，这时必须做出选择：既然希望管道符用来作为过滤器的分界线那就抛弃它按位或运算符的意义。有的人说，这不是得不到完全的语言能力了吗？实际上问题一点都不大，因为任何绑定属性的值理论上都可以通过计算属性实现，而不是直接将表达式写在属性值的位置。话虽这么说，但是还是需要做一些基本的处理，比如以上列出的五种管道符存在歧义的地方都是有能力处理的。

接下来思考如何判断一个管道符到底是不是表达式与过滤器的分界线，依据五种情况逐个分析，首先对于单引号的管道符：

```html
<div :key="'id | featId'"></div>  <!-- 单引号内的管道符 -->
```

思路是如果发现管道符存在于由两个单引号组成的字符串内，则认为其只是一个普通字符串而非过滤器的分界线。对于双引号 (`""`) 和模板字符串内的管道符也是同样的道理。所以问题的关键在于要能够识别单引号、双引号、模板字符串才行，这部分内容放到后边的 `parseFilters` 函数时再仔细讲解。

对于存在于正则表达式中的管道符，如下：

```html
<div :key="/id|featId/.test(id).toString()"></div>  <!-- 正则表达式内的管道符 -->
```

这种情况会比较复杂，因为要有能力识别出管道符是否存在于正则表达式中才行，难点就在于如何识别正则表达式，可以知道正则表达式由斜杠 (`/`) 开头，并以斜杠 (`/`) 结尾，但不要忘了斜杠在 JS 这门语言中还被用作除法运算符。所以归根结底难点在于需要识别一个斜杠它所代表的意义到底是除法还是正则。

实际上，这是一个相当复杂的事情，引用一个例子：

```javascript
a = b
/hi/g.exec(c).map(d)
```

思考一个问题，上方代码中第二行代码开头的斜杠 (`/`) 是除法运算符还是正则表达式的开头？答案是除法，因为如上代码等价于：

```javascript
a = b / hi / g.exec(c).map(d)
```

除此之外再来看一个例子：

```javascript
// 第一段代码
function f() {}
/1/g

// 第二段代码
var a = {}
/1/g
```

如上两段代码所示，这两段代码具有相同的特点，即第一行代码的最后一个字符为 `}`，第二句代码的第一个字符为 `/`。思考一下哪一段代码中的斜杠是除法运算符，哪一段代码中的斜杠是正则表达式的开头？实际上第一段代码中的斜杠是正则，因为该斜杠之前的语境是函数定义，而第二段代码中的斜杠是除法，因为该斜杠之前的语境为表达式并且花括号 `{}` 的意义为对象字面量。

实际上判断一个斜杠到底代表什么意义，应该综合考虑上下文语境，ECMA 规范中清楚的已经告诉大家需要多种标志符号类型 `goal symbols` 来综合判断，并且还要考虑 JS 这门语言的自动插入分号机制，以及其他可能产生歧义的地方。

如果要实现一个完整的能够精确识别斜杠意义的解析器需要花费大量的精力并且编写大量的代码，但对于 Vue 来讲，去实现一个完整的解析器是一个收入和回报完全不对等的事情。后面在分析 `parseFilters` 函数时可以看到，`parseFilters` 函数对于正则的处理仅仅考虑了很小的一部分，但对于 Vue 来讲已经足够了。还是那句话：**为什么一定要在绑定的表达式中写正则呢？用计算属性就可以了**。

以上就是对 `parseFilters` 函数的作用和一些基本实现思路的讲解，接下来就会具体到 `parseFilters` 函数中去，看看真正的实现和最终的效果。

打开 `src/compiler/parser/filter-parser.js` 文件找到 `parseFilters` 函数，如下时其函数的签名：

```javascript
export function parseFilters (exp: string): string {}
```

`parseFilters` 函数接收绑定的属性值作为参数，在 `parseFilters` 函数的开头定义了一些变量，先看如下这组变量：

```javascript
let inSingle = false
let inDouble = false
let inTemplateString = false
let inRegex = false
```

这里定义了四个变量，分别是 `inSingle`、`inDouble`、`inTemplateString` 以及 `inRegex`，并且它们的出事值都为 `false`。大家需要知道的是，大部分解析器在解析一段字符串的时候，都会将字符串当做一个字符流，从头到尾逐个字符读取。`parseFilters` 函数也不例外，`parseFilters` 函数会把接收到的字符串从头到尾逐个字符串读取，当读取到字符 `'` 并且该字符串前不是 `\` 时，则会将这个单引号字符作为字符串的开始，这时会把 `inSingle` 变量设置为 `true`，代表当前解析进入了单引号包裹的字符串。所以对于后续读取的任何字符来讲，由于 `inSingle` 变量为 `true`，所以这些字符都会被当做普通字符串的一部分来处理，直到解析器遇到了下一个能够代表字符串结束的单引号为止，此时会重新将 `inSingle` 变量的值设置为 `false`。

所以可以理解为 `inSingle` 变量的作用是用来标识当前读取的字符是否在单引号包裹的字符串中。同样的：

- `inDouble` 变量是用来标识当前读取字符是否在由 **双引号** 包裹的字符串中。
- `inTemplateString` 变量用来标识当前读取的字符是否在 **模板字符串** 中。
- `inRegex` 变量是用来标识当前读取的字符是否在 **正则表达式** 中。

接着再来看如下这三个变量：

```javascript
let curly = 0
let square = 0
let paren = 0
```

如上三个变量的初始值都为 `0`，其作用如下：

- 在解析绑定的属性值时，每遇到一个左花括号 `{`，则 `curly` 变量的值就会加一，每遇到一个右花括号 `}`，则 `curly` 变量的值就会减一。
- 在解析绑定的属性值时，每遇到一个左方括号 `[`，则 `square` 变量的值就会加一，每遇到一个右方括号 `]`，则 `square` 变量的值就会减一。
- 在解析绑定的属性值时，每遇到一个左圆括号 `(`，则 `paren` 变量的值就会加一，每遇到一个右圆括号 `)`，则 `paren` 变量的值就会减一。

当 `parseFilters` 函数在解析属性值字符串并遇到一个管道符时，该管道符应不应该作为过滤器的分界线还要看以上三个变量是否为 `0`，如果以上三个变量至少有一个不为 `0`，则说明该管道符存在于花括号或方括号或圆括号之内，这时该管道符是不会作为过滤器的分界线的，如下：

```html
<div :key="(aa | bb)"></div>
```

以上代码中绑定属性 `key` 的属性值中包含一个管道符，但是由于该管道符存在于圆括号内，所以它不会被作为过滤器的分界线。

再往下定义了如下这些变量：

```javascript
let lastFilterIndex = 0
let c, prev, i, expression, filters
```

这里简单介绍一下这些变量的作用，更具体的会在源码中讲解。`lastFilterIndex` 变量的初始值为 `0`，它的值是属性值字符串中字符的索引，将会被用来确定过滤器的位置。变量 `c` 为当前字符对应的 `ASCII` 码，我们知道在解析属性值时会以字符流的方式逐个字符读入，而变量 `c` 就是当前读入字符所对应的 `ASCII` 码。变量 `prev` 保存的则是当前字符的前一个字符所对应的 `ASCII` 码。变量 `i` 为当前读入字符的位置索引。变量 `expression` 将是 `parseFilters` 函数的返回值。变量 `filters` 将来会是一个数组，它保存着所有过滤器函数名。

再往下将进入一个 `for` 循环：

```javascript
for (i = 0; i < exp.length; i++) {
  // 省略...
}
```

这个 `for` 循环是整个 `parseFilters` 函数的核心，它的作用就是将属性值字符串作为字符读入，从第一个字符开始一直读到字符串的末尾，在 `for` 循环的开头执行的是如下两行代码：

```javascript
for (i = 0; i < exp.length; i++) {
  prev = c
  c = exp.charCodeAt(i)
  // 省略...
}
```

可以看到每次循环的开始，都会将上一次读取的字符所对应的 `ASCII` 码赋值给 `prev` 变量，然后再将变量 `c` 的值设置为当前读取字符所对应的 `ASCII` 码。所以说 `prev` 变量中保存的是上一个字符的 `ASCII` 码。

在这两行代码的下面是一连串的 `if...elseif...else` 语句，如下：

```javascript
for (i = 0; i < exp.length; i++) {
  prev = c
  c = exp.charCodeAt(i)
  if (inSingle) {
    // 如果当前读取的字符存在于由单引号包裹的字符串内，则会执行这里的代码
  } else if (inDouble) {
    // 如果当前读取的字符存在于由双引号包裹的字符串内，则会执行这里的代码
  } else if (inTemplateString) {
    // 如果当前读取的字符存在于模板字符串内，则会执行这里的代码
  } else if (inRegex) {
    // 如果当前读取的字符存在于正则表达式内，则会执行这里的代码
  } else if (
    c === 0x7C && // pipe
    exp.charCodeAt(i + 1) !== 0x7C &&
    exp.charCodeAt(i - 1) !== 0x7C &&
    !curly && !square && !paren
  ) {
    // 如果当前读取的字符是过滤器的分界线，则会执行这里的代码
  } else {
    // 当不满足以上条件时，执行这里的代码
  }
}
```

首先来看第一段 `if` 条件语句的判断：

```javascript
if (inSingle) {
  if (c === 0x27 && prev !== 0x5C) inSingle = false
}
```

该条件检测了 `inSingle` 变量是否为 `true`，如果为 `true` 则说明当前读入的字符存在于由单引号包裹的字符串内，此时会执行 `if` 语句块内的代码，可以看到在 `if` 条件语句块内的同样是一个 `if` 判断语句，它的判断条件为：

```javascript
c === 0x27 && prev !== 0x5C
```

可以看到如上判断条件中有两个十六进制的数字：`0x27` 和 `0x5C`，这两个十六进制的数字实际上就是字符的 `ASCII` 码，其中 `0x27` 为字符单引号 (`'`) 所对应的 `ASCII` 码，而 `0x5C` 则是字符反斜杠 (`\`) 所对应的 `ASCII` 码。所以如上判断条件翻译过来就是：当前字符是单引号 (`'`)，并且当前字符的前一个字符不是反斜杠 (`\`)，也就是说当前字符 (`单引号`) 就是字符串的结束。该判断条件的关键在于不仅要当前字符串时单引号 (`'`)，同时前一个字符也一定不能够是反斜杠才行，这时因为反斜杠在字符串内具有转义的作用，如果判断条件成立，则将 `inSingle` 变量的值设置为 `false`，代表接下来的解析工作已经不处于由单引号所包裹的字符串环境中了。

再来看下一个 `else if` 判断分支：

```javascript
else if (inDouble) {
  if (c === 0x22 && prev !== 0x5C) inDouble = false
}
```

和单引号的情况类似，该 `else if` 条件语句检查了变量 `inDouble` 是否为 `true`，如果为 `true` 则说明当前字符处于由双引号包裹的字符串中，此时会检查当前字符所对应的 `ASCII` 码是否等于 `0x22`，这里的数字 `0x22` 就是字符双引号 (`"`) 所对应的 `ASCII` 码。所以如上判断语句则等价于：当前字符是双引号，并且前一个字符不是转义字符 (`\`)。这说明当前字符 (`双引号`) 就应该是字符串的结束，此时会将变量 `inDouble` 的值设置为 `false`，代表接下来的解析工作已经不处于由双引号所包裹的字符串环境中了。

再接着就是如下判断分支，它同时是一个 `else if` 语句块：

```javascript
else if (inTemplateString) {
  if (c === 0x60 && prev !== 0x5C) inTemplateString = false
}
```

这个判断语句和前两个判断语句类似，如果该 `else if` 语句的条件成立，则说明当前字符处于模板字符串中，此时会继续检查当前字符所对应的 `ASCII` 码是否等于 `0x60`，这里的数字 `0x60` 就是模板引用字符的 `ASCII` 码，所以如上判断语句成立则等价于：当前字符是 \`，并且前一个字符不是转义字符 `\`。这说明当前字符 \` 就应该是模板字符串的结束，此时会将变量 `inTemplateString` 的值设置为 `false`，代表接下来的解析工作已经不处于模板字符串环境中了。

再来看下一个 `else if` 条件语句块：

```javascript
else if (inRegex) {
  if (c === 0x2f && prev !== 0x5C) inRegex = false
}
```

如果该 `else if` 语句的条件成立，则说明当前字符处于正则表达式中，此时会继续检测当前字符所对应的 `ASCII` 码是否等于 `0x2f`，这里的数字 `0x2f` 就是字符 `/` 所对应的 `ASCII` 码。所以如上判断语句成立则等价于：当前字符是 `/`，并且前一个字符不是转义字符 `\`，这说明当前字符 `/` 就应该是正则表达式的结束，此时会将变量 `inRegex` 的值设置为 `false`，代表接下来的解析工作已经不处于正则表达式的环境中了。

再往下的一个 `else if` 条件语句的判断条件稍微复杂一些，如下：

```javascript
else if (
  c === 0x7C && // pipe
  exp.charCodeAt(i + 1) !== 0x7C &&
  exp.charCodeAt(i - 1) !== 0x7C &&
  !curly && !square && !paren
)
```

如上判断条件中的数字 `0x7C` 为管道符 (`|`) 所对应的 `ASCII` 码，如果以上条件成立，则说明当前字符为管道符，实际上这个判断条件是用来检测当前遇到的管道符是否是过滤器的分界线。如果一个管道符是过滤器的分界线则必须满足以上条件，即：

- 当前字符所对应的 `ASCII` 码必须是 `0x7C`，即当前字符必须是管道符
- 该字符的后一个字符不能是管道符
- 该字符的前一个字符不能是管道符
- 该字符不能处于花括号、方括号、圆括号之内

如果一个字符满足以上条件，则说明该字符就是用来作为过滤器分界线的管道符。此时该 `else if` 语句块的代码将被执行，不过暂时跳过，来看最后一个 `else` 语句。

当以上所有判断分支全部失效之后，代码会来到 `else` 分支，假设有如下代码：

```html
<div :key="'id'"></div>
```

此时传递给 `parseFilters` 函数的字符串应该是 `'id'`，该字符串有四个字符，第一个字符为单引号，我们尝试按照 `parseFilters` 函数的执行过程多该字符串进行解析。首先读取该字符串的第一个字符，即单引号 `'`，接着会判断 `inSingle` 变量是否为 `true`，由于 `inSingle` 变量的初始值为 `false`，所以会继续判断下一个条件分支，同样地由于 `inDouble`、`inTemplateString`、`inRegex` 等变量的初始值都为 `false`，并且该字符是单引号而不是管道符，所以接下来的任何一个 `else if` 分支语句块内的代码都不会执行。所以最终 `else` 语句块内的代码将被执行。

在 `else` 语句块内，首先执行的是一段 `switch` 语句，如下：

```javascript
switch (c) {
  case 0x22: inDouble = true; break         // "
  case 0x27: inSingle = true; break         // '
  case 0x60: inTemplateString = true; break // `
  case 0x28: paren++; break                 // (
  case 0x29: paren--; break                 // )
  case 0x5B: square++; break                // [
  case 0x5D: square--; break                // ]
  case 0x7B: curly++; break                 // {
  case 0x7D: curly--; break                 // }
}
```

这段 `switch` 语句的作用总结如下：

- 如果当前字符为双引号 (`"`)，则将 `inDouble` 变量的值设置为 `true`
- 如果当前字符为单引号 (`'`)，则将 `inSingle` 变量的值设置为 `true`
- 如果当前字符为模板字符串的定义字符 \`，则将 `inTemplateString` 变量的值设置为 `true`
- 如果当前字符是左圆括号 (`(`)，则将 `paren` 变量的值加一
- 如果当前字符是右圆括号 (`)`)，则将 `paren` 变量的值减一
- 如果当前字符是左方括号 (`[`)，则将 `square` 变量的值加一
- 如果当前字符是右方括号 (`]`)，则将 `square` 变量的值减一
- 如果当前字符是左花括号 (`{`)，则将 `curly` 变量的值加一
- 如果当前字符是右花括号 (`}`)，则将 `curly` 变量的值减一

假设还是解析字符串 `'id'`，该字符串的第一个字符为单引号，可以知道当解析字符串的第一个字符时会执行 `else` 语句块内的代码，所以如上 `switch` 语句将被执行，并且 `inSingle` 变量的值被设置为 `true`。接着会解析第二个字符 `i`，由于此时 `inSingle` 变量的值已经为真，所以如下代码将被执行：

```javascript
if (inSingle) {
  if (c === 0x27 && prev !== 0x5C) inSingle = false
}
```

显然字符 `i` 所对应的 `ASCII` 码不等于 `0x27`，所以这里直接跳过解析下一个字符。下一个字符是 `d`，它的情况和字符 `i` 一样，也会被跳过。直到遇到最后一个字符 `'`，该字符同样是单引号，所以此时会将 `inSingle` 变量的值设置为 `false`，意味着由单引号包裹着的字符串结束了。所以通过以上的分析可以知道一件事情，即只要存在于单引号包裹的字符串内的字符都被跳过。这样做的目的就是为了避免误把存在于字符串中的管道符当做过滤器的分界线，如下代码所示：

```html
<div :key="'id|featId'"></div>
```

可看到绑定属性 `key` 的属性值为 `'id|featId'`，由于管道符 `|` 存在于由单引号所包裹的字符串内，所以该管道符不会作为过滤器的分界线，这时非常合理的。

同样的道理，对于存在于双引号包裹的字符串中或模板字符串中或正则表达式中的管道符，也不会作为过滤器的分界线。对于双引号和模板字符串的判断是很容易的，它们的原理和单引号类似。难点在于如何判断正则，或者换句话说应该在什么情况下才能将 `inRegex` 变量的值设置为 `true`。如下是 `else` 语句块内用来判断是否即将进入正则环境的代码：

```javascript
if (c === 0x2f) { // /
  let j = i - 1
  let p
  // find first non-whitespace prev char
  for (; j >= 0; j--) {
    p = exp.charAt(j)
    if (p !== ' ') break
  }
  if (!p || !validDivisionCharRE.test(p)) {
    inRegex = true
  }
}
```

如上代码是一个 `if` 判断语句，用来判断当前字符所对应的 `ASCII` 码是否等于数字 `0x2f`，其中数字 `0x2f` 就是字符 `/` 所对应的 `ASCII` 码。可以知道正则表达式就是以字符 `/` 开头的，所以当遇到字符 `/` 时，则说明该字符有可能就是正则的开始。但是至于到底是不是正则的开始还真不一定，前面已经提到过了，字符 `/` 还有除法的意义。而判断字符 `/` 到底是正则的开始还是除法却是一件不容易的事情。实际上如上代码根本不足以保证所遇到的字符 `/` 就是正则表达式，但还是那句话，这对于 Vue 而言已经足够了，没有必要花大力气在收益很小的地方。

来看如上代码是如何确定字符 `/` 正则的开始的，首先要明确如果上方这段 `if` 条件语句成立，则说明当前字符为 `/`，此时 `if` 语句块内的代码将被执行，在 `if` 语句块内定义了变量 `j`，它的值为 `i - 1`，也就是说变量 `j` 是 `/` 字符的前一个字符的索引。然后又定义了变量 `p`，接着开启一个 `for` 循环，这个 `for` 循环的作用是找到 `/` 字符之前第一个不为空的字符。如果没找到则说明字符 `/` 之前的所有字符都是空格，或根本没有字符，如下：

```html
<div :key="/a/.test('abc')"></div>      <!-- 第一个 `/` 之前就没有字符  -->
<div :key="    /a/.test('abc')"></div>  <!-- 第一个 `/` 之前都是空格  -->
```

所以以上两种情况，第一个 `/` 都应该是正则的开始，而非除法。

但是假如字符 `/` 之前有非空的字符，则只有在该字符不满足正则 `validDivisionCharRE` 的情况下，才会认为字符 `/` 为正则的开始。`validDivisionCharRE` 正则常量定义在 `parseFilters` 函数的前面，如下：

```javascript
const validDivisionCharRE = /[\w).+\-_$\]]/
```

该正则用来匹配一个字符，这个字符应该是字母、数字、`)`、`.`、`+`、`-`、`_`、`$`、`]` 之一。再看如下代码：

```javascript
if (c === 0x2f) { // /
  let j = i - 1
  let p
  // find first non-whitespace prev char
  for (; j >= 0; j--) {
    p = exp.charAt(j)
    if (p !== ' ') break
  }
  if (!p || !validDivisionCharRE.test(p)) {
    inRegex = true
  }
}
```

可以看到如果条件 `!validDivisionCharRE.test(p)` 成立则也会认为当前字符 `/` 是正则的开始。条件 `!validDivisionCharRE.test(p)` 成立说明字符 `/` 之前的字符不能是正则 `validDivisionCharRE` 所匹配的任何一个字符，否则当前字符 `/` 就不被认为是正则的开始。

以上是 Vue 的做法，但之前已经说过了，这不足以对字符 `/` 的意义做出精准的判断，但是对 Vue 而言够了，其实很容易可以找到反例，如下：

```html
<div :key="a + /a/.test('abc')"></div>
```

实际上在表达式 `a + /a/.test('abc')` 中出现的斜杠 (`/`) 的确是定义了正则，但 Vue 却不认为它是正则，因为第一个斜杠之前的第一个不为空的字符为加号 `+`。加号存在于正则 `validDivisionCharRE` 中，所以 Vue 不认为这里的斜杠是正则的定义。但实际上如上代码简直就是没有任何意义的，假如非得这么写，那也可以完全使用计算属性替代。

了解了这些，可以发现 `else` 语句块内的代码就是用来检查环境的，这些环境指的是字符串环境或正则环境，或圆括号、方括号以及花括号等环境，这些环境信息将会被用到其他判断分支的条件语句。

接着看一下之前没有讲解的一段 `else if` 语句块，如下：

```javascript
else if (
  c === 0x7C && // pipe
  exp.charCodeAt(i + 1) !== 0x7C &&
  exp.charCodeAt(i - 1) !== 0x7C &&
  !curly && !square && !paren
) {
  if (expression === undefined) {
    // first filter, end of expression
    lastFilterIndex = i + 1
    expression = exp.slice(0, i).trim()
  } else {
    pushFilter()
  }
}
```

在本节的前边，已经讲解过了该条件语句块的判断条件，如果以上条件成立，则说明当前字符为管道符，并且该管道符就是过滤器的分界线。接着来看 `else if` 语句块内的代码，首先判断了 `expression` 变量是否存在，可以知道 `expression` 变量的初始值为 `undefined`，所以当程序在解析字符串时第一次遇到作为过滤器分界线的管道符时，将会执行如下：

```javascript
if (expression === undefined) {
  // first filter, end of expression
  lastFilterIndex = i + 1
  expression = exp.slice(0, i).trim()
} else {
  // 省略...
}
```

如上两行代码所示，首先将变量 `lastFilterIndex` 的值设置为 `i + 1`，变量 `i` 就是当前遇到的管道符的位置索引，所以 `i + 1` 就应该是管道符下一个字符的位置索引，所以可以把 `lastFilterIndex` 变量理解为过滤器的开始。接着对字符串 `exp` 进行截取，其截取的位置恰好是索引为 `i` 的字符，也就是管道符，当然了截取后生成的新字符串是不包含管道符的，同时对截取后生成的新字符串使用 `trim` 方法去除前后空格，最后将处理后的结果赋值给 `expression` 表达式。

为了更直观理解 `lastFilterIndex` 变量和 `expression` 变量，举个例子，假设有如下代码：

```html
<div :key="id | featId"></div>
```

对于字符串 `'id | featId'` 来讲，其中的管道符是过滤器的分界线，其位置索引为 `3`，所以 `lastFilterIndex` 的值应该是管道符后一个字符的位置索引 `4`。此时 `expression` 变量的值就应该是 `exp.slice(0, 3).trim`，所以 `expression` 的值就应该是字符串 `'id'`，这样就把表达式提取了出来了，并使用 `expression` 变量保存。

此时对于管道符的解析工作就结束了，`for` 循环开始解析下一个字符，知道所有字符解析完毕。当 `for` 循环结束时，变量 `i` 的值应该是字符串的长度。同时 `expression` 变量中保存着过滤器分界线之前的字符串，也就是表达式。接下来看 `for` 循环之后的这段代码，如下：

```javascript
if (expression === undefined) {
  expression = exp.slice(0, i).trim()
} else if (lastFilterIndex !== 0) {
  pushFilter()
}
```

在 `for` 循环结束之后将会执行如上代码，这段代码由 `if...else if` 条件语句块组成，首先检查 `expression` 是否存在，还是拿如下这个例子来说：

```html
<div :key="id | featId"></div>
```

可以知道在解析字符串 `'id | featId'` 之后，`expression` 的值应该是字符串 `'id'`，所以 `if` 条件语句块的内容不会被执行，此时会进入 `else if` 条件语句的判断，即：`lastFilterIndex !== 0`，还是拿上例来说，此时 `lastFilterIndex` 变量的值应该是作为过滤器分界线的管道符后一个字符的位置索引，所以 `lastFilterIndex` 变量的值为 `4`，由于它不等于 `0`，所以 `else if` 语句块内的代码将被执行，可以看到在 `else if` 语句块内直接调用了 `pushFilter` 函数。该函数的源码如下：

```javascript
function pushFilter () {
  (filters || (filters = [])).push(exp.slice(lastFilterIndex, i).trim())
  lastFilterIndex = i + 1
}
```

首先检查变量 `filters` 是否存在，如果不存在则将其初始化为空数组，接着使用 `slice` 方法对字符串 `exp` 进行截取，截取的开始和结束位置恰好是 `lastFilterIndex` 和 `i`。还是拿之前的例子来说，`lastFilterIndex` 指向的是管道符后面的空格，这里大家需要注意的是变量 `i` 指向的既不是字符 `d` 也不是引号 `"`，而是字符 `d` 后面的字符，这个字符是不存在的。知道了这些，就可以知道如下表达式的值：

```javascript
exp.slice(lastFilterIndex, i).trim()
```

该表达式的值就应该是字符串 `'featId'`，而这个字符串就代表着过滤器函数的名字，它将被添加到数组 `filters` 中。不过可以看到 `pushFilter` 函数的第二行代码：`lastFilterIndex = i + 1`，这里又将 `lastFilterIndex` 变量的值设置为 `i + 1`，实际上之前所举的例子不足以体现这行代码的作用，接着看如下的例子：

```html
<div :key="id | featId | featId2"></div>
```

如上代码所示，不仅仅拥有一个过滤器，而是有两个过滤器，分别是 `featId` 和 `featId2`。当 `parseFilters` 函数在解析字符串 `'id | featId | featId2'` 时，会遇到两个被作为过滤器分界线的管道符，再来看如下代码：

```javascript
else if (
  c === 0x7C && // pipe
  exp.charCodeAt(i + 1) !== 0x7C &&
  exp.charCodeAt(i - 1) !== 0x7C &&
  !curly && !square && !paren
) {
  if (expression === undefined) {
    // first filter, end of expression
    lastFilterIndex = i + 1
    expression = exp.slice(0, i).trim()
  } else {
    pushFilter()
  }
}
```

当遇到第一个管道符时 `lastFilterIndex` 变量是第一个管道符后一个字符的索引。当遇到第二个管道符时由于此时变量 `expression` 已经保存了表达式字符串 `'id'`，所以将会执行 `else` 分支的代码，即调用 `pushFilter` 函数，要知道此时变量 `i` 已经是第二个管道符的位置索引了。再来看 `pushFilter` 函数：

```javascript
function pushFilter () {
  (filters || (filters = [])).push(exp.slice(lastFilterIndex, i).trim())
  lastFilterIndex = i + 1
}
```

在 `pushFilter` 函数内会先将字符串 `'featId'` 添加到数组，接着设置 `lastFilterIndex` 变量的值为 `i + 1`，由于此时变量 `i` 已经是第二个管道符的位置索引，所以此时 `i + 1` 就应该是第二个管道符后一个字符串的位置索引。

接着解析工作会继续进行，知道解析结束，当解析结束时变量 `i` 的值就应该是字符串的长度。此时 `for` 循环也将结束，会继续执行 `for` 循环之后的代码，如下：

```javascript
if (expression === undefined) {
  expression = exp.slice(0, i).trim()
} else if (lastFilterIndex !== 0) {
  pushFilter()
}
```

此时代码依然会执行 `else if` 分支，再次调用 `pushFilter` 函数，可以知道到目前为止我们只将字符串 `'featId'` 添加到了 `filters` 数组中，但此时是拥有两个过滤器，所以还需要将字符串 `'featId2'` 也添加到 `filters` 数组才行，所以这里需要再次执行 `pushFilter` 函数，不过不同的是，此时在 `pushFilter` 函数中，`lastFilterIndex` 变量已经指向了第二个管道符的后一个字符，而变量 `i` 的值也变成了字符串的长度，所以此时被添加到 `filters` 数组的字符串将会是 `'featId2'`。这样两个过滤器的名字就都被添加到了 `filters` 数组了。

经过以上代码的处理，对我们来说最重要的两个变量分别是 `expression` 和 `filters`，前者保存着表达式，后者则保存着所有过滤器的名字，假设有如下代码：

```html
<div :key="id | a | b | c"></div>
```

那么经过解析，变量 `expression` 的值将是字符串 `'id'`，且 `filters` 数组中将包含三个元素：`['a', 'b', 'c']`。

有了这些基础，代码将来到最关键的一步，即如下代码：

```javascript
if (filters) {
  for (i = 0; i < filters.length; i++) {
    expression = wrapFilter(expression, filters[i])
  }
}
```

