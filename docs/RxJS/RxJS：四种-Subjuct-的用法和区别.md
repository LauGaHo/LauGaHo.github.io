---
title: RxJS：四种 Subjuct 的用法和区别
date: 2021/7/7 11:54:00
tag: [RxJS]
---

# RxJS：四种 Subject 的用法和区别

在 RxJS 中有四种 Subject 分别是：Subject、BehaviorSubject、AsyncSubject、ReplaySubject；这四种 Subject 都是特殊的 Observable。

四种 Subject 与普通 Observable 的区别：

|         Observable         |                 Subject                 |                 BehaviorSubject                 |                 AsyncSubject                 |             ReplaySubject             |
| :------------------------: | :-------------------------------------: | :---------------------------------------------: | :------------------------------------------: | :-----------------------------------: |
|            Cold            |                   Hot                   |                       Hot                       |                     Hot                      |                  Hot                  |
|         数据生成者         |           数据生成者和消费者            |               数据生成者和消费者                |              数据生成者和消费者              |          数据生成者和消费者           |
|            单播            |                  多播                   |                      多播                       |                     多播                     |                 多播                  |
| 每次从头开始把值发给观察者 | 将值多播给已注册监听该 Subject 的观察者 | 把最后一个值 (当前值) 发送给观察者 (需要初始值) | 执行的最后一个值发送给观察者，相当于 last( ) | 可以把之前的值发送给观察者 (错过的值) |

## Cold vs Hot

前面介绍了 Cold Observables 与 Hot Observables 的区别，除了 Observable 之外，这四种 Subject 都是 Hot Observables。

## 数据生产者 vs 数据消费者

- ***数据生产者*** 是指 Observable (可观察对象)，产生数据的一方
- ***数据消费者*** 是指 Observer (观察者)，接收数据的一方

普通的 Observable 只是数据生产者，发送数据。而 Subject、BehaviorSubject、AsyncSubject 和 ReplaySubject 即是生产者有时消费者。比如在 Angular 项目中，我们经常是哟 Subject 在多个 component 中共享数据：

```javascript
// messageService.ts
private shareData: Subject<string> = new Subject<string>();

sendData(value: string) {
    this.shareData.next(value);
}
getData(): Observable<string> {
    return this.shareData.asObservable();
}
```

```javascript
// RxjsSubjectBComponent
// send data from RxjsSubjectBComponent
  this.messageService.sendData("this message from subject b");
```

```javascript
// RxjsSubjectCComponent
// get data in RxjsSubjectCComponent
this.subscription = this.messageService.getData()
    .subscribe(val => {
        this.message = val;
        console.log(val);
    });
```

在这两个 component 中通过定义在 messageService 的 Subject 共享数据，通过 next( ) 方法使得 Subject 是数据生产者 (可观察对象) ，通过 asObservable( ) 使得 Subject 是数据消费者 (Observers)。

## 单播 vs 多播

在上面的代码例子中，不仅是 component c 可以订阅拿到数据，其他 component 中也可以订阅拿到数据。比如在 RxjsSubjectAComponent 中定义如下代码：

```javascript
// RxjsSubjectAComponent
// get data in RxjsSubjectAComponent
this.subscription = this.messageService.getData()
    .subscribe(val => {
        console.log("this is in componet A and " + val);
    })
```

从 RxjsSubjectComponent 发出值，在 RxJSSubjectCComponent 和 RxjsSubjectAComponent 能拿到同样的值，这个就是多播。

普通的 Observables 是 Cold Observables 并且是单播的，其他四种 Subject 是 Hot Observables 并且是多播的。

单播的意思是，每个普通的 Observables 实例都只能被一个观察者订阅，当它被其他观察者订阅的时候，会产生一个新的实例。也就是普通的 Observables 被不同的观察者订阅的时候，会有多个实例，不管观察者是从何时开始订阅，每个实例都是从头开始把值发给对应的观察者。

## BehaviorSubject

Subject 其中的一个变体就是 BehaviorSubject，它有一个“当前值”的概念。它保存了发送给消费者的最新值，当有新的观察者订阅时，会立即从 BehaviorSubject 那接收到“当前值”，在定义一个 BehaviorSubject 时需要有初始值。

在 messageService 定义一个 BehaviorSubject，初始值是 1，updateBehaviorSubject 方法把值更新为 2，代码如下：

```javascript
//messageService.ts
private behaviorSubject: BehaviorSubject<number> = new BehaviorSubject<number>(1);
currentBehaviorSubject = this.behaviorSubject.asObservable();
updateBehaviorSubject() {
    this.behaviorSubject.next(2);
}
```

并在 RxjsSubjectBComponent 中调用执行了 updateBehaviorSubject 方法，代码如下：

```javascript
//RxjsSubjectBComponent
this.messageService.updateBehaviorSubject();
```

