# JS：作用域链和闭包，代码出现相同的变量，JS引擎如何选择

理解作用域链时理解闭包的基础，而闭包在 JavaScript 中几乎无处不在，同时作用域和作用域链还是所有编程语言的基础。

首先先来看这一段代码：

```javascript
function bar() {
    console.log(myName)
}
function foo() {
    var myName = " 极客邦 "
    bar()
}
var myName = " 极客时间 "
foo()
```

你觉得这段代码中的 bar 函数和 foo 函数打印出来的是什么吗？

分析代码，当代码执行到 bar 函数内部，调用栈的状态如图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/7amvbs.jpg)

从图中可以看出，全局执行上下文和 foo 函数的执行上下文中都包含变量 myName，那 bar 函数中的 myName应该选择哪一个呢？这里就需要了解作用域链了。

## 作用域链

其实每一个执行上下文的变量环境中，都包含了一个外部引用，用来指向外部的执行上下文，我们把这个外部因你用哪个称为 outer。

当一段代码使用了一个变量，JavaScript 引擎首先会在“当前执行上下文”中查找该变量，比如上面的那段代码在查找 myName 变量时，如果在当前的变量环境中没有查到，那么 JavaScript 引擎会继续在 outer 所指向的执行上下文中查找。为了直观理解，你可以直接看下图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/zRdlhw.jpg)

从图中可以看到，bar 函数和 foo 函数的 outer 都是指向全局上下文的，这也意味着如果在 bar 函数或者 foo 函数中使用了外部变量，那么 JavaScript 引擎就会去全局执行上下文中查找。我们把这个查找的链条就称作为作用域链。

## 词法作用域

***词法作用域就是指作用域是由代码中函数声明的位置来决定的，所以词法作用域是静态的作用域，通过它就能够预测代码在执行过程中如何查找标识符。***

可以直接看下图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/zcmf66.jpg)

从图中可以看出，词法作用域就是根据代码的位置来决定的，其中 main 函数包含了 bar 函数，bar 函数包含了 foo 函数，因为 JavaScript 作用域链是由词法作用域决定的，所以整个词法作用域链的顺序是：foo 函数作用域 -> bar 函数作用域 -> main 函数作用域 -> 全局作用域。

回过头看上边的代码，foo 函数调用了 bar 函数，但是 bar 函数的外部引用是全局执行上下文。

因为词法作用域，foo 和 bar 的上级作用域都是全局作用域，所以如果 foo 或者 bar 函数使用了一个没有定义的变量，那么它们就会到全局作用域中寻找，故，词法作用域是代码阶段就决定好的，和函数的调用没有关系。

## 块级作用域中的变量查找

```javascript
function bar () {
  var myName = " 极客世界 "
    let test1 = 100
    if (1) {
        let myName = "Chrome 浏览器 "
        console.log(test)
    }
}
function foo() {
    var myName = " 极客邦 "
    let test = 2
    {
        let test = 3
        bar()
    }
}
var myName = " 极客时间 "
let myAge = 10
let test = 1
foo()
```

ES6 是支持块级作用域的，当执行到代码块时，如果代码块中有 let 或者 const 声明的变量，那么变量就会存放到该函数的词法环境中，对于上面这段代码，当执行到 bar 函数内部的 if 语句块时，调用栈的情况如下图所示：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/下载.png)

现在是执行到 bar 函数的 if 语句块之内，需要打印出来的变量 test，那么久需要查找到 test 变量的值，其查找过程我已经在上图中使用了序号1、2、3、4、5标记出来了。

下面我就来解释一下这个过程。首先是在 bar 函数的执行上下文中查找，但因为 bar 函数的执行上下文中没有 定义 test 变量，所以根据词法作用域的规则，下一步就在 bar 函数的外部作用域中查找，也就是全局作用域。

## 闭包

了解了作用域链，接着就可以了解闭包了。关于闭包，理解起来稍微有一点难度。结合以下代码就可以理解什么是闭包了：

