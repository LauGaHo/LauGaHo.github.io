# Vue源码解读—patch

## `patch` 入口

`/src/core/instance/lifecycle.js`

```javascript
const updateComponent = () => {
  // 执行 vm._render() 函数，得到 VNode，并将 VNode 传递给 _update 方法，接下来就该到 patch 阶段了
  vm._update(vm._render(), hydrating)
}
```

此处的 `vm._render()` 函数是根据组件模板，将其转换成对应的 `VNode`



## `vm._update()`

`/src/core/instance/lifecycle.js`

```javascript
/**
 * 页面首次渲染和后续更新的入口位置，也是 patch 的入口位置 
 */
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  // 页面的挂载点，真实的元素
  const prevEl = vm.$el
  // 老 VNode
  const prevVnode = vm._vnode
  const restoreActiveInstance = setActiveInstance(vm)
  // 新 VNode
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // 老 VNode 不存在，表示首次渲染，即初始化页面时走这里
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // 响应式数据更新时，即更新页面时走这里
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  restoreActiveInstance()
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```



## `vm.__patch__()`

`/src/platforms/web/runtime/index.js`

```javascript
// 在 Vue 原型链上安装 web 平台的 patch 函数
Vue.prototype.__patch__ = inBrowser ? patch : noop
```



## `patch()`

`/src/platforms/web/runtime/patch.js`

```javascript
// patch 工厂函数，为其传入平台特有的一些操作，然后返回一个 patch 函数
export const patch: Function = createPatchFunction({ nodeOps, module })
```

**注意：此处的 `patch` 函数对应的就是实现多端的关键，因为我们可以传入不同平台的操作，从而得到不同平台对应的 `patch` 函数**



## `nodeOps()`

`/src/platforms/web/runtime/node-ops.js`

```javascript
/**
 * web 平台的 DOM 操作 API
 */

// 创建标签名为 tagName 的元素节点
export function createElement (tagName: string, vnode: VNode): Element {
  // 创建元素节点
  const elm = document.createElement(tagName)
  if (tagName !== 'select') {
    return elm
  }
  // false or null will remove the attribute but undefined will not
  // 如果是 select 元素，则为它设置 multiple 属性
  if (vnode.data && vnode.data.attrs && vnode.data.attrs.multiple !== undefined) {
    elm.setAttribute('multiple', 'multiple')
  }
  return elm
}

// 创建带命名空间的元素节点
export function createElementNS (namespace: string, tagName: string): Element {
  return document.createElementNS(namespaceMap[namespace], tagName)
}

// 创建文本节点
export function createTextNode (text: string): Text {
  return document.createTextNode(text)
}

// 创建注释节点
export function createComment (text: string): Comment {
  return document.createComment(text)
}

// 在指定节点前插入节点
export function insertBefore (parentNode: Node, newNode: Node, referenceNode: Node) {
  parentNode.insertBefore(newNode, referenceNode)
}

// 移除指定子节点
export function removeChild (node: Node, child: Node) {
  node.removeChild(child)
}

// 添加子节点
export function appendChild (node: Node, child: Node) {
  node.appendChild(child)
}

// 返回指定节点的父节点
export function parentNode (node: Node): ?Node {
  return node.parentNode
}

// 返回指定节点的下一个兄弟节点
export function nextSibling (node: Node): ?Node {
  return node.nextSibling
}

// 返回指定节点的标签名 
export function tagName (node: Element): string {
  return node.tagName
}

// 为指定节点设置文本 
export function setTextContent (node: Node, text: string) {
  node.textContent = text
}

// 为节点设置指定的 scopeId 属性，属性值为 ''
export function setStyleScope (node: Element, scopeId: string) {
  node.setAttribute(scopeId, '')
}
```



## `module`

`/src/platforms/web/runtime/modules` 和 `/src/core/vdom/modules`

**这里具体指的是平台一些特有的操作，比如 attr、class、style、event 等，还有核心的 directive 和 ref，他们会向外暴露一些特有的方法，比如：create、active、update、remove、destroy，这些方法在 patch 阶段会被调用，从而做相应的操作，比如创建 attr、指令等。**



## `createPatchFunction()`

`/src/core/vdom/patch.js`

```javascript
const hooks = ['create', 'activate', 'update', 'remove', 'destroy']

/**
 * 工厂函数，注入平台特有的一些功能操作，并定义一些方法，然后返回 patch 函数
 */
export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  /**
   * modules: { ref, directives, 平台特有的一些操纵，比如 attr、class、style 等 }
   * nodeOps: { 对元素的增删改查 API }
   */
  const { modules, nodeOps } = backend

  /**
   * hooks = ['create', 'activate', 'update', 'remove', 'destroy']
   * 遍历这些钩子，然后从 modules 的各个模块中找到相应的方法，比如：directives 中的 create、update、destroy 方法
   * 让这些方法放到 cb[hook] = [hook 方法] 中，比如: cb.create = [fn1, fn2, ...]
   * 然后在合适的时间调用相应的钩子方法完成对应的操作
   */
  for (i = 0; i < hooks.length; ++i) {
    // 比如 cbs.create = []
    cbs[hooks[i]] = []
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        // 遍历各个 modules，找出各个 module 中的 create 方法，然后添加到 cbs.create 数组中
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  /**
   * vm.__patch__
   *   1、新节点不存在，老节点存在，调用 destroy，销毁老节点
   *   2、如果 oldVnode 是真实元素，则表示首次渲染，创建新节点，并插入 body，然后移除老节点
   *   3、如果 oldVnode 不是真实元素，则表示更新阶段，执行 patchVnode
   */
  return patch
}
```