在 RxjsSubjectAComponent 中订阅 BehaviorSubject 的值，代码如下：

```javascript
let subscription1 = this.messageService.currentBehaviorSubject
    .subscribe(val => {
        console.log("this is in RxjsSubjectAComponent and Current behavior subject value is  " + val)
    });

this.subscription.add(subscription1);
```

在 RxjsSubjectCComponent 中，等 5 秒以后再订阅 BehaviorSubject 的值，代码如下：

```javascript
setTimeout(() => {
    let subscription1 = this.messageService.currentBehaviorSubject
        .subscribe(val => {
             console.log("this is in RxjsSubjectCComponent and Current behavior subject value is  " + val)
        });

}, 5000);
```

运行效果：在 RxJSSubjectAComponent 中会拿到 1 和 2 两个值，在RxJSSubjectCComponent中会拿到 2 这个值。

```javascript
this is in RxjsSubjectAComponent and Current behavior subject value is  1
this is in RxjsSubjectAComponent and Current behavior subject value is  2
this is in RxjsSubjectCComponent and Current behavior subject value is  2
```

如果是正常的 Hot Observable 在 5 秒之后，拿不到任何值，因为在 5 秒内已经将所有值发送出去了，5 秒之后观察者再订阅，没有任何值可以发送了。但是用 BehaviorSubject 保存了发送给消费者的最新值，当有新的观察者订阅时，会立即从 BehaviorSubject 那接收到“当前值”，所以 RxJSSubjectCComponent 的订阅者会拿到 2 这个值。

## ReplaySubject

类似于 BehaviorSubject，可以发送旧值给新的订阅者，但是不仅是“当前值”，还可以是之前的旧值。

在 messageService 定义一个 ReplaySubject，<u>为新的订阅者缓冲 3 个值</u>，sendReplaySubject 方法把 1、2、3、4、5 五个值发送给订阅者，代码如下：

```javascript
//messageService.ts
private replaySubject: ReplaySubject<number> = new ReplaySubject<number>(3);
sendReplaySubject() {
    this.replaySubject.next(1);
    this.replaySubject.next(2);
    this.replaySubject.next(3);
    this.replaySubject.next(4);
    this.replaySubject.next(5);
}

getReplaySubject(): Observable<any> {
    return this.replaySubject.asObservable();
}

```

在 RxjsSubjectBComponent 中，调用 sendReplaySubject 方法：

```javascript
this.messageService.sendReplaySubject();
```

在 RxjsSubjectAComponent 中，订阅 ReplaySubject 的值，代码如下：

```javascript
let subscription2 = this.messageService.getReplaySubject()
    .subscribe(val => {
        console.log("this is in RxjsSubjectAComponent and Current Replay subject value is  " + val)
    });

this.subscription.add(subscription2);
```

在 RxjsSubjectCComponent 中，等5秒以后在订阅 replaySubject 的值，代码如下：

```javascript
setTimeout(() => {
    let subscription2 = this.messageService.getReplaySubject()
        .subscribe(val => {
            console.log("this is in RxjsSubjectCComponent and Current repaly subject value is  " + val)
        });
    this.subscription.add(subscription2);

}, 5000);
```

## AsyncSubject

只有当 Observable 执行完成之后 (执行 complete ( ))，它才会将执行的最后一个值发送给观察者。

在 messageService 定义一个 AsyncSubject，sendAsyncSubject 方法把 10001 10002 10003 三个值发给订阅者，代码如下：

```javascript
private asyncSubject: AsyncSubject<number> = new AsyncSubject<number>();

sendAsyncSubject() {
    this.asyncSubject.next(10001);
    this.asyncSubject.next(10002);
    this.asyncSubject.next(10003);
    this.asyncSubject.complete();
}
getAsyncSubject() {
    return this.asyncSubject.asObservable();
}
```

在 RxjsSubjectBComponent 中，调用 sendAsyncSubject 方法：

```javascript
this.messageService.sendAsyncSubject();
```

在 RxjsSubjectAComponent 中，订阅 asyncSubject 的值，代码如下：

```javascript
let subscription3 = this.messageService.getAsyncSubject()
    .subscribe(val => {
        console.log("this is in RxjsSubjectAComponent and Current Async subject value is  " + val)
    });

this.subscription.add(subscription3);
```

在 RxjsSubjectCComponent 中，等5秒以后在订阅 asyncSubject 的值，代码如下：

```javascript
setTimeout(() => {
    let subscription3 = this.messageService.getAsyncSubject()
        .subscribe(val => {
            console.log("this is in RxjsSubjectCComponent and Current async subject value is  " + val)
        });
    this.subscription.add(subscription3);

}, 5000);
```

效果如下：

```javascript
this is in RxjsSubjectAComponent and Current Async subject value is  10003
this is in RxjsSubjectCComponent and Current async subject value is  10003
```

