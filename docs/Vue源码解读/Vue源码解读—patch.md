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
- `isDef(vnode) && isDef(oldVNode) && isDef(oldVNode.nodeType)`：这种情况就是对应初始化 App 组件 (根组件) 的时候，传入的 `oldVNode` 是一个 `Element` 类型的情况，反正就是初始化的情况，他需要为传进来的 `Element` 类型的 `oldVNode` 创建一个 `VNode` 类型的数据替代原本的 `oldVNode`，并且将原本的 `Element` 类型的数据挂载到新创建的 `VNode` 类型的 `oldVNode` 上



## `sameVnode()`

`/src/core/vdom/patch.js`

```javascript
/**
 * 判读两个节点是否相同 
 */
function sameVnode (a, b) {
  return (
    // key 必须相同，需要注意的是 undefined === undefined => true
    a.key === b.key && (
      (
        // 标签相同
        a.tag === b.tag &&
        // 都是注释节点
        a.isComment === b.isComment &&
        // 都有 data 属性
        isDef(a.data) === isDef(b.data) &&
        // input 标签的情况
        sameInputType(a, b)
      ) || (
        // 异步占位符节点
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

- **注意：这里需要注意，关于 key 的比较，`undefined === undefined` 返回的是 true ！！！**



## `emptyNodeAt()`

`/src/core/vdom/patch.js`

```javascript
/**
 * 为元素(elm)创建一个空的 vnode
 */
function emptyNodeAt(elm) {
  return new VNode(nodeOps.tagName(elm).toLowerCase(), {}, [], undefined, elm)
}
```

- **由于第一次初始化 Vue 的时候，执行 `patch()` 函数的时候，函数中的 `oldVNode` 形参对应的类型是 `Element`，并不是 `VNode` 类型，所以此时需要为 `Element` 生成一个 `VNode` 类型的对象，并把 `Element` 赋值给 `VNode` 中的 `elm` 属性中。**



## `createElm()`

`/src/core/vdom/patch.js`

```javascript
/**
 * 基于 vnode 创建整棵 DOM 树，并插入到父节点上
 */
