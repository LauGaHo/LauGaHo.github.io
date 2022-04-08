# Vue动态组件\<component\>的思考

## 解耦代码需求

由于在公司开发动态组件的项目，项目的初始阶段，急于做出一个 Demo 出来，所以导致项目没有经过深入的框架设计。随着项目一直向前迭代，新增的需求组件越来越多，导致项目中有几十个组件，每个组件对应一段 HTML 代码，因此到了最后就导致几十个组件的 HTML 全部冗余在一个 Vue 文件的模板标签当中，总共有几百行，全部都用 `v-if` 来进行控制，这样一来，每次模板中关联的变量一旦更新，就会导致几百行的模板变量的组件重新计算一遍，计算量之大可想而知。并且，对于一些弹框式的模态框，项目中也是把模态框的模板写死在某些组件中，只是用一个 `visible` 变量来进行控制。整个模板文件就十分庞杂。

## 研究 `Vue.extend()`

为了寻求解耦的方式，我思路一开始转到了以前作为一个 `Nger` 最基本的思路，就是在使用到组件的时候在 JS 或者 TS 代码中把组件实例化出来，然后将其挂载到项目中的 `DOM` 结构下。所以我一开始将我的目光瞄准了 Vue 中用于动态创建组件的 API ：`Vue.extend()`。这个 API ：

- 第一步，就是传入一个组件配置对象，然后 Vue 根据对应的对象，构建一个子类构造器出来，也就是 `const Constructor = Vue.extend(XXXComponent)` 这样就得到了组件的构造器。
- 第二步，通过 `new` 操作符就能创建对应的组件实例，也就是：`const comInstance = new Constructor({ propsData: {} })` 其中还可以传入对应所需要的 `props`。
- 第三步，对新建出来的组件订阅事件就是 `comInstance.$on('some_event', () => {})` 进行事件的订阅。
- 第四步，挂载组件则是：`comInstance.$mount('#app')`

这样咋眼看确实功能都可以实现，但是项目中的组件还使用了 `provide` 和 `inject`。我在网上找遍了大江南北都找不到对应的设置方法，所以我放弃了，后面找到了一个，`inject` 的来源是在 `Root` 组件中，但是我想要的并非是在 `Root` 组件中，而是在祖父组件中的 `provide`。最终功能达不到，被迫放弃。

## 转移目光，聚焦 `<component>`

后面某个时刻，想了一想 Vue 中的动态组件的原理，其实这个是 Vue 实现动态组件不同于 Angular 的一种思路。Vue 的实现思路是这样的，在模板上放置一个 `<component></component>` 占位标签，然后将一些变量绑定在 `<component>` 标签中，例子如下：

```vue
<template>
	<component v-bind="input"></component>
</template>

<script>
	export default {
    name: 'TestComponent',
    data() {
      return {
        input: {
          is: 'HelloComponent',
          message: 'Hi'
        }
      }
    }
  }
</script>
```

形如上方的例子中，将 `input` 变量绑定了 `<component>` 标签，然后每当绑定的变量的改变之后，Vue 识别到之后就会根据 `is` 属性中的值来选择对应的组件自动完成实例化，数据绑定，组件挂载到 `<component>` 占位符所在的位置中。所以本质上和 Angular 的动态组件是一样的原理，但是 Vue 的话更加智能化，直接用占位符占据对应的位置，然后改变组件类型，就能够自动完成实例化，挂载等操作。对比之下，Vue 的这种实现方式，更加能体现 MVVM 框架中的一大思想：***数据驱动视图层的改变。***由此感叹这是一个多么伟大的思想，太你妈的牛逼了这个设计。

所以只需要选好动态组件渲染的位置，然后不断更改动态组件绑定的数据，Vue 就能够不断销毁创建组件，生生不息。

## 实践

回到开头中提到的模态框写死，还有几十个组件全写在一个文件中，然后通过 `v-if` 指令来控制。

### 模态框写死

```vue
<template>
	<!--各种弹窗 Modal 组件-->
  <component
  	ref='modalComponent'
    v-if='modalOptions'
    v-bind='modalOptions.props'
    v-on='modalOptions.events'>
  </component>
</template>
```

在上方代码中，用 `modalOptions` 来控制弹窗 Modal 组件的创建和销毁，每次 `DOM` 更新，就会看 `modalOptions` 变量有无更改，如果有，则销毁原来的组件，然后实例化新的所需要的组件，并完成挂载，如果 `modalOptions` 为 `null` 值，那就是销毁原本的组件，这样子 `<component>` 节点就会仿佛不再存在一般。所以这样就像德芙一样丝滑。

### 几十个组件写死

通过使用动态组件就可以将原本写死的几百行 HTML 浓缩成如下：

```vue
<template>
	<!--根据 element 获取当前动态组件的组件类型，形如：ElInput, ElSelect, ElDivider 等-->
  <component v-bind='element.getComProps()'>
    <!--满足 element 渲染过程中的这种情况：<el-button>{{textContent}}</el-button>-->
    <template v-if='element.hasOwnProperty("getComTextContent")'>
      {{ typeof element.getComTextContent() === 'function' ? element.getComTextContent().call(this) : element.getComTextContent()
      }}
  	</template>
    <!--
    满足 element 渲染过程中的这种情况：
    <el-select>
    <el-option v-for="(item) in list></el-option>
    </el-select>
    -->
    <template v-if='element.hasOwnProperty("getComChildren")'>
      <component v-for='(com, index) in element.getComChildren()' v-bind='com'>
        <template v-if='com.hasOwnProperty("textContent")'>{{ com.textContent }}</template>
      </component>
    </template>
  </component>
</template>
```

上方动态组件的销毁实例化同样的也是由 `element` 实例来控制，我们只需要控制 `element` 就等于控制了视图层，因为在上方，我们指定了 `element` 映射到视图层的一个规则，只要我们改变了 `element` 这一数据，Vue 就会按照我们指定的规则，在规定的位置渲染我们规定的组件，一切仅在我们掌握之中。

另外，模板体积减少，也让 Vue 在 `patch()` 和 `patchVNode()` 阶段的开销减少，某种程度上，算是优化了页面渲染的性能。

由此可以知道 Vue 也是一个优秀的前端框架，把前端框架的思想死死把握住，尤雨溪牛逼！

所以我懂得了一个道理，其实三大框架都是优秀的框架，没有必要拉踩任何一个，优秀的开发者应该是掌握底层的思想和原理，以不变应万变，毕竟只要知道原理，这些东西都是信手拈来，各位开发者共勉，一起加油！！！

