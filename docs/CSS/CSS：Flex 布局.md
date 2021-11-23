---
title: CSS：Flex 布局
date: 2021/7/9 14:35:00
tag: [CSS]
---

# CSS：Flex 布局

## 盒子模型

### 盒模型

盒模型由 `content`、`padding`、`border`、`margin` 组成

### margin

- margin 可以让块状元素居中显示，注意是块状元素。实现方式是：`margin: 10px, auto`

- 设置 margin 可能会导致上下的两个元素出现外间距合并的情况，类似的上边的 div (块级元素) 设置了 margin 为 100px，然后下边的 div 设置了 50px，那么就会导致这两个元素间的间距并不是 150px，而是变成了 100px，这个就是外间距合并，这种的解决方式就是在这两个 div 的外边的父元素触发成一个 BFC。这样就可以解决了。
- 行内元素实际上不占上下外边距。行内元素的左右外边距不合并
- 浮动元素的外边距也不会合并
- 允许指定负的外边距，不过使用的时候要小心

### border

padding 的宽高要是记录在盒子模型的宽高之内，与此同时的是 border 也要记录在盒子模型的宽高之内，但是 margin 并不算在宽高之内。所以各位在书写宽高时注意减掉内边距和边框。box-sizing 属性有两个值，一个是 content-box (标准)，此时 padding 和 border 不被包含在 width 和 height 内，元素的实际大小为宽高 + border + padding，此为标准模式下的盒模型。border-boxing (怪异) padding 和 border 被包含在定义的 width 和 height 中。元素实际的大小为你定义了多宽就是多宽。此属性为怪异模式下的盒模型。

## Flex 布局

### 是什么

Flex是“弹性布局”，用来为盒状模型提供最大的灵活性，任何一个容器都可以指定为Flex布局。

```css
.box{
  display: flex;
}
```

行内元素也可以使用Flex布局

```css
.box{
	display: inline-flex
}
```

注意，设为Flex布局后，子元素的float、clear和vertical-align属性将失效。

### 基本概念

采用了 Flex 布局的元素，称为 Flex 容器，简称“容器”。它的所有子元素自动称为容器成员，称为 Flex 项目，简称“项目”

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/4COzFw.jpg)

容器默认存在两根轴：水平方向的主轴 (main axis) 和垂直方向的交叉轴 (cross axis)。主轴的开始位置 (与边框的交叉点) 叫 main start，结束位置叫做 main end；交叉轴的开始位置叫做 (cross end)。

项目默认沿主轴排列。单个项目占据的主轴空间叫做 (main size)，占据的交叉轴空间叫做 (cross size)。

### 容器的属性

以下 6 个属性设置在容器上

- flex-direction
- flex-wrap
- flex-flow
- justify-content
- align-items
- align-content

#### flex-direction

flex-direction 属性决定主轴的方向 (即项目的排列方向)

```css
.box {
  flex-direction: row | row-reverse | column | column-reverse
}
```

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/7t8Gu5.jpg)

它的取值有 4 种可能：

- row (默认值)：主轴为水平方向，起点在左端
- row-reverse：主轴为水平方向，起点在右边
- column：主轴为垂直方向，起点在上方
- column-reverse：主轴为垂直方向，起点在下方

#### flex-wrap

默认情况下，项目都排在一条线 (又称“轴线”) 上。flex-wrap属性定义，如果一条轴线排不下，如何换行

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/2FW5XI.jpg)

它的取值有三种可能：

- norwrap (默认值)：不换行

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/6d2FWW.jpg)

- wrap：换行，第一行在上方

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/JA3nNx.jpg)

- wrap-reverse：换行，第一行在下方

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/hOo7m9.jpg)

#### flex-flow

flex-flow 属性是 flex-direction 属性和 flex-wrap 属性的简写形式，默认值为 row norwrap

```css
.box {
  flex-flow: <flex-direction> || <flex-wrap>;
}
```

#### justify-content

justify-content 属性定义了项目在主轴的对齐方式

