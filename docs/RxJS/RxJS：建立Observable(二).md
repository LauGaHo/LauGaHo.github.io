# RxJS：建立 Observable (二)

## 创建运算符 `Creation Operator`

Observable 有许多创建实例的方法，称为创建运算符。下面是常用的创建运算符：

- create
- of
- from
- fromEvent
- fromPromise
- never
- empty
- throw
- interval
- timer

### of

Observable 最基本的创建方式是使用 `create` 来进行创建，代码如下：

```typescript
var source = Rx.Observable
    .create(function(observer) {
        observer.next('Jerry');
        observer.next('Anna');
        observer.complete();
    });

source.subscribe({
    next: function(value) {
    	console.log(value)
    },
    complete: function() {
    	console.log('complete!');
    },
    error: function(error) {
    console.log(error)
    }
});

// Jerry
// Anna
// complete!
```

这里先后传递了 `'Jerry'`, `'Anna'` 然后结束 `complete`，当我们想要同步的传递几个值，就可以用 `of` 这个 `operator` 来简洁的表达。

下面代码效果同上：

```typescript
var source = Rx.Observable.of('Jerry', 'Anna');

source.subscribe({
    next: function(value) {
        console.log(value)
    },
    complete: function() {
        console.log('complete!');
    },
    error: function(error) {
        console.log(error)
    }
});

// Jerry
// Anna
// complete!
```

### from

`of` 这个 operator 的参数其实就是一个 `list`，而 `list` 在 JavaScript 中最常见的形式是数组，那有没有什么方法是直接传一个已知的数组当做参数？

使用 `from` 运算符，该运算符可以接受任何可列举的参数：

```typescript
var arr = ['Jerry', 'Anna', 2016, 2017, '30 days'] 
var source = Rx.Observable.from(arr);

source.subscribe({
    next: function(value) {
        console.log(value)
    },
    complete: function() {
        console.log('complete!');
    },
    error: function(error) {
        console.log(error)
    }
});

// Jerry
// Anna
// 2016
// 2017
// 30 days
// complete!
```

注意，任何可列举的参数都可以使用，也就是说像 `Set`, `WeakSet`, `Iterator` 等都可当做参数。

另外 `from` 还能接收字符串 (string) ，如下：

```typescript
var source = Rx.Observable.from('铁人赛');

source.subscribe({
    next: function(value) {
        console.log(value)
    },
    complete: function() {
        console.log('complete!');
    },
    error: function(error) {
        console.log(error)
    }
});
// 铁
// 人
// 赛
// complete!
```

上面的代码会把字符串中的字一一印出来。

我们也可以传入 Promise 事件，如下：

```typescript
var source = Rx.Observable
  .from(new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve('Hello RxJS!');
    },3000)
  }))

source.subscribe({
    next: function(value) {
    	console.log(value)
    },
    complete: function() {
    	console.log('complete!');
    },
    error: function(error) {
    console.log(error)
    }
});

// Hello RxJS!
// complete!
```

如果我们传入 Promise 事件实例，当正常回传时，就会被传送到到 `next`，并立即发送完成通知，如果有错误，则会发送到 error 中。

> 这里也可以使用 `fromPromise`，结果一样。

### fromEvent

我们也可以使用 Event 来建立 Observable，通过 `fromEvent` 方法，如下：

```typescript
var source = Rx.Observable.fromEvent(document.body, 'click');

source.subscribe({
    next: function(value) {
        console.log(value)
    },
    complete: function() {
        console.log('complete!');
    },
    error: function(error) {
        console.log(error)
    }
});

// MouseEvent {...}
```

`fromEvent` 的第一个参数要传入 DOM 事件，第二个参数传入要监听的时间名称。上面的代码会针对 `body` 的 `click` 事件做监听，每当点击 `body` 就会打印出 `event`。

> 取得 DOM 事件的常用方法： `document.getElementById()` `document.querySelector()` `document.getElementsByTagName()` `document.getElementsByClassName()`

补充： `fromEventPattern`

要用 Event 来建立 Observable 实例还有另外一个方法，就是 `fromEventPattern`，这个方法是给类事件使用。所谓的类事件就是指其行为跟事件相像，同时具有注册监听及移除监听两种行为，就像 DOM Event 有 `addEventListener` 和 `removeEventListener` 一样。代码如下：

```typescript
class Producer {
	constructor() {
		this.listeners = [];
	}
	addListener(listener) {
		if(typeof listener === 'function') {
			this.listeners.push(listener)
		} else {
			throw new Error('listener 必须是 function')
		}
	}
	removeListener(listener) {
		this.listeners.splice(this.listeners.indexOf(listener), 1)
	}
	notify(message) {
		this.listeners.forEach(listener => {
			listener(message);
		})
	}
}
// ------- 以上都是之前的代码 -------- //

var egghead = new Producer(); 
// egghead 同时具有 注册观察者及移除观察者 两种方法

var source = Rx.Observable
    .fromEventPattern(
        (handler) => egghead.addListener(handler), 
        (handler) => egghead.removeListener(handler)
    );

source.subscribe({
    next: function(value) {
        console.log(value)
    },
    complete: function() {
        console.log('complete!');
    },
    error: function(error) {
        console.log(error)
    }
})

egghead.notify('Hello! Can you hear me?');
// Hello! Can you hear me
```

