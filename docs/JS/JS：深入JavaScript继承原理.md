# JS：深入 JavaScript 继承原理

## 类

我们回顾一下 ES6 /  TypeScript /  ES5 类的写法以做对比。首先我们创建一个 GithubUser 类，它拥有一个 login 方法，和一个静态方法 getPublicServices，用于获取 public 的方法列表：

```javascript
class GithubUser {
    static getPublicServices() {
        return ['login']
    }
    constructor(username, password) {
        this.username = username
        this.password = password
    }
    login() {
        console.log(this.username + '要登录Github，密码是' + this.password)
    }
}
```

实际上，ES6 这个类的写法有一个弊病，密码 password 应该是 Github 用户一个私有变量，接下来用 TypeScript 重写一下：

```typescript
class GithubUser {
    static getPublicServices() {
        return ['login']
    }
    public username: string
    private password: string
    constructor(username, password) {
        this.username = username
        this.password = password
    }
    public login(): void {
        console.log(this.username + '要登录Github，密码是' + this.password)
    }
}
```

如此一来，password 就只能在类的内部访问了。

结合原型讲解那一篇文章讲解的知识，来用 ES5 来实现这个类：

```javascript
function GithubUser(username, password) {
    // private属性
    let _password = password 
    // public属性
    this.username = username 
    // public方法
    GithubUser.prototype.login = function () {
        console.log(this.username + '要登录Github，密码是' + _password)
    }
}
// 静态方法
GithubUser.getPublicServices = function () {
    return ['login']
}
```

***值得注意的是，我们一般都会把共有的方法放在类的原型上，而不会采用 this.login = function () {} 这种写法。因为只有这样，才能让多个实例引用同一个共有方法，从而避免了重复创建方法的浪费。***

留下两个疑问：

1、如何实现 private 方法

2、能否实现 protected 属性 / 方法 

## 继承 

如果创建了一个 JuejinUser 来继承 GithubUser，那么 JuejinUser 及其实例就可以调用 Github 的login 方法了。首先，先写出这个简单的 JuejinUser 类：

```javascript
function JuejinUser(username, password) {
    // TODO need implementation
    this.articles = 3 // 文章数量
    JuejinUser.prototype.readArticle = function () {
        console.log('Read article')
    }
}
```

先概述几种继承方法：

- 类式继承
- 构造函数式继承
- 组合式继承
- 原型继承
- 寄生式继承
- 寄生组合式继承

## 类式继承

***若通过 new Parent( ) 创建了 Child，则 Child.\_\_proto\_\_ = Parent.prototype，而原型链则是顺着 \_\_proto\_\_ 依次向上查找。因此，可以通过修改 子类的原型为父类的实例来实现继承***

第一直觉的实现如下：

```javascript
function GithubUser(username, password) {
    let _password = password 
    this.username = username 
    GithubUser.prototype.login = function () {
        console.log(this.username + '要登录Github，密码是' + _password)
    }
}

function JuejinUser(username, password) {
    this.articles = 3 // 文章数量
    JuejinUser.prototype = new GithubUser(username, password)
    JuejinUser.prototype.readArticle = function () {
        console.log('Read article')
    }
}

const juejinUser1 = new JuejinUser('ulivz', 'xxx', 3)
console.log(juejinUser1)
```

在浏览器中查看原型链：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/WMixMl.jpg)

从上图可以看出，明显 juejinUser1.\_\_proto\_\_ 并不是 GithubUser 的一个实例。

实际上，JuejinUser.prototype 在定义函数的时候就已经将其指针指向对应的地址，利用构造函数实例化对象的时候，同样地也将 JuejinUser.prototype 当前指向的地址赋给了 juejinUser.\_\_proto\_\_ 这个变量，然而，在构造函数中，将 JuejinUser.prototype 指向了一个新对象的地址，就造成了 JuejinUser.prototype 和 juejinUser.\_\_proto\_\_的指向不一致，一个指向新对象，一个指向旧对象。所以重新赋值一下实例的 \_\_proto\_\_ 就可以解决这个问题：

```javascript
function GithubUser(username, password) {
    let _password = password 
    this.username = username 
    GithubUser.prototype.login = function () {
        console.log(this.username + '要登录Github，密码是' + _password)
    }
}

function JuejinUser(username, password) {
    this.articles = 3 // 文章数量
    const prototype = new GithubUser(username, password)
    // JuejinUser.prototype = prototype // 这一行已经没有意义了
    prototype.readArticle = function () {
        console.log('Read article')
    }
    this.__proto__ = prototype
}

const juejinUser1 = new JuejinUser('ulivz', 'xxx', 3)
console.log(juejinUser1)
```

接着查看原型链：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/71V7S6.jpg)

原型链已经出来了，问题好想解决了，但实际上还有问题：

1、在原型链上创建了属性（这不是一个好的实践）

