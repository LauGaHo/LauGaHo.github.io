---
title: RxJS：Cold vs Hot Observables
date: 2021/7/9 14:35:00
tag: [RxJS]
---

# RxJS：Cold vs Hot Observables

RxJS 中 Observables 分为两种：Cold Observables 和 Hot Observables，这两个的区别：

- Cold Observables：只有被 observers 订阅的时候，才会开始产生值。是单播的，有多少个订阅就会生成多少个订阅实例，每个订阅都是从第一个产生的值开始接受值，所以每个订阅收到的值都是一样的。
- Hot Observables：不管有没有被订阅都会产生值。是多播的，多个订阅共享同一个，是从订阅开始接受到值，每个订阅接收到的值是不同的，取决于它们是从什么时候开始订阅的。

## Cold Observables

```javascript
let obs$ = from([1, 2, 3, 4, 5]);

obs$.subscribe(data => { console.log("1st subscriber:" + data) });
obs$.subscribe(data => { console.log("2nd subscriber:" + data) });
```

把一个数组转换成 Observables，然后分别用不同的 Subcription 订阅它，在 console.log 结果如下：

```javascript
1st subscriber:1
1st subscriber:2
1st subscriber:3
1st subscriber:4
1st subscriber:5
2nd subscriber:1
2nd subscriber:2
2nd subscriber:3
2nd subscriber:4
2nd subscriber:5
```

我们把代码改一下，让第二个 Subscription 延迟一秒订阅：

```javascript
let obs$ = from([1, 2, 3, 4, 5]);

obs$.subscribe(data => { console.log("1st subscriber:" + data) });
setTimeout(() => {
    obs$.subscribe(data => { console.log("2nd subscriber:" + data) });
}, 1100);
```

结果还是跟上面一样，没有任何区别。这个是 Cold Observables，代码如下，先忽略 publish，ConnectableObservable，connect，稍后再详细解释。

```javascript
let obs$ = from([1, 2, 3, 4, 5]).pipe(
    publish()
) as ConnectableObservable<any>;
obs$.connect();

obs$.subscribe(data => { console.log("1st subscriber:" + data) });
setTimeout(() => {
    obs$.subscribe(data => { console.log("2st subscriber:" + data) });
}, 2100);
```

根据 Hot Observables 的定义，期待的效果是第一个订阅收到 1，2，3，4，5；第二个订阅收到 3，4，5。实际上两个订阅都没有接收任何值。因为 Hot Observables 是不管有没有被订阅，都会发送值，数组中有 5 个值，等到开始订阅的时候，这 5 个值已经发送完成了，所以在订阅中接收不到任何值了。

为了看到效果，把数组替换成了 interval，每隔一秒钟就发送一个值，第一个订阅等了一秒开始接受值，第二订阅等了两秒开始接收值，代码如下：

```javascript
let obs$ = interval(1000).pipe(
    publish()
) as ConnectableObservable<any>;
obs$.connect();
setTimeout(() => {
    obs$.subscribe(data => { console.log("1st subscriber:" + data) });
    setTimeout(() => {
        obs$.subscribe(data => { console.log("2st subscriber:" + data) });
    }, 1100);
}, 2100);	
```

效果是

```javascript
1st subscriber:2
1st subscriber:3
2st subscriber:3
1st subscriber:4
2st subscriber:4
1st subscriber:5
2st subscriber:5
1st subscriber:6
2st subscriber:6

......
```

## Cold Observables vs Hot Observables

- 我们可以把 Cold Observables 理解为在手机网易云音乐 APP 上听歌：APP 里的歌曲资源是 Cold Observables，听歌的人是 observers。如果没有人打开 APP 去播放这首歌，这首歌不会自己播放。每个人从自己 APP 打开播放这首歌的时候，先把歌曲从网上下载到本地，每份都是单独的歌曲实例，都是从头开始听，互相不会影响。
- 而 Hot Observables 可以理解为演唱会：比如我们去看一场演唱会，没有迟到的小伙伴（A）可以从第一首歌开始听，迟到的小伙伴就（B）只能从第二首或者更晚的歌开始听；演唱会就是 Hot Observables，小伙伴 A 和 B 就是 observers。小伙伴 A 和 B 共享同一个演唱会实例，是从订阅开始接受到值，每个订阅接收到的值是不同的，取决于它们是从什么时候开始订阅。

## 如何创建 Hot Observables

在上方的 Hot Observables 代码里，用到了 publish 操作符，ConnectableObservable，以及 connect( ) 方法创建 Hot Observables；看一下对应的方法是什么意思。

