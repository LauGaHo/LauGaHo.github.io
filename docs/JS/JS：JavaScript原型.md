# JS：JavaScript 原型

## 引入：普通对象与函数对象

在 JavaScript 中，一直有一个说法，万物皆对象，事实上，对象也是有区别的，我们可以将其划分为 ***普通对象*** 和 ***函数对象***。Object 和 Function 便是 JavaScript 自带的两个典型的函数对象，而函数对象就是一个纯函数，所谓的函数对象，其实就是使用 JavaScript 在模拟类。

那么究竟什么是普通对象，什么是函数对象，请看下方例子：

```javascript
function fn1() {}
const fn2 = function() {}
const fn3 = new Function('language', 'console.log(language)')

const ob1 = {}
const ob2 = new Object()
const ob3 = new fn1()
```

打印以下结果，可以得到：

```javascript
console.log(typeof Object); // function
console.log(typeof Function); // function
console.log(typeof ob1); // object
console.log(typeof ob2); // object
console.log(typeof ob3); // object
console.log(typeof fn1); // function
console.log(typeof fn2); // function
console.log(typeof fn3); // function
```

在上述例子中，ob1、ob2、ob3为普通对象（均为 Object 的实例），而 fn1、fn2、fn3 均是 Function 的实例，称之为 函数对象。

如何区分，记住这句话就可以了：

所有的 Function 的实例都是函数对象，而其他的都是普通对象。

上面提到，Object 和 Function 均是函数对象，我们也说到了，所有的 Function 的实例都是函数对象，难道 Function 也是 Function 的实例？

保留这个疑问，我们做一个总结：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/S7IhPO.jpg)

从图中可以看出，对象本身的实现还是要依靠构造函数。那原型链到底是用来干嘛的？

众所周知，作为一门面向对象的语言，必定具有以下特征：

- 对象唯一性
- 抽象性
- 继承性
- 多态性

而原型链最大的目的，就是为了**实现继承**。

## 进阶：prototype 和 \_\_proto\_\_

原型链究竟是如何实现继承的？首先，先引入两兄弟：prototype 和 \_\_proto\_\_，这是在 JavaScript 中无处不在的两个变量（如果你经常调试的话），然而，这两个变量并不是在所有的对象上都存在，先看一张表：

​					prototype       	\_\_proto\_\_

普通对象			X					V

函数对象			V					V

首先，我们先给出结论：

1、只有函数对象具有 prototype 这个属性；

2、prototype 和 \_\_proto\_\_ 都是 JavaScript 在定义一个函数对象时自动创建的预定义属性；

接下来我们验证上述的两个结论：

```javascript
function fn() {}
console.log(typeof fn.__proto__); // function
console.log(typeof fn.prototype); // object

const ob = {}
console.log(typeof ob.__proto__); // function
console.log(typeof ob.prototype); // undefined，哇！果然普通对象没有 prototype
```

既然是语言层面的预置属性，那么两者究竟有什么区别？我们依然从结论出发，给出以下两个结论：

1、prototype 被实例的 \_\_proto\_\_ 所指向（被动）

2、\_\_proto\_\_ 指向构造函数的 prototype（主动）

也就是说以下代码成立：

```javascript
console.log(fn.__proto__ === Function.prototype); // true
console.log(ob.__proto__ === Object.prototype); // true
```

那么问题来了，既然 fn 是一个函数对象，那么 fn.prototype.\_\_proto\_\_到底等于什么？

```javascript
console.log(fn.prototype.__proto__ === Object.prototype) // true
```

创建一个函数时，JavaScript 对该函数原型的初始化代码如下：

```javascript
// 实际代码
function fn1() {}

// JavaScript 自动执行
fn1.protptype = {
    constructor: fn1,
    __proto__: Object.prototype
}

fn1.__proto__ = Function.prototype
```

普通对象就是通过函数对象实例化得到的，而一个实例不可能再次进行实例化，也就不会让另一个对象的 \_\_proto\_\_ 指向它的 prototype，因此普通对象没有 prototype 属性这个结论就非常好理解了。而且我们也看出来了，fn1.prototype 就是一个普通对象，它也不存在 prototype 属性。

我们在上面还留下了一个疑问

- 难道 Function 也是 Function 的实例？

```javascript
console.log(Function.__proto__ === Function.prototype) // true
```

## 重点：原型链

上面我们了解了 prototype 和 \_\_proto\_\_，实际上，这两兄弟主要就是为了构造原型链而存在的。

先看一段代码

```javascript
const Person = function(name, age) {
    this.name = name
    this.age = age
} /* 1 */

Person.prototype.getName = function() {
    return this.name
} /* 2 */

Person.prototype.getAge = function() {
    return this.age
} /* 3 */

const ulivz = new Person('ulivz', 24); /* 4 */

console.log(ulivz) /* 5 */
console.log(ulivz.getName(), ulivz.getAge()) /* 6 */
```

解释一下执行细节：

- 执行 1，创建了一个构造函数 Person，要注意，前面已经提到，此时 Person.prototype 已经被自动创建，它包含 constructor 和 \_\_proto\_\_这两个属性；
- 执行2，给对象 Person.prototype 增加了一个方法 getName()；
- 执行3，给对象 Person.prototype 增加了一个方法 getAge()；
- 执行4, 由构造函数 Person 创建了一个实例 ulivz，值得注意的是，一个构造函数在实例化时，一定会自动执行该构造函数。
- 在浏览器得到 5 的输出，即 ulivz 应该是：

```javascript
{
     name: 'ulivz',
     age: 24
     __proto__: Object // 实际上就是 `Person.prototype`
}
```

结合上面的结论，以下等式成立：

```javascript
console.log(ulivz.__proto__ == Person.prototype)  // true
```

