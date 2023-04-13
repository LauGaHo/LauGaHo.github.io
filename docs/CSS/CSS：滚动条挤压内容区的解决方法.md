# CSS: 滚动条挤压内容区的解决方法

在项目开发的过程中遇到了一个这样的问题，父容器设置了 `overflow-x: hidden`，然后子元素内容的高度大于父容器的高度，所以父容器中出现了滚动条，但是滚动条的出现占据了原本子元素的空间，导致子元素的内容发生了挤压和变形，代码如下：

```html
<div class="father">
  <div class="son"></div>
</div>
```

```css
.father {
  width: 200px;
  height: 200px;
  overflow-x: hidden;
}

.son {
  width: 200px;
  height: 500px;
}
```

> 这里当时我自己也有一个疑惑，为什么我没有设置 `overflow-y` 属性，为什么会自动出现滚动条，那是因为，在默认情况下，`overflow-x` 和 `overflow-y` 的默认值都是 `visible`，但是在 `.father` 属性上，我们设置了 `overflow-x: hidden`，MDN 官网上说道：`如果设置了 overflow-x 或者 overflow-y 其中一个方向的值不为 visible 的话，那么另外的一个方向会被自动设置为 auto`。所以这里设置了 `overflow-x: hidden`，因而 `overflow-y` 也会被自动设置为 `auto`。

对应于上述滚动条占据子元素的情况，我们可以使用 `overflow-y: overlay` 来进行处理。

## overflow: overlay

`overflow: overlay` 这个设置的意思是：当出现滚动条的时候，滚动条不会实际占用父容器的空间位置，就好像移动端的滚动条一样，只会浮在容器上，但是不会像移动端那样会自动消失，他会遮盖一些位置，但是不会对父容器中的子元素产生挤压的效果，因为他就像悬浮一样，浮在父容器的表面。

经过在浏览器中的查看，大概知道了滚动条的宽度为 `6px`，所以我们的做法是给父容器添加 `6px` 的 `padding-right`，然后给父容器设置 `overflow-y: overlay` 这样父容器的滚动条就不会占据父容器的空间，不过滚动条在这里虽然不会占据父容器的空间，但是他的悬浮会遮盖父容器内容，导致可能会影响用户在父容器的交互操作，所以我们这里给父容器增加了 `6px` 的 `padding-right` 这样滚动条就会出现浮在 `padding-right` 的区域上，并不会对父容器的内容区造成遮盖。

```css
.father {
  width: 200px;
  height: 200px;
  overflow-y: overlay;
  padding-right: 6px;
}

.son {
  width: 200px;
  height: 500px;
}
```

经过这样就能够完美解决滚动条占据父容器空间的问题，设置了 `padding-right` 属性之后，就算滚动条浮了起来也不会遮盖到父容器的内容区，滚动条只会乖乖占据 `padding-right` 的位置。
