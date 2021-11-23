# JS：箭头函数

先来看下 ES6 中箭头函数的基本语法：

```javascript
let func = value => value;
```

相当于

```javascript
let func = function (value) {
    return value
}
```

如果需要传入多个参数：

```javascript
let func = (value, num) => value * num;
```

上面箭头函数例子中都省略了 return 关键字和代码的花括号，在箭头函数中如果方法体中只有一行的代码，可以省略关键字和方法的花括号，直接简化成 value => value。

如果函数的代码块需要多条语句：

```javascript
let func = (value, num) => {
    return value * num;
}
```

如果需要返回一个对象，箭头函数的方法体必须放在大括号 ( ) 中，这样做的原因是：没有大括号，JS 引擎没办法区分是正常定义一个对象还是一个箭头函数体：

```javascript
let func = (value, num) => ({ value: value, num: num }); //正确写法

let func = (value, num) => { value: value, num: num }; //会报错
```

## 与普通函数的区别

### 没有 this

箭头函数式的 this 需要通过查找作用域链来确定，它的 this 是指包在它外面的作用域的 this，我们来看下以下代码中的 this 分别指的是：

```javascript
const obj = {
    a: function () {
        console.log(this); // obj
    },
    b: () => {
        console.log(this); // windows
    }
}
```

obj.b 是一个箭头函数，它的 this 是包在外层的词法作用域的 this ，obj 对象不是可执行代码，所以它不是离箭头函数最近的词法作用域，再往外就是全局作用域 window 了，所以 obj.b 的 this 指的是 windows。

再来看一段代码：

```javascript
var pageHandler = {

    id: "123456",

    init: function () {
        document.addEventListener("click", function (event) {
            this.doSomething(event.type);     // error
        }, false);
    },

    doSomething: function (type) {
        console.log("Handling " + type + " for " + this.id);
    }
};
```

在调用 pageHandler.init 方法的时候会报错，报错的函数是个回调函数，这个回调函数的 this 指的是全局变量 windows，在全局变量里没有 doSomething 这个方法，所以会报错，有两种方式处理这种错误：

第一种方式就是通过 bind 来指定 this：

```javascript
var pageHandler = {
    id: "123456",
    init: function () {
        document.addEventListener("click", (function (event) {
            this.doSomething(event.type)
        }).bind(this), false)

    },
    doSomething: function () {
        console.log("Handling" + type + " for " + this.id)
    }
}
```

.bind(this) 中的 this 是指 pageHandler 这个对象，通过 bind 生成一个 this 指向 pageHandler 的新函数，这样执行 init 方法的时候不会报错，看起来有点奇怪。

第二种方式就是通过箭头函数：

```javascript
var pageHandler = {
    id: "12345",
    init: function () {
        document.addEventListener("click", () => {
            this.doSomething();
        }, false)
    },
    doSomething: function () {
        console.log("Handling" + type + " for " + this.id);
    }
}
```

我们把 init 中的回调函数改成了箭头函数，箭头函数的 this 是它最近的作用域链上的 this，也就是 init 这个方法的 this，也就是 pageHandler 这个对象，这样就可以达到目的不报错。

***需要注意的是，箭头函数不能改变 this 的值，普通函数可以通过 call、apply、bind 来指定 this，但是箭头函数的 this 是不能改变的。***

***上面所说，箭头函数的 this 不能改变某种程度上是不准确的，因为箭头函数中的 this 其实是跟随其外一层的函数的 this，所以如果外层函数中的 this 改变了，对应里边的箭头函数中的 this 指向也是跟随着外层的函数的 this 的。***

观察下方的代码：

```javascript
var pageHandler = {
    id: "12345",
    init: function () {
        var func = () => {
          console.log(this);
        }
    },
    doSomething: function () {
        console.log("Handling" + type + " for " + this.id);
    }
}

pageHandler.init();	// pageHandler

var testFunc = pageHandler.init;
testFunc.init();	// window
```

上方代码中的两种调用方式，得到的结果却不一样。

第一种方式，通过对象来调用方法，相当于把 pageHandler 绑定到了 init 方法中，然后 init 方法中的 箭头函数中的 this 是跟随 init 方法中的 this，因而箭头函数中的 this 也是指向 pageHandler 的。

第二种方式，通过将对象中的方法赋值给 testFunc 变量，然后在调用 testFunc( )，这时 testFunc 函数中的 this 指针未被指定，因而是默认值 window。 然后 testFunc 方法中的箭头函数中的 this 是跟随外一层的 this 指针，故箭头函数中的 this 也是指向 window 的。

由此可以看出，箭头函数的 this 指针并非完全不能够改变的，因为箭头函数的 this 指向是跟随外一层函数的 this 指针的，所以如果外一层函数的 this 指针改变了，对应的，其可执行代码中的箭头函数的 this 指针也会跟随改变。

### 没有 arguments

访问箭头函数的 arguments，其实也是访问包在它外面的非箭头函数的 arguments。

```javascript
function foo(x, y) {
    return () => arguments[0];
}
console.log(foo(1,2)()); //1
```

### 不能通过 new 关键字调用

```javascript
var Foo = () =>{};
var foo = new Foo(); // TypeError: Foo is not a constructor
```

### 没有原型

没有 prototype，但是有 \_\_proto\_\_ 指向 function。

