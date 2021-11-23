# RxJS：Operator - switchMap, mergeMap, concatMap

## Operator 

### concatMap

`concatMap` 其实就是 `map` 加上 `concatAll` 的简化写法，直接看一个例子

```typescript
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source
                .map(e => Rx.Observable.interval(1000).take(3))
                .concatAll();

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

上面这个示例可以简化成

```typescript
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source
                .concatMap(
                    e => Rx.Observable.interval(100).take(3)
                );

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

前后两个行为是一致的，`concatMap` 也会先处理前一个送出的 Observable 在处理下一个 Observable，对应 Marble Diagrams

```typescript
source : -----------c--c------------------...
        concatMap(c => Rx.Observable.interval(100).take(3))
example: -------------0-1-2-0-1-2---------...
```

这样的行为也很常被用在发送 request 如下

```typescript
function getPostData() {
    return fetch('https://jsonplaceholder.typicode.com/posts/1')
    .then(res => res.json())
}
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source.concatMap(
                    e => Rx.Observable.from(getPostData()));

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

这里我们每点击一下画面就会送出一个 HTTP request，如果我们快速的连续点击，大家可以在开发者工具的 network 看到每个 request 是等到前一个 request 完成才会送出下一个 request，如下图

![alt](https://raw.githubusercontent.com/LauGaHo/blog-img/master/uPic/wd5oo0.jpg)

从 network 的图形可以看得出来，第二个 request 的发送时间是接在第一个 request 之后的，我们可以确保每一个 request 会等前一个 request 完成才做处理。

`concatMap` 还有第二个参数是一个 `selector callback`，这个 `callback` 会传入四个参数，分别是

- 外部 Observable 送出的元素
- 内部 Observable 送出的元素
- 外部 Observable 送出元素的 index
- 内部 Observable 送出元素的 index

回传我们想要的值，示例如下

```typescript
function getPostData() {
    return fetch('https://jsonplaceholder.typicode.com/posts/1')
    .then(res => res.json())
}
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source.concatMap(
                e => Rx.Observable.from(getPostData()), 
                (e, res, eIndex, resIndex) => res.title);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

这个示例的外部 Observable 送出的元素就是 `click event` 实例，内部 Observable 送出的元素就是 `response` 实例，这里我们回传 `response` 实例的 `title` 属性，这样一来我们就可以直接收到 `title`，这个方法很适合用在 `response` 要选取的值跟前一个事件或顺位相关。

### switchMap

`switchMap` 其实就是 `map` 加上 `switch` 简化的写法，如下

```typescript
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source
                .map(e => Rx.Observable.interval(1000).take(3))
                .switch();

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

上面的代码可以简化成

```typescript
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source
                .switchMap(
                    e => Rx.Observable.interval(100).take(3)
                );

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

对应的 Marble Diagrams

```typescript
source : -----------c--c-----------------...
        concatMap(c => Rx.Observable.interval(100).take(3))
example: -------------0--0-1-2-----------...
```

只要注意一个重点 `switchMap` 会在下一个 Observable 被送出后直接退订前一个未完成的 Observable。

另外我们也可以把 `switchMap` 用在发送 HTTP request。

```typescript
function getPostData() {
    return fetch('https://jsonplaceholder.typicode.com/posts/1')
    .then(res => res.json())
}
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source.switchMap(
                    e => Rx.Observable.from(getPostData()));

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

如果我们快速的连续点击 5 下，可以在开发者工具的 network 看到每个 request 会在点击时发送，如下图

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/4MdIAL.jpg)

> 灰色是浏览器原生的停顿行为，实际上灰色的一开始就是 `fetch` 执行送出 request，只是卡在浏览器等待发送。

从上图可以看到，虽然我们发送了多个 request 但最后真正印出来的 `log` 只会有一个，代表前面发送的 request 已经不会造成任何的 side-effect 了，这个很适合用在只看最后一次 request 的情景，比如说 自动完成 (Auto Complete)，我们只需要显示使用者最后一次打在画面上的文字，来做建议选项而不用每一次的。

`switchMap` 和 `concatMap` 一样有第二个参数 `selector callback` 可用来回传我们想要的值，这部分的行为跟 `concatMap` 是一样的。

### mergeMap

`mergeMap` 其实就是 `map` 加上 `mergeAll` 简化的写法，如下

```typescript
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source
                .map(e => Rx.Observable.interval(1000).take(3))
                .mergeAll();

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