## `patch()`

`/src/core/vdom/patch.js`

```javascript
/**
 * vm.__patch__
 *   1、新节点不存在，老节点存在，调用 destroy，销毁老节点
 *   2、如果 oldVnode 是真实元素，则表示首次渲染，创建新节点，并插入 body，然后移除老节点
 *   3、如果 oldVnode 不是真实元素，则表示更新阶段，执行 patchVnode
 */
function patch(oldVnode, vnode, hydrating, removeOnly) {
  // 如果新节点不存在，老节点存在，则调用 destroy，销毁老节点
  if (isUndef(vnode)) {
    if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
    return
  }

  let isInitialPatch = false
  const insertedVnodeQueue = []

  if (isUndef(oldVnode)) {
    // 新的 VNode 存在，老的 VNode 不存在，这种情况会在一个组件初次渲染的时候出现，比如：
    // <div id="app"><comp></comp></div>
    // 这里的 comp 组件初次渲染时就会走这儿
    // empty mount (likely as component), create new root element
    isInitialPatch = true
    createElm(vnode, insertedVnodeQueue)
  } else {
    // 判断 oldVnode 是否为真实元素
    const isRealElement = isDef(oldVnode.nodeType)
    if (!isRealElement && sameVnode(oldVnode, vnode)) {
      // 不是真实元素，但是老节点和新节点是同一个节点，则是更新阶段，执行 patch 更新节点
      patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
    } else {
      // 是真实元素，则表示初次渲染
      if (isRealElement) {
        // 挂载到真实元素以及处理服务端渲染的情况
        // mounting to a real element
        // check if this is server-rendered content and if we can perform
        // a successful hydration.
        if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
          oldVnode.removeAttribute(SSR_ATTR)
          hydrating = true
        }
        if (isTrue(hydrating)) {
          if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
            invokeInsertHook(vnode, insertedVnodeQueue, true)
            return oldVnode
          } else if (process.env.NODE_ENV !== 'production') {
            warn(
              'The client-side rendered virtual DOM tree is not matching ' +
              'server-rendered content. This is likely caused by incorrect ' +
              'HTML markup, for example nesting block-level elements inside ' +
              '<p>, or missing <tbody>. Bailing hydration and performing ' +
              'full client-side render.'
            )
          }
        }
        // 走到这儿说明不是服务端渲染，或者 hydration 失败，则根据 oldVnode 创建一个 vnode 节点
        // either not server-rendered, or hydration failed.
        // create an empty node and replace it
        oldVnode = emptyNodeAt(oldVnode)
      }

      // 拿到老节点的真实元素
      const oldElm = oldVnode.elm
      // 获取老节点的父元素，即 body
      const parentElm = nodeOps.parentNode(oldElm)

      // 基于新 vnode 创建整棵 DOM 树并插入到 body 元素下
      createElm(
        vnode,
        insertedVnodeQueue,
        // extremely rare edge case: do not insert if old element is in a
        // leaving transition. Only happens when combining transition +
        // keep-alive + HOCs. (#4590)
        oldElm._leaveCb ? null : parentElm,
        nodeOps.nextSibling(oldElm)
      )

      // 递归更新父占位符节点元素
      if (isDef(vnode.parent)) {
        let ancestor = vnode.parent
        const patchable = isPatchable(vnode)
        while (ancestor) {
          for (let i = 0; i < cbs.destroy.length; ++i) {
            cbs.destroy[i](ancestor)
          }
          ancestor.elm = vnode.elm
          if (patchable) {
            for (let i = 0; i < cbs.create.length; ++i) {
              cbs.create[i](emptyNode, ancestor)
            }
            // #6513
            // invoke insert hooks that may have been merged by create hooks.
            // e.g. for directives that uses the "inserted" hook.
            const insert = ancestor.data.hook.insert
            if (insert.merged) {
              // start at index 1 to avoid re-invoking component mounted hook
              for (let i = 1; i < insert.fns.length; i++) {
                insert.fns[i]()
              }
            }
          } else {
            registerRef(ancestor)
          }
          ancestor = ancestor.parent
        }
      }

      // 移除老节点
      if (isDef(parentElm)) {
        removeVnodes([oldVnode], 0, 0)
      } else if (isDef(oldVnode.tag)) {
        invokeDestroyHook(oldVnode)
      }
    }
  }

  invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
  return vnode.elm
}
```

**注意：上述代码，通过 `if else` 分了好几种情况，如下：**

- `isUndef(vnode) && isDef(oldVNode)`：移除旧 DOM，然后直接返回
- `isDef(vnode) && isUnDef(oldVNode)`：这种情况对应是创建新组件的情况，例如：`<div><com></com></div>`，渲染 `<com><com/>` 组件的时候，走的就是这一个分支，因为渲染组件之前，对应的位置之前是不存在任何结构的。
- `isDef(vnode) && isDef(oldVNode) && !isDef(oldVNode.nodeType) && sameVNode(vnode, oldVNode)`：这种情况就是对应组件每次更新，进行旧 vnode 和新 vnode 比对的过程
- `isDef(vnode) && isDef(oldVNode) && isDef(oldVNode.nodeType)`：这种情况就是对应初始化 App 组件的时候，传入的 `oldVNode` 是一个 `Element` 类型的情况，反正就是初始化的情况，他需要为传进来的 `Element` 类型的 `oldVNode` 创建一个 `VNode` 类型的数据替代原本的 `oldVNode`，并且将原本的 `Element` 类型的数据挂载到新创建的 `VNode` 类型的 `oldVNode` 上

