---
title: JS异步
date: 2020-08-05 18:53:01
tags: 异步
category: 异步
---

# `JS`异步

## `Primose`源码

在`core.js`文件中找到`Promise`的源码

![](E:\poetry\log_code\log_img\promise\文件位置.PNG)

```javascript
module.exports = Promise;

//const back = new Promise((res , rej)=>{
//    res('data')
//})

//首先在new Promise的时候会传如一个函数，在这个函数中会有两个参数，都是函数。res代表
//运行成功之后取得的结果值，rej代表就是发生错误的结果值。调用这两个函数之后就可以在
//back.then((data)=>{})中取得相应的值
//Promise有三个状态pending , fulfilled , reject , pending 只会向fulfilled , reject
//其中一个方向进行转化，也不可反向


function Promise(fn) {  //首先是Promise的原型
  // 判断 this一定得是object不然就会报错，这个方法一定得要new出来
  if (typeof this !== 'object') {
    throw new TypeError('Promises must be constructed via new');
  }
  if (typeof fn !== 'function') {
    throw new TypeError('Promise constructor\'s argument is not a function');
  }
  this._deferredState = 0;
  this._state = 0;  //判断结果传入值的类型 ， 1代表普通的值 ， 2代表出错 ， 3代表传入的是一个Promise
  this._value = null;   //用来记录新的结果值的，不管是报错提示还是成功的返回的结果值
  this._deferreds = null;
  if (fn === noop) return;
  doResolve(fn, this);
}
```

在源码中可以看到当传入`fn 即(res , rej)=>{}`之后，进行了一些初始值的重置和检查执行了`doResolve(fn , this)`函数,`fn 指的就是传入的函数 ， this指的就是Promise`本身，

现在我们还不知道

```javascript
this._deferredState
this._state
this._value
this._deferreds
```

他们分别代表的信息是啥？

接下来就是开始看看`doResolve`函数做了什么事情

```javascript
function doResolve(fn, promise) {
  //因为只能向reslove , reject方向进行转变，所以reslove,reject只能执行其中的一个
  var done = false;      
  var res = tryCallTwo(fn, function (value) {
    if (done) return;
    done = true;
    resolve(promise, value);
  }, function (reason) {
    if (done) return;
    done = true;
    reject(promise, reason);
  });
  if (!done && res === IS_ERROR) {
    done = true;
    reject(promise, LAST_ERROR);
  }
}

//相当于对fn中传入的reslove , reject赋值然后执行真正的reslove 和 reject函数
function tryCallTwo(fn, a, b) {
  try {
    fn(a, b);
  } catch (ex) {
    LAST_ERROR = ex;
    return IS_ERROR;
  }
}
```

接下来看看真正的`reslove`和`reject`函数

```javascript
function resolve(self, newValue) {//self指的是Promise
  // Promise Resolution Procedure: https://github.com/promises-aplus/promises-spec#the-promise-resolution-procedure
  if (newValue === self) {
    return reject(
      self,
      new TypeError('A promise cannot be resolved with itself.')
    );
  }
  if (
    newValue &&
    (typeof newValue === 'object' || typeof newValue === 'function')
  ) {  //如果这个传入Promise
    var then = getThen(newValue); //判断有没有then属性
    if (then === IS_ERROR) {
      return reject(self, LAST_ERROR);
    }
    if (
      then === self.then &&
      newValue instanceof Promise  //判断是否进行链式调用，一般调用完then后执行
    ) {
      self._state = 3;
      self._value = newValue;
      finale(self);
      return;
    } else if (typeof then === 'function') {  //then属性是一个函数
      doResolve(then.bind(newValue), self);
      return;
    }
  }
  //普通的情况
  self._state = 1;
  self._value = newValue;
  finale(self);
}


function reject(self, newValue) {
  self._state = 2;
  self._value = newValue;
  if (Promise._onReject) {
    Promise._onReject(self, newValue);
  }
  finale(self);
}

//现在我们知道self._value就是用来记录新的结果值的，不管是报错提示还是成功的返回的结果值
//self._state用来判断结果传入值的类型 ， 1代表普通的值 ， 2代表出错 ， 3代表传入的是一个Promise

//看完两个函数发现，执行的最后都执行了finale这个函数应该是对结果值的一个赋值
```