- publish：这个操作符把正常的 Observable (Cold Observables) 转换成 ConnectableObservable。
- ConnectableObservable：ConnectableObservable 是多播的共享 Observable，可以同时被多个 observers 共享订阅，是 Hot Observables。ConnectableObservable 是订阅者和真正源头 Observables (上面例子中的 interval，每隔一秒发送一个值，就是源头 Observable) 的中间人。ConnectableObservable 从源头 Observables 接收到值然后再把值转发给订阅者。
- connect( )：ConnectableObservable 并不会主动发送值，它有个 connect 方法，通过调用 connect 方法，可以启动共享 ConnectableObservable 发送值。当我们调用 ConnectableObservable.prototype.connect 方法，不管有没有被订阅，都会发送值。订阅者共享同一个实例，订阅者接收到的值取决于它们何时开始订阅。在我们的例子中，第一个订阅等了一秒从 2 开始接收值，第二个订阅等了两秒从 3 开始接收值。

## connect vs refCount

除了 connect，还有一个 refCount 方法，在比较这两个区别之前，我们先看一下 refCount 的用法和效果：

```javascript
let obs$ = interval(1000).pipe(
    publish(),
    refCount()
)

setTimeout(() => {
    obs$.subscribe(data => { console.log("1st subscriber:" + data) });
    setTimeout(() => {
        obs$.subscribe(data => { console.log("2st subscriber:" + data) });

    }, 1100);

}, 2000);
```

按之前对 publish 会生成一个 ConnectableObservable，它是一个 Hot Observables，那么预期结果是：第一个订阅者是从 2 开始接收值，第二个订阅者是从 3 开始接收值。

实际的输出：

```javascript
1st subscriber:0
1st subscriber:1
2st subscriber:1
1st subscriber:2
2st subscriber:2
1st subscriber:3
2st subscriber:3
1st subscriber:4
2st subscriber:4
1st subscriber:5
2st subscriber:5

.....
```

跟预想的不一样，第一个订阅者从 0 开始接收值，第二个订阅者比第一个订阅者晚了一秒，从 1 开始接收值。

当使用 refCount，是引用计数的 Observable。它表示当第一个订阅者开始订阅的时候，开始发送和产生值；第二个订阅者 (之后的订阅者) 共享第一个订阅者的 Observables 实例，没有订阅者的时候，会自动取消订阅；之后再重新订阅，又从头开始发送值。

它不是 Hot Observables 也不是 Cold Observables，因为它是从有第一个订阅者的时候才开始发送值，没有订阅者的时候会自动取消订阅，而且之后的订阅者共享第一个订阅者的 Observables 实例。它是基于 Hot Observables 与 Cold Observables 之间的 Observables，可以理解为 Warm Observables。

## share

publish 和 refCount 可以生成一个 Warm Observables，实际单独使用 share 操作符也可以达到同样的效果，share 实际上是 publish( ).refCount( ) 的简写，这次把第二个订阅比第一个订阅晚 5 秒再开始订阅，具体代码如下：

```javascript
let obs$ = interval(1000).pipe(
    share()
)

setTimeout(() => {
    obs$.subscribe(data => { console.log("1st subscriber:" + data) });
    setTimeout(() => {
        obs$.subscribe(data => { console.log("2st subscriber:" + data) });

    }, 5100);

}, 2000);
```

输出结果：

```javascript
1st subscriber:0
1st subscriber:1
1st subscriber:2
1st subscriber:3
1st subscriber:4
1st subscriber:5
2st subscriber:5
1st subscriber:6
2st subscriber:6
1st subscriber:7
2st subscriber:7

......
```

和使用 publish( ).refCount( ) 效果完全一样。

## shareReplay

在用 share( ) 的时候，第二个或者更后面的订阅者开始订阅者，都是共享第一个订阅者的 Observables，比如上面的例子中，第二个订阅比第一个订阅晚 5 秒再开始订阅，那么第二个订阅者从 5 开始接收值。但是实际情况中，如果我想让第二个订阅者也能够拿得到前面的值，那怎么办？用 shareReplay( ) 可以实现。具体代码如下：

```javascript
let obs$ = interval(1000).pipe(
    shareReplay(1)
)

setTimeout(() => {
    obs$.subscribe(data => { console.log("1st subscriber:" + data) });
    setTimeout(() => {
        obs$.subscribe(data => { console.log("2st subscriber:" + data) });

    }, 5100);

}, 2000);
```

输出结果：

```javascript
1st subscriber:0
1st subscriber:1
1st subscriber:2
1st subscriber:3
1st subscriber:4
2st subscriber:4
1st subscriber:5
2st subscriber:5
1st subscriber:6
2st subscriber:6
1st subscriber:7
2st subscriber:7

......
```

shareReplay(1) 中的 1 表示拿到错过的前一个值，在我们的例子就是第二个订阅从 4 开始接收值。如果改成 shareReplay (2) 就表示从错过的前两个值开始接收值，也就是第二个订阅会从 3 开始接收值。

shareReplay(1) 其实也是 publishReplay(1).refCount( ) 的缩写，用 publishReplay(1).refCount( ) 有同样的效果。