上面的代码可以看到，`egghead` 是 `Producer` 的实例，同时具有注册监听和移除监听的两种方法，我们可以把这两个方法传入 `fromEventPattern` 来建立 Observable 的事件实例。

> 这里注意不要直接将方法传入，避免 this 出错，也可以用 `bind` 来写。

```typescript
Rx.Observable
    .fromEventPattern(
        egghead.addListener.bind(egghead), 
        egghead.removeListener.bind(egghead)
    )
    .subscribe(console.log)
```

### empty、never、throw

接下来我们来看几个比较无趣的 operator，之后我们会讲到很多 observables 合并 (combine)、转换 (transform) 的方法，到那个时候无趣的 observable 也会很有用。

有点像数学上的 0，虽然有时候没什么，但却非常重要。在 Observable 的世界也有类似的东西，就是 `empty`。

```typescript
var source = Rx.Observable.empty();

source.subscribe({
    next: function(value) {
        console.log(value)
    },
    complete: function() {
        console.log('complete!');
    },
    error: function(error) {
        console.log(error)
    }
});
// complete!
```

`empty` 会给我们一个空的 observable，如果我们订阅这个 observable，它会立即发送 complete 信息。

数学上还有一个东西跟 0 很像，那就是无穷 ∞，在 Observable 的世界我们用 `never` 来建立无穷的 Observable。

```typescript
var source = Rx.Observable.never();

source.subscribe({
    next: function(value) {
        console.log(value)
    },
    complete: function() {
        console.log('complete!');
    },
    error: function(error) {
        console.log(error)
    }
});
```

`never` 会给我们一个无穷的 observable，它就是一个一直存在但却什么都不做的 observable。

最后还有一个 operator `throw`，它也就只做一件事就是抛出错误。

```typescript
var source = Rx.Observable.throw('Oop!');

source.subscribe({
	next: function(value) {
		console.log(value)
	},
	complete: function() {
		console.log('complete!');
	},
	error: function(error) {
    console.log('Throw Error: ' + error)
	}
});
// Throw Error: Oop!
```

上面这段代码之后 log 出 `'Throw Error Oop'`

### interval、timer

JS 中的 `setInterval` 在 RxJS 中也有对应的 operator `interval`

```typescript
var source = Rx.Observable.create(function(observer) {
    var i = 0;
    setInterval(() => {
        observer.next(i++);
    }, 1000)
});

source.subscribe({
	next: function(value) {
		console.log(value)
	},
	complete: function() {
		console.log('complete!');
	},
	error: function(error) {
    console.log('Throw Error: ' + error)
	}
});
// 0
// 1
// 2
// .....
```

在 Observable 的世界也有一个更方便的 operator 可以做到这件事，就是 `interval`

```typescript
var source = Rx.Observable.interval(1000);

source.subscribe({
	next: function(value) {
		console.log(value)
	},
	complete: function() {
		console.log('complete!');
	},
	error: function(error) {
    console.log('Throw Error: ' + error)
	}
});
// 0
// 1
// 2
// ...
```

`interval` 有一个参数必须是数值 (Number)，这个数值代表发出信号的间隔时间 (ms)。这两段代码基本等价，会持续每隔一秒发送一个从零开始递增的数值

另外一个很相似的 operator 叫 `timer`，`timer` 可以给出两个参数：

```typescript
var source = Rx.Observable.timer(1000, 5000);

source.subscribe({
	next: function(value) {
		console.log(value)
	},
	complete: function() {
		console.log('complete!');
	},
	error: function(error) {
    console.log('Throw Error: ' + error)
	}
});
// 0
// 1
// 2 ...
```

当 `timer` 有两个参数时，第一个参数是发出第一个值的等待时间 (ms)，第二个参数代表第一次之后发送值的间隔时间，所以上面这段代码会等一秒发送 1，之后每 5 秒发送 2、3、4、5......

`timer` 第一个参数除了可以是数值 (Number) 之外，也可以是日期 (Date)，就会等到指定的时间发送第一个值。

另外 `timer` 也可以只接收一个参数

```typescript
var source = Rx.Observable.timer(1000);

source.subscribe({
	next: function(value) {
		console.log(value)
	},
	complete: function() {
		console.log('complete!');
	},
	error: function(error) {
    console.log('Throw Error: ' + error)
	}
});
// 0
// complete!
```

上面这段代码就会等一秒后发送 1 同时通知结束。

### Subscription

我们有时候需要在某些行为之后不需要这些资源，要做到这件事最简单的方式是 `unsubscribe`。

其实在订阅 Observable 后，会回传一个 Subscription 事件，这个事件具有释放资源的 `unsubscribe` 方法，如下：

```typescript
var source = Rx.Observable.timer(1000, 1000);

// 取得 subscription
var subscription = source.subscribe({
	next: function(value) {
		console.log(value)
	},
	complete: function() {
		console.log('complete!');
	},
	error: function(error) {
    console.log('Throw Error: ' + error)
	}
});

setTimeout(() => {
    subscription.unsubscribe() // 停止订阅(退订)， RxJS 4.x 以前的版本用 dispose()
}, 5000);
// 0
// 1
// 2
// 3
// 4
```

这里我们用到了 `setTimeout` 在 5 秒之后，执行了 `subscription.unsubscribe( )` 来停止订阅并释放资源，另外 subscription 事件还有其他合并订阅等作用，这个之后会提到。