- 执行6的时候，由于在 ulivz 中找不到 getName() 和 getAge() 这两个方法，就会继续朝着原型链向上查找，也就是通过 \_\_proto\_\_ 向上查找，于是，很快在 ulviz.\_\_proto\_\_ 中，即 Person.prototype 中找到了这两个方法，于是停止查找并执行得到结果。

***这便是 JavaScript 的原型继承。准确的说，JavaScript 的原型继承时通过 \_\_proto\_\_ 并借助 prototype 来实现的。***

于是便有以下总结：

1、函数对象的 \_\_proto\_\_ 指向 Function.prototype；

2、instance.\_\_proto\_\_ 指向函数对象的 prototype；

3、普通对象的 \_\_proto\_\_ 指向 Object.prototype；

4、普通对象没有 prototype 属性；

5、在访问一个对象的某个属性 / 方法的时候，如果在当前对象上没有找到，则会尝试 ob.\_\_proto\_\_，也就是访问该对象的构造函数的原型 obCtr.prototype，如果还是找不到，会继续查找 obCtr.prototype.\_\_proto\_\_，依次查找下去，若在某一刻，找到了该属性，则会立刻返回值并停止对原型链的搜索，若找不到，则返回 undefined。

为了加深理解，参考以下代码：

```javascript
console.log(ulivz.__proto__ == Function.prototype);	// false
console.log(Person.__proto__ == Function.prototype);	// true
console.log(Person.prototype.__proto__ == Object.prototype);	// true
console.log(Person.__proto__ == Object.prototype);	// false
console.log(Person.prototype.__proto__ == Function.prototype);	// false
```

## 终极：原型链图

上面我们还遗留了一个问题：

- 如果原型链一直找不到的话，那么什么时候停止，原型链的尽头在哪？

我们可以用代码验证一下：

```javascript
function Person() {}
const ulivz = new Person();
console.log(ulivz.name);
```

很明显，会输出 undefined。简述查找过程：

```javascript
ulivz                // 是一个对象，可以继续 
ulivz['name']           // 不存在，继续查找 
ulivz.__proto__            // 是一个对象，可以继续
ulivz.__proto__['name']        // 不存在，继续查找
ulivz.__proto__.__proto__          // 是一个对象，可以继续
ulivz.__proto__.__proto__['name']     // 不存在, 继续查找
ulivz.__proto__.__proto__.__proto__       // null !!!! 停止查找，返回 undefined
```

最后来看一下上一节的代码：

```javascript
const Person = function(name, age) {
    this.name = name
    this.age = age
} /* 1 */

Person.prototype.getName = function() {
    return this.name
} /* 2 */

Person.prototype.getAge = function() {
    return this.age
} /* 3 */

const ulivz = new Person('ulivz', 24); /* 4 */

console.log(ulivz) /* 5 */
console.log(ulivz.getName(), ulivz.getAge()) /* 6 */
```

我们来画一个原型链图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/MbRe8T.jpg)

画完这张图，所有疑问都已经得到了解答了。

## 调料：Constructor

前面已经有所提及了，只有原型对象才具有 constructor 这个属性，constructor 用来指向引用它的函数对象。

```javascript
Person.prototype.constructor === Person //true
console.log(Person.prototype.constructor.prototype.constructor === Person) //true
```

这是一种循环引用。

## 补充：JavaScript 中的 6 大内置（函数）对象的原型继承

通过前文的论述，结合相应的代码验证，整理出以下原型链图：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/dsJdGM.jpg)

由此可见，我们更加强化了这两个观点；

1、任何内置函数对象（类）本身的 \_\_proto\_\_ 都指向 Function 的原型对象；

2、除了 Object 的原型对象的 \_\_proto\_\_ 指向 null，其他所有内置函数对象的原型对象的 \_\_proto\_\_ 都指向 Object.prototype；

请看以下代码：

***Array***

```javascript
var arr = [];		
		console.log(arr.__proto__)
		console.log(arr.__proto__ == Array.prototype)   // true 
		console.log(Array.prototype.__proto__== Object.prototype)  // true 
		console.log(Object.prototype.__proto__== null)  // true
```

***RegExp***

```javascript
var reg = new RegExp;
    console.log(reg.__proto__)
    console.log(reg.__proto__ == RegExp.prototype)  // true 
    console.log(RegExp.prototype.__proto__== Object.prototype)  // true
```

***Date***

```javascript
var date = new Date;
    console.log(date.__proto__)
    console.log(date.__proto__ == Date.prototype)  // true 
    console.log(Date.prototype.__proto__== Object.prototype)  // true
```

***Boolean***

```javascript
var boo = new Boolean;
    console.log(boo.__proto__)
    console.log(boo.__proto__ == Boolean.prototype) // true 
    console.log(Boolean.prototype.__proto__== Object.prototype) // true
```

***Number***

```javascript
var num = new Number;
    console.log(num.__proto__)
    console.log(num.__proto__ == Number.prototype)  // true 
    console.log(Number.prototype.__proto__== Object.prototype)  // true
```

***String***

```javascript
var str = new String;
    console.log(str.__proto__)
    console.log(str.__proto__ == String.prototype)  // true 
    console.log(String.prototype.__proto__== Object.prototype)  // true
```

## 总结

- 若 A 通过 new 创建了 B，那么 B.\_\_proto\_\_ == A.prototype；
- \_\_proto\_\_ 是原型链查找的起点；
- 执行 B.a，若在 B 中找不到 a，则会在 B.\_\_proto\_\_ 中，也就是 A.prototype 中查找，若 A.prototype 中仍然没有，则会继续向上查找，最终，一定会找到 Object.prototype，倘若还找不到，因为 Object.prototype.\_\_proto\_\_ 指向 null，因此一定会返回 undefined。