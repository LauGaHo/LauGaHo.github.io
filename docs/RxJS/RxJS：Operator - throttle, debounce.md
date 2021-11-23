---
title: RxJS：Operator - throttle, debounce
date: 2021/7/7 11:51:00
tag: [RxJS]
---

# RxJS：Operator - throttle, debounce

## Operator

### debounce

跟 `buffer`，`bufferTime` 一样，Rx 有 `debounce` 跟 `debounceTime` 一个是传入 Observable，另一个则是传入毫秒，比较常用到的是 `debounceTime`，直接看一个示例

```typescript
var source = Rx.Observable.interval(300).take(5);
var example = source.debounceTime(1000);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// 4
// complete
```

这里只印出了 `4` 然后就结束了，因为 `debounce` 运行的方式是每次收到元素，会先把元素 cache 住并等待一段时间，如果这段时间内没有收到任何元素，则把元素送出；如果这段时间内又收到新的元素，则会把原本 cache 住的元素释放掉并重新计时，不断重复。

以现在这个示例来说，我们每 300 毫秒就会送出一个数值，但我们的 `debounceTime` 是 1000 毫秒，也就是说每次 `debounce` 收到元素还等不到 1000 毫秒，就会收到下一个新元素，然后重新等待 1000 毫秒，如此重复知道第五个元素送出时，Observable 结束了。

Marble Diagrams

```typescript
source : --0--1--2--3--4|
        debounceTime(1000)
example: --------------4|        
```

`debounce` 会在收到元素后等待一段时间，很适合用来处理间歇行为，间歇行为就是指这个行为是一段一段的，例如要做 Auto Complete 时，我们要打字搜寻不会一直不断打字，可以等我们听了一小段时间再送出，才不会每打一个字就送出一次 request！

例子，假设我们想要自动传送使用者打的字到后端

```typescript
const searchInput = document.getElementById('searchInput');
const theRequestValue = document.getElementById('theRequestValue');

Rx.Observable.fromEvent(searchInput, 'input')
  .map(e => e.target.value)
  .subscribe((value) => {
    theRequestValue.textContent = value;
    // 在这里发 request
  })
```

如果用上面这段代码，就会每打一个字就送一次 request，当很多人在使用时就会对 server 造成很大的负担，实际上我们只需要使用者最后打出来的问题就好了，不用每次都送，这时就能用 `debounceTime` 做优化。

```typescript
const searchInput = document.getElementById('searchInput');
const theRequestValue = document.getElementById('theRequestValue');

Rx.Observable.fromEvent(searchInput, 'input')
  .debounceTime(300)
  .map(e => e.target.value)
  .subscribe((value) => {
    theRequestValue.textContent = value;
    // 在这里发 request
  })
```

### throttle

基本上每次看到 `debounce` 就会看到 `throttle`，他们两个的作用都是降低事件的触发频率，但行为上有很大的不同。

跟 `debounce` 一样 Rx 有 `throttle` 跟 `throttleTime` 两个方法，一个是传入 Observable，另一个是传入毫秒，比较常用到的也是 `throttleTime`，直接看示例

```typescript
var source = Rx.Observable.interval(300).take(5);
var example = source.throttleTime(1000);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// 0
// 4
// complete
```

跟 `debounce` 的不同是 `throttle` 会先开放送出元素，等到有元素被送出就会沉默一段时间，等到时间过了又会开放发送元素。

`throttle` 比较像是控制行为的最高频率，也就是说如果我们设定 1000 毫秒，那该事件频率的最大值就是每秒触发一次，不会再更快，`debounce` 则比较像是必须等待的时间，要等到一定的时间过了才会收到元素。

`throttle` 更适合用在连续性行为，比如说 UI 动画的运算过程，因为 UI 动画是连续的，像我们之前在做拖拉时，就可以加上 `throttleTime(12)` 让 `mousemove` 事件不要发送得太快，避免画面更新的速度跟不上样式的切换速度。