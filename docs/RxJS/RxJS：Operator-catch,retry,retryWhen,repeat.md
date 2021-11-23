# RxJS：Operator - catch, retry, retryWhen, repeat

## Operators

### catch

`catch` 是很常见的非同步错误处理方法，在 RxJS 中也能够直接用 `catch` 来处理错误，在 RxJS 中的 `catch` 可以回传一个 Observable 来送出新的值，让我们直接看示例

```typescript
var source = Rx.Observable.from(['a','b','c','d',2])
            .zip(Rx.Observable.interval(500), (x,y) => x);

var example = source
                .map(x => x.toUpperCase())
                .catch(error => Rx.Observable.of('h'));

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

这个示例我们每隔 500 毫秒会送出一个字符串 (String)，并用字符串的方法 `toUpperCase()` 来吧字符串的英文字母改成大写，过程中可能未知的原因送出了一个数值 `2` 导致发生例外，这时我们在后面接的 `catch` 就能抓到错误。

`catch` 可以回传一个新的 Observable、Promise、Array 或任何 Iterable 的事件，来传送之后的元素。

以我们的例子来说就会在送出 `X`，Marble Diagrams 如下：

```typescript
source : ----a----b----c----d----2|
        map(x => x.toUpperCase())
         ----a----b----c----d----X|
        catch(error => Rx.Observable.of('h'))
example: ----a----b----c----d----h|
```

这里可以看到，当错误发生后就会进入到 `catch` 并重新处理一个新的 Observable，我们可以利用这个新的 Observable 来送出我们想送的值。

也可以在遇到错误之后，让 Observable 结束，如下：

```typescript
var source = Rx.Observable.from(['a','b','c','d',2])
            .zip(Rx.Observable.interval(500), (x,y) => x);

var example = source
                .map(x => x.toUpperCase())
                .catch(error => Rx.Observable.empty());

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

回传一个 `empty` 的 Observable 来直接结束。

另外 `catch` 的 `callback` 能接收第二个参数，这个参数会接收当前的 Observable，我们可以回传当前的 Observable，我们可以回传当前的 Observable 来做到重新执行，示例如下：

```typescript
var source = Rx.Observable.from(['a','b','c','d',2])
            .zip(Rx.Observable.interval(500), (x,y) => x);

var example = source
                .map(x => x.toUpperCase())
                .catch((error, obs) => obs);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

这里可以看到我们直接回传了当前的 Observable (其实就是 example) 来重新执行，Marble Diagrams 如下：

```typescript
source : ----a----b----c----d----2|
        map(x => x.toUpperCase())
         ----a----b----c----d----X|
        catch((error, obs) => obs)
example: ----a----b----c----d--------a----b----c----d--..
```

因为是我们只是简单的示范，所以这里会一直无限循环，实际上通常会用在断线重连的情景。

上面的处理方式还有一种进化的写法，叫做 `retry()`。

### retry

如果我们想要一个 Observable 发生错误时，重新尝试就可以用 `retry` 这个方法，跟我们前一个讲的示例的行为是一致的

```typescript
var source = Rx.Observable.from(['a','b','c','d',2])
            .zip(Rx.Observable.interval(500), (x,y) => x);

var example = source
                .map(x => x.toUpperCase())
                .retry();

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
}); 
```

通常这种无限的 `retry` 会放在几时同步的重新连接，让我们在连线断掉后，不断尝试重连。另外我们也可以设定只尝试几次，如下

```typescript
var source = Rx.Observable.from(['a','b','c','d',2])
            .zip(Rx.Observable.interval(500), (x,y) => x);

var example = source
                .map(x => x.toUpperCase())
                .retry(1);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
}); 
// a
// b
// c
// d
// a
// b
// c
// d
// Error: TypeError: x.toUpperCase is not a function
```

这里我们对 `retry` 传入一个数值 `1`，能够让我们只重复尝试 1 次后送出错误，对应 Marble Diagrams 如下：

```typescript
source : ----a----b----c----d----2|
        map(x => x.toUpperCase())
         ----a----b----c----d----X|
                retry(1)