## `Reslove`传入的普通值的情况

传入的普通的值的情况,执行情况

```javascript
self._state = 1;
self._value = newValue;   
```

简简单单的直接赋值



## `Reslove`传入函数/`object`的情况

```javascript
//首先他们都执行了一个函数
var then = getThen(newValue);
//getThen就是通过.then的取值简简单单的判断传入的是Promise还是function
function getThen(obj) {
  try {
    return obj.then;
  } catch (ex) {
    LAST_ERROR = ex;
    return IS_ERROR;
  }
}
```

### `Function`的情况

```javascript
doResolve(then.bind(newValue), self);  //self代表Promise
```

这里无非又是执行`reslove`和`reject`函数，最终的判断还是在是非为`Promise`上

### `Promise`的情况

```javascript
self._state = 3;
self._value = newValue;
finale(self);
```

也是简简单单的赋值



## `finale`

接下来就看看`finale`

```javascript
function finale(self) {
    //self._deferredState 默认值为0，但是一路上也未见
  if (self._deferredState === 1) {
    handle(self, self._deferreds);
    self._deferreds = null;
  }
  if (self._deferredState === 2) {
    for (var i = 0; i < self._deferreds.length; i++) {
      handle(self, self._deferreds[i]);
    }
    self._deferreds = null;
  }
}
```

所以这个函数暂时看来被没有时候作用



## `then`

`new`初始化完成，函数执行后，就需要通过`.then`进行取值

先看一个完整的实例吧

```javascript
const fs = require('fs')

const fileReader = new Promise((res , rej)=>{
    fs.readFile('../' , (data)=>{
        res(data)
    })
})

fileReader.then((data)=>{
    console.log(data)
})
```



```javascript
Promise.prototype.then = function(onFulfilled, onRejected) {
  if (this.constructor !== Promise) {
    return safeThen(this, onFulfilled, onRejected);
  }
  var res = new Promise(noop); //用于链式调用，返回的仍然是一个Promise
  handle(this, new Handler(onFulfilled, onRejected, res));
  return res;
};
```

在`handle`中就需要去看看`Handler`这个函数了

```javascript
function Handler(onFulfilled, onRejected, promise){
  this.onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : null;
  this.onRejected = typeof onRejected === 'function' ? onRejected : null;
  this.promise = promise;
}
```

`handle`函数

```javascript
function handle(self, deferred) {
  while (self._state === 3) { //传入的是Promise，直接让传入的Promise替代原本的Promise
    self = self._value;
  }
  if (Promise._onHandle) {
    Promise._onHandle(self);
  }
  if (self._state === 0) { //这表示并没有执行reslove 与 reject函数
    if (self._deferredState === 0) {
      self._deferredState = 1;
      self._deferreds = deferred;
      return;
    }
    if (self._deferredState === 1) {
      self._deferredState = 2;
      self._deferreds = [self._deferreds, deferred];
      return;
    }
    self._deferreds.push(deferred);
    return;
  }
  handleResolved(self, deferred);
}
```

如果传入的是普通的值

