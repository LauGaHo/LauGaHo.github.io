# JS：JavaScript 中有可能模拟出私有变量吗？

JavaScript 中谈私有属性和私有方法就是扯淡，不知道什么时候提上来真正的私有，我们来看看 JS 是如何根据当前特性来实现私有成员。

## 闭包

JavaScript 实现私有属性必须依赖闭包特性，看以下例子：

```javascript
var uniqueId;
  uniqueId = (function() {
    var index;
    index = 0;
    return function(prefix) {
      return prefix + "_" + index++;
    };
  })();
  //c_0
  console.log(uniqueId("c"));
  //c_1
  console.log(uniqueId("c"));
```

通常我们所说的或者看到的闭包就是这样子的 — (function() {}) ()，但这不是它的全部或者本质。在定义 uniqueId 这个函数的时候，我们使用了匿名函数表达式（请注意 (function( ){}) 是函数表达式）定义了一个函数且立即执行，把此时这个 function(prefix) {/*some code*/} 已经生成了函数实例，在函数实例生成的过程中：

1、通俗的讲，将 index 这个外部函数定义的变量记住了。

2、再次我们没法通过什么 this.index 或者 someObj.index 引用到 index，改变其值了，(function( ) { })( )这个一执行完，局部变量 index 在外面没有办法访问到。

3、怎么调得到，只能靠 function(prefix) {/*sone code*/}，因为我们还能通过它间接的取得或改变 index 值，这就是闭包了。

比较学术的解释：

1、JS 是词法作用域（就是程序上看上去是怎么样就是怎么样），使用一个叫 [[scope]] 的内部属性来标识每个执行上下文的作用域（我们可以读写哪些变量，调用哪些函数）；每个函数执行时都由该 [[scope]] 作用域加上活动对象构成真实的执行上下文。

2、而这个执行上下文 [[scope]] 属性是在函数生成时就指定的了。

3、于是 function(prefix){/*some code*/} 生成时其内部的 [[scope]] 属性引用了 (function( ){ })( ) 执行上下文的 scope 链；该 scope 链即包含了该函数的 [[scope]] 和活动对象，且活动对象包含了 index 的定义引用。

4、GC 的回收规则，没人用我，我就是垃圾。因此 uniqueId 引用了 function(prefix) {/*some code*/} 函数实例，而该函数的实例 [[scope]] 引用了 (function( ){ })( ) 执行期的 scope 链，其包含活动对象，即有 index 的引用；还有人引用它，它就不是垃圾，因此闭包就形成了，我们可以通过 uniqueId 函数间接地读取或者修改 index。

总结：其实学术解释和通俗解释都是一个意思，不过闭包其实是相对的，并不是我们不能修改 index，只是需要间接方法。

## 私有属性和私有方法

构造单例对象的私有属性和私有方法都比较简单。

```javascript
var aira;
aira = (function () {
    var __getName, __name;
    //private variable
    __name = "HTC mobile";
    //private method
    __getName = function () {
        return __name;
    };
    aira = {
        init: function () {
            //change private variable inner
            __name = "aira";
        },
        hello: function () {
            //execute private method inner
            console.log("hello,my name is " + (__getName()));
        }
    };
    return aira;
})();
aira.init();
//hello,my name is aira
aira.hello();
```

使用下划线 "\_" 表示私有；aira 手机有一个私有属性 "\_name" 和私有方法 "\_getName"；我们可以在 init 中修改 "\_name"，在 hello 中调用 "\_getName"，且在闭包外面无法直接调用和修改这两个成员。这样就可以实现私有变量和私有方法了。

但是确切的说，其实 aira 能够有私有属性和方法仅仅是因为它有一个私有的一个闭包，即 init 和 hello 成员的 [[scope]] 都引用了闭包的活动对象。

然而，一个构造函数（类）的私有属性和方法就是这么简单。

```javascript
var Phone, aira;
//wrap by function
Phone = function (name) {
    var phone;
    phone = (function () {
        var __getName, __name;
        __name = name;
        __getName = function () {
            return __name;
        };
        phone = {
            init: function (number) {
                __name += "#" + number;
            },
            hello: function () {
                console.log("hello,my name is " + (__getName()));
            }
        };
        return phone;
    })();
    return phone;
};
aira1 = Phone("aira");
aira1.init(1);
//hello,my name is aira#1
aira1.hello();

aira2 = Phone("aira");
aira2.init(2);
//hello,my name is aira#2
aira2.hello();
```

我们先来简单的将单例对象的构造包裹一个函数，实现产生不同的对象。我们可以说 Phone 是一个类，因为它可以产生不同的对象，有类似的功能。同样 aria1 和 aria2 都有自己的闭包，于是都有自己的私有属性和私有方法。

JS 中类的概念就是构造函数。

