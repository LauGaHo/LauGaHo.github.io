# Vue计算属性computed的妙用

## 问题场景

在公司的开发中，有一个情况是数据在爷父子组件中层层通过 `props` 来进行数据的传递，具体的场景如下：

```vue
// Container.vue
<WidgetForm :data='data'></WidgetForm>

// WidgetForm.vue
<WidgetFormList :list='data.list'></WidgetFormList>

// WidgetFormList.vue
<draggable v-model='???'></draggable>
```

在上方的代码当中，数据通过在 `Container.vue` 文件层层传递到 `WidgetFormList.vue` 当中，然后在 `WidgetFormList.vue` 文件当中又使用了第三方组件 `vue-draggable` 用于实现拖曳排序的功能。但是该组件有一个问题需要注意。

在上方的代码中，`WidgetFormList.vue` 文件中存在一个名为 `list` 的 `props` 变量，当这个时候，如果我们将上方第三行 `HTML` 变量补全 `<draggable v-model='list'></draggable>`，这个时候就会喜提错误，如下：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/BkLaql.png)

上方的错误提示所说的就是：不能够使用 `props` 变量对 `vue-draggable` 组件中的 `v-model` 进行赋值。所以到了这里就需要思考其他的方法为 `vue-draggable` 的 `v-model` 进行变量绑定。

## 不成熟的解决方法

首当其冲就是想到了在 `WidgetFormList.vue` 文件当中，重新定义一个名为 `dataList` 的 `data` 变量，在 `created()` 钩子中进行如下的赋值语句：`this.dataList = this.list`，然后模板上绑定变量就使用 `dataList` 进行变量绑定，然后为 `dataList` 定义一个监视属性，当发现 `dataList` 发生改变之后，就会执行 `this.$emit('update:list', value)` 从而改变外层组件对应的 `list` 变量。

这种做法咋眼看是没有问题，但是倘若遇到了一种情况就是：在 `Container.vue` 中对 `data` 变量进行重新赋值的时候，这个时候在 `WidgetFormList.vue` 是不能够或最新的 DOM 的。具体的原因就在于，我们在 `WidgetFormList.vue` 文件中，是使用 `this.dataList` 绑定模板的，只有 `WidgetFormList.vue` 文件中的 `this.dataList` 发生改变，才会触发视图层的更新。但是此时的问题是，`WidgetFormList.vue` 文件中的 `this.list` 变更是不会触发 `this.dataList` 重新赋值，所以这里就需要当 `WidgetFormList.vue` 中的 `this.list` 更新的时候，可以为 `this.dataList` 重新赋值。

一般人可能会说，可不可以简单粗暴一点，同样的监听 `WidgetFormList.vue` 文件中的 `this.list`，当发生改变，就为 `this.dataList` 重新赋值，但是，这种做法真的很有问题，因为一开始监听了 `this.dataList`，如果发生改变就会改变外层的 `list` 变量，那不就是相当于改变自身的 `this.list`，然后又监视 `this.list`，当发生改变的时候又改变 `this.dataList`。这里就变成一个死循环，一直在改变。再者，监视属性本身的性能开销就比较大，而且叠加这样死循环，系统不慢才怪。

## 计算属性 `computed` 剑指胜利

晚上下班之后，我的脑子闪过了一丝的念头，不能够使用 `this.list` 进行 `v-model` 进行绑定，主要都是因为变量名的原因，因为 `list` 在组件的 `props` 选项中声明了，倘若用另外的变量名，在 `this.list` 的外层包一层，那这样的话，是不是问题就可以得到解决了吗？

所以马上拿出了电脑，去除了 `WidgetFormList.vue` 中 `data` 选项中的 `dataList` 变量，改而把 `dataList` 声明在 `computed` 中，然后绑定在 `vue-draggable` 中的 `v-model` 中，这个时候确实成功了，没有报错，功能同样也正常，但是还有一个问题，就是要同步外层组件的 `list`，最后使用了 `computed` 计算属性的完整定义方式，为一个计算属性定义了完整的 `get/set` 函数。定义如下：

```javascript
dataList: {
  get() {
    return this.list;
  },
  set(val) {
    this.$emit('update:list', val);
  }
}
```

这样问题就得到了完美的解决！

## 关于 `vue-draggable` 的坑

后来在一些重复的实验下，得知了为什么 `vue-draggable` 会不允许绑定 `props`，正常来说，如果父组件传了一个引用类型的变量给子组件的 `props`，那其实是可以在子组件修改该引用类型变量里边的值的，但是 `vue-draggable` 不允许传 `props` 是因为 `vue-draggable` 其实是对传入的变量进行了一个重新赋值。例如：

```vue
<template>
	<draggable v-model='list'></draggable>
</template>
```

在这里传入了一个名为 `list` 的变量，但其实在 `vue-draggable` 组件中，会出现如下的赋值方式：

```javascript
let list = [...list, somethingElse];
```

这种重新赋值就会导致 `list` 变量所指向的堆的地址不一样。从而会导致一些问题出现。这个就看每个人如何使用这个组件了。举个简单的例子：

```javascript
let list = ....;
let temp = list;
list = [...list, somethingElse];

console.log(list === temp)	// false
```

这个问题一般情况不会对代码有什么影响，这里只是涉及到对 `vue-draggable` 底层实现原理的一些猜测。