function createElm(
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // This vnode was used in a previous render!
    // now it's used as a new node, overwriting its elm would cause
    // potential patch errors down the road when it's used as an insertion
    // reference node. Instead, we clone the node on-demand before creating
    // associated DOM element for it.
    vnode = ownerArray[index] = cloneVNode(vnode)
  }

  vnode.isRootInsert = !nested // for transition enter check
  /**
   * 重点
   * 1、如果 vnode 是一个组件，则执行 init 钩子，创建组件实例并挂载，
   *   然后为组件执行各个模块的 create 钩子
   *   如果组件被 keep-alive 包裹，则激活组件
   * 2、如果是一个普通元素，则什么也不错
   */
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }

  // 获取 data 对象
  const data = vnode.data
  // 所有的孩子节点
  const children = vnode.children
  const tag = vnode.tag
  if (isDef(tag)) {
    // 未知标签
    if (process.env.NODE_ENV !== 'production') {
      if (data && data.pre) {
        creatingElmInVPre++
      }
      if (isUnknownElement(vnode, creatingElmInVPre)) {
        warn(
          'Unknown custom element: <' + tag + '> - did you ' +
          'register the component correctly? For recursive components, ' +
          'make sure to provide the "name" option.',
          vnode.context
        )
      }
    }

    // 创建新节点
    vnode.elm = vnode.ns
      ? nodeOps.createElementNS(vnode.ns, tag)
      : nodeOps.createElement(tag, vnode)
    setScope(vnode)

    // 递归创建所有子节点（普通元素、组件）
    createChildren(vnode, children, insertedVnodeQueue)
    if (isDef(data)) {
      invokeCreateHooks(vnode, insertedVnodeQueue)
    }
    // 将节点插入父节点
    insert(parentElm, vnode.elm, refElm)

    if (process.env.NODE_ENV !== 'production' && data && data.pre) {
      creatingElmInVPre--
    }
  } else if (isTrue(vnode.isComment)) {
    // 注释节点，创建注释节点并插入父节点
    vnode.elm = nodeOps.createComment(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  } else {
    // 文本节点，创建文本节点并插入父节点
    vnode.elm = nodeOps.createTextNode(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  }
}
```

- **这里需要注意：在 `createElm()` 函数中，会根据 `VNode` 类型进行分类创建，如果 `VNode` 是组件类型，则会进入 `createComponent()` 函数中，如果 `VNode` 类型是元素的话，则在 `createComponent()` 函数中不会执行，而且会返回 `false` 直接跳出 `if` 分支，直接执行下面创建元素的逻辑，元素创建完之后，如果自身有子元素就会递归创建，反之就会直接插入到父元素中。**



## `createComponent()`

`/src/core/vdom/patch.js`

```javascript
/**
 * 如果 vnode 是一个组件，则执行 init 钩子，创建组件实例，并挂载
 * 然后为组件执行各个模块的 create 方法
 * @param {*} vnode 组件新的 vnode
 * @param {*} insertedVnodeQueue 数组
 * @param {*} parentElm oldVnode 的父节点
 * @param {*} refElm oldVnode 的下一个兄弟节点
 * @returns 如果 vnode 是一个组件并且组件创建成功，则返回 true，否则返回 undefined
 */
function createComponent(vnode, insertedVnodeQueue, parentElm, refElm) {
  // 获取 vnode.data 对象
  let i = vnode.data
  if (isDef(i)) {
    // 验证组件实例是否已经存在 && 被 keep-alive 包裹
    const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
    // 执行 vnode.data.init 钩子函数，该函数在讲 render helper 时讲过
    // 如果是被 keep-alive 包裹的组件：则再执行 prepatch 钩子，用 vnode 上的各个属性更新 oldVnode 上的相关属性
    // 如果是组件没有被 keep-alive 包裹或者首次渲染，则初始化组件，并进入挂载阶段
    if (isDef(i = i.hook) && isDef(i = i.init)) {
      i(vnode, false /* hydrating */)
    }
    // after calling the init hook, if the vnode is a child component
    // it should've created a child instance and mounted it. the child
    // component also has set the placeholder vnode's elm.
    // in that case we can just return the element and be done.
    if (isDef(vnode.componentInstance)) {
      // 如果 vnode 是一个子组件，则调用 init 钩子之后会创建一个组件实例，并挂载
      // 这时候就可以给组件执行各个模块的的 create 钩子了
      initComponent(vnode, insertedVnodeQueue)
      // 将组件的 DOM 节点插入到父节点内
      insert(parentElm, vnode.elm, refElm)
      if (isTrue(isReactivated)) {
        // 组件被 keep-alive 包裹的情况，激活组件
        reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
      }
      return true
    }
  }
}
```

- **如果 `VNode` 的类型是组件的话，那么 `vnode.data` 中必定会存在组件对应的钩子函数，则直接执行 `init()` 钩子函数，也就是 `vnode.data.init()` 进行创建组件实例，得到组件实例之后就可以给组件执行各个模块中的 `create()` 钩子，然后将组件对应的 DOM 节点插入到父节点中。**
- **在对组件进行 `init()` 操作的过程中，组件还会进行相对应的以下操作：`Component.$mount()`、`mountComponent()`、`updateComponent()`、`vm._render()`、`vm._update()`、`patch()` 等操作，这里需要注意，每个组件都有属于他自己的 `patch()` 函数的，然后如果该 `patch()` 函数的调用者不是根组件的话，在 `invokeInsertHook()` 函数中，就会将对应的 `insertedVnodeQueue` 赋值给 `vnode.parent.pendingInsert` ，然后返回上一层组件的时候，调用 `initComponent()` 的时候就会将 `pendingInsert` 加到 `insertedVnodeQueue` 中，因为每一层组件的 `patch()` 函数中都持有一个 `insertVnodeQueue` ，所以只能通过这种方法将他们进行合并操作**

- **在组件自己的 `init()` 钩子函数执行完毕之后，就会轮到执行 `initComponent()` 函数，如下面的代码所示，在 `initComponent()` 中，会将当前 `vnode` 的 `pendingInsert` 赋值给 `insertedVnodeQueue` ，本质就是合并父子组件中的 `insertedVnodeQueue`！！！！！！！！！文末会给出一个详细的流程图说明整个流程！！！！！！！！！**

```javascript
function initComponent (vnode, insertedVnodeQueue) {
    if (isDef(vnode.data.pendingInsert)) {
      insertedVnodeQueue.push.apply(insertedVnodeQueue, vnode.data.pendingInsert)
      vnode.data.pendingInsert = null
    }
    vnode.elm = vnode.componentInstance.$el
    if (isPatchable(vnode)) {
      invokeCreateHooks(vnode, insertedVnodeQueue)
      setScope(vnode)
    } else {
      // empty component root.
      // skip all element-related modules except for ref (#3455)
      registerRef(vnode)
      // make sure to invoke the insert hook
      insertedVnodeQueue.push(vnode)
    }
  }
```



## `insert()`

`/src/core/vdom/patch.js`

```javascript
/**
 * 向父节点插入节点 
 */
function insert(parent, elm, ref) {
  if (isDef(parent)) {
    if (isDef(ref)) {
      if (nodeOps.parentNode(ref) === parent) {
        nodeOps.insertBefore(parent, elm, ref)
      }
    } else {
      nodeOps.appendChild(parent, elm)
    }
  }
}
```

- **该方法在 `createElm()` 和 `createComponent()` 中都有调用，他是将创建出来的元素、组件对应的 DOM 结构插入到父节点的 DOM 结构当中**



## `patchVnode()`

`/src/core/vdom/patch.js`

```javascript
/**
 * 更新节点
 *   全量的属性更新
 *   如果新老节点都有孩子，则递归执行 diff
 *   如果新节点有孩子，老节点没孩子，则新增新节点的这些孩子节点
 *   如果老节点有孩子，新节点没孩子，则删除老节点的这些孩子
 *   更新文本节点
 */
function patchVnode(
  oldVnode,
  vnode,
  insertedVnodeQueue,
  ownerArray,
  index,
  removeOnly
) {
  // 老节点和新节点相同，直接返回
  if (oldVnode === vnode) {
    return
  }

  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // clone reused vnode
    vnode = ownerArray[index] = cloneVNode(vnode)
  }

  const elm = vnode.elm = oldVnode.elm

  // 异步占位符节点
  if (isTrue(oldVnode.isAsyncPlaceholder)) {
    if (isDef(vnode.asyncFactory.resolved)) {
      hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
    } else {
      vnode.isAsyncPlaceholder = true
    }
    return
  }

  // 跳过静态节点的更新
  // reuse element for static trees.
  // note we only do this if the vnode is cloned -
  // if the new node is not cloned it means the render functions have been
  // reset by the hot-reload-api and we need to do a proper re-render.
  if (isTrue(vnode.isStatic) &&
    isTrue(oldVnode.isStatic) &&
    vnode.key === oldVnode.key &&
    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
  ) {
    // 新旧节点都是静态的而且两个节点的 key 一样，并且新节点被 clone 了 或者 新节点有 v-once指令，则重用这部分节点
    vnode.componentInstance = oldVnode.componentInstance
    return
  }

  // 执行组件的 prepatch 钩子
  let i
  const data = vnode.data
  if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
    i(oldVnode, vnode)
  }

  // 老节点的孩子
  const oldCh = oldVnode.children
  // 新节点的孩子
  const ch = vnode.children
  // 全量更新新节点的属性，Vue 3.0 在这里做了很多的优化
  if (isDef(data) && isPatchable(vnode)) {
    // 执行新节点所有的属性更新
    for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
    if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
  }
  if (isUndef(vnode.text)) {
    // 新节点不是文本节点
    if (isDef(oldCh) && isDef(ch)) {
      // 如果新老节点都有孩子，则递归执行 diff 过程
      if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
    } else if (isDef(ch)) {
      // 老孩子不存在，新孩子存在，则创建这些新孩子节点
      if (process.env.NODE_ENV !== 'production') {
        checkDuplicateKeys(ch)
      }
      if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
      addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
    } else if (isDef(oldCh)) {
      // 老孩子存在，新孩子不存在，则移除这些老孩子节点
      removeVnodes(oldCh, 0, oldCh.length - 1)
    } else if (isDef(oldVnode.text)) {
      // 老节点是文本节点，则将文本内容置空
      nodeOps.setTextContent(elm, '')
    }
  } else if (oldVnode.text !== vnode.text) {
    // 新节点是文本节点，则更新文本节点
    nodeOps.setTextContent(elm, vnode.text)
  }
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
  }
}
```

- **该方法是执行 `vnode` 和 `oldVnode` 的比较，并进行更新操作，如果 `vnode` 和 `oldVnode` 中都含有 `children` 的话，那么就会执行 `updateChildren()` 的函数进行更新操作，如果 `vnode` 是组件的话，那么将执行组件中的 `vnode.data.hook.update()` 方法，该 `vnode.data.hook.update()` 方法就是执行组件自身的 `update()` 函数，该过程会触发对应的 `patch()` 函数，这里的更新都是使用递归的形式进行深度遍历的更新方式，一直找到最底层开始进行更新。 **



## `updateChildren()`

`/src/core/vdom/patch.js`

```javascript
/**
 * diff 过程:
 *   diff 优化：做了四种假设，假设新老节点开头结尾有相同节点的情况，一旦命中假设，就避免了一次循环，以提高执行效率
 *             如果不幸没有命中假设，则执行遍历，从老节点中找到新开始节点
 *             找到相同节点，则执行 patchVnode，然后将老节点移动到正确的位置
 *   如果老节点先于新节点遍历结束，则剩余的新节点执行新增节点操作
 *   如果新节点先于老节点遍历结束，则剩余的老节点执行删除操作，移除这些老节点
 */