```javascript
var Phone, aira1, aira2;
Phone = function (name) {
    var __getName, __name;
    __name = name;
    __getName = function () {
        return __name;
    };
    this.init = function (number) {
        __name += "#" + number;
    };
    this.hello = function () {
        console.log("hello,my name is " + (__getName()));
    };
};
aira1 = new Phone("aira");
aira1.init(1);
//hello,my name is aira#1
aira1.hello();

aira2 = new Phone("aira");
aira2.init(1);
//hello,my name is aira#2
aira2.hello();
```

Phone 构造函数其实就是闭包的功能，每个 Phone 实例的 init 和 hello 都能引用其构造期间的形成的私有的 "\_name" 和 "\_getName"。

每个实例都必须由闭包产生私有属性和方法，因此只能在该闭包中定义公共方法暴露出来（比如说 init 和 hello），这就意味着每次构造一个实例我们都必须生成 init 和 hello 的函数实例。

```javascript
var Phone, aira;
Phone = function (name) {
    var __getName, __name;
    __name = name;
    __getName = function () {
        return __name;
    };
};
Phone.prototype.init = function (number) {
    __name += "#" + number;
};
Phone.prototype.hello = function () {
    console.log("hello,my name is " + (__getName()));
};
aira = new Phone("aira");
```

上面的代码是错误的（在 init 中 "\_name" 是全局的，hello 中的 "\_getName" 方法因为不存在，所以会报错），这就是问题所在，能够引用私有属性和变量的公共方法必须在闭包中定义，然后暴露出来，然而原型方法并不能在闭包中定义。

下面这段代码是私有方法吗？

```javascript
var Phone, aira1, aira2;
Phone = (function () {
    var __getName, __name;
    __getName = function () {
        return __name;
    };
    Phone = function (name) {
        __name = name;
    };
    Phone.prototype.init = function (number) {
        __name += "#" + number;
    };
    Phone.prototype.hello = function () {
        console.log("hello,my name is " + (__getName()));
    };
    return Phone;
})();
aira1 = new Phone("aira");
aira1.init(1);
//hello,my name is aira#1 right!
aira1.hello();
aira2 = new Phone("aira");
aira2.init(2);
//hello,my name is aira#2 right!
aira2.hello();
//hello,my name is aira#2 wrong!
aira1.hello();
```

试图用闭包包住构造函数，形成闭包，但是得到的结果是 "\_name" 和 "\_getName" 其实都是类的私有属性，而不是实例。aira1 和 aira2 共用了 "\_name" 和 "\_getName"。

再来确定一下什么是私有属性和私有方法，即每个类实例都拥有且只能在类内访问的变量和函数。也就是说变量和方法只能由类的方法来调用。说到这里，我们或许可以尝试一下，不让类外的方法调用类的私有方法。

```javascript
var inner, outer;
outer = function () {
    inner();
};
inner = function () {
    console.log(arguments.callee.caller);
};
/*
  function(){
      inner();
  }
  */
outer();
```

从 arguments 的 callee 中可获取当前的执行函数 inner，而 inner 的动态属性 caller 指向了调用 inner 的外层函数 outer，由此看来我们可以使用 arguments.callee.caller 来确定函数的执行环境，实现私有方法和属性。

```javascript
var Phone, aira1, aira2;
Function.prototype.__public = function (klass) {
    this.klass = klass;
    return this;
};
Function.prototype.__private = function () {
    var that;
    that = this;
    return function () {
        if (this.constructor === arguments.callee.caller.klass) {
            return that.apply(this, arguments);
        } else {
            throw new Error("" + that + " is a private method!");
        }
    };
};
Phone = function (name) {
    var __name;
    __name = name;
    this.__getName = (function () {
        return __name;
    }).__private();
    this.__setName = (function (name) {
        __name = name;
    }).__private();
};
Phone.prototype.init = (function (number) {
    this.__setName(this.__getName() + "#" + number);
}).__public(Phone);
Phone.prototype.hello = (function () {
    console.log("hello,my name is " + (this.__getName()));
}).__public(Phone);
aira1 = new Phone("aira");
aira1.init(1);
//hello,my name is aira#1
aira1.hello();
aira2 = new Phone("aira");
aira2.init(2);
//hello,my name is aira#2
aira2.hello();
//hello,my name is aira#1
aira1.hello();

try {
    aira1.__getName();
} catch (e) {
		/*
			Error Object
    				message:"function () {return __name;} is a private method!"
			*/
    console.log(e);
}
```

1、我给 Function 原型上添加了两个方法 \_public 和 \_private 以此来实现私有方法的调用环境测试；

2、其次我无法给私有属性添加检测，所以私有属性直接不可见，使用私有 get，set 方法访问；

3、本身在 aira1 外部调用时，我们还是能看到 \_getName 和 \_setName 方法，只是不能调用而已；

4、唯一好的一点是原型方法（公共方法）终于可以从构造函数闭包中解放出来了；