```javascript
function foo() {
    var myName = " 极客时间 "
    let test1 = 1
    const test2 = 2
    var innerBar = {
        getName:function(){
            console.log(test1)
            return myName
        },
        setName:function(newName){
            myName = newName
        }
    }
    return innerBar
}
var bar = foo()
bar.setName(" 极客邦 ")
bar.getName()
console.log(bar.getName())
```

首先我们看看当执行到 foo 函数内部的 return innerBar 这行代码时，调用栈的情况你可以参考此图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/uPznOY.jpg)

从上面的代码可以看出，innerBar 是一个对象，包含了 getName 和 setName 的两个方法（通常我们把对象内部的函数称为方法）。你可以看到，这两个方法都是在 foo 函数内部定义的，并且这两个方法内部都使用了 myName 和 test1 两个变量。

根据词法作用域的规则，内部函数 getName 和 setName 总是可以访问它们的外部函数 foo 中的变量，所以当 innerBar 对象返回给全局变量 bar 时，虽然 foo 函数结束了，但是 getName 和 setName 函数依然可以使用 foo 函数中的变量 myName 和 test1。所以当 foo 函数执行完成之后，其整个调用栈的状态如下图所示：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/20NJwJ.jpg)

从上图可以看出，foo 函数执行完成之后，其执行上下文从栈顶弹出了，但是由于返回的 setName 和 getName 方法中使用了 foo 函数内部的变量 myName 和 test1，所以这两个变量依然存在在内存当中。这像极了 setName 和 getName 方法背的一个专属背包，无论在哪里调用 setName 和 getName 方法，它们都会背着这个 foo 函数的专属背包。

之所以时专属背包，是因为除了 setName 和 getName 函数之外，其他任何地方都是无法访问该背包的，我们就可以把这个背包称为 foo 函数的闭包。

最后给闭包完整的定义，在 JavaScript 中，根据词法作用域的规则，内部函数总是可以访问其外部函数中声明的变量，当通过一个外部函数返回一个内部函数后，即使该外部函数已经执行结束了，但是内部函数引用外部函数的变量依然保存在内存中，我们就把这些变量的集合统称为闭包。比如外部函数 foo，那么这些变量的集合就称为 foo 函数的闭包。

闭包的使用。当执行到 bar.setName 方法中的 myName = "极客邦" 这句代码时，JavaScript 引擎会沿着 “当前执行上下文 -> foo 函数闭包 -> 全局执行上下文” 的顺序来查找 myName 变量，你可以参考下面调用栈状态图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/RxMYdM.jpg)

从图中可以看出，setName 的执行上下文中没有 myName 变量，foo 函数的闭包中包含了变量 myName，所以当调用 setName 时，会修改 foo 闭包中的 myName 变量的值。

同样的流程，当调用 bar.getName 的时候，所访问的变量 myName 也是位于 foo 函数闭包中的。

可以通过 “开发者工具” 来看看闭包的情况，打开 Chrome 的 “开发者工具”，在 bar 函数任意地方打上断点，然后刷新页面，可以看到如下内容：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/cEAEuG.jpg)

从图中可以看出来，当调用 bar.getName 的时候，右边 Scope 就体现了作用域链的情况：Local 就是当前的 getName 函数的作用域，Closure( foo )是指 foo 函数的闭包，最下面的 Global 就是指全局作用域，从 “Local -> Closure( foo ) -> Global” 就是一个完整的作用域链。

所以，可以通过 Scope 来查看实际代码作用域链的情况。



## 闭包的本质

一般来说，当某个函数被调用时，会创建 <span style="background: yellow">一个执行环境</span>  (excution context) 及 <span style="background: yellow">相应的作用域链</span> 。然后使用 <span style="background: green">arguments</span> 和 <span style="background: green">其他命名参数的值</span> 来初始化 函数的 <span style="background: green">活动对象 (activation object)</span> 。

```javascript
function compare(value1, value2) {
  if (value1 < value2) {
    return -1;
  } else if (value1 > value2) {
    return 1;
  } else {
    return 0;
  }
}

var result = compare(5, 10);
```

