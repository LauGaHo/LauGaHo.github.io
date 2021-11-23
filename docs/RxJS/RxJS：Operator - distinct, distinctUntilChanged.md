---
title: RxJS：Operator - distinct, distinctUntilChanged
date: 2021/7/7 11:51:00
tag: [RxJS]
---

# RxJS：Operator - distinct, distinctUntilChanged

## Operator

### distinct

`distinct` 可以帮我们把相同值的值滤掉只留一个

```typescript
var source = Rx.Observable.from(['a', 'b', 'c', 'a', 'b'])
            .zip(Rx.Observable.interval(300), (x, y) => x);
var example = source.distinct()

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// a
// b
// c
// complete
```

Marble Diagrams

```typescript
source : --a--b--c--a--b|
            distinct()
example: --a--b--c------|
```

从上面可以看出，当使用了 `distinct` 之后，只要有重复出现的值就会被滤掉。

另外还可以传入一个 `selector callback function`，这个 `callback` 会传入一个接收到的元素，并回传我们真正希望比对的值，举例如下：

```typescript
var source = Rx.Observable.from([{ value: 'a'}, { value: 'b' }, { value: 'c' }, { value: 'a' }, { value: 'c' }])
            .zip(Rx.Observable.interval(300), (x, y) => x);
var example = source.distinct((x) => {
    return x.value
});

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// {value: "a"}
// {value: "b"}
// {value: "c"}
// complete
```

这里可以看到，因为 `source` 送出的都是实例，而 JS 事件的比对是比对内存位置，所以在这个例子中这些实例永远不会相等，但实际上我们想比对的是实例中的 `value`，这时我们可以传入 `selector callback`，来选择我们要比对的值。

> `distinct` 传入的 `callback` 在 RxJS 5 几个 beta 版本中有过很多改变，现在网络上很多文章跟教学都是过时的，务必小心

实际上 `distinct` 会在背地里建立一个 `Set`，当接收元素时先去判断 `Set` 内是否有相同的值，如果有就不送出，如果没有则存到 `Set` 并送出。所以尽量不要直接把 `distinct` 用在一个无限的 Observable 里，这样很可能会让 `Set` 越来越大，建议大家可以放第二个参数 `flushes`，或用 `distinctUntilChanged`。

> 这里指的 `Set` 其实是 RxJS 自己实现的，跟 ES6 原生的 Set 行为也都一致，只是因为 ES6 的 `Set` 支持程度还并不理想，所以这里是直接用 JS 实现。

`distinct` 可以传入第二个参数 `flushes`，用来清除暂存的资料，示例如下：

```typescript
var source = Rx.Observable.from(['a', 'b', 'c', 'a', 'c'])
            .zip(Rx.Observable.interval(300), (x, y) => x);
var flushes = Rx.Observable.interval(1300);
var example = source.distinct(null, flushes);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// a
// b
// c
// c
// complete
```

对应的 Marble Diagrams

```typescript
source : --a--b--c--a--c|
flushes: ------------0---...
        distinct(null, flushes);
example: --a--b--c-----c|
```

其实 `flushes` 就是 Flushes Observable 在送出元素时，会把 `distinct` 的暂存清空，所以之后的暂存就会从头来过，这样就不用担心暂存的 `Set` 越来越大的问题，但其实我们平常不太会用这种方式来处理，通常用另一个方法 `distinctUntilChanged`。

### distinctUntilChanged

`distinctUntilChanged` 跟 `distinct` 一样会把相同的元素过滤掉，但 `distinctUntilChanged` 只会跟最后一次送出的元素比较，不会每个都比，举例如下：

```typescript
var source = Rx.Observable.from(['a', 'b', 'c', 'c', 'b'])
            .zip(Rx.Observable.interval(300), (x, y) => x);
var example = source.distinctUntilChanged()

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// a
// b
// c
// b
// complete
```