2、私自篡改 \_\_proto\_\_，导致 juejinUser1.\_\_proto\_\_ === JuejinUser.prototype 不成立！从而导致了 juejinUser1 instanceof JuejinUser也不成立。

造成这个问题的根本原因就在于，我们在实例化的时候动态修改了原型，那有没有一种方法可以在实例化之前就固定好类的原型的  reference 呢？

事实上，我们可以考虑把类的原型的赋值挪出来：

```javascript
function JuejinUser(username, password) {
    this.articles = 3 // 文章数量
}

// 此时构造函数还未运行，无法访问 username 和 password !!
JuejinUser.prototype =  new GithubUser() 

prototype.readArticle = function () {
    console.log('Read article')
}
```

但是这样做又有更明显的缺点：

1、父类过早被创建，导致无法接受子类的动态参数；

2、仍然在原型上创建了属性，此时，多个子类的实例将共享一个父类属性，会互相影响；

举例说明缺点2：

```javascript
function GithubUser(username) {
    this.username = 'Unknown' 
}

function JuejinUser(username, password) {}

JuejinUser.prototype =  new GithubUser() 
const juejinUser1 = new JuejinUser('ulivz', 'xxx', 3)
const juejinUser2 = new JuejinUser('egoist', 'xxx', 0)

//  这就是把属性定义在原型链上的致命缺点，你可以直接访问，但修改就是一件难事了！
console.log(juejinUser1.username) // 'Unknown'
juejinUser1.__proto__.username = 'U' 
console.log(juejinUser1.username) // 'U'

// 卧槽，无情地影响了另一个实例!!!
console.log(juejinUser2.username) // 'U'
```

由此可见，类式继承的两种方式缺陷太多！

## 构造函数式继承

通过 call( ) 来实现继承（相应的，你也可以用 apply）：

```javascript
function GithubUser(username, password) {
    let _password = password 
    this.username = username 
    GithubUser.prototype.login = function () {
        console.log(this.username + '要登录Github，密码是' + _password)
    }
}

function JuejinUser(username, password) {
    GithubUser.call(this, username, password)
    this.articles = 3 // 文章数量
}

const juejinUser1 = new JuejinUser('ulivz', 'xxx')
console.log(juejinUser1.username) // ulivz
console.log(juejinUser1.username) // xxx
console.log(juejinUser1.login()) // TypeError: juejinUser1.login is not a function
```

当然，如果继承那么简单，那么本文就没有存在的必要了，本继承方法也有一个明显的缺陷，那就是 ***构造函数式继承*** 并没有继承父类原型上的方法。

## 组合式继承

既然上述两种方法都各有缺点，又各有所长，能否将其结合起来，这种方式就叫做 ***组合式继承***。

```javascript
function GithubUser(username, password) {
    let _password = password 
    this.username = username 
    GithubUser.prototype.login = function () {
        console.log(this.username + '要登录Github，密码是' + _password)
    }
}

function JuejinUser(username, password) {
    GithubUser.call(this, username, password) // 第二次执行 GithubUser 的构造函数
    this.articles = 3 // 文章数量
}

JuejinUser.prototype = new GithubUser(); // 第二次执行 GithubUser 的构造函数
const juejinUser1 = new JuejinUser('ulivz', 'xxx')
```

虽然这种方式弥补了上述两种方式的一些缺陷，但是仍然存在着问题：

1、子类仍旧无法动态传递参数给父类

2、父类的构造函数被调用了两次

本方法很明显执行了两次父类的构造函数，因此，这也不是我们最终想要的继承方式。

## 原型继承

原型继承实际上是对类式继承的一种封装，只不过其独特之处在于，定义了一个干净的中间类，如下：

```javascript
function createObject(o) {
    // 创建临时类
    function f() {
        
    }
    // 修改类的原型为o, 于是f的实例都将继承o上的方法
    f.prototype = o
    return new f()
}
```

熟悉 ES5 的同学会注意到，这不就是 Object.create( ) 吗？确实可以这样认为。

既然是类式继承的一种封装，其使用方式自然就是如下：

```javascript
JuejinUser.prototype = createObject(GithubUser)
```

也就是仍然没有解决类式继承的一些问题。

## 寄生继承

寄生继承是依托于一个对象而生的一种继承方式，因此称之为寄生。

```javascript
const juejinUserSample = {
    username: 'ulivz',
    password: 'xxx'
}

function JuejinUser(obj) {
    var o = Object.create(obj)
     o.prototype.readArticle = function () {
        console.log('Read article')
    }
    return o;
}

var myComputer = new CreateComputer(computer);
```

由于实际生产中，继承一个单例对象的场景实在是太少了，因此还没有找到最佳的实现方法。

## 寄生组合式继承

先上代码

