---
title: 从0到1实现Promise
date: 2019-11-24 15:43:46
tags:
  - 前端
  - 规范
categories:
	- 技术文档
---
## Promise 是什么

在实现 Promise 之前，我们首先要了解两点：

- Promise 是什么
- 官方的 Promise 有哪些东西

对于第一个问题，在 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise) 的解释，简的来说 Promise 就是一个为了解决异步调用问题，让使用者能以同步方式抽象处理异步调用的代理对象。通常会将其与 `callback` 的回调地狱进行对比，这里我们给出一个简单的小例子。

```typescript
// 使用回调
asyncFunc((err, res) => {
  // ...
  otherAsyncFunc((err2, res2) => {
    // ...
    // 继续回调将导致嵌套层级更深而形成回调地狱
  })
});

// 使用 Promise, 彻底解决了回调地狱问题
asyncFunc()
  .then(res => otherAsyncFunc())
  .then(res2 => { /* do something*/ })
  .catch(err => {})
```

接下来让我们看看第二个问题，ES6 中的 Promise 为我们提供了哪些标准 API。基于 [Promises/A+](https://promisesaplus.com/) 规范，ES6 提供了 Promise 的构造函数，用于处理异步回调的 `then`，进行错误处理的 `catch`，进行清理的清道夫`finally` 等函数。除此外，还提供了部分便捷的辅助功能静态函数诸如：`Promise.all`, `Promise.resolve` 等等。

而我们今天要实现的 Promise，就只基于 [Promises/A+](https://promisesaplus.com/) 规范将核心功能实现，暂且不考虑各种辅助功能。

## Promise 使用

在了解一个东西的原理之前，我们得先学会使用。因此，让我们先看一个简单的使用 Promise 解决异步问题的小例子。该例子发起一个异步请求，如果请求成功，输出 `success`，失败则输出 `failed`。

```typescript
new Promise((resolve, reject) => {
  asyncFunction((err, data) => {
    if (err) { // 失败
      reject('failed');
    } else {
      resolve('success');
    }
  })
})
.then(console.log)
.catch(console.log)
```

## Promise 类型定义

为了更便于理解和展现，笔者将通过 [Typescript](http://www.typescriptlang.org/index.html) 进行本次代码书写。首先，让我们先通过 Promise/A+ 的规范进行类型定义。

```typescript
declare enum PromiseState {
  PENDING = 'pending',
  FULFILLED = 'fulfilled',
  REJECTED = 'rejected'
};

declare interface Thenable {
  then: (onFulfilled?: Function, onRejected?: Function) => Thenable;
};

declare type PromiseFunction = (onFulfilled?: Function, onRejected?: Function) => void
```

Promise 的本质是一个具有三个状态的单向状态机。仅允许从 `pending` 状态转向 `fulfilled` 的成功状态或者 `rejected` 的失败状态。

![](https://wiki.jikexueyuan.com/project/javascript-promise-mini-book/images/1.1.png)

紧接着，我们定义上 Promise 的相关属性。

```typescript
declare interface MyPromise extends Thenable {
  state: PromiseState;
  data: any;
  // 入参
  fn: PromiseFunction;
  // 处理回调序列，后文具体描述
  resolvedCbArray: Function[];
  rejectedCbArray: Function[];
  // 核心函数
  resolve: (value: any) => void;
  reject: (value: any) => void;
  catch: (err: Exception) => Thenable
};
```

上述 MyPromise 就是我们今天最终要实现的版本，现在就让我们一起进入 Promise 的实现。

## 简易 Promise 的实现

```typescript
class MyPromise {
  data = undefined;

  constructor(fn) {
    fn(this.resolve, this.reject);
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      if (onFulfilled) {
        resolve(onFulfilled(this.data));
      }
      if (onRejected) {
        reject(onRejected(this.data));
      }
    });
  }

  catch(onRejected) {
    return this.then(null, onRejected);
  }

  resolve(value) {
    this.data = value;
  }

  reject(value) {
    this.data = value;
  }
}
```

以上就是最基础版本的 Promise 结构，咋一看什么都没有，实际.. 是的。连最基本的状态都没有。但却实现了 Promise 的基本功能，我们可以通过一个小例子校验下：

```typescript
new MyPromise((resolve, reject) => {
  resolve('success');
}).then(console.log);

new MyPromise((resolve, reject) => {
  reject('failed');
}).then(console.log);
```

在这个简易版本中，`resolve` 和 `reject` 仅仅是记录缓存入参的 data 数据而未做状态转移。重点让我们看看 `then` 和 `catch` 方法。

参看 `then` 方法的函数定义，这是一个接收两个可选函数，返回一个新的 Promise 的方法。其中，第一个方法代表成功，第二个方法代表失败。我们定义了 `promise2` 和 `data2` 存储返回的数据结构，判断成功则构建一个新的作为 resolve 处理的 promise，否则构建一个作为 reject 处理的 promise。

在理解了 `then` 方法后，再看 `catch` 方法其实就很明了了，就是将 `then` 方法中的 resolve 部分去除即可，因为进入该方法则代表了失败。

## 添加状态

紧接着，我们为 MyPromise 添加上状态机。

```typescript
class MyPromise {
  // ...
  state = 'pending'

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const resolved = onFulfilled && typeof onFulfilled === 'function' ? onFulfilled : () => { };
      const rejected = onRejected && typeof onRejected === 'function' ? onRejected : () => { };
      if (onFulfilled) {
      }
      if (onRejected) {
        reject(onRejected(this.data));
      }
      switch (this.state) {
        case 'pending':
          // do something
          break;
        case 'fulfilled':
          resolve(resolved(this.data));
          break;
        case 'rejected':
          reject(rejected(this.data));
        default:
          break;
      }
    });
  }

  resolve(value) {
    this.state = 'fulfilled';
    //...
  }

  reject(value) {
    this.state = 'rejected';
    //...
  }
}
```

状态的添加，主要位于几个地方

- 状态初始化为 `pending`
- `resolve` 触发后，状态改变为 `fulfilled`
- `reject` 触发后，状态改变为 `rejected`
- 在 `then` 函数中，根据状态进行处理，`pending` 状态，我们暂且留个悬念之后说明。因为无论何时，Promise 只可能存在于某个固定状态下。因此，当处于 `fulfilled` 状态下，我们只需要考虑 `onFulfilled` 函数，同理，只在 `rejected` 状态下考虑 `onRejected` 函数。

至此，状态机的添加完成，细心的读者可能会发现，在我们使用简易版 MyPromise 的时候，采用的并非异步函数而是同步函数，这是因为在运行 `MyPromise(fn).then(() => {/*...*/})` 的时候，`then` 函数是紧接着 `fn` 函数运行的，如此一来，我们就只能处理同步的情况，对于异步的情况，现有的情况下数据将始终是 `undefined`。

至于如何解决这个问题，就轮到我们之前埋下伏笔的 `pending` 状态下的 `then` 函数的对应操作了。

## 解决异步处理

解决异步问题的思路，我们可以采用先存储，执行的时候调用来解决。在 `pending` 状态下，先存储对应回调，待最后状态终结的时候再执行。

根据思路，我们对之前代码进一步进行改进：

```typescript
class MyPromise {
  // ...
  resolvedCbArray = [];
  rejectedCbArray = [];

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      // ...
      switch (this.state) {
        case 'pending':
          this.resolvedCbArray.push(() => resolved(this.data));
          this.rejectedCbArray.push(() => rejected(this.data));
          break;
        // ......
      }
    });
  }

  resolve(value) {
    //...
    this.resolvedCbArray.forEach(cb => cb(value));
  }

  reject(value) {
    //...
    this.rejectedCbArray.forEach(cb => cb(value));
  }
}
```

通过添加两个回调函数存储数组，之所以是数组是因为可能存在多个 then 函数，在 pending 状态下，将回调暂存对应数组。之后在终结状态时（resolve 和 reject 调用改变状态）根据状态依次调用对应的函数。

## ResolvePromise 实现

在目前的实现中，我们都没有考虑到 Promise 在处理过程中结果为 Promise 的情况，如果是一个 Promise，那么我们应该进行该 Promise 的执行并取其结果作为最终结果传递。在 Promise/a+ 规范中，Promise 的处理过程用 ResolvePromise 表示，具体规则如下：

1. 如果`promise` 和 `x` 指向相同的值, 使用 `TypeError`做为原因将`promise`拒绝。
2. 如果 `x` 是一个`promise`, 根据状态:
    1. 如果 `x` 是 pending 状态，`promise` 必须保持 pending 直到 `x` fulfilled 或 rejected.
    2. 如果 `x` 是 fulfilled 状态，将 `x` 的值用于 fulfill `promise`.
    3. 如果 `x` 是 rejected 状态, 将 `x` 的原因用于reject `promise`..
3. 如果 `x` 是一个对象或一个函数：
    1. 将 `then` 赋为 `x.then`. [3.5]
    2. 如果在取 `x.then` 值时抛出了异常，则以这个异常做为原因将 `promise` 拒绝。
    3. 如果 `then` 是一个函数， 以 `x` 为 `this` 调用 `then` 函数， 且第一个参数是 `resolvePromise`，第二个参数是 `rejectPromise`，且：
        1. 当 `resolvePromise` 被以 `y`为参数调用, 执行 `[[Resolve]](promise, y)`.
        2. 当 `rejectPromise` 被以 `r` 为参数调用, 则以`r`为原因将`promise`拒绝。
        3. 如果 `resolvePromise` 和 `rejectPromise` 都被调用了，或者被调用了多次，则只第一次有效，后面的忽略。
        4. 如果在调用 `then` 时抛出了异常，则：
            1. 如果 `resolvePromise` 或 `rejectPromise` 已经被调用了，则忽略它。
            2. 否则, 以 `e` 为 reason 将 `promise` 拒绝。
    4. 如果 `then` 不是一个函数，则 以 `x` 为值 fulfill `promise`。
4. 如果 `x` 不是对象也不是函数，则以 `x` 为值 fulfill `promise`。

将其用程序语言翻译如下：

```typescript
resolvePromise(x: any, promise: MyPromise, resolve: HandleFunction, reject: HandleFunction ): void {
  let called = false;
  if (promise === x) { // promise and x refer to the same object
    reject(new Error('TypeError'));
  } else {
    try {
      // x has a then method
      if (typeof x === "object" && x.then && typeof x.then === 'function') { 
        x.then(res => {
          if (called) return;
          called = true;
          this.resolvePromise(res, x, resolve, reject);
        }, (rej) => {
          if (called) return;
          called = true;
          reject(rej);
        });
      } else {
        if (called) return;
        called = true;
        resolve(x);
      }
    } catch(e) {
      if (called) return;
      called = true;
      reject(e);
    }
  }
}
```

在这里，我们额外定义了一个 called 变量用来确保过程中的 resolvePromise 或者 rejectPromise 只被调用一次。

至此，我们从一个最简易版本的 MyPromise，一步一步进行迭代，依次加上了单向状态机和对异步调用状态的处理。如果按照画马的步骤，那么这时候我们应该算是已经完成了前四步，就差最后一步了（笑）。

[](https://www.notion.so/af561f2b95f746458f937b1d3b116da8#be58a20f43a34477b7f0e7e07f73fbe3)

当然笔者不可能真的如上图所描述一般。在最后一步，笔者会带领大家在之前基础上，添加上对应的Typescript 类型校验，错误处理和一些小细节，就实现了属于我们自己的 Promise。

## 完整实现

```typescript
declare enum PromiseState {
  PENDING = 'pending',
  FULFILLED = 'fulfilled',
  REJECTED = 'rejected'
};

declare interface Thenable {
  then: (onFulfilled?: HandleFunction, onRejected?: HandleFunction) => Thenable;
};

declare type PromiseFunction = (onFulfilled?: HandleFunction, onRejected?: HandleFunction) => void

declare type HandleFunction = HandleFunction;

declare interface MyPromise extends Thenable {
  state: PromiseState;
  data: any;
  // 入参
  fn: PromiseFunction;
  // 处理回调序列，后文具体描述
  resolvedCbArray: HandleFunction[];
  rejectedCbArray: HandleFunction[];
  // 核心函数
  resolvePromise: (x: any, promise: MyPromise, resolve: HandleFunction, reject: HandleFunction) => void;
  resolve: HandleFunction;
  reject: HandleFunction;
  catch: (err: Exception) => Thenable
};

declare type Exception = Error;

class MyPromise implements MyPromise {
  state: PromiseState;
  fn: PromiseFunction;
  data: any;
  resolvedCbArray: HandleFunction[] = [];
  rejectedCbArray: HandleFunction[] = [];

  constructor(fn: PromiseFunction) {
    this.state = PromiseState.PENDING;
    try {
      fn(this.resolve, this.reject);
    } catch(err) {
      this.reject(err);
    }
  }

  then(onFulfilled?: HandleFunction, onRejected?: HandleFunction): Thenable {
    return new MyPromise((resolve, reject) => {
      let x;
      const resolved = onFulfilled && typeof onFulfilled === 'function' ? onFulfilled : () => { };
      const rejected = onRejected && typeof onRejected === 'function' ? onRejected : () => { };

      switch (this.state) {
        case 'pending':
          this.resolvedCbArray.push(() => {
            try {
              this.resolvePromise(resolved(this.data), this, resolve, reject);
            } catch (e) {
              reject(e);
            }
          });
          this.rejectedCbArray.push(() => {
            try {
              this.resolvePromise(rejected(this.data), this, resolve, reject);
            } catch (e) {
              reject(e);
            }
          });
          break;
        case 'fulfilled':
          try {
            x = resolved(this.data);
            this.resolvePromise(x, this, resolve, reject);
          } catch(reason) {
            reject(reason);
          }

          break;
        case 'rejected':
          try {
            x = rejected(this.data);
            this.resolvePromise(x, this, resolve, reject);
          } catch(reason) {
            reject(reason);
          }
          break
        default:
          break;
      }
    });
  }
  
  resolvePromise(x: any, promise: MyPromise, resolve: HandleFunction, reject: HandleFunction ): void {
    let called = false;
    if (promise === x) { // promise and x refer to the same object
      reject(new Error('TypeError'));
    } else {
      try {
        // x has a then method
        if (typeof x === "object" && x.then && typeof x.then === 'function') { 
          x.then(res => {
            if (called) return;
            called = true;
            this.resolvePromise(res, x, resolve, reject);
          }, (rej) => {
            if (called) return;
            called = true;
            reject(rej);
          });
        } else {
          if (called) return;
          called = true;
          resolve(x);
        }
      } catch(e) {
        if (called) return;
        called = true;
        reject(e);
      }
    }
  }

  catch(onRejected: HandleFunction) {
    return this.then(null, onRejected);
  }

  resolve(value: any) {
    if (this.state === PromiseState.PENDING) {
      this.state = PromiseState.FULFILLED;
      this.data = value;
      this.resolvedCbArray.forEach(cb => cb(value));
    }
  }

  reject(value: any) {
    if (this.state === PromiseState.PENDING) {
      this.state = PromiseState.REJECTED;
      this.data = value;
      this.rejectedCbArray.forEach(cb => cb(value));
    }
  }
}
```

## 参考文献

1. [Promise/ A+ 规范](https://promisesaplus.com/)
2. [MDN Promise](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)
3. [Promise 实现](https://segmentfault.com/a/1190000016550260#articleHeader3)