# JS：Promise实现原理

## 观察者模式

**我们先来看一个最简单的 Promise 使用**

```javascript
const p1 = new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve('result')
    },
    1000);
}) 

p1.then(res => console.log(res), err => console.log(err))
```

**由上述代码中，可以得知 Promise 的调用顺序：**

- **Promise 的构造方法接收一个 `executor()`，在 `new Promise()` 时就立即执行这个 `executor` 回调**
- **`executor()` 内部的异步任务被放入宏 / 微任务队列，等待执行**
- **`then()` 执行，收集成功失败回调，放入成功 / 失败队列**
- **`executor()` 的异步任务被执行，触发 resolve / reject，从成功 / 失败队列中取出回调依次执行**

**其实这个就是观察者模式的一个应用例子，在 Promise 中，执行的顺序为：**

- **`then()` 收集依赖**
- **异步触发 `resolve`**
- **`resolve()` 执行依赖**

**根据这个基本思路，可以基本上勾勒出 Promise 的代码实现：**

```javascript
class MyPromise {
  // 构造方法接收一个回调
  constructor(executor) {
    this._resolveQueue = []    // then收集的执行成功的回调队列
    this._rejectQueue = []     // then收集的执行失败的回调队列

    // 由于resolve/reject是在executor内部被调用, 因此需要使用箭头函数固定this指向, 否则找不到this._resolveQueue
    let _resolve = (val) => {
      // 从成功队列里取出回调依次执行
      while(this._resolveQueue.length) {
        const callback = this._resolveQueue.shift()
        callback(val)
      }
    }
    // 实现同resolve
    let _reject = (val) => {
      while(this._rejectQueue.length) {
        const callback = this._rejectQueue.shift()
        callback(val)
      }
    }
    // new Promise()时立即执行executor,并传入resolve和reject
    executor(_resolve, _reject)
  }

  // then方法,接收一个成功的回调和一个失败的回调，并push进对应队列
  then(resolveFn, rejectFn) {
    this._resolveQueue.push(resolveFn)
    this._rejectQueue.push(rejectFn)
  }
}
```



## Promise A+ 规范

**上面我们已经简单地实现了一个超低配版Promise，但我们会看到很多文章和我们写的不一样，他们的Promise实现中还引入了各种状态控制，这是由于ES6的Promise实现需要遵循Promise/A+规范，是规范对Promise的状态控制做了要求。Promise/A+的规范比较长，这里只总结两条核心规则：**

- **Promise 的本质是一个状态机，且状态只能为以下三种：Pending (等待态)、Fulfilled (执行态)、Rejected (拒绝态)，状态的变更是单向的，只能够是从 Pending 到 Fulfilled，Pending 到 Rejected，状态变更不可逆。**

