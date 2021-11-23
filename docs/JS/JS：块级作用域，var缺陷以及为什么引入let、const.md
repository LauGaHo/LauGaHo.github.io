# JS：块级作用域，var缺陷以及为什么引入let、const

## 作用域（scope）

**作用域是指在程序中定义变量的区域，该位置决定了变量的声明周期，通俗地理解，作用域就是变量与函数的可访问范围，即作用域控制着变量和函数的可见性和生命周期。**

在 ES6 之前，ES 的作用域只有两种，全局作用域和函数作用域。

- 全局作用域中的对象在代码中的任何地方都能访问，其声明周期伴随着页面的声明周期。
- 函数作用域就是在函数内部定义的变量或者函数，并且定义的变量或函数只能在函数内部被访问。函数执行结束后，函数内部定义的变量就会被销毁。

在 ES6 之前，JavaScript 只支持这两种作用域，相较而言，其他语言则普遍支持块级作用域。块级作用域就是使用一对大括号包裹的代码，比如函数、判断语句、循环语句、甚至单独的一个 { } 都可以被看作一个块级作用域。

为了更好地理解块级作用域，参考下面的代码：

```javascript
//if块
if(1){}

//while块
while(1){}

//函数块
function foo(){
 
//for循环块
for(let i = 0; i<100; i++){}

//单独一个块
{}
```

简单来说，如果一个语言支持块级作用域，那么其代码块内部定义的变量在代码块外是访问不到的，并且等该代码块执行完成后，代码块中定义的变量会被销毁。

## 变量提升所带来的问题

由于变量提升的问题，使用 JavaScript 来编写和其他语言相同的逻辑的代码，都有可能产生不一样的结果。主要有以下两种原因

### 1、变量容易在不被察觉的情况下被覆盖掉

比如我们使用 JavaScript 实现以下代码

```javascript
var myname = "极客时间"
function showName(){
  console.log(myname);
  if(0){
   var myname = "极客邦"
  }
  console.log(myname);
}
showName()
```

打印出来的结果是 undefined 。

直接展示最终的调用栈状态如下图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/iZeQCb.jpg)

showName 函数的执行上下文创建后，JavaScript 引擎便开始执行 showName 函数内部的代码了。首先执行的是：

```javascript
console.log(myname);
```

JavaScript 会优先从当前的执行上下文中查找变量，由于变量提升，所以当前执行上下文就包括了变量 myname，并且值是 undefined。所以获取得到的值就是 undefined。

### 2、本应销毁的变量没有销毁

再看看这一段误解更大的代码：

```javascript
function foo(){
  for (var i = 0; i < 7; i++) {
  }
  console.log(i); 
}
foo();
```

如果使用 C语言或者其他大部分的语言实现类似的代码，在for循环结束之后，i 就已经被销毁了，但是在JavaScript 代码中，i 的值未被销毁，所以最后打印出来的值为 7 。

## ES6 如何解决变量提升带来的问题

为了解决变量提升带来的问题，ES6 引入了 let 和 const 关键字，使得 JavaScript 也像其他语言一般拥有了块级作用域。

```javascript
let x = 5
const y = 6
x = 7
y = 9 //报错，const声明的变量不可以修改
```

从这段代码中，可以看出使用 let 关键字的变量可以改变值，使用 const 关键字的变量则不可以改变的。

参考一段存在变量提升的代码：

```javascript
function varTest() {
  var x = 1;
  if (true) {
    var x = 2;  // 同样的变量!
    console.log(x);  // 2
  }
  console.log(x);  // 2
}
```

这段代码有两个地方定义了变量 x，由于使用 var 关键字，在编译阶段会生成如下的执行上下文

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/6r82N9.jpg)

最终只生成了一个 x 值，并且函数体内所有对 x 的赋值操作都会直接改变变量环境中的 x 值。所以上述代码最后通过console.log( x )输出的应该是 2 。而其他语言最后一步输出的应该是 1。如果要让他支持块级作用域只需要将 var 关键字改成 let 关键字即可。

```javascript
function letTest() {
  let x = 1;
  if (true) {
    let x = 2;  // 不同的变量
    console.log(x);  // 2
  }
  console.log(x);  // 1
}
```

执行这段代码的输出结果就和我们的预期是一致的。

## JavaScript 是如何支持块级作用域的

观察以下代码

```javascript
function foo(){
    var a = 1
    let b = 2
    {
      let b = 3
      var c = 4
      let d = 5
      console.log(a)
      console.log(b)
    }
    console.log(b) 
    console.log(c)
    console.log(d)
}   
foo()
```

当执行上面这段代码的时候，JavaScript 引擎线对其进行编译并创建上下文，然后再按照顺序执行代码，关于如何创建上下文已经分析过了，但是现在的情况有点不一样，let 关键字会创建块级作用域，那么 let 关键字如何影响执行上下文的，接下来一步步分析

**第一步、编译并创建执行上下文**

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/ZuNPJi.jpg)

通过上图，可以得出结论：

- 函数内部通过 **var** 声明的变量，在编译阶段全被存到变量环境当中。
- 通过 **let** 声明的变量，在编译阶段会存放到 **词法环境（Lexical Environment）**中。
- 在函数的作用域内部，通过 **let** 声明的变量没有被存放到词法环境中。
- 继续执行代码，当执行到代码块中，变量环境中的a已经设置为1，词法环境中的b设置为了2。

这时函数的执行上下文就如下图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/yjmTSJ.jpg)

从图中可以看出，当进入了函数的作用域块时，作用域块中通过 let 声明的变量，会被存放在词法环境的一个单独的区域中，这个区域中的变量并不影响作用域块外面的变量，比如在作用域外面声明了变量 b，在该作用域块内部也声明了变量 b，当执行到作用域内部时，它们都是独立的存在。

其实，在词法环境内部，维护了一个小型栈结构，栈底是最外层的变量，进入了一个作用域中，就会把该作用域内部的变量压到栈顶；当作用域执行完成之后，该作用域的信息就会从栈顶弹出，这就是词法环境的结构。需要注意一下，我这里所讲的变量是指通过 let 或者 const 声明的变量。

再接下来，当执行到作用域块中的console.log( a )这行代码时，就需要在词法环境和变量环境中查找变量 a 的值了，具体查找方法：沿着词法环境的栈顶向下查询，如果在词法环境中的某个块中查找到了，就直接返回给 JavaScript 引擎，如果没有找到，那么继续在变量环境中查找。

这样一个变量查找的过程就完成了，你可以参考下图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/ChGluo.jpg)

从上图可以清晰看到查找变量的流程，不过要完整理解查找变量或者查找函数的流程，就需要涉及到作用域链了。

当作用域块执行结束之后，其内部定义的变量就会从词法环境的栈顶弹出，最终执行上下文如图所示：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/vpNfIR.jpg)