Promise 异步执行是通过 [asap](https://github.com/kriskowal/asap) 这个库来实现的,后面将对`asap`进行解析

```javascript
function handleResolved(self, deferred) {
  asap(function() {
    //这里主要是判断Promise状态, _state 1代表执行成功 , 2代表执行失败
    var cb = self._state === 1 ? deferred.onFulfilled : deferred.onRejected;
    if (cb === null) { //代表它的异步任务并没有完成，持续的递归等待直到任务完成，因为asap是利yong用setImmediate或process.nextTick不断的提升当前回调函数的等级，相当于一直将handleResolved函数一直卡死在check阶段或者nextTick，浏览器中就是微任务一直不断地都有，等待异步结果出现才会进行真的回到主线程中继续执行,为啥会使用setImmediate或nextTick，因为setImmediate是在check阶段，在数据更新之后，UI更新之前。nextTick是在下一次事件循环之前出现的，所以是卡在了当前事件循环的最后，下一次事件循环之前
        
      if (self._state === 1) {
        resolve(deferred.promise, self._value);
      } else {
        reject(deferred.promise, self._value);
      }
      return;
    }
    var ret = tryCallOne(cb, self._value);
    if (ret === IS_ERROR) {
      reject(deferred.promise, LAST_ERROR);
    } else {
      resolve(deferred.promise, ret);
    }
  });
}

function tryCallOne(fn, a) {
  try {
    return fn(a);
  } catch (ex) {
    LAST_ERROR = ex;
    return IS_ERROR;
  }
}
```



`Promise`的大致的流程就是如此,接下来我们就去看看`asap`吧

## `asap`分析

`asap `是 `as soon as possible` 的简称，在 `Node` 和浏览器环境下，能将回调函数以高优先级任务来执行（下一个事件循环之前），即把任务放在微任务队列中执行。

在分析`asap`之前先来回顾一下浏览器与`Node`下的`js`执行机制

### 浏览器执行机制

在`js`中有同步任务以及异步任务，因为`js`是单线程语言，所以在避免大量复杂的计算工作的时候造成的页面卡顿，有了同步以及异步，但是带来的麻烦就是函数回调的控制问题

大致的流程如下:

![](E:\poetry\log_code\log_img\promise\browser_eventLoop.jpg)

但是在异步任务中又分为宏任务和微任务，

- 宏任务 ： 优先级低，先定义的先执行。包括：ajax，setTimeout，setInterval，事件绑定，postMessage，MessageChannel（用于消息通讯）
- 微任务 ： 优先级高，并且可以插队，不是先定义先执行。包括：promise 中的 then，observer，MutationObserver，setImmediate

一般的执行流程

![](E:\poetry\log_code\log_img\promise\宏_微任务.jpg)

实例如下：

```javascript
console.log('1');
// 1 6 7 2 4 5 9 10 11 8 3
// 记作 set1
setTimeout(function () {
    console.log('2');
    // set4
    setTimeout(function() {
        console.log('3');
    });
    // pro2
    new Promise(function (resolve) {
        console.log('4');
        resolve();
    }).then(function () {
        console.log('5')
    })
})

// 记作 pro1
new Promise(function (resolve) {
    console.log('6');
    resolve();
}).then(function () {
    console.log('7');
    // set3
    setTimeout(function() {
        console.log('8');
    });
})

// 记作 set2
setTimeout(function () {
    console.log('9');
    // 记作 pro3
    new Promise(function (resolve) {
        console.log('10');
        resolve();
    }).then(function () {
        console.log('11');
    })
})
```

首先应该将执行代码，遇到`console.log`在主线程中，`setTimeout`为宏任务，`Promise`为微任务进行相应的函数注册，进入相应的队列中

![](E:\poetry\log_code\log_img\promise\browser_1.jpg)

执行完`console.log()`之后，主线程就无任务了，这个时候就会执行微任务队列中的所有任务,注册相应的函数之后，又回到了主线程空闲，检查微任务的队列。因为微任务中没有任务就会从宏任务队列中进行任务调度.

![](E:\poetry\log_code\log_img\promise\browser_2.jpg)

![](E:\poetry\log_code\log_img\promise\browser_3.jpg)

主线程执行完成之后就又会去检查微任务的队列，并且执行微任务队列中的所有的任务

![](E:\poetry\log_code\log_img\promise\browser_4.jpg)

之后在去一次执行宏任务队列中的任务.

### `Node`执行机制

`node`环境中的`JavaScript`执行机制与浏览器中的执行机制略有不同

Node.js是一个事件驱动I/O服务端JavaScript环境，基于Google的V8引擎，V8引擎执行Javascript的速度非常快，性能非常好。将libuv作为跨平台抽象层，libuv是用c/c++写成的高性能事件驱动的程序库。nodejs的原理类似c/c++系统编程中的epoll

#### 执行机制如下:

![](E:\poetry\log_code\log_img\promise\node_.jpg)

- 首先`node`是基于`chrome`的`V8`引擎的,所以`V8`引擎的用于解析`JavaScript`的脚本
- 被解析的脚本，调用`node`的`api`
- `libuv`库负责`Node API`的执行。它将不同的任务分配给不同的线程，形成一个`Event Loop`（事件循环），以异步的方式将任务的执行结果返回给`V8`引擎
- `V8`引擎再将结果返回给用户
- `node.js`的主进程和`Event Loop`是分开的，主进程执行完同步代码就不执行了，剩下的就交给事件循环去做了

#### 事件循环

 ![](E:\poetry\log_code\log_img\promise\node_loop.webp)

在`node`的事件循环中一共分为上述的几个阶段，然后在上述的阶段中循环执行

- `timers` : 
  - 执行 `setTimeout()` 和 `setInterval()`的回调
  - 一个`timer`指定一个下限时间而不是准确时间，在达到这个下限时间后执行回调。在指定时间过后，`timers`会尽可能早地执行回调，但系统调度或者其它回调的执行可能会延迟它们。
  - 注意：这个下限时间有个范围：[1, 2147483647]，如果设定的时间不在这个范围，将被设置为1。
- `I/O callbacks:`
  - 此阶段执行某些系统操作的回调，例如`TCP`错误的类型。 例如，如果`TCP`套接字在尝试连接时收到 `ECONNREFUSED`，则某些系统希望等待报告错误。 这将操作将等待在`I/O`回调阶段执行
- `idle, prepare` :
  - 仅为`node`内部使用
- `poll` ： 
  - 获取新的`I/O`事件, 例如操作读取文件等等，适当的条件下`node`将阻塞在这里
  - 两个主要功能:
    - 执行下限时间已经达到的`timers`的回调
    - 然后处理 `poll` 队列里的事件。 当`event loo`进入 `poll` 阶段，并且 没有设定的 `timers（there are no timers scheduled）`，会发生下面两件事之一：
      - 如果 `poll`队列不空，event loop会遍历队列并同步执行回调，直到队列清空或执行的回调数到达系统上限；
      - 如果 poll 队列为空，则发生以下两件事之一
        - 如果代码已经被`setImmediate()`设定了回调, `event loop`将结束`poll` 阶段进入 `check `阶段来执行 `check` 队列（里面的回调 `callback`）
        - 如果代码没有被`setImmediate()`设定回调，event loop将阻塞在该阶段等待回调被加入 poll 队列，并立即执行。
  - 但是，当`event loop`进入 `poll `阶段，并且 有设定的`timers`，一旦`poll` 队列为空（`poll` 阶段空闲状态）： `event loop`将检查`timers`,如果有1个或多个`timers`的下限时间已经到达，`event loop`将绕回 `timers` 阶段，并执行 `timer` 队列
- `check`:
  - 执行 `setImmediate()` 设定的callbacks , 这个阶段允许在 `poll `阶段结束后立即执行回调。如果 poll 阶段空闲，并且有被`setImmediate()`设定的回调，`event loop`会转到 `check` 阶段而不是继续等待。
  - `setImmediate()` 实际上是一个特殊的`timer`，跑在`event loop`中一个独立的阶段。它使用`libuv`的`API `来设定在 `poll `阶段结束后立即执行回调。
  - 通常上来讲，随着代码执行，`event loop`终将进入`poll `阶段，在这个阶段等待`ncoming connection`, `request` 等等。但是，只要有被`setImmediate()`设定了回调，一旦` poll `阶段空闲，那么程序将结束 `poll `阶段并进入`check `阶段，而不是继续等待` poll `事件们 `（poll events）`
- `close callbacks` :
  - 比如 `socket.on(‘close’, callback)` 的callback会在这个阶段执行

##### `nextTick `

`process.nextTick` 不属于事件循环的任何一个阶段，它属于该阶段与下阶段之间的过渡, 即本阶段执行结束, 进入下一个阶段前, 所要执行的回调。在`timers`之前

`nextTick`的递归危害

因为`nextTick`可以插队，在一次循环结束,`timers`之前进行执行，他的执行可能会阻塞后面`node`的事件循环,导致了其它事件处理程序处于饥饿状态. 为了防止递归产生的问题, `Node.js `提供了一个 `process.maxTickDepth (默认 1000)`。

##### `nextTick` 与 `Promise` 

`nextTick`与`Promise`在`node`中称为微任务，不在事件循环之中。执行在同一阶段。关于`nextTick`与`Promise`先后顺序的问题 , `nextTick` 永远先于`Promise`关于`Promise`的实现与`nextTick`也有关系



### `asap`源码

这里分析的主要是在`node`环境下进行的`asap`实现

主要是下面的两个文件

[`asap.js`](https://github.com/kriskowal/asap/blob/master/asap.js)

```javascript
"use strict";

var rawAsap = require("./raw");
var freeTasks = [];
/**
在自身事件中返回以下内容后，尽快调用任务优先于IO事件。 任务中引发的异常可以由`process.on（“ uncaughtException”）或`domain.on（“ error”）`，否则会导致进程崩溃。 如果错误得到解决，所有后续任务将恢复。
 */
module.exports = asap;
function asap(task) {
    var rawTask;
    if (freeTasks.length) {
        rawTask = freeTasks.pop();
    } else {
        rawTask = new RawTask();
    }
    rawTask.task = task;
    rawTask.domain = process.domain;
    rawAsap(rawTask);
}

function RawTask() {
    this.task = null;
    this.domain = null;
}

RawTask.prototype.call = function () {
    if (this.domain) {
        this.domain.enter();
    }
    var threw = true;
    try {
        this.task.call();
        threw = false;
        // If the task throws an exception (presumably) Node.js restores the
        // domain stack for the next event.
        if (this.domain) {
            this.domain.exit();
        }
    } finally {
        // We use try/finally and a threw flag to avoid messing up stack traces
        // when we catch and release errors.
        if (threw) {
            // In Node.js, uncaught exceptions are considered fatal errors.
            // Re-throw them to interrupt flushing!
            // Ensure that flushing continues if an uncaught exception is
            // suppressed listening process.on("uncaughtException") or
            // domain.on("error").
            rawAsap.requestFlush();
        }
        // If the task threw an error, we do not want to exit the domain here.
        // Exiting the domain would prevent the domain from catching the error.
        this.task = null;
        this.domain = null;
        freeTasks.push(this);
    }
};
```

[raw.js]( https://github.com/kriskowal/asap/blob/master/raw.js)

```javascript
"use strict";

var domain; // The domain module is executed on demand
var hasSetImmediate = typeof setImmediate === "function";  //版本兼容

module.exports = rawAsap;
function rawAsap(task) {
    if (!queue.length) {
        requestFlush();
        flushing = true;
    }
    // Avoids a function call
    queue[queue.length] = task;
}

var queue = [];
// Once a flush has been requested, no further calls to `requestFlush` are
// necessary until the next `flush` completes.
var flushing = false;
// The position of the next task to execute in the task queue. This is
// preserved between calls to `flush` so that it can be resumed if
// a task throws an exception.
var index = 0;
// If a task schedules additional tasks recursively, the task queue can grow
// unbounded. To prevent memory excaustion, the task queue will periodically
// truncate already-completed tasks.
var capacity = 1024;

function flush() {
    while (index < queue.length) {
        var currentIndex = index;
        index = index + 1;
        queue[currentIndex].call();
        if (index > capacity) {
            // 防止内存泄漏
            for (var scan = 0, newLength = queue.length - index; scan < newLength; scan++) {
                queue[scan] = queue[scan + index];
            }
            queue.length -= index;
            index = 0;
        }
    }
    queue.length = 0;
    index = 0;
    flushing = false;
}

rawAsap.requestFlush = requestFlush;
function requestFlush() {
    // Ensure flushing is not bound to any domain.
    // It is not sufficient to exit the domain, because domains exist on a stack.
    // To execute code outside of any domain, the following dance is necessary.
    var parentDomain = process.domain;
    if (parentDomain) {
        if (!domain) {
            // Lazy execute the domain module.
            // Only employed if the user elects to use domains.
            domain = require("domain");
        }
        domain.active = process.domain = null;
    }

    // `setImmediate` is slower that `process.nextTick`, but `process.nextTick`
    // cannot handle recursion.
    // `requestFlush` will only be called recursively from `asap.js`, to resume
    // flushing after an error is thrown into a domain.
    // Conveniently, `setImmediate` was introduced in the same version
    // `process.nextTick` started throwing recursion errors.
    if (flushing && hasSetImmediate) {
        setImmediate(flush);
    } else {
        process.nextTick(flush);
    }

    if (parentDomain) {
        domain.active = process.domain = parentDomain;
    }
}
```

总结来说就是`asap`将添加的任务在事件循环的`check`阶段或者在事件循环之前进行.`rawAsap `方法是通过 `setImmediate` 或 `process.nextTick` 来实现异步执行的任务栈，而 `asap `方法是对 `rawAsap` 方法的进一步封装，通过缓存的 `domain` 和 `try/finally` 实现了即使某个任务抛出异常也可以恢复任务栈的继续执行（再次调用`rawAsap.requestFlush`）。



## `Generator`与`async/await`

首先简单回顾一下`Generator`用法

```javascript
let count = 0

function* G(){
    const a = yield ++count
    console.log(a)
}

const num =  G()

console.log(num.next().value)   //1
```

`Generator`既是一个迭代器又是一个状态机，`ES6`[Iterator的介绍](https://es6.ruanyifeng.com/#docs/iterator)

**生成器（Generator）**对象是ES6中新增的语法，和Promise一样，都可以用来异步编程。但与Promise不同的是，它不是使用JS现有能力按照一定标准制定出来的，而是一种新型底层操作，async/await就是在它的基础上实现的。实际上是对于函数栈的操作。[具体详情请看](https://github.com/sisterAn/blog/issues/20)

### `co`函数

`co`函数用于 Generator 函数的自动执行，co 函数返回一个 Promise 对象，因此可以用 then 方法添加回调函数。

实例：

```javascript
// 还是之前的thunk函数
function Thunk(fn) {
  return function(...args) {
    return function(callback) {
      return fn.call(this, ...args, callback)
    }
  }
}

// 将我们需要的request转换成thunk
const request = require('request');
const requestThunk = Thunk(request);

const baiduRequest = requestThunk('https://www.baidu.com');

// 引入co执行, co的参数是一个Generator
// co的返回值是一个Promise，我们可以用then拿到他的结果
const co = require('co');
co(function* () {
  const r1 = yield baiduRequest;
  const r2 = yield baiduRequest;
  const r3 = yield baiduRequest;
  
  return {
    r1,
    r2,
    r3,
  }
}).then((res) => {
  // then里面就可以直接拿到前面返回的{r1, r2, r3}
  console.log(res);
});
```

 #### 源码分析

```javascript
function co(gen) {
  var ctx = this;
  var args = slice.call(arguments, 1); //拿到gen后面的函数的传参

  return new Promise(function(resolve, reject) {
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    onFulfilled();

    function onFulfilled(res) {
      var ret;
      try {
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      next(ret);
      return null;
    }

    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    function next(ret) {
      if (ret.done) return resolve(ret.value);
      var value = toPromise.call(ctx, ret.value);
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
    }
  });
}
```

`co`函数就是一直递归执行`next`完成`Generator`函数，并返回一个`Promise`

### `async/await`

用法:

```javascript
const fetch = require('fetch')  //执行返回promise

const ajax = async(url1 , url2){
    const res1 = await fetch(url1)
    if(...) return res1
    const res2 = await fetch(url2)
    return [url1 , url2]
}
```

其实`async/await`与上面的`co`函数十分的相似，也是`Generator`的语法糖而已