- **`then()` 方法接收两个可选参数，分别是状态改变时触发的回调。`then()` 方法返回一个 Promise。`then()` 方法可以被同一个 Promise 调用多次。**

  ![alt](https://cdn.jsdelivr.net/gh/LauGaHo/blog-img@master/uPic/m78azq.png)

  **根据规范，再次补充一下 Promise 的代码：**

  ```javascript
  //Promise/A+规范的三种状态
  const PENDING = 'pending'
  const FULFILLED = 'fulfilled'
  const REJECTED = 'rejected'
  
  class MyPromise {
    // 构造方法接收一个回调
    constructor(executor) {
      this._status = PENDING     // Promise状态
      this._resolveQueue = []    // 成功队列, resolve时触发
      this._rejectQueue = []     // 失败队列, reject时触发
  
      // 由于resolve/reject是在executor内部被调用, 因此需要使用箭头函数固定this指向, 否则找不到this._resolveQueue
      let _resolve = (val) => {
        if(this._status !== PENDING) return   // 对应规范中的"状态只能由pending到fulfilled或rejected"
        this._status = FULFILLED              // 变更状态
  
        // 这里之所以使用一个队列来储存回调,是为了实现规范要求的 "then 方法可以被同一个 promise 调用多次"
        // 如果使用一个变量而非队列来储存回调,那么即使多次p1.then()也只会执行一次回调
        while(this._resolveQueue.length) {    
          const callback = this._resolveQueue.shift()
          callback(val)
        }
      }
      // 实现同resolve
      let _reject = (val) => {
        if(this._status !== PENDING) return   // 对应规范中的"状态只能由pending到fulfilled或rejected"
        this._status = REJECTED               // 变更状态
        while(this._rejectQueue.length) {
          const callback = this._rejectQueue.shift()
          callback(val)
        }
      }
      // new Promise()时立即执行executor,并传入resolve和reject
      executor(_resolve, _reject)
    }
  
    // then方法,接收一个成功的回调和一个失败的回调
    then(resolveFn, rejectFn) {
      this._resolveQueue.push(resolveFn)
      this._rejectQueue.push(rejectFn)
    }
  }
  ```



## then 的链式调用

**我们来看一下如何实现 Promise 的链式调用，这个是 Promise 实现的重点和难点，`then()` 的链式调用如下面的代码：**

```javascript
const p1 = new Promise((resolve, reject) => {
  resolve(1)
})

p1
  .then(res => {
    console.log(res)
    //then回调中可以return一个Promise
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        resolve(2)
      }, 1000);
    })
  })
  .then(res => {
    console.log(res)
    //then回调中也可以return一个值
    return 3
  })
  .then(res => {
    console.log(res)
  })

// 输出
1
2
3
```

**实现这种链式调用的思路：**

- **显然需要 `.then()` 返回一个 Promise，这样才能找到 `then()` 方法，所以我们会将 `then()` 方法的返回值包装成 Promise**

- **`.then()` 的回调需要顺序执行，以上面这段代码为例，虽然中间 return 了一个 Promise，但执行顺序仍然要保证是：1、2、3。我们要等待当前 Promise 状态变更后，再执行下一个 `then()` 收集的回调，这就要求我们对 `then()` 的返回值进行分类讨论**

  ```javascript
  // then方法
  then(resolveFn, rejectFn) {
    //return一个新的promise
    return new MyPromise((resolve, reject) => {
      //把resolveFn重新包装一下,再push进resolve执行队列,这是为了能够获取回调的返回值进行分类讨论
      const fulfilledFn = value => {
        try {
          //执行第一个(当前的)Promise的成功回调,并获取返回值
          let x = resolveFn(value)
          /**
           * 分类讨论返回值,如果是Promise,那么等待Promise状态变更,否则直接resolve
           *
           *     这里是难点，如果 回调函数的返回值 x 是一个promise，则我们需要通过 x.then再
           *  注册一个（两个）回调，目的是为了保证只有当 x 这个promise 的状态改变以后
           * （promise 就绪）才执行后续的回调（即后面 then 注册的回调）；如果 x 不是
           *  promise，则直接执行 resolve(x), 比如 resolve(number) 或者 
           *  resolve(undefined)
           *
           *  这里如果读者理解困难，一定要自己敲一遍，执行一遍示例 + console.log打日志，基本
           *  上就能理解了
           */
          x instanceof MyPromise ? x.then(resolve, reject) : resolve(x)
        } catch (error) {
          reject(error)
        }
      }
      //把后续then收集的依赖都push进当前Promise的成功回调队列中(_rejectQueue), 这是为了保证顺序调用，理解这句对于理解 then 的顺序执行以及 _resolve 函数中回调队列的执行很重要
      this._resolveQueue.push(fulfilledFn)
  
      //reject同理
      const rejectedFn  = error => {
        try {
          let x = rejectFn(error)
          x instanceof MyPromise ? x.then(resolve, reject) : resolve(x)
        } catch (error) {
          reject(error)
        }
      }
      this._rejectQueue.push(rejectedFn)
    })
  }
  ```



## 值穿透 & 状态已变更的情况

**我们已经初步完成了链式调用，但是还有两个细节需要处理：**

- **值穿透**

  根据规范，如果 `then()` 接收的参数不是 function，那么我们应该忽略他，如果没有忽略，当 `then()` 回调不为 function 时将会抛出异常，导致链式调用终端

- **处理状态为 resolve / reject 的情况**

  其实我们上边的 `then()` 的写法是对应 pending 的情况，但是有些时候，resolve / reject 在 `then()` 之前就被执行了 (比如：`Promise.resolve().then()`)，如果这个时候还把 `then()` 回调 push 进 resolve / reject 的执行队列中，那么回调将不会被执行，因此对于状态已经变为 fulfilled 或 rejected 的情况，我们直接执行 `then()` 回调：

  ```javascript
  // then方法,接收一个成功的回调和一个失败的回调
  then(resolveFn, rejectFn) {
    // 根据规范，如果then的参数不是function，则我们需要忽略它, 让链式调用继续往下执行(通过重新定义这两个函数来实现该目的)
    typeof resolveFn !== 'function' ? resolveFn = value => value : null
    typeof rejectFn !== 'function' ? rejectFn = reason => {
      throw new Error(reason instanceof Error? reason.message:reason);
    } : null
    
    // return一个新的promise
    return new MyPromise((resolve, reject) => {
      // 把resolveFn重新包装一下,再push进resolve执行队列,这是为了能够获取回调的返回值进行分类讨论
      const fulfilledFn = value => {
        try {
          // 执行第一个(当前的)Promise的成功回调,并获取返回值
          let x = resolveFn(value)
          // 分类讨论返回值,如果是Promise,那么等待Promise状态变更,否则直接resolve
          x instanceof MyPromise ? x.then(resolve, reject) : resolve(x)
        } catch (error) {
          reject(error)
        }
      }
    
      // reject同理
      const rejectedFn  = error => {
        try {
          let x = rejectFn(error)
          x instanceof MyPromise ? x.then(resolve, reject) : resolve(x)
        } catch (error) {
          reject(error)
        }
      }
    
      switch (this._status) {
        // 当状态为pending时,把then回调push进resolve/reject执行队列,等待执行
        case PENDING:
          this._resolveQueue.push(fulfilledFn)
          this._rejectQueue.push(rejectedFn)
          break;
        // 当状态已经变为resolve/reject时,直接执行then回调
        case FULFILLED:
          fulfilledFn(this._value)    // this._value是上一个then回调return的值(见完整版代码)
          break;
        case REJECTED:
          rejectedFn(this._value)
          break;
      }
    })
  }
  ```



## 兼容同步任务

**上文我们提到过，Promise 的执行顺序是：**

- **`new Promise`**
- **`then()` 收集回调**
- **resolve / reject 执行回调**

**但是如果 executor 是一个同步任务，那么顺序就会变成：**

- **`new Promise`**
- **resolve / reject 执行回调**
- **`then()` 收集回调**

**在这种情况之下，`resolve()` 的执行就会跑到了 `then()` 之前了，为了兼容这种情况，我们给 resolve / reject 执行回调的操作包一个 `setTimeout()`，让他异步执行，这里需要注意，添加 `setTimeout()` 是将任务放在了宏任务队列当中。**

```javascript
//Promise/A+规定的三种状态
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

class MyPromise {
  // 构造方法接收一个回调
  constructor(executor) {
    this._status = PENDING     // Promise状态
    this._value = undefined    // 储存then回调return的值
    this._resolveQueue = []    // 成功队列, resolve时触发
    this._rejectQueue = []     // 失败队列, reject时触发

    // 由于resolve/reject是在executor内部被调用, 因此需要使用箭头函数固定this指向, 否则找不到this._resolveQueue
    let _resolve = (val) => {
      //把resolve执行回调的操作封装成一个函数,放进setTimeout里,以兼容executor是同步代码的情况
      const run = () => {
        if(this._status !== PENDING) return   // 对应规范中的"状态只能由pending到fulfilled或rejected"
        this._status = FULFILLED              // 变更状态
        this._value = val                     // 储存当前value

        // 这里之所以使用一个队列来储存回调,是为了实现规范要求的 "then 方法可以被同一个 promise 调用多次"
        // 如果使用一个变量而非队列来储存回调,那么即使多次p1.then()也只会执行一次回调
        while(this._resolveQueue.length) {    
          const callback = this._resolveQueue.shift()
          callback(val)
        }
      }
      setTimeout(run)
    }
    // 实现同resolve
    let _reject = (val) => {
      const run = () => {
        if(this._status !== PENDING) return   // 对应规范中的"状态只能由pending到fulfilled或rejected"
        this._status = REJECTED               // 变更状态
        this._value = val                     // 储存当前value
        while(this._rejectQueue.length) {
          const callback = this._rejectQueue.shift()
          callback(val)
        }
      }
      setTimeout(run)
    }
    // new Promise()时立即执行executor,并传入resolve和reject
    executor(_resolve, _reject)
  }

  // then方法,接收一个成功的回调和一个失败的回调
  then(resolveFn, rejectFn) {
    // 根据规范，如果then的参数不是function，则我们需要忽略它, 让链式调用继续往下执行
    typeof resolveFn !== 'function' ? resolveFn = value => value : null
    typeof rejectFn !== 'function' ? rejectFn = reason => {
      throw new Error(reason instanceof Error? reason.message:reason);
    } : null
  
    // return一个新的promise
    return new MyPromise((resolve, reject) => {
      // 把resolveFn重新包装一下,再push进resolve执行队列,这是为了能够获取回调的返回值进行分类讨论
      const fulfilledFn = value => {
        try {
          // 执行第一个(当前的)Promise的成功回调,并获取返回值
          let x = resolveFn(value)
          // 分类讨论返回值,如果是Promise,那么等待Promise状态变更,否则直接resolve
          x instanceof MyPromise ? x.then(resolve, reject) : resolve(x)
        } catch (error) {
          reject(error)
        }
      }
  
      // reject同理
      const rejectedFn  = error => {
        try {
          let x = rejectFn(error)
          x instanceof MyPromise ? x.then(resolve, reject) : resolve(x)
        } catch (error) {
          reject(error)
        }
      }
  
      switch (this._status) {
        // 当状态为pending时,把then回调push进resolve/reject执行队列,等待执行
        case PENDING:
          this._resolveQueue.push(fulfilledFn)
          this._rejectQueue.push(rejectedFn)
          break;
        // 当状态已经变为resolve/reject时,直接执行then回调
        case FULFILLED:
          fulfilledFn(this._value)    // this._value是上一个then回调return的值(见完整版代码)
          break;
        case REJECTED:
          rejectedFn(this._value)
          break;
      }
    })
  }
}
```

**此外 Promise 中还具有其他方法，将在文章的下面列出实现的源码：**



## Promise.prototype.catch()

**`catch()` 方法返回一个 Promise，并且处理拒绝的情况。他的行为和调用 `Promise.prototype.then(undefined, onRejected)` 相同**

```javascript
//catch方法其实就是执行一下then的第二个回调
catch (rejectFn) {
  return this.then(undefined, rejectFn)
}
```



## Promise.prototype.finally()

**`finally()` 方法返回一个 Promise。在 Promise 结束时，无论结果是 fulfilled 还是 rejected，都会执行执行指定的回调函数，在 `finally()` 之后，还可以继续 `then()`。并且会将值原封不动的传递给后面 `then()`**

```javascript
//finally方法
finally(callback) {
  return this.then(
    // MyPromise.resolve执行回调,并在then中return结果传递给后面的Promise
    value => MyPromise.resolve(callback()).then(() => value),
    // reject同理
    reason => MyPromise.resolve(callback()).then(() => { throw reason })  
  )
}
```



## Promise.resolve()

**`Promise.resolve(value)` 方法返回一个以给定值解析后的 Promise 对象。如果该值为 Promise，则返回这个 Promise；如果这个值是 thenable (即带有 ”then“ 方法)，返回的 Promise 会跟随这个 thenable 的对象，采用他的最终状态；否则返回的 Promise 将以此值完成。此类函数将类 Promise 对象的多层嵌套展平**

```javascript
//静态的resolve方法
static resolve(value) {
  if(value instanceof MyPromise) return value // 根据规范, 如果参数是Promise实例, 直接return这个实例
  return new MyPromise(resolve => resolve(value))
}
```



## Promise.reject()

**`Promise.reject()` 方法返回一个带有拒绝原因的 Promise 对象**

```javascript
//静态的reject方法
static reject(reason) {
  return new MyPromise((resolve, reject) => reject(reason))
}
```



## Promise.all()

**`Promise.all(iterable)` 方法返回一个 Promise 实例，此实例在 iterable 参数内所有的 Promise 都完成或参数中不包含 Promise 时完成回调成 (resolve)；如果参数中 Promise 有一个失败了，此实例回调失败，失败原因是第一个失败 Promise 的结果。**

```javascript
//静态的all方法
static all(promiseArr) {
  let index = 0
  let result = []
  return new MyPromise((resolve, reject) => {
    promiseArr.forEach((p, i) => {
      //Promise.resolve(p)用于处理传入值不为Promise的情况
      MyPromise.resolve(p).then(
        val => {
          index++
          result[i] = val
          //所有then执行后, resolve结果
          if(index === promiseArr.length) {
            resolve(result)
          }
        },
        err => {
          //有一个Promise被reject时，MyPromise的状态变为reject
          reject(err)
        }
      )
    })
  })
}
```



## Promise.race()

**`Promise.race(iterable)` 方法返回一个 Promise，一旦迭代器中的某个 Promise 解决或拒绝，返回的 Promise 就会解决或拒绝。**

```javascript
static race(promiseArr) {
  return new MyPromise((resolve, reject) => {
    //同时执行Promise,如果有一个Promise的状态发生改变,就变更新MyPromise的状态
    for (let p of promiseArr) {
      MyPromise.resolve(p).then(  //Promise.resolve(p)用于处理传入值不为Promise的情况
        value => {
          resolve(value)        //注意这个resolve是上边new MyPromise的
        },
        err => {
          reject(err)
        }
      )
    }
  })
}
```

 





