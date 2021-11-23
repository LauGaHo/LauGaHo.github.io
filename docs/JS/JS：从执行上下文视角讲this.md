# JS：从执行上下文视角讲 this

观察以下代码

```javascript
var bar = {
    myName:"time.geekbang.com",
    printName: function () {
        console.log(myName)
    }    
}
function foo() {
    let myName = " 极客时间 "
    return bar.printName
}
let myName = " 极客邦 "
let _printName = foo()
_printName()
bar.printName()
```

这里其实是个障眼法，只需要确定好函数调用栈就可以轻松地回答，调用了 foo( ) 后，返回的是 bar.printName，后续就跟 foo 函数没有关系了，所以结果就是调用了两次 bar.printName( ) 函数，根据词法作用域，结果都是 “极客邦”。这是因为 JavaScript 语言的作用域链是由词法作用域决定的，而词法作用域是由代码结构确定的。

不过按照常理来说，调用 bar.printName 方法时，该方法内部的变量 myName 应该使用 bar 对象中的，因为它们是一个整体，大多数面向对象语言都是这样设计的。所以在对象内部的方法中使用对象内部的属性是一种非常普遍的的需求，但是 JavaScript 的作用域机制并不支持这一点，基于这个需求，JavaScript 又搞出了另外一套 this 机制。

所以在 JavaScript 中可以使用 this 实现在 printName 函数中访问到 bar 对象中的 myName 属性了。具体的代码，如下显示：

```javascript
printName: function () {
  console.log(this.myName)
}
```

## JavaScript 中的 this 

关于 this，我们还是得从执行上下文说起，其实执行上下文中包含了变量环境、词法环境、外部环境，但其实还有一个 this 没有提及，具体可以参考下图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/VZ8Dym.jpg)

从图中可以看出，this 是和执行上下文绑定的，也就是说每个执行上下文都有一个 this。执行上下文包含了：全局执行上下文、函数执行上下文、eval执行上下文。

## 全局执行上下文中的 this

全局执行上下文中输入 console.log( this )，最终输出的是 window 对象，这也是 this 和作用域链的唯一交点，作用域链的最底端包含了 window 对象，全局执行上下文中的 this 也是指向 window 对象。

## 函数执行上下文中的 this

重点分析函数执行上下文，先看代码

```javascript
function foo() {
  console.log(this);
}
foo();
```

这段代码打印出来的也是 window 对象，这说明在默认情况下调用一个函数，其执行上下文也是指向 window 对象的。但是我们可以设置执行上下文中的 this 来指向其他对象，通常有三种方式来设置函数执行上下文中的 this 值。

### 1、通过函数的 call 方法设置

可以通过函数的 call 方法来设置函数执行上下文的 this 指向，比如下面的这段代码，我们就并没有直接调用 foo 函数，而是调用了 foo 的call 函数，并将 bar 对象作为 call 方法的参数。

```javascript
let bar = {
  myName : " 极客邦 ",
  test1 : 1
}
function foo(){
  this.myName = " 极客时间 "
}
foo.call(bar)
console.log(bar)
console.log(myName)
```

执行这段代码，观察输出，就可以发现 foo 函数内部的 this 已经指向了 bar 对象，因为通过打印 bar 对象，可以看出 bar 的 myName 属性已经由 “极客邦” 变为 “极客时间” 了，同时在全局执行上下文中打印 myName，JavaScript 引擎提示该变量未定义。

其实除了 call 方法，还可以使用 bind 和 apply 方法来设置函数执行上下文的 this，它们在使用上还是有一些区别的。

### 2、通过对象调用方法设置

要改变函数执行上下文中的 this 指向，除了通过函数的 call 方法来实现以外，还可以通过对象调用的方式，比如下面这段代码：

```javascript
var myObj = {
  name : " 极客时间 ", 
  showThis: function(){
    console.log(this)
  }
}
myObj.showThis()
```

在这段代码中，我们定义了一个 myObj 对象，该对象是由一个 name 属性和一个 showThis 方法组成的，然后再通过 myObj 对象来调用 showThis 方法。执行这段代码，你可以看到，最终输出的 this 值是指向 myObj 的。

所以，你可以得出这样的结论：使用对象来调用其内部的方法，该方法的 this 是指向对象本身的。

也可以这样认为 JavaScript 引擎在执行 myObject.showThis( ) 时，将其转化为了：

```javascript
myObject.showThis.call(myObj)
```

接下来我们稍微改变下调用方式，把 showThis 赋给一个全局对象，然后再调用该对象，代码如下：

```javascript
var myObj = {
  name : " 极客时间 ",
  showThis: function(){
    this.name = " 极客邦 ";
    console.log(this);
  }
}
var foo = myObj.showName;
foo();
```

执行这段代码，你会发现 this 指向又是全局 window 对象。

