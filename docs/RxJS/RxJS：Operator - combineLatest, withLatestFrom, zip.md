---
title: RxJS：Operator - combineLatest, withLatestFrom, zip
date: 2021/7/7 11:51:00
tag: [RxJS]
---

# RxJS：Operator - combineLatest, withLatestFrom, zip

> 非同步最难的地方在于，当有多个非同步行为同时触发且相互依赖，这时候我们要处理的逻辑跟状态就会变得极其复杂，甚至程序很可能会在完成的一两天就成了 Legacy Code (遗留代码)。

我们讲过 `merge` 的用法，它的逻辑就像是 OR (||) 一样，可以把多个 Observable 合并且同时处理，当其中任何一个 Observable 送出元素时，我们都做相同的处理。

本文讲的三个 Operator 则像是 AND (&&) 逻辑，它们都是多个元素送进来时，只输出一个新元素，但各自的行为上仍有差异，需要花点时间思考。

## Operator

### combineLatest

首先介绍的是 `combineLatest`，它会取得各个 Observable 最后送出的值，再输出成一个值，直接看示例比较容易解释：

```typescript
var source = Rx.Observable.interval(500).take(3);
var newest = Rx.Observable.interval(300).take(6);

var example = source.combineLatest(newest, (x, y) => x + y);

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
// 7
// complete
```

相信读者第一次看到这个 `output` 应该都会很困惑，直接看 Marble Diagrams

```typescript
source : ----0----1----2|
newest : --0--1--2--3--4--5|

    combineLatest(newest, (x, y) => x + y);

example: ----01--23-4--(56)--7|
```

首先 `combineLatest` 可以接收多个 Observable，最后一个参数是 `callback function`，这个 `callback function` 接收的参数数量跟合并的 Observable 数量相同，依照示例来说，因为我们这里合并了两个 Observable 所以后面的 `callback function` 就接收 `x`，`y` 两个参数，`x` 会接收从 source 发送来的的值，`y` 会接收从 newest 发送过来的值。

最后一个重点就是一定会等两个 Observable 都曾有送值出来才会呼叫我们传入的 `callback function`，所以这段代码是这样运行的

- `newest` 送出了 `0`，但此时 `source` 并没有送出过任何值，所以不会执行 `callback`
- `source` 送出了 `0`，此时 `newest` 最后一次送出的值为 `0`，把这两个数传入 `callback` 得到 `0`
- `newest` 送出了 `1`，此时 `source` 最后一次送出的值为 `0`，把这两个数传入 `callback` 得到 `1`
- `newest` 送出了 `2`，此时 `source` 最后一次送出的值为  `0`，把这两个数传入 `callback` 得到`2`
- `source` 送出了 `1`，此时 `newest` 最后一次送出的值为 `2`，把这两个数传入 `callback` 得到 `3`
- `newest` 送出了 `3`，此时 `source` 最后一次送出的值为 `1`，把这两个数传入 `callback` 得到 `4`
- `source` 送出了 `2`，此时 `newest` 最后一次送出的值为 `3`，把这两个数传入 `callback` 得到 `5`
- `source` 结束，但 `newest` 还没结束，所以 example 还不会结束
- `newest` 送出了 `4`，此时 `source` 最后一次发送的值为 `2`，把这两个数传入 `callback` 得到 `6`
- `newest` 送出了 `5`，此时 `source` 最后一次发送的值为 `2`，把这两个数传入 `callback` 得到 `7`
- `newest` 结束，因为 `source` 也结束了，所以 example 结束

不管是 `source` 还是 `newest` 送出值来，只要另一方曾有送出过值来 (有最后的值)，就会执行 `callback` 并送出新的值，这就是 `combineLatest`

`combineLatest` 很常用在运算多个因子的结果，例如最常见的 BMI 计算，我们身高变动时就拿上一次的体重计算新的 BMI，当体重变动时则拿上一次身高计算 BMI，这就很适合 `combineLatest` 来处理。

### zip

`zip` 会取每个 Observable 相同顺位的元素并传入 `callback`，也就是说每个 Observable 的第 n 个元素会一起被传入 `callback`，这里我们同样直接用示例讲解会比较清楚

```typescript
var source = Rx.Observable.interval(500).take(3);
var newest = Rx.Observable.interval(300).take(6);

var example = source.zip(newest, (x, y) => x + y);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// 0
// 2
// 4
// complete
```

