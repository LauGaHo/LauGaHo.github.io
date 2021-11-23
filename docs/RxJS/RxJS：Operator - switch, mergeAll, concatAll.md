---
title: RxJS：Operator - switch, mergeAll, concatAll
date: 2021/7/7 11:51:00
tag: [RxJS]
---

# RxJS：Operator - switch, mergeAll, concatAll

本文讲解的三个 Operator 都是用来处理 Higher Order Observable。所谓的 Higher Order Observable 就是指一个 Observable 送出的元素还是一个 Observable，就像是二维数组一样，一个数组中的每个元素都是数组。如果用泛型来表达就像是

```typescript
Observable<Observable<T>>
```

通常我们需要的是第二层 Observable 送出的元素，所以我们希望可以把二位的 Observable 改成一维的，像是下面这样

```typescript
Observable<Observable<T>> => Observable<T>
```

其实想要做到这件事有三个方法 `switch`、`mergeAll`、`concatAll`，其中 `concatAll` 我们在之前的文章已经稍微讲过了，今天这篇文章会讲解这三个 Operator 各自的效果和差异。

## Operator

### concatAll

`concatAll` 最重要的重点是他处理完前一个 Observable 才会再处理下一个 Observable，直接看示例

```typescript
var click = Rx.Observable.fromEvent(document.body, 'click');
var source = click.map(e => Rx.Observable.interval(1000));

var example = source.concatAll();
example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// (点击后)
// 0
// 1
// 2
// 3
// 4
// 5 ...
```

上面这段代码，当我们点击画面就会开始送出数值，对应 Marble Diagrams

```typescript
click  : ---------c-c------------------c--.. 
        map(e => Rx.Observable.interval(1000))
source : ---------o-o------------------o--..
                   \ \
                    \ ----0----1----2----3----4--...
                     ----0----1----2----3----4--...
                     concatAll()
example: ----------------0----1----2----3----4--..
```

从 Marble Diagrams 可以看出，当我们点击一下 `click` 事件就会被转成一个 Observable 而这个 Observable 会每一秒送出一个递增的数值，当我们用 `concat` 之后会把二维的 Observable 摊平成一维的 Observable，但 `concatAll` 会一个一个处理，一定是等前一个 Observable 完成才会处理下一个 Observable，因为现在送出 Observable 是无限的永远不会完成，就导致他永远不会处理第二个送出的 Observable。

再看一个例子

```typescript
var click = Rx.Observable.fromEvent(document.body, 'click');
var source = click.map(e => Rx.Observable.interval(1000).take(3));

var example = source.concatAll();
example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

这里我们把送出的 Observable 变成有限的，只会送出三个元素，这时就能看得出来 `concatAll` 不管两个 Observable 送出的时间多么相近，一定会先处理前一个 Observable 再处理下一个。

### switch

`switch` 同样能把二维的 Observable 摊平成一维的，但他们在行为上有很大的不同，我们看以下示例

```typescript
var click = Rx.Observable.fromEvent(document.body, 'click');
var source = click.map(e => Rx.Observable.interval(1000));

var example = source.switch();
example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

对应的 Marble Diagrams 如下

```typescript
click  : ---------c-c------------------c--.. 
        map(e => Rx.Observable.interval(1000))
source : ---------o-o------------------o--..
                   \ \                  \----0----1--...
                    \ ----0----1----2----3----4--...
                     ----0----1----2----3----4--...
                     switch()
example: -----------------0----1----2--------0----1--...
```

`switch` 最重要的就是他会在新的 Observable 送出后直接处理新的 Observable 不管前一个 Observable 是否完成，每当有新的 Observable 送出就会直接把旧的 Observable 退订 (unsubscribe)，永远只处理最新的 Observable。

所以在这上面的 Marble Diagrams 可以看得出来第一次送出的 Observable 跟 第二次送出的 Observable 时间点太相近了，导致第一个 Observable 还来不及送出元素就直接退订了，当下一次 Observable 就又把前一次的 Observable 退订了。

### mergeAll

我们之前讲过 `mergeAll` 他可以让多个 Observable 同时送出元素，`mergeAll` 也是同样的道理，它会把二维的 Observable 转成一维的，并且能够同时处理所有的 Observable，看看这个示例

```typescript
var click = Rx.Observable.fromEvent(document.body, 'click');
var source = click.map(e => Rx.Observable.interval(1000));

var example = source.mergeAll();
example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

对应的 Marble Diagrams

```typescript
click  : ---------c-c------------------c--.. 
        map(e => Rx.Observable.interval(1000))
source : ---------o-o------------------o--..
                   \ \                  \----0----1--...
                    \ ----0----1----2----3----4--...
                     ----0----1----2----3----4--...
                     switch()
example: ----------------00---11---22---33---(04)4--...
```

从 Marble Diagrams 可以看出，所有的 Observable 是并行处理的，也就是 `mergeAll` 不会像 `switch` 一样退订原先的 Observable，而是并行处理多个 Observable。以我们的示例来说，当我们点击越多下，最后送出的频率就会越快。

另外 `mergeAll` 可以传入一个数值，这个数值代表他可以同时处理的 Observable 数量，看一个例子：

```typescript
var click = Rx.Observable.fromEvent(document.body, 'click');
var source = click.map(e => Rx.Observable.interval(1000).take(3));

var example = source.mergeAll(2);
example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

这里我们送出的 Observable 改成取前三个，并且让 `mergeAll` 最多只能同时处理 2 个Observable，对应的 Marble Diagrams

```typescript
click  : ---------c-c----------o----------.. 
        map(e => Rx.Observable.interval(1000))
source : ---------o-o----------c----------..
                   \ \          \----0----1----2|     
                    \ ----0----1----2|  
                     ----0----1----2|
                     mergeAll(2)
example: ----------------00---11---22---0----1----2--..
```

当 `mergeAll` 传入参数后，就会等处理中的其中一个 Observable 完成，再去处理下一个。以我们的例子来说，前面两个 Observable 可以被并行处理，但第三个 Observable 必须等到第一个 Observable 结束后，才会开始。

我们可以利用这个参数来决定要同时处理几个 Observable，如果我们传入 `1` 其行为就会跟 `concatAll` 是一模一样，这点在源码可以看到他们是完全相同的。