function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  // 老节点的开始索引
  let oldStartIdx = 0
  // 新节点的开始索引
  let newStartIdx = 0
  // 老节点的结束索引
  let oldEndIdx = oldCh.length - 1
  // 第一个老节点
  let oldStartVnode = oldCh[0]
  // 最后一个老节点
  let oldEndVnode = oldCh[oldEndIdx]
  // 新节点的结束索引
  let newEndIdx = newCh.length - 1
  // 第一个新节点
  let newStartVnode = newCh[0]
  // 最后一个新节点
  let newEndVnode = newCh[newEndIdx]
  let oldKeyToIdx, idxInOld, vnodeToMove, refElm

  // removeOnly是一个特殊的标志，仅由 <transition-group> 使用，以确保被移除的元素在离开转换期间保持在正确的相对位置
  const canMove = !removeOnly

  if (process.env.NODE_ENV !== 'production') {
    // 检查新节点的 key 是否重复
    checkDuplicateKeys(newCh)
  }

  // 遍历新老两组节点，只要有一组遍历完（开始索引超过结束索引）则跳出循环
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (isUndef(oldStartVnode)) {
      // 如果节点被移动，在当前索引上可能不存在，检测这种情况，如果节点不存在则调整索引
      oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
    } else if (isUndef(oldEndVnode)) {
      oldEndVnode = oldCh[--oldEndIdx]
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
      // 老开始节点和新开始节点是同一个节点，执行 patch
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      // patch 结束后老开始和新开始的索引分别加 1
      oldStartVnode = oldCh[++oldStartIdx]
      newStartVnode = newCh[++newStartIdx]
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      // 老结束和新结束是同一个节点，执行 patch
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      // patch 结束后老结束和新结束的索引分别减 1
      oldEndVnode = oldCh[--oldEndIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
      // 老开始和新结束是同一个节点，执行 patch
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
      // 处理被 transtion-group 包裹的组件时使用
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
      // patch 结束后老开始索引加 1，新结束索引减 1
      oldStartVnode = oldCh[++oldStartIdx]
      newEndVnode = newCh[--newEndIdx]
    } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
      // 老结束和新开始是同一个节点，执行 patch
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
      // patch 结束后，老结束的索引减 1，新开始的索引加 1
      oldEndVnode = oldCh[--oldEndIdx]
      newStartVnode = newCh[++newStartIdx]
    } else {
      // 如果上面的四种假设都不成立，则通过遍历找到新开始节点在老节点中的位置索引

      // 找到老节点中每个节点 key 和 索引之间的关系映射 => oldKeyToIdx = { key1: idx1, ... }
      if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
      // 在映射中找到新开始节点在老节点中的位置索引
      idxInOld = isDef(newStartVnode.key)
        ? oldKeyToIdx[newStartVnode.key]
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
      if (isUndef(idxInOld)) { // New element
        // 在老节点中没找到新开始节点，则说明是新创建的元素，执行创建
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
      } else {
        // 在老节点中找到新开始节点了
        vnodeToMove = oldCh[idxInOld]
        if (sameVnode(vnodeToMove, newStartVnode)) {
          // 如果这两个节点是同一个，则执行 patch
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
          // patch 结束后将该老节点置为 undefined
          oldCh[idxInOld] = undefined
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
        } else {
          // 最后这种情况是，找到节点了，但是发现两个节点不是同一个节点，则视为新元素，执行创建
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        }
      }
      // 老节点向后移动一个
      newStartVnode = newCh[++newStartIdx]
    }
  }
  // 走到这里，说明老姐节点或者新节点被遍历完了
  if (oldStartIdx > oldEndIdx) {
    // 说明老节点被遍历完了，新节点有剩余，则说明这部分剩余的节点是新增的节点，然后添加这些节点
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
  } else if (newStartIdx > newEndIdx) {
    // 说明新节点被遍历完了，老节点有剩余，说明这部分的节点被删掉了，则移除这些节点
    removeVnodes(oldCh, oldStartIdx, oldEndIdx)
  }
}
```