Marble Diagrams

```typescript
source : ----0----1----2|
newest : --0--1--2--3--4--5|
    zip(newest, (x, y) => x + y)
example: ----0----2----4|
```

`zip` 会等到 `source` 和 `newest` 都送出了第一个元素，再传入 `callback`，下次则等到 `source` 和 `newest` 都送出了第二个元素再一起传入 `callback`，运行步骤如下：

- `newest` 送出了第一个值 `0`，此时 `source` 没有送出第一个值，所以不执行 `callback`
- `source` 送出了第一个值 `0`，`newest` 之前送出的第一个值为 `0`，把这两个数传入 `callback` 得到 `0`
- `newest` 送出了第二个值 `1`，此时 `source` 没有送出第二个值，不执行 `callback`
- `newest` 送出了第三个值 `2`，此时 `source` 没有送出第三个值，不执行 `callback`
- `source` 送出了第二个值 `1`，`newest` 之前送出的第二个值为 `1`，把这两个数传入 `callback` 得到 `2`
- `newest` 送出了第四个值 `3`，此时 `source` 并没有送出第四个值，不执行 `callback`
- `source` 送出了第三个值 `2`，`newest` 之前送出的第二个值为 `2`，把这两个数传入 `callback` 得到 `4`
- `source` 结束 example 就直接结束，因为 `source` 和 `newest` 不会再有对应顺位的值

`zip` 会把各个 Observable 相同顺位送出的值传入 `callback`，这很常拿来做 demo 使用，比如我们想间隔 100ms 送出 'h' 'e' 'l' 'l' 'o'，就可以这么做

```typescript
var source = Rx.Observable.from('hello');
var source2 = Rx.Observable.interval(100);

var example = source.zip(source2, (x, y) => x);
```

对应 Marble Diagrams

```typescript
source : (hello)|
source2: -0-1-2-3-4-...
        zip(source2, (x, y) => x)
example: -h-e-l-l-o|
```

这里利用 `zip` 来达到原本只能同步发送的资料变成了非同步，适合建立示范用的资料

> 平常没事不要乱用 `zip`，除非真的需要，因为 `zip` 必须 cache 住还没处理的元素，当两个 Observable 一个很快一个很慢的时候，就会 cache 住非常多的元素，等待比较慢的那个 Observable。可能会造成内存相关的问题。

### withLatestFrom

`withLatestFrom` 运行方式跟 `combineLatest` 有点像，只是他又有主从的关系，只有在主要的 Observable 送出新的值时，才会执行 `callback`，附随的 Observable 只是在背景下运行。

```typescript
var main = Rx.Observable.from('hello').zip(Rx.Observable.interval(500), (x, y) => x);
var some = Rx.Observable.from([0,1,0,0,0,1]).zip(Rx.Observable.interval(300), (x, y) => x);

var example = main.withLatestFrom(some, (x, y) => {
    return y === 1 ? x.toUpperCase() : x;
});

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

对应的 Marble Diagrams

```typescript
main   : ----h----e----l----l----o|
some   : --0--1--0--0--0--1|

withLatestFrom(some, (x, y) =>  y === 1 ? x.toUpperCase() : x);

example: ----h----e----l----L----O|
```

`withLatestFrom` 会在 `main` 送出值的时候执行 `callback`，注意如果 `main` 送出值时 `some` 之前没有送出任何值 `callback` 仍然不会执行。

在 `main` 送出值时，判断 `some` 最后一次送的是不是 `1` 来决定是否切换大小写，执行步骤如下：

- `main` 送出了 `h`，此时 `some` 上一次送出的值为 `0`，把这两个参数传入 `callback` 得到 `h`
- `main` 送出了 `e`，此时 `some` 上一次送出的值为 `0`，把这两个参数传入 `callback` 得到 `e`
- `main` 送出了 `l`，此时 `some` 上一次送出的值为 `0`，把这两个参数传入 `callback` 得到 `l`
- `main` 送出了 `l`，此时 `some` 上一次送出的值为 `1`，把这两个参数传入 `callback` 得到 `L`
- `main` 送出了 `o`，此时 `some` 上一次送出的值为 `1`，把这两个参数传入 `callback` 得到 `O`

`withLatestFrom` 很常用在一些 `checkbox` 型的功能，例如说一个编辑器，开启粗体之后，打出来的字就都要变粗体，粗体就像 `some` Observable，打字就像 `main` Observable。