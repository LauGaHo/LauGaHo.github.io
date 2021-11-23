# CSS：选择器

## 基本选择器

1、标签选择器

```css
p {
	color: red
}
```

2、类选择器 class

```css
.active {
	color: red
}
```

3、Id 选择器

```css
#id1 {
	color: red
}
```

<u>***总结：Id 选择器 > class 选择 > 标签选择器***</u>

## 层次选择器

1、后代选择器：body 中的所有子孙 p 标签都被设置为红色

```css
body p{
	color: red
}
```

2、子选择器：body 中的子 p 标签才被设置为红色，孙 p 标签不被设置为红色

```css
body > p{
	color:red
}
```

3、相邻兄弟选择器：只使用类别为 active 并且是他下边一个相邻的 p 标签才被设置为红色

```css
.active + p{
	color:red
}
```

4、通用选择器：通用兄弟选择器，当前选中元素的向下的所有兄弟元素

```css
.active ~ p{
	color:red
}
```

## 结构伪类选择器

```html
<body>
  <p>p1</p>
  <p>p2</p>
  <p>p3</p>
  <ul>
    <li>li1</li>
    <li>li2</li>
    <li>li3</li>
  </ul>
</body>
```

```css
/*选择ul中的子孙li元素，选中最后一个li*/
ul li:last-child {
  background: red;
}

/*选择ul中的子孙li元素，选中第一个li*/
ul li:first-child {
  background: blue;
}

/*选择当前p标签的父标签，选中父级元素的第一个，并且是当前元素才生效*/
p:nth-child(1) {
  background: yellow
}

/*定位到当前元素的父元素，并且将父元素中为p标签类型第二个设置为黄色*/
p:nth-of-type(2) {
  background: yellow
}
```

## 属性选择器

HTML 内容如下：

```html
<p class="demo">
  <a href="http://www.baidu.com" class="links item first" id="first">1</a>
  <a href="" class="links item active" target="_blank" title="test">2</a>
  <a href="images/123.html" class="links item">3</a>
  <a href="images/123.png" class="links item">4</a>
  <a href="images/123.jpg" class="links item">5</a>
  <a href="abc" class="links item">6</a>
  <a href="/a.pdf" class="links item">7</a>
  <a href="/abc.pdf" class="links item">8</a>
  <a href="abc.doc" class="links item">9</a>
  <a href="abcd.doc" class="links item last">10</a>
</p>
```

CSS：

```css
/*存在id属性的元素*/
a[id] {
  background: yellow;
}

/*选中id为first的元素*/
a[id=first] {
  background: black;
}

/*选中class中含有links的元素*/
a[class*="links"] {
  background: blue;
}

/*选中href属性中以http开头的元素*/
a[href^=http] {
  background: yellow;
}

/*选中href属性中以pdf结尾的元素*/
a[href$=pdf] {
  background: blue;
}
```

=：绝对等于

*=：包含这个元素

^=：以这个开头

$=：以这个结尾