```javascript
// 寄生组合式继承的核心方法
function inherit(child, parent) {
    // 继承父类的原型
    const p = Object.create(parent.prototype)
    // 重写子类的原型
    child.prototype = p
    // 重写被污染的子类的constructor
    p.constructor = child
}

// GithubUser, 父类
function GithubUser(username, password) {
    let _password = password 
    this.username = username 
}

GithubUser.prototype.login = function () {
    console.log(this.username + '要登录Github，密码是' + _password)
}

// GithubUser, 子类
function JuejinUser(username, password) {
    GithubUser.call(this, username, password) // 继承属性
    this.articles = 3 // 文章数量
}

// 实现原型上的方法
inherit(JuejinUser, GithubUser)

// 在原型上添加新方法
JuejinUser.prototype.readArticle = function () {
    console.log('Read article')
}

const juejinUser1 = new JuejinUser('ulivz', 'xxx')
console.log(juejinUser1)
```

浏览器查看结果：

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/k1AGHC.jpg)

简单说明一下：

- 子类继承了父类的属性和方法，同时属性没有被创建在原型链上，因此多个子类不会共享同一个属性。
- 子类可以传递动态参数给父类。
- 父类构造函数只执行了一次。

然而，还是存在一个美中不足的问题：

- 子类想要在原型上添加方法，必须在继承之后添加，否则将覆盖原有原型上的方法，这样的话，如果已经存在的两个类，就不好办了。

## 终极版继承

为了让代码更清晰，我用 ES6 的一些 API，写出了这个我认为最合理的继承方法：

- 用 Reflect 代替了 Object；
- 用 Reflect.getPrototypeOf 来代替 ob.\_\_proto\_\_
- 用 Reflect.ownKeys 来读取所有可枚举 / 不可枚举 / Symbol 的属性
- 用 Reflect.getOwnPropertyDescriptor 读取属性描述符
- 用 Reflect.setPrototypeOf 来设置 \_\_proto\_\_

源代码如下：

```javascript
/*!
 * fancy-inherit
 * (c) 2016-2018 ULIVZ
 */
 
// 不同于object.assign, 该 merge方法会复制所有的源键
// 不管键名是 Symbol 或字符串，也不管是否可枚举
function fancyShadowMerge(target, source) {
    for (const key of Reflect.ownKeys(source)) {
        Reflect.defineProperty(target, key, Reflect.getOwnPropertyDescriptor(source, key))
    }
    return target
}

// Core
function inherit(child, parent) {
    const objectPrototype = Object.prototype
    // 继承父类的原型
    const parentPrototype = Object.create(parent.prototype)
    let childPrototype = child.prototype
    // 若子类没有继承任何类，直接合并子类原型和父类原型上的所有方法
    // 包含可枚举/不可枚举的方法
    if (Reflect.getPrototypeOf(childPrototype) === objectPrototype) {
        child.prototype = fancyShadowMerge(parentPrototype, childPrototype)
    } else {
        // 若子类已经继承子某个类
        // 父类的原型将在子类原型链的尽头补全
        while (Reflect.getPrototypeOf(childPrototype) !== objectPrototype) {
		childPrototype = Reflect.getPrototypeOf(childPrototype)
        }
	Reflect.setPrototypeOf(childPrototype, parent.prototype)
    }
    // 重写被污染的子类的constructor
    parentPrototype.constructor = child
}
```

测试：

```javascript
// GithubUser
function GithubUser(username, password) {
    let _password = password
    this.username = username
}

GithubUser.prototype.login = function () {
    console.log(this.username + '要登录Github，密码是' + _password)
}

// JuejinUser
function JuejinUser(username, password) {
    GithubUser.call(this, username, password)
    WeiboUser.call(this, username, password)
    this.articles = 3
}

JuejinUser.prototype.readArticle = function () {
    console.log('Read article')
}

// WeiboUser
function WeiboUser(username, password) {
    this.key = username + password
}

WeiboUser.prototype.compose = function () {
    console.log('compose')
}

// 先让 JuejinUser 继承 GithubUser，然后就可以用github登录掘金了
inherit(JuejinUser, GithubUser) 

// 再让 JuejinUser 继承 WeiboUser，然后就可以用weibo登录掘金了
inherit(JuejinUser, WeiboUser)  

const juejinUser1 = new JuejinUser('ulivz', 'xxx')

console.log(juejinUser1)

console.log(juejinUser1 instanceof GithubUser) // true
console.log(juejinUser1 instanceof WeiboUser) // true
```

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/KQi6Uj.jpg)

## 总结：

- 我们可以使用 function 来模拟一个类
- JavaScript 类的继承是基于原型的，一个完善的继承方法，其继承过程是相当复杂的
- 虽然建议实际生产中直接使用 ES6 的继承，但仍建议深入内部继承机制
- 在 ES6 中，默认所有类都是不可枚举的