所以通过以上两个例子的对比，你可以得出下面这样的结论：

在全局环境中调用一个函数，函数内部的 this 指向的是全局变量 window。

通过一个对象来调用其内部的一个方法，该方法的执行上下文中的 this 指向对象本身。

### 3、通过构造函数中设置

```javascript
function CreateObj() {
  this.name = "极客时间";
}
var myObje = new CreateObj();
```

在这段代码中，使用了 new 创建了对象 myObj。

其实，当执行 new CreateObj( ) 的时候，JavaScript 引擎做了如下四件事：

- 首先创建了一个空对象 tempObj；
- 接着调用 CreateObj.call 方法，并将 tempObj 作为 call 方法的参数，这样当 CreateObj 的执行上下文创建时，它的 this 就指向 tempObj 对象；
- 然后执行 CreateObj 函数，此时的 CreateObj 函数执行上下文中的 this 指向了 tempObj 对象；
- 最后返回 tempObj 对象；

为了直观理解，我们用代码来演示一下

```javascript
var tempObj = {}
CreateObj.call(tempObj)
return tempObj
```

这样，我们就通过 new 关键字构建好了一个新对象，并且构造函数中的 this 其实就是新对象本身。

## this 的设计缺陷以及应对方案

this 并不是一个很好的设计，因为它很多使用方法都冲击人的直觉，在使用过程中比较多坑，下面来看看 this 的设计缺陷

### 1、嵌套函数中的 this 不会从外层函数中继承

分析一段代码

```javascript
var myObj = {
  name : " 极客时间 ", 
  showThis: function(){
    console.log(this)
    function bar(){console.log(this)}
    bar()
  }
}
myObj.showThis()
```

我们在这段代码的 showThis 方法中添加了一个 bar 方法，然后接着在 showThis 函数中调用了 bar 函数，那么现在的问题是：bar 函数中的 this 是什么？

如果是刚刚接触 JavaScript，那么会很自然地觉得，bar 中的 this 应该和其外层 showThis 函数中的 this 是一致的都是指向 myObj 对象，这很符合人的直觉。但是实际情况却并非如此，执行这段代码后，发现函数 bar 中的 this 指向的是全局 window 对象，而函数 showThis 中的 this 指向的是 myObj 对象，这就是 JavaScript 中容易让人迷惑的地方之一，也是很多问题的源头。

可以通过一个小技巧来解决这一个问题，比如在 showThis 函数中声明一个变量 self 用来保存 this，然后在 bar 函数中使用 self，代码如下：

```javascript
var myObj = {
  name : " 极客时间 ", 
  showThis: function(){
    console.log(this)
    var self = this
    function bar(){
      self.name = " 极客邦 "
    }
    bar()
  }
}
myObj.showThis()
console.log(myObj.name)
console.log(window.name)
```

执行这段代码，你会发现它也输出了我们想要的结果，最终 myObj 中的 name 属性值变成了 “极客邦”。其实这个方法的本质就是把 this 体系转换成了作用域体系。

同样的也可以使用 ES6 中的箭头函数来解决这个问题，结合一下代码：

```javascript
var myObj = {
  name : " 极客时间 ", 
  showThis: function(){
    console.log(this)
    var bar = () => {
      this.name = " 极客邦 "
      console.log(this)
    }
    bar()
  }
}
myObj.showThis()
console.log(myObj.name)
console.log(window.name)
```

执行这段代码，会发现它也输出了我们想要的结果，也就是说箭头函数 bar 里面的 this 是指向 myObj 对象的。这是因为 ES6 中的箭头函数并不会创建其自身的执行上下文，所以箭头函数中的 this 取决于它的外部函数。

通过上面的讲解，你应该知道了 this 没有作用域的限制，这点和变量不一样，所以嵌套函数不会从调用它的函数中继承 this，这样会造成很多不符合直觉的代码。解决这个问题，有两种思路：

- 把 this 保存为一个 self 变量，再利用变量的作用域机制传递给嵌套函数。
- 继续使用 this，但是需要把嵌套函数改为箭头函数，因为箭头函数没有自己的执行上下文，所以它会继承调用函数中的 this。

### 2、普通函数中的 this 默认指向全局对象 window

在默认情况下，调用一个函数，其执行上下文中的 this 是默认指向全局对象 window 的。

不过这个设计也是一种缺陷，因为在实际工作中，我们并不希望函数执行上下文中的 this 默认指向全局对象，因为这样会打破数据边界，造成一些误操作。如果要让函数执行上下文中的 this 指向某个对象，最好的方式就是说通过 call 方法来实现调用。

这个问题可以通过设置 JavaScript 的 “严格模式” 来解决。在严格模式下，默认执行一个函数，其函数的执行上下文中的 this 值是undefined，这就解决上面的问题了。