上面代码可以简化成

```typescript
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source
                .mergeMap(
                    e => Rx.Observable.interval(100).take(3)
                );

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

对应 Marble Diagrams

```typescript
source : -----------c-c------------------...
        concatMap(c => Rx.Observable.interval(100).take(3))
example: -------------0-(10)-(21)-2----------...
```

`mergeMap` 可以并行处理多个 Observable，以这个例子来讲当我们快速点按 2 下，元素发送的时间点是有机会重叠的，这个部分的细节大家可以看上一篇文章 `merge` 部分。

另外也可以把 `switchMap` 用在发送 HTTP request

```typescript
function getPostData() {
    return fetch('https://jsonplaceholder.typicode.com/posts/1')
    .then(res => res.json())
}
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source.mergeMap(
                    e => Rx.Observable.from(getPostData()));

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

如果我们快速的连续点击 5 下，大家可以在开发者工具的 network 看到每个 request 会在点击时发送并且会 `log` 出 5 个实例，如下图

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/ohyqkB.jpg)

`mergeMap` 也能传入第二个参数 `selector callback`，这个 `selector callback` 跟 `concatMap` 第二个参数也是完全一致的，但 `mergeMap` 的重点我们可以传入第三个参数，来限制并行处理的数量。

```typescript
function getPostData() {
    return fetch('https://jsonplaceholder.typicode.com/posts/1')
    .then(res => res.json())
}
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source.mergeMap(
                e => Rx.Observable.from(getPostData()), 
                (e, res, eIndex, resIndex) => res.title, 3);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

这里我们传入 3 就能限制，HTTP request 最多只能同时送出 3 个，并且要等其中一个完成再处理下一个，如下图

![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/fonvDO.jpg)

可以注意一下这张图，连续点了 5 下，但第四个 request 是在第一个完成之后才送出，这个适合在特殊的需求下，可以限制同时发送 request 的数量

> RxJS 还保留了 `mergeMap` 的别名叫 `flatMap`，虽然官方文档上没有，但这两个方法是完全一致的

### switchMap, mergeMap,  concatMap

这三个 Operator 还有一个共同的特性，那就是这三个 Operator 可以把第一个参数所回传的 `Promise` 实例直接转成 Observable，这样我们就不用再用 `Rx.Observable.from` 转一次，如下

```typescript
function getPersonData() {
    return fetch('https://jsonplaceholder.typicode.com/posts/1')
    .then(res => res.json())
}
var source = Rx.Observable.fromEvent(document.body, 'click');

var example = source.concatMap(e => getPersonData());
                                    //直接回传 promise 实例

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

至于在上面如何选择三个 Operator？其实还是要看具体使用情景而定，这里简单列一下大部分的使用场景

- `concatMap` 用在可以确定内部的 Observable 结束时间比外部 Observable 发送时间快的情景，并且不希望有任何并行处理行为，适合少数要一次一次完成到底的 UI 动画或特别的 HTTP request 行为。
- `switchMap` 用在只要最后一次行为的结果，适合绝大多数的使用情景。
- `mergeMap` 用在并行处理多个 Observable，适合需要并行处理的行为，像是多个 I/O 的并行处理。

> 不确定选择哪一种时，使用 `switchMap`

> 在使用 `concatAll` 或 `concatMap` 时，请注意内部的 Observable 一定要能够结束，且外部的 Observable 发送元素的速度不能俾内部的 Observable 结束时间快太多，不然会有内存问题。

