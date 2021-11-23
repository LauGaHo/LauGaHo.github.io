# RxJS：Operator - scan, buffer

## Operator

### scan

`scan` 其实就是 Observable 版本的 `reduce` 只是命名不同。如果熟悉数组操作的话，应该知道原生的 JS Array 就有 `reduce` 方法，使用方式如下：

```typescript
var arr = [1, 2, 3, 4];
var result = arr.reduce((origin, next) => { 
    console.log(origin)
    return origin + next
}, 0);

console.log(result)
// 0
// 1
// 3
// 6
// 10
```

`reduce` 方法需要传两个参数，第一个是 `callback` 第二个则是起始状态，这个 `callback` 执行时，会传入两个参数，一个是原本的状态，另一个是修改原本状态的参数，最后回传一个新的状态，再继续执行。

所以这段代码是这样执行的：

- 第一次执行 `callback` 起始状态是 `0`，所以 `origin` 传入 `0`，`next` 为 `arr` 的第一个元素 `1`，相加之后变成 `1` 回传并当做下一次的状态。
- 第二次执行 `callback`，这时原本的状态 (`origin`) 就变成了 `1`，`next` 为 `arr` 的第二个元素 `2`，相加之后变成 `3` 回传并当做下一次的状态。
- 第三次执行 `callback`，这时原本的状态 (`origin`) 就变成了 `3`，`next` 为 `arr` 的第三个元素 `3`，相加之后变成 `6` 回传并当做下一次的状态。
- 第四次执行 `callback`，这时原本的状态 (`origin`) 就变成了 `6`，`next` 为 `arr` 的第四个元素 `4`，相加之后变成 `10` 回传并当做下一次的状态。
- 这时 `arr` 的元素都已经遍历过了，所以不会直接把 `10` 回传。

`scan` 整体的运行方式都跟 `reduce` 一样，示例如下：

```typescript
var source = Rx.Observable.from('hello')
             .zip(Rx.Observable.interval(600), (x, y) => x);

var example = source.scan((origin, next) => origin + next, '');

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// h
// he
// hel
// hell
// hello
// complete
```

画成 Marble Diagram

```typescript
source : ----h----e----l----l----o|
    scan((origin, next) => origin + next, '')
example: ----h----(he)----(hel)----(hell)----(hello)|
```

这里可以看到第一次传入 `'h'` 跟 `' '` 相加，返回 `'h'` 当做下一次的初始状态，一直重复下去。

> `scan` 跟 `reduce` 最大的差别就在 `scan` 一定会回传一个 Observable 实例，而 `reduce` 最后回传的值有可能是任何类型，必须看使用者传入的 `callback` 才能决定 `reduce` 最后的返回值。

> Jafar Husain 曾说过：JavaScript 的 `reduce` 是错的，它最后应该永远回传数组才对！

`scan` 很常用在状态的计算处理，最简单的就是对一个数字的加减，我们可以绑定一个 button 的 `click` 事件，并用 `map` 把 `click` 事件转成 `1`，之后送到 `scan` 计算值再做显示。

最简单的加减示例：

```typescript
const addButton = document.getElementById('addButton');
const minusButton = document.getElementById('minusButton');
const state = document.getElementById('state');

const addClick = Rx.Observable.fromEvent(addButton, 'click').mapTo(1);
const minusClick = Rx.Observable.fromEvent(minusButton, 'click').mapTo(-1);

const numberState = Rx.Observable.empty()
  .startWith(0)
  .merge(addClick, minusClick)
  .scan((origin, next) => origin + next, 0)

numberState
  .subscribe({
    next: (value) => { state.innerHTML = value;},
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
  });
```

这里我们使用了两个 button，一个是 add 按钮，另一个是 minus 按钮。

我们把这两个按钮的点击事件各自建立了 `addClick`, `minusClick` 两个 Observable，这两个 Observable 直接 `mapTo(1)` 和 `map(-1)`，代表点击后各自送出的数字！

接着我们用 `empty()` 建立一个空的 Observable 代表画面上数字的状态，搭配 `startWith(0)` 来设定初始值，接着用 `merge` 把两个 Observable 合并通过 `scan` 处理之后的逻辑，最后在 Observable 来更改画面的显示。

### buffer

buffer 是一整个家族，总共有五个相关的 Operator

- buffer
- bufferCount
- bufferToggle
- bufferWhen

这里比较常用的是 `buffer`, `bufferCount` 跟 `bufferTime` 这三个，我们直接看示例。

```typescript
var source = Rx.Observable.interval(300);
var source2 = Rx.Observable.interval(1000);
var example = source.buffer(source2);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// [0,1,2]
// [3,4,5]
// [6,7,8]...
```

画成 Marble Diagrams 则像是

```typescript
source : --0--1--2--3--4--5--6--7..
source2: ---------0---------1--------...
            buffer(source2)
example: ---------([0,1,2])---------([3,4,5])
```

`buffer` 要传入一个 Observable，它会把原来的 Observable 送出的元素缓存在数组中，等到传入的 Observable 送出元素时，就会触发把缓存的元素送出。

这里的示例 `source2` 是每一秒就会送出一个元素，我们可以改用 `bufferTime` 简洁表达，如下：

```typescript
var source = Rx.Observable.interval(300);
var example = source.bufferTime(1000);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// [0,1,2]
// [3,4,5]
// [6,7,8]...
```

除了用时间来做缓存外，我们更常用数量来做缓存，示例如下：

```typescript
var source = Rx.Observable.interval(300);
var example = source.bufferCount(3);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
// [0,1,2]
// [3,4,5]
// [6,7,8]...
```

在示例上，我们可以用 `buffer` 来做某个事件的过滤，例如像是滑鼠连点才能真的执行，这里我们一样写了一个小示例

```typescript
const button = document.getElementById('demo');
const click = Rx.Observable.fromEvent(button, 'click')
const example = click
                .bufferTime(500)
                .filter(arr => arr.length >= 2);

example.subscribe({
    next: (value) => { console.log('success'); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

这里我们只有在 500 毫秒内连点两下，才能成功印出 `'success'`，这个功能在某些特殊的需求中非常好用，也能用在批次处理降低 request 传送的次数。