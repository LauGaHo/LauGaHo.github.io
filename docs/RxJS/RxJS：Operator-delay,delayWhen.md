# RxJS：Operator - delay, delayWhen

## Operators

### delay

`delay` 可以延迟 Observable 一开始发送元素的时间点，示例如下：

```typescript
var source = Rx.Observable.interval(300).take(5);

var example = source.delay(500);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// 0
// 1
// 2
// 3
// 4
```

当然直接从 log 出来的信息来看，是完全看不出差异的。

Marble Diagrams

```typescript
source : --0--1--2--3--4|
        delay(500)
example: -------0--1--2--3--4|
```

从 Marble Diagrams 可以看得出来，第一次送出元素的时间变慢了，虽然在这里看起来没什么用，但是在 UI 操作上是非常有用的，这部分最后展示。

`delay` 除了可以传入毫秒之外，也可以传入 `Date` 类型的变量，如下使用方式

```typescript
var source = Rx.Observable.interval(300).take(5);

var example = source.delay(new Date(new Date().getTime() + 1000));

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

### delayWhen

`delayWhen` 的作用跟 `delay` 很像，最大的区别是 `delayWhen` 可以影响每个元素，而且需要传一个 `callback` 并回传一个 Observable，示例如下：

```typescript
var source = Rx.Observable.interval(300).take(5);

var example = source
              .delayWhen(
                  x => Rx.Observable.empty().delay(100 * x * x)
              );

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

这时我们的 Marble Diagrams 如下

```typescript
source : --0--1--2--3--4|
    .delayWhen(x => Rx.Observable.empty().delay(100 * x * x));
example: --0---1----2-----3-----4|
```

这里传进来的 `x` 就是 `source` 送出的每个元素，这样我们就能对每一个做延迟。