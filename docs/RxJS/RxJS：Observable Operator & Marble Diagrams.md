---
title: RxJS：Observable Operator & Marble Diagrams
date: 2021/7/7 11:51:00
tag: [RxJS]
---

# RxJS：Observable Operator & Marble Diagrams

## 什么是 Operator

Operator 就是一个个被附加到 Observable 的函数，例如：map、filter、concatAll 等等，所有这些函数都会拿到原本的 Observable 并回传一个新的 Observable。

```typescript
var people = Rx.Observable.of('Jerry', 'Anna');

function map(source, callback) {
    return Rx.Observable.create((observer) => {
        return source.subscribe(
            (value) => { 
                try{
                    observer.next(callback(value));
                } catch(e) {
                    observer.error(e);
                }
            },
            (err) => { observer.error(err); },
            () => { observer.complete() }
        )
    })
}

var helloPeople = map(people, (item) => item + ' Hello~');

helloPeople.subscribe(console.log);
// Jerry Hello~
// Anna Hello~
```

这里可以看到写了一个 `map` 函数，它接收两个参数，第一个是原本的 `Observable`，第二个是 `map` 的 `callback function`。`map` 内部第一件事就是用 `create` 建立一个新的 `Observable` 并回传，并且在内部订阅原本的 `Observable`。

当然我们也可以直接把 `map` 塞到 `Observable.prototype`

```typescript
function map(callback) {
    return Rx.Observable.create((observer) => {
        return this.subscribe(
            (value) => { 
                try{
                    observer.next(callback(value));
                } catch(e) {
                    observer.error(e);
                }
            },
            (err) => { observer.error(err); },
            () => { observer.complete() }
        )
    })
}
Rx.Observable.prototype.map = map;
var people = Rx.Observable.of('Jerry', 'Anna');
var helloPeople = people.map((item) => item + ' Hello~');

helloPeople.subscribe(console.log);
// Jerry Hello~
// Anna Hello~
```

这里有两个重点必须知道，每个 `operator` 都会回传一个新的 `observable`，而我们可以通过 `create` 的方法建立各种 `operator`。

> 在 RxJS 5 的实例中，其实每个 Operator 是透过原来的 Observable 的 `lift` 方法来建立新的 Observable，这个方法会在新回传的 Observable 事件内偷塞两个属性，分别是 `source` 和 `operator`，记录原本的资料和当前使用的 `operator`。

> 其实 `lift` 方法还是用 `new Observable` (跟 create 一样)。至于为什么要独立这个方法，除了更好的封装之外，主要的原因是为了让 RxJS 5 的使用者能更好的 debug。关于 RxJS 5 的除错方式，之后会讲。

在学习 `operator` 之前，为了更好地理解，我们需要订定一个简单的方式来表达 Observable。

## Marble Diagrams

我们把描绘 Observable 的图示称为 Marble Diagrams，采用类似 ASCAII 的绘画方式。

我们用 `-` 来表达一小段时间，这些 `-` 串起来就代表一个 Observable

```typescript
-------------------------
```

`X` (大写 X) 表示有错误发生

```typescript
------------------------X
```

`|` 表示 Observable 结束

```typescript
------------------------|
```

用这个时间序当中，我们可能会发送值，如果值是数字则是直接用阿拉伯数字取代，其他的资料类型则用相近的英文符号代表，这里我们用 `interval` 来举例：

```typescript
-----0-----1-----2-----3--...
```

当 Observable 是同步发送值的时候，例如：

```typescript
var source = Rx.Observable.of(1, 2, 3, 4);
```

`source`  的图形就会长这样：

```typescript
(1234)|
```

小括号代表同步发生。

另外，Marble Diagrams 也能够表达 operator 的前后转换，例如：

```typescript
var source = Rx.Observable.interval(1000);
var newest = source.map(x => x + 1); 
```

这时 Marble Diagrams 就会长这样

```typescript
source: -----0-----1-----2-----3--...
							map(x => x + 1)
newest: -----1-----2-----3-----4--...
```

最上面是原本的 Observable，中间是 Operator，下面则是最新的 Observable。

以上就是 Marble Diagrams 如何表示 Operator 对 Observable 的操作，这能让我们更好的理解各个 Operator。

## Operator

### `map`

Observable 的 `map` 方法使用跟数组中的 `map` 是一样的，我们传入一个 `callback function`，这个 `callback function` 会带入每次的发送值，然后再回传一个经过处理的值，如下：

```typescript
var source = Rx.Observable.interval(1000);
var newest = source.map(x => x + 1); 

newest.subscribe(console.log);
// 1
// 2
// 3
// 4
// 5..
```

用 Marble Diagrams 表达就是：

```typescript
source: -----0-----1-----2-----3--...
            map(x => x + 1)
newest: -----1-----2-----3-----4--...
```

还有一个方法，名为 `mapTo`，跟 `map` 方法很像

### `mapTo`

`mapTo` 可以把传进来的值改成一个固定的值，如下：

```typescript
var source = Rx.Observable.interval(1000);
var newest = source.mapTo(2); 

newest.subscribe(console.log);
// 2
// 2
// 2
// 2..
```

`mapTo` 用 Marble Diagrams 表示：

```typescript
source: -----0-----1-----2-----3--...
                mapTo(2)
newest: -----2-----2-----2-----2--...
```

### `filter`

`filter` 在使用上也跟数组的相同，我们需要传入一个 `callback function`，这个 `function` 会传入每个被发送的元素，并且回传一个 `boolean` 值，如果为 `true` 则会保留，如果为 `false` 就会被过滤掉，如下：

```typescript
var source = Rx.Observable.interval(1000);
var newest = source.filter(x => x % 2 === 0); 

newest.subscribe(console.log);
// 0
// 2
// 4
// 6..
```

`filter` 用 Marble Diagrams 表达：

```typescript
source: -----0-----1-----2-----3-----4-...
            filter(x => x % 2 === 0)
newest: -----0-----------2-----------4-...
```