example: ----a----b----c----d--------a----b----c----d----X|
```

这种处理方式很适合在 HTTP request 失败的场景中，我们可以设定重新发送几次后，再抛出错误信息。

### retryWhen

RxJS 还提供了另一种方法 `retryWhen`，他可以把另外发生的元素放到一个 Observable 中，让我们可以直接操作这个 Observable，并等到整个 Observable 操作完后再重新订阅一次原本的 Observable。

直接看代码

```typescript
var source = Rx.Observable.from(['a','b','c','d',2])
            .zip(Rx.Observable.interval(500), (x,y) => x);

var example = source
                .map(x => x.toUpperCase())
                .retryWhen(errorObs => errorObs.delay(1000));

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
}); 
```

这里 `retryWhen` 我们传入一个 `callback`，这个 `callback` 有一个参数会传入一个 Observable，这个 Observable 不是原本的 Observable (example) 而是另外事件送出的错误所组成的一个 Observable，我们可以对这个由错误所组成的 Observable 做操作，等到这次的处理完成后就会重新订阅我们原本的 Observable。

这个示例我们是把错误的 Observable 送出错误延迟 1 秒，这会使后面重新订阅的动作延迟 1 秒才执行，对应 Marble Diagrams 如下：

```typescript
source : ----a----b----c----d----2|
        map(x => x.toUpperCase())
         ----a----b----c----d----X|
        retryWhen(errorObs => errorObs.delay(1000))
example: ----a----b----c----d-------------------a----b----c----d----...
```

从上图可以看到后续重新订阅的行为就被延后了，但实际上我们不太会用 `retryWhend` 来做重新订阅的延迟，通常是直接用 `catch` 做到这件事。这里只是为了示范 `retryWhen` 的行为，实际上我们

### repeat

我们有时候可能想要 `retry` 一直重复订阅的效果，但没有错误发生，这时就可以用 `repeat` 来做到这件事，示例如下：

```typescript
var source = Rx.Observable.from(['a','b','c'])
            .zip(Rx.Observable.interval(500), (x,y) => x);

var example = source.repeat(1);

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});

// a
// b
// c
// a
// b
// c
// complete
```

这里 `repeat` 的行为跟 `retry` 基本一致，只是 `retry` 只有在例外发生时才出发，对应 Marble Diagrams

```typescript
source : ----a----b----c|
            repeat(1)
example: ----a----b----c----a----b----c|
```

同样，我们也可以加参数不让他无限循环，如下

```typescript
var source = Rx.Observable.from(['a','b','c'])
            .zip(Rx.Observable.interval(500), (x,y) => x);

var example = source.repeat();

example.subscribe({
    next: (value) => { console.log(value); },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
});
```

这样我们就可以做不断重复的行为，这个可以在建立轮询时使用，让我们不断地发 request 来更新画面。

最后看一下错误处理在实际应用的小示例

```typescript
const title = document.getElementById('title');

var source = Rx.Observable.from(['a','b','c','d',2])
            .zip(Rx.Observable.interval(500), (x,y) => x)
            .map(x => x.toUpperCase()); 
            // 通常 source 会是建立即时同步的连线，像是 web socket

var example = source.catch(
                (error, obs) => Rx.Observable.empty()
                               .startWith('连线发生错误： 5秒后重连')
                               .concat(obs.delay(5000))
                 );

example.subscribe({
    next: (value) => { title.innerText = value },
    error: (err) => { console.log('Error: ' + err); },
    complete: () => { console.log('complete'); }
}); 
```

这个示例其实就是模仿在即使同步断线时，利用 `catch` 返回一个新的 Observable，这个 Observable 会先送出错误信息并且把原本的 Observable 延迟 5 秒再作合并，虽然这只不过是一个模仿，但它清除展示了 RxJS 在做错误处理时的灵活性。