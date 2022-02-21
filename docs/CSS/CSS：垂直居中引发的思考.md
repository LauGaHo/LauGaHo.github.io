# CSS：垂直居中引发的思考



## 水平居中

- 横向格式化属性：他包含七个属性：margin-left、border-left-width、padding-left、width、padding-left、border-right-width、margin-right，这七个属性影响着块级框的横向布局。他们七个的值加起来要等于元素容纳块的宽度，而这个宽度通常就是块级父元素的 width 值。用公式来表示就是：

  ```javascript
  margin-left + border-left-width + padding-left + width + padding-right + border-right-width + margin-right = father-width
  ```

  这七个值中可以设置为 auto 的值只有三个，margin-left、margin-right、width，如果把其中一个值设置为 auto，其他两个值设置为具体值，那么设置为 auto 的那个属性具体长度要能满足元素框的宽度等于父元素的宽度，即满足那个等式，那么如果把其中的两个设置为 auto，下面我们就来讨论这个情况：

  - 把 margin-left 和 margin-right 全部设置为 auto，width 设置为具体值，这个时候两个外边距的长度相等，具体表现就是元素在父元素中居中显示，也就是我们想要的水平居中效果

  - 把某一边的外边距和 width 设置为 auto，此时设置为 auto 的外边距等于 0，宽度根据等式计算，举个简单的例子：

    ```css
    .center {
      margin-left: 100px;
      margin-right: auto;
      width: auto;
      height: 300px;
      background-color: red;
    }
    ```

    此时我们将 margin-right 设置为了 auto，那它就会变成 0，由于没有设置 border 和 padding，那么他们默认为 0，所以 `width + margin-left` 应该等于父元素的宽度，这里我的父元素就是 body，所以表现出来应该就是 `margin-left` 等于  `100px` 然后 `div` 占满了剩下的空间，如图：

    ![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/TIua17.jpg)

  - 还有一种情况，我们把三个值全部都设置为 auto，这时候会把左右边距全部设置为 0，width 则是要多宽就有多宽。显示的效果如下图。因为去掉宽度后宽度的默认值为 auto，你又把左右边距设置成了 auto，那么左右边距自然变成 0 了，这也是为什么居中失效的原因。接下来你可能会想，如果我一个 auto 都不设置，全部设置成具体值，那上面的那个等式不就不成立了吗？这种情况在 css 中被称为过约束，在这种情况下 margin-right 会被强制设置为 auto



## 垂直居中

了解完水平居中，我们探讨一下垂直居中，有了上面的结论，很容易会想到，垂直居中不就设置为 `margin: auto 0` 不就行了吗

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
    <style>
        .center {
            margin: auto 0;
            width: 100px;
            height: 100px;
            background-color: red;
        }
    </style>
</head>
<body>
    <div class="center">
        hello world
    </div>
</body>
</html>
```

这段代码产生的效果是下面这样的：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/ba9Gbz.jpg)

这里为什么不会自动设置上边距和下边距？这里是跟 auto 的默认行为有关，上下外边距如果都被设置为 auto，最终会变成 0，就像跟没有外边距是一样的效果。这很容易想明白，父元素的高度都是不确定的，如何去自动居中？所以不可能像横向 margin 那样计算，这也就解释了为什么我们不能使用 auto 来垂直居中元素。那么我们如何实现垂直居中？通常我们的代码是这样的：

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
    <style>
        .father {
            position: relative;
            width: 500px;
            height: 500px;
            background-color: blue;
        }
        .center {
            position: absolute;
            left: 0;
            right: 0;
            top: 0;
            bottom: 0;
            margin: auto;
            width: 100px;
            height: 100px;
            background-color: red;
        }
    </style>
</head>
<body>
    <div class="father">
        <div class="center"></div>
    </div>
</body>
</html>
```

我们为 center 元素加上绝对定位，并且把他的各个值全部都设置为 0，这个时候神奇的事情发生了，我们成功居中了子元素了，如下图

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/GNavip.jpg)

- 纵向格式化属性
  - 在纵向同样也是存在一个式子：`top + margin-top + border-top-width + padding-top + height + padding-bottom + border-bottom-width + margin-bottom + bottom = father-height ` 在前方的式子中，top 和 bottom 都被设置为了 0，margin 等于 auto，这时候浏览器为了满足这个等式会把上下的距离均匀分给 margin，即达到我们想要的居中效果。同时横向也是一样的道理，所以我们可以看到其实这个元素时水平垂直居中的，如果此时存在过约束，一般会忽略 right 属性。

