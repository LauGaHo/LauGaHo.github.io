# JS：深入理解 JavaScript - 执行上下文

JS 每次执行回调函数，会把方法以 **执行上下文** 的方式压入 **执行栈** ，执行完会被弹出执行栈。

## 执行上下文

而 **执行上下文** 的结构如下图所示：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/Wm3r0E.jpg)

从上图可以了解到，执行上下文分为了两个环境，一个是 **变量环境（VariableEnvironment）**，另一个是 **词法环境（LexicalEnvironment）**，其中这两个环境之间的差别就在于，**变量环境** 是登记对应的 **var、function** 的声明。而另外的 **词法环境** 是用来登记对应的 **let、const、class** 等变量声明。**词法环境** 的出现是为了实现块级作用域的同时不影响 **var、function** 声明。

## 变量提升

先看一段代码，观察其输出结果是什么

```javascript
showName();
console.log(myname);
var myname = "极客时间";
function showName() {
  console.log("函数showName被执行");
}
// 函数showName被执行
// undefined
```

按照顺序执行的逻辑来看，这段代码是无法执行的，但是结果这段代码不但没有报错，并且能够正常输出。

出现这个非正常的现象的原因就在于一段 JavaScript 代码在执行之前需要被 JavaScript 引擎编译，编译完成后，才会进入执行阶段。

## 编译阶段

### 第一部分：变量提升的代码。

```javascript
var myname = undefined;
function showName() {
  console.log("函数showName被执行");
}
```

### 第二部分：执行部分代码。

```javascript
showName();
console.log(myname);
myname = "极客时间"
```

下图就是把 Javascript 的执行流程细化，如下图所示：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/vNRkwQ.jpg)

你可以简单认为变量环境对象是如下结构：

```javascript
VariableEnvironment:
     myname -> undefined, 
     showName ->function : {console.log(myname)
```

我们详细分析一下代码是如何生成变量环境对象的。

```javascript
showName()
console.log(myname)
var myname = '极客时间'
function showName() {
    console.log('函数showName被执行');
}
```

- 第1行和第2行，由于这两行代码不是声明操作，所以JavaScript引擎不会做任何处理；
- 第3行，由于这行是经过var声明的，因此JavaScript引擎将在环境对象中创建一个名为myname的属性，并使用undefined对其初始化；
- 第4行，JavaScript引擎发现了一个通过function定义的函数，所以它将函数定义存储到堆(HEAP）中，并在环境对象中创建一个showName的属性，然后将该属性值指向堆中函数的位置（不了解堆也没关系，JavaScript的执行堆和执行栈我会在后续文章中介绍）。 这样就生成了变量环境对象。接下来JavaScript引擎会把声明以外的代码编译为字节码，至于字节码的细节，我也会在后面文章中做详细介绍，你可以类比如下的模拟代码

```javascript
showName()
console.log(myname)
myname = '极客时间'
```

## 执行阶段

- 当执行到showName函数时，JavaScript引擎便开始在变量环境对象中查找该函数，由于变量环境对象中存在该函数的引用，所以JavaScript引擎便开始执行该函数，并输出“函数showName被执行”结果。
- 接下来打印“myname”信息，JavaScript引擎继续在变量环境对象中查找该对象，由于变量环境存在myname变量，并且其值为undefined，所以这时候就输出undefined。
- 接下来执行第3行，把“极客时间”赋给myname变量，赋值后变量环境中的myname属性值改变为“极客时间”，变量环境如下所示：

```javascript
VariableEnvironment:
     myname -> "极客时间", 
     showName ->function : {console.log(myname)
```

