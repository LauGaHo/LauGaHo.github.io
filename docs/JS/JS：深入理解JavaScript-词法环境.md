# JS：深入理解JavaScript - 词法环境

JS引擎中的栈桢有一个专业名称叫做**执行上下文 (Execution Context)**，紧接着就是一大串的名词：Lexical Environment Execution Context 变量对象 作用域链 原型链 this 闭包等等。

## Lexical Environment

V8引擎JS的编译执行过程，大致上分为了三个阶段：

- 第一步：V8刚拿到执行上下文的时候，会将代码从上到下一行一行的先做分词 / 词法分析。分词指的是：比如 var a =    2；这段代码，会被分词成：var a 2和；这样的原子符号；词法分析是指：登记变量声明，函数声明，函数声明的行参。
- 第二步：在分词结束后，会做代码解析，引擎将token解析翻译成一个AST (抽象语法树)，这一步的时候，如果发现语法错误，就会直接报错，不再往下执行。
- 第三步：引擎生成 CPU 可以执行的机器码

在第一步有个词法分析，它用来登记变量声明，函数声明，函数声明的形参，后续代码执行的时候就知道去哪里拿变量的值和函数了，这个登记的地方就是**Lexical Environment (词法环境)**

词法环境有两个组成部分：

- **1：环境记录（Environment Record）**，这个就是真正登记变量的地方
  - **1.1：声明式环境记录（Declarative Environment Record）**，用来记录直接有标识符定义的元素，比如变量、常量、let、class、module、import以及函数声明。
  - **1.2：对象式环境记录（Object Environment Record）**，主要用于with、global的词法环境。
- **2：对外部词法环境的引用（outer）**，它是作用域链能够连起来的关键。

其中 **声明式环境记录（Declarative Environment Record）**，又分为两种类型：

- **函数环境记录（Function Environment Record）**：用于函数作用域。
- **模块环境记录（Module Environment Record）**：模块环境记录用于体现一个模块的外部作用域，即模块 export 所在环境

词法环境与我们自己写的代码结构相对应，也就是我们自己代码写成什么样子，词法环境就成什么样子。词法环境式在代码定义的时候决定的，跟代码在哪调用没有关系，所以说JavaScript采用的是词法作用域（静态作用域）。

我们来看个例子：

```javascript
var a = 2;
let x = 1;
const y = 5;

function foo() {
    console.log(a);

    function bar() {
        var b = 3;
        console.log(a * b);
    }

    bar();
}
function baz() {
    var a = 10;
    foo();
}
baz();
```

它的词法环境关系图如下：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/qvxwBg.jpg)

我们可以用伪代码来模拟上面代码的词法环境：

```javascript
// 全局词法环境
GlobalEnvironment = {
    outer: null, //全局环境的外部环境引用为null
    GlobalEnvironmentRecord: {
        //全局this绑定指向全局对象
        [[GlobalThisValue]]: ObjectEnvironmentRecord[[BindingObject]],
        //声明式环境记录，除了全局函数和var，其他声明都绑定在这里
        DeclarativeEnvironmentRecord: {
            x: 1,
            y: 5
        },
        //对象式环境记录，绑定对象为全局对象
        ObjectEnvironmentRecord: {
            a: 2,
            foo:<< function>>,
            baz:<< function>>,
            isNaNl:<< function>>,
            isFinite: << function>>,
            parseInt: << function>>,
            parseFloat: << function>>,
            Array: << construct function>>,
            Object: << construct function>>
            ...
            ...
        }
    }
}
//foo函数词法环境
fooFunctionEnviroment = {
    outer: GlobalEnvironment,//外部词法环境引用指向全局环境
    FunctionEnvironmentRecord: {
        [[ThisValue]]: GlobalEnvironment,//this绑定指向全局环境
        bar:<< function>> 
    }
}
//bar函数词法环境
barFunctionEnviroment = {
    outer: fooFunctionEnviroment,//外部词法环境引用指向foo函数词法环境
    FunctionEnvironmentRecord: {
        [[ThisValue]]: GlobalEnvironment,//this绑定指向全局环境
        b: 3
    }
}

//baz函数词法环境
bazFunctionEnviroment = {
    outer: GlobalEnvironment,//外部词法环境引用指向全局环境
    FunctionEnvironmentRecord: {
        [[ThisValue]]: GlobalEnvironment,//this绑定指向全局环境
        a: 10
    }
}
```

我们可以看到词法环境和我们定义代码定义一一对应，每个词法环境都有一个outer指向上一层的词法环境，当运行上方的代码，函数 bar 的词法环境里没有变量 a，所以就会到它的上一层词法环境去找（foo 词法环境）。foo 函数词法环境中也没有 a。然后沿着 foo词法环境一直往上找，在全局词法环境中找到了 var a = 2，沿着 outer 一层层找变量的值就是作用域链。如果找到第一个就会停止，如果在全局词法环境还是找不到，就会停止查找并返回 null，因为全局词法环境里的 outer 是 null。就会报 ReferenceError。

## 变量提升 vs 函数提升

V8 引擎执行代码分为三步，先做分词和词法分析，然后解析生成AST，最后生成机器码执行代码，词法分析时会生成词法环境登记变量，对于变量声明和函数声明，词法环境的处理不一样的。

在词法分析的时候：

- 对于变量声明，**var a = 2；**，**let x = 1；**，给变量分配内存并初始化为 undefined，赋值语句是在第三步生成机器码真正执行代码的时候才执行的。
- 对于函数声明，**function foo ( ) { . . . }**，会在内存里创建函数对象，并且直接初始化为该函数对象。

这就是 JS 的**变量提升和函数提升**，我们来看个例子：

```javascript
var c;

function functionDec() {
    console.log(c)
    c = 30;
}

functionDec();
```

最后运行的结果是：undefined

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/5eLpjb.jpg)

如果整个变量就没有定义，如下：

```javascript
function functionDec() {
    console.log(c)
    c = 30;
}

functionDec();
```

运行代码，结果如下：

```javascript
Uncaught ReferenceError: c is not defined
    at functionDec (<anonymous>:4:17)
    at <anonymous>:8:1
```

