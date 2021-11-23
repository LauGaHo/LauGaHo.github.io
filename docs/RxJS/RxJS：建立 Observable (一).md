---
title: RxJS：建立 Observable (一)
date: 2021/7/7 11:51:00
tag: [RxJS]
---

# RxJS：建立 Observable (一)

## 建立 Observable：`create`

建立 Observable 的方法有非常多种，其中 `create` 是最基本的方法。`create` 方法在 `Rx.Observable` 事件中，要传入一个 `callback function`，这个 `callback function` 会接收一个 `observer` 参数，如下：

```typescript
var observable = Rx.Observable
	.create(function(observer) {
		observer.next('Jerry'); // RxJS 4.x 以前的版本用 onNext
		observer.next('Anna');
	})
```

这个 `callback function` 会定义 `observable` 将如何发送值。

> 虽然 `Observable` 可以被 `create`，但实际上我们通常都使用 create operator 像是 `from`, `of`, `fromEvent`, `fromPromise` 等。这里只是为了从基本的开始讲解才用 `create`。

我们可以订阅这个 `observable`，来接收他发送的值，代码如下：

```typescript
var observable = Rx.Observable
	.create(function(observer) {
		observer.next('Jerry'); // RxJS 4.x 以前的版本用 onNext
		observer.next('Anna');
	})

// 订阅这个 observable	
observable.subscribe(function(value) {
	console.log(value);
})
```

当我们订阅这个 `observable`，就会依序发送 `'Jerry'`, `'Anna'` 两个字符串。

> 订阅 `Observable` 跟 `addEventListener` 在实例上其实有非常大的不同。虽然行为上很像，但实际上 `Observable` 根本没有管理一个订阅清单，最后会说明

这里有一个重点，很多人认为，RxJS 是在做非同步处理，所以所有行为都是非同步的。但其实这个观念是错误的，RxJS 确实主要在处理非同步的行为没错，但也同时能处理同步行为，像是上面的代码就是同步执行的。

证明如下：

```typescript
var observable = Rx.Observable
	.create(function(observer) {
		observer.next('Jerry'); // RxJS 4.x 以前的版本用 onNext
		observer.next('Anna');
	})

console.log('start');
observable.subscribe(function(value) {
	console.log(value);
});
console.log('end');
```

上面这段代码会打印出

```typescript
start
Jerry
Anna
end
```

而不是

```typescript
start
end
Jerry
Anna
```

所以明显的，这段代码是同步执行的，当然我们也可以拿它来处理非同步的行为：

```typescript
var observable = Rx.Observable
	.create(function(observer) {
		observer.next('Jerry'); // RxJS 4.x 以前的版本用 onNext
		observer.next('Anna');

		setTimeout(() => {
			observer.next('RxJS 30 days!');
		}, 30)
	})

console.log('start');
observable.subscribe(function(value) {
	console.log(value);
});
console.log('end');
```

这样就会打印：

```typescript
start
Jerry
Anna
end
RxJS 30 days!
```

从上可见，`Observable` 同时可以处理同步与非同步的行为

## 观察者 `Observer`

`Observable` 可以被订阅 `subscribe`，或说可以说被观察，而订阅 `Observable` 的事件又称为观察者 `Observer`。观察者是一个具有三个方法的事件，每当 `Observable` 发生事件时，便会呼叫观察者相对应的方法。

> 这里说的观察者 `Observer` 跟上一篇讲的观察者模式无关，观察者模式是一种设计模式，是思考问题的解决过程，而这里讲的观察者是一个被定义的事件。

观察者的三个方法：

- `next`：每当 `Observable` 发送新的值，`next` 方法就会被呼叫

- `complete`：在 `Observable` 没有其他资料可以取得时，`complete` 方法就会被呼叫，在 `complete` 被呼叫之后，`next` 方法就不会再起作用。
- `error`：每当 `Observable` 内发生错误时，`error` 方法就会被呼叫。

直接建立一个观察者：

```typescript
var observable = Rx.Observable
	.create(function(observer) {
			observer.next('Jerry');
			observer.next('Anna');
			observer.complete();
			observer.next('not work');
	})

// 建立一个观察者，具备 next, error, complete 三个方法
var observer = {
	next: function(value) {
		console.log(value);
	},
	error: function(error) {
		console.log(error)
	},
	complete: function() {
		console.log('complete')
	}
}

// 用我们定义好的观察者，来订阅这个 observable	
observable.subscribe(observer)
```

上面这段代码印出

```typescript
Jerry
Anna
complete
```

执行完 `complete` 执行后，`next` 就会自动失效。

下面则是发送错误的示例

```typescript
var observable = Rx.Observable
  .create(function(observer) {
    try {
      observer.next('Jerry');
      observer.next('Anna');
      throw 'some exception';
    } catch(e) {
      observer.error(e)
    }
  });

// 建立一个观察者，具备 next, error, complete 三个方法
var observer = {
	next: function(value) {
		console.log(value);
	},
	error: function(error) {
		console.log('Error: ', error)
	},
	complete: function() {
		console.log('complete')
	}
}

// 用我们定义好的观察者，来订阅这个 observable	
observable.subscribe(observer)
```

这里就会执行 `error` 的 function 印出 `Error：some exception`。

另外观察者可以是不完整的，可以只具有一个 `next` 方法，如下：

```typescript
var observer = {
	next: function(value) {
		//...
	}
}
```

> 有时候 Observable 会是一个无限的序列，例如 `click` 事件，这时 `complete` 方法就有可能永远不会被呼叫！

我们也可以直接把 `next`, `error`, `complete` 三个 function 依序传入 `Observable.subscribe`，如下：

```typescript
observable.subscribe(
    value => { console.log(value); },
    error => { console.log('Error: ', error); },
    () => { console.log('complete') }
)
```

`Observable.subscribe` 会在内部自动生成 Observer 事件来操作。

## 实例细节

我们前边提到了，其实 `Observable` 的订阅和 `addEventListener` 在实例上有挺大的差异，虽然他们的行为很像

`addEventListener` 本质上就是观察者模式的实例，在内部会有一份订阅清单。

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
```

我们在内部存储了一份所有的观察者清单 `this.listener`，在要发布通知时会逐一的呼叫这份清单的观察者。

但在 Observable 不是这样的实例，在内部并没有一份订阅者的清单。订阅 Observable 的行为比较像是执行一个事件的方法，并把资料传进这个方法中。

以下代码做说明：

```typescript
var observable = Rx.Observable
	.create(function (observer) {
			observer.next('Jerry');
			observer.next('Anna');
	})

observable.subscribe({
	next: function(value) {
		console.log(value);
	},
	error: function(error) {
		console.log(error)
	},
	complete: function() {
		console.log('complete')
	}
})
```

像上面这段代码，他的行为比较像这样：

```typescript
function subscribe(observer) {
		observer.next('Jerry');
		observer.next('Anna');
}

subscribe({
	next: function(value) {
		console.log(value);
	},
	error: function(error) {
		console.log(error)
	},
	complete: function() {
		console.log('complete')
	}
});
```

这里可以看到 `subscribe` 是一个 function，这个 function 执行时会传入观察者，而我们在这个 function 内部去执行观察者的方法。

订阅一个 `Observable` 就像是执行一个 function。

## 总结

- Observable 可以同时处理同步与非同步行为
- Observer 是一个事件，这个事件具有三个方法，分别是：`next`, `error`, `complete`
- 订阅一个 Observable 就像在执行一个 function