代码的作用域链如下图所示：

1、<span style="color: green">活动对象</span> 包含 <span style="color: green">arguments</span> 和 <span style="color: green">参数</span> 

2、全局 <span style="color: orange">变量对象</span> 包含 <span style="color: orange">函数名</span> 和 <span style="color: orange">变量</span> 

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/iRrx6Q.jpg)

在创建 `compare()` 函数时，会创建一个预先包含全局变量对象的作用域链，这个作用域链被保存在内部的 `[[Scope]]` 属性中，当调用 `compare()` 函数时，会为函数创建一个执行环境，然后通过复制函数的 `[[Scope]]` 属性中的对象来构建起执行环境的作用域链。此后，又有一个活动对象 (在此作为变量对象使用) 被创建并且被推入执行环境作用域的前端。对于这个例子中的 `compare()` 函数的执行环境而言，其作用域链中包含两个变量对象：**本地活动对象**和**全局变量对象**。显然，作用域链本质上是一个指向变量对象的指针列表，它只引用但不实际包含变量对象！！！

无论什么时候在函数中访问一个变量时，就会从作用域链中搜索具有相应名字的变量。一般来讲，当函数执行完毕后，局部活动对象就会被销毁，内存只能够仅保存全局作用域 (全局执行环境的变量对象) 。但是，**闭包的情况又有所不同** 。看下面的代码：

```javascript
function createComparisonFunction(propertyName) {
  return function(object1, object2) {
    var value1 = object1[propertyName];
    var value2 = object2[propertyName];
    
    if (value1 < value2) {
      return -1;
    } else if (value1 > value2) {
      return 1;
    } else {
      return 0;
    }
  };
}
```

在另一个函数内部定义的函数会将包含函数 (即外部函数) 的活动对象添加到它的作用域链中，因此，在 `createComparisonFunction()` 函数内部定义的匿名函数的作用域链中，实际上将会包含外部函数 `createComparisonFunction()` 的活动对象。(活动对象包含 arguments 和 参数)

```javascript
var compareNames = createComparisonFunction("name");
var result = compareNames({name:"Nicholas"}, {name:"Greg"});
```

在匿名函数从 `createComparisonFunction()` 中被返回后，他的作用域链被初始化为包含 `createComparisonFunction()` 函数的活动对象和全局变量对象。这样，匿名函数就可以访问在 `createComparisonFunction()` 中定义的所有变量。更为重要的是，`createComparisonFunction()` 函数在执行完毕后，其活动对象也不会被销毁，因为匿名函数的作用域链仍然在引用这个活动对象，换句话说，当 `createComparisonFunction()` 函数返回后，其执行环境的作用域链就会被销毁，但他的活动对象还会留在内存；直到匿名函数被销毁后，`createComparisonFunction()` 的活动对象才会被销毁，例如：

```javascript
// 创建函数，这里 返回的 是一个 匿名函数 赋值给 compareNames
var compareNames = createComparisonFunction("name");
// 调用 匿名函数
var result = compareNames({name:"Nicholas"}, {name:"Greg"});

// 解除 对 匿名函数 的 引用（以便 释放 内存）
compareNames = null;
```

首先，`createComparisonFunction()` 返回的匿名函数保存在变量 `compareNames` 中，而通过将 `compareNames` 设置为 `null` 解除该函数的引用，就等于通知垃圾回收线程将其清除。随着匿名函数的作用域被销毁，其他作用域 (除了全局作用域) 也都可以安全地销毁了。



## 闭包的回收

通常如果引用闭包的函数是一个全局变量，那么闭包会一直存在直到页面关闭；但是如果这个闭包以后不再使用就会导致内存泄漏。

所以在使用闭包的时候，尽量注意一个原则：如果该闭包会一直使用，那么它可以作为全局变量而存在；但如果使用频率不高，而且占用内存比较大的话，那就尽量让它成为一个局部变量。
