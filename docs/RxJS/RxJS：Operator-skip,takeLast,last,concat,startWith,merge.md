# RxJS：Operator - skip, takeLast, last, concat, startWith, merge

## Operator

### skip

```typescript
var source = Rx.Observable.interval(1000);
var example = source.skip(3);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// 3
// 4
// 5...
```

原本从 0 开始的就会变成 3 开始，但是记得原本元素的等待时间仍然存在，也就是说此示例第一个取得的元素需要等待 4 秒，用 Marble Diagrams 表示如下。

```typescript
source : ----0----1----2----3----4----5--....
                    skip(3)
example: -------------------3----4----5--...
```

### takeLast

除了可以用 take 取前几个之外，我们也可以倒过来取最后几个，示例如下：

```typescript
var source = Rx.Observable.interval(1000).take(6);
var example = source.takeLast(2);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// 4
// 5
// complete
```

这里我们先取了前 6 个元素，再取最后 2 个。所以最后会发送 4、5 complete，这里有一个重点，就是 `takeLast` 必须等到整个 Observable 完成，才知道最后的元素有哪些，并且 **同步发送**，如果用 Marble Diagrams 表示如下：

```typescript
source : ----0----1----2----3----4----5|
                takeLast(2)
example: ------------------------------(45)|
```

这里可以看到 `takeLast` 后，必须等到原来的 Observable 完成后，才立即同步发送 4、5、complete。

### last

跟 `take(1)` 相同，我们有一个 `takeLast(1)` 的简化写法，那就是 `last(1)` 用来取得最后一个元素。

```typescript
var source = Rx.Observable.interval(1000).take(6);
var example = source.last();

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// 5
// complete
```

用 Marble Diagrams 表示如下

```typescript
source : ----0----1----2----3----4----5|
                    last()
example: ------------------------------(5)|
```

### concat

`concat` 可以把多个 Observable 实例合并成一个，示例如下：

```typescript
var source = Rx.Observable.interval(1000).take(3);
var source2 = Rx.Observable.of(3)
var source3 = Rx.Observable.of(4,5,6)
var example = source.concat(source2, source3);

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
// 5
// 6
// complete
```

和 `concatAll` 一样，必须等到前一个 Observable 完成，才会继续下一个，用 Marble Diagrams 表示如下：

```typescript
source : ----0----1----2|
source2: (3)|
source3: (456)|
            concat()
example: ----0----1----2(3456)|
```

另外 `concat` 还可以当做静态方法来使用

```typescript
var source = Rx.Observable.interval(1000).take(3);
var source2 = Rx.Observable.of(3);
var source3 = Rx.Observable.of(4,5,6);
var example = Rx.Observable.concat(source, source2, source3);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

### startWith

`startWith` 可以在 Observable 的一开始塞要发送的元素，有点像 `concat` 但参数不是 Observable 而是要发送的元素，使用示例如下：

```typescript
var source = Rx.Observable.interval(1000);
var example = source.startWith(0);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// 0
// 0
// 1
// 2
// 3...
```

这里可以看到我们在 source 的一开始塞了一个 `0`，让 example 会在一开始就立即发送 `0`，用 Marble Diagrams 表示如下：

```typescript
source : ----0----1----2----3--...
                startWith(0)
example: (0)----0----1----2----3--...
```

记得 `startWith` 的值是一开始就同步发出的，这个 Operator 很常被用来保存程序的起始状态。

### merge

`merge` 跟 `concat` 一样都是用来合并 Observable，但他们行为上有非常大的不同

直接看例子

```typescript
var source = Rx.Observable.interval(500).take(3);
var source2 = Rx.Observable.interval(300).take(6);
var example = source.merge(source2);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// 0
// 0
// 1
// 2
// 1
// 3
// 2
// 4
// 5
// complete
```

上面可以看得出来，`merge` 把多个 Observable 同时处理，这跟 `concat` 一次处理一个 Observable 是完全不一样的，由于是同时处理行为会变得较为复杂，这里我们用 Marble Diagrams 会比较好解释。

```typescript
source : ----0----1----2|
source2: --0--1--2--3--4--5|
            merge()
example: --0-01--21-3--(24)--5|
```

这里可以看到 `merge` 之后的 example 在时间序上同时跑 source 与 source2，当两件事情同时发生时，会同步发送资料，当两个 Observable 都结束时才会真正结束。

`merge` 同样也可以当做静态方法用

```typescript
var source = Rx.Observable.interval(500).take(3);
var source2 = Rx.Observable.interval(300).take(6);
var example = Rx.Observable.merge(source, source2);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

`merge` 的逻辑有点像是 OR (||)，就当做两个 Observable 其中一个被处罚时都可以被处理，这很常用在一个以上的按钮具有部分相同的行为。

例如一个影片播放器，一个是暂停 (||)，另一个是结束播放。这两个俺妞妞都具有相同的行为就是影片会被停止，只是结束播放会让影片回到 00 秒，这时我们就可以把这两个按钮的事件 merge 起来处理影片暂停这件事。

```typescript
var stopVideo = Rx.Observable.merge(stopButton, endButton);

stopVideo.subscribe(() => {
    // 暂停播放影片
})
```