```css
.box{
	justify-content: flex-start | flex-end | center | space-between
}
```

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/NFWbgj.jpg)

它的取值有 5 种可能，具体对齐方式跟轴的方向有关，下面假设主轴为从左到右

- flex-start (默认值)：左对齐
- flex-end：右对齐
- center：居中
- space-between：两端对齐，项目之间的间隔相等
- space-around：每个项目两侧的间隔相等，所以，项目之间的间隔比项目与边框的间隔大一倍

#### align-items

align-items 属性定义项目在交叉轴上如何对齐

```css
.box{
	align-items: flex-start | flex-end | center | baseline | stretc
}
```

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/rttbFb.jpg)

它的取值有 5 种可能：

- flex-start：交叉轴起点对齐
- flex-end：交叉轴终点对齐
- center：交叉轴中点对齐
- baseline：项目的第一项文字的基线
- stretch (默认值)：如果项目未设置高度或设为 auto，将占满整个容器的高度

#### align-content

align-content 属性定义了多根轴线的对齐方式。如果项目只有一个轴线，该属性不起作用

```css
.box{
	align-content: flex-start | flex-end | center | space-between;
}
```

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/W8DGmG.jpg)

- flex-start：与交叉轴的起点对齐。
- flex-end：与交叉轴的终点对齐。
- center：与交叉轴的中点对齐。
- space-between：与交叉轴两端对齐，轴线之间的间隔平均分配
- space-around：每根轴线两侧的间隔都相等，所以，轴线之间的间隔比轴线与边框间隔大一倍
- stretch（默认值）：轴线占满整个交叉轴

### 项目的属性

以下 6 个属性设置在项目上

- order
- flex-grow
- flex-shrink
- flex
- align-self

#### order

order 属性定义项目的排列顺序。数值越小，排列越靠前，默认为 0。

```css
.item{
	order: <integer>;
}
```

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/iTaKZ4.jpg)

#### flex-grow

flex-grow 属性定义项目的放大比例，默认为 0，即如果存在剩余空间，也不放大。

```css
.item{
	flex-grow: <number>;
}
```

如果所有项目的 flex-grow 属性都为 1，则他们将等分剩余空间 (如果有的话)，如果一个项目的 flex-grow 属性为 2，其他项目都为 1，则前者占据的剩余空间将比其他项多一倍

#### flex-shrink

flex-shrink 属性定义了项目的缩小比例，默认为 1，即如果空间不够，该项目会缩小

```css
.item{
	flex-shrink: <number>;
}
```

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/sn1tSc.jpg)

如果所有项目 flex-shrink 属性为 1，那么当空间不够的时候，都将等比例缩小。如果一个项目的 flex-shrink 属性为 0，其他项目都为 1，则空间不足时，前者不缩小。负值对该属性无效。

#### flex-basis

flex-basis 属性定义了在分配多余空间之前，项目占据的主轴空间 (main size)。浏览器根据这个属性，计算主轴是否有多余空间。它的默认值为 auto，即项目本来大小。

```css
.item{
	flex-basis: <length> | auto;
}
```

它可以设为跟 width 或 height 属性一样的值，比如 350px，则项目将占据固定空间。

#### flex

flex 属性是 flex-grow、flex-shrink、flex-basis 的简写，默认值为 0、1、auto。后两个属性可选。

```css
.item{
	flex: none | [ <'flex-grow'> <'flex-shrink'>? || <'flex-basis'>]
}
```

该属性有两个快捷键：auto(1 1 auto) 和 auto(0 0 auto)。

建议优先使用这个属性，而不是单独写是哪个分离的属性，因为浏览器会推算相关值。

#### align-self

align-self 属性允许单个项目有与其他项目不一样的对齐方式，可覆盖 align-items 属性。默认值为 auto，表示继承父元素的 align-items，如果没有父元素，则等同 stretch。

```css
.item{
	align-self: auto | flex-start | flex-end | center | baseline |
}
```

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/KeVUWJ.jpg)

该属性可能取值为 6 个，除了 auto，其他都与 align-items 属性完全一致。