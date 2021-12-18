## 链接

- [MDN - Promise 文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [Promise 规范](https://promisesaplus.com/)
- [Bilibili - 手写 Promise](https://www.bilibili.com/video/BV1BK4y1D7Wc/)
- [Bilibili - 函数式 & Promise](https://www.bilibili.com/video/BV1vv411N7jt)

## 核心目的

- 只为了解 Promise 的常见用法，有时候，简单实现一个 Promise 比看各种文档更有用。

- 不去完善详细的细节；

    - 比如只用 instanceof 判断是否为 promise，不去根据 ducking type 判断。

    - 比如不解决 promise 各种嵌套问题，像是 resolve 中传了一个 promise 对象。

## 实现简单的构造函数

### 代码

```ts
// Promise 有三种状态：pending、fulfilled、rejected
// [A promise must be in one of three states: pending, fulfilled, or rejected.]
// (https://promisesaplus.com)
enum Status {
  pending = "pending",
  rejected = "rejected",
  fulfilled = "fulfilled",
}

class MiniPromise<Type> {
  status: Status = Status.pending;
  // 因为状态不可能共存，所以只需要一个变量存储结果
  result: any = null;

  constructor(
    // `executor()` 接收两个函数参数，这两个函数参数又分别接收一个任意类型的参数。
    // [resolutionFunc and rejectionFunc are also functions, and you can give them whatever actual names you want.
    // Their signatures are simple: they accept a single parameter of any type.]
    // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise#parameters)
    executor: (
      resolve: (value: Type) => void,
      reject: (reason: any) => void
    ) => void
  ) {
    const resolve = (value: Type) => {
      // 只有 pending 状态下才能改变状态
      // [When fulfilled, a promise: must not transition to any other state.]
      // (https://promisesaplus.com)
      if (this.status === Status.pending) {
        this.status = Status.fulfilled;
        this.result = value;
      }
    };

    const reject = (reason: any) => {
      // 只有 pending 状态下才能改变状态
      // [When rejected, a promise: must not transition to any other state.]
      // (https://promisesaplus.com)
      if (this.status === Status.pending) {
        this.status = Status.rejected;
        this.result = reason;
      }
    };

    try {
      // `executor()` 会在构造函数中执行
      // [A function to be executed by the constructor,
      // during the process of constructing the new Promise object.]
      // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise#parameters)
      executor(resolve, reject);
    } catch (error) {
      // `executor()` 中如果有异常，那状态为 `rejected`。
      // [If an error is thrown in the executor, the promise is rejected.]
      // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise#parameters)
      reject(error);
    }
  }
}
```

### 测试用例

```ts
const P = MiniPromise;

console.log("\n1. `executor()` 立即执行");
new P(() => console.log("立即执行！！！"));

console.log("\n2. `resolve()` & `reject()` & `throw` 三种改变状态的方式");
console.log(new P<number>((resolve) => resolve(100)));
console.log(new P((resolve, reject) => reject(-100)));
console.log(
  new P(() => {
    throw new Error("错误");
  })
);

console.log("\n3. 只有 pending 状态下才能改变状态");
console.log(
  new P<number>((resolve, reject) => {
    resolve(100);
    reject(-100);
  })
);
console.log(
  new P<number>((resolve, reject) => {
    reject(-100);
    resolve(100);
  })
);
```

### 结果

![](./手写`Promise()`-basic_constructor.png)

## 实现简单的 then

### 代码

```ts
...

class MiniPromise<Type> {
  ...
  fulfilledCallbacks: (() => void)[] = [];
  rejectedCallbacks: (() => void)[] = [];

  constructor(...) {
    const resolve = (value: Type) => {
      ...
      if (this.status === Status.pending) {
        ...
        this.fulfilledCallbacks.forEach((func) => func());
      }
    };

    const reject = (reason: any) => {
      ...
      if (this.status === Status.pending) {
        ...
        this.rejectedCallbacks.forEach((func) => func());
      }
    };
    ...
  }

  then = (
    // `then()` 接收两个函数参数
    // [A promise’s then method accepts two arguments:]
    // (https://promisesaplus.com/#the-then-method)
    // 两个函数参数都是可选参数
    // [Both onFulfilled and onRejected are optional arguments:]
    // (https://promisesaplus.com/#point-23)
    // `onFulfilled()` 的默认值是返回参数
    // [If it is not a function, it is internally replaced with an
    // "Identity" function (it returns the received argument).]
    // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then#parameters)
    // `onRejected()` 的默认值是抛出异常
    // [If it is not a function, it is internally replaced with a "Thrower"
    // function (it throws an error it received as argument).]
    // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then#parameters)
    onFulfilled: (value: Type) => unknown = (value: Type) => value,
    onRejected: (reason: any) => unknown = (reason: any) => {
      throw reason;
    }
  ) => {
    if (this.status === Status.fulfilled) {
      // 如果状态是 fulfilled，调用 `onFulfilled()`。
      // [If onFulfilled is a function: it must be called after promise is rejected,
      // with promise’s reason as its first argument.]
      // (https://promisesaplus.com/#point-26)
      onFulfilled(this.result);
    } else if (this.status === Status.rejected) {
      // 如果状态是 rejected，调用 `onRejected()`。
      // [If onRejected is a function: it must be called after promise is rejected,
      // with promise’s reason as its first argument.]
      // (https://promisesaplus.com/#point-30)
      onRejected(this.result);
    } else {
      // this.status === Status.pending
      // 发布 - 订阅
      // `then()` 可以被调用多次，因此需要
      // [then may be called multiple times on the same promise.]
      // (https://promisesaplus.com/#point-36)
      this.fulfilledCallbacks.push(() => onFulfilled(this.result));
      this.rejectedCallbacks.push(() => onRejected(this.result));
    }
  };
}
```

### 测试用例

```ts
const P = MiniPromise;

console.log("\n1. 同步情况，一个 promise 调用多次 `then()`");
const promise1 = new P<number[]>((resolve) => resolve([]));
console.log('%cApp.tsx line:10 promise1', 'color: #007acc;', promise1);
promise1.then((value) => {
  value.push(1);
  console.log(value);
});
promise1.then((value) => {
  value.push(2);
  console.log(value);
});

const promise2 = new P((resolve, reject) => {
  setTimeout(() => {
    console.log("\n2. 异步情况下，一个 promise 调用多次 `then()`");
    reject([]);
  }, 200);
});
promise2.then(undefined, (reason) => {
  reason.push(-1);
  console.log(reason);
});
promise2.then(undefined, (reason) => {
  reason.push(-2);
  console.log(reason);
});

console.log("\n3. `then()` 不传参数");
const promise3 = new P<number>((resolve) => resolve(100));
promise3.then();
promise3.then((value) => console.log(value));
```

### 结果

![](./手写`Promise()`-basic_then.png)

## 实现简单的链式 then

### 代码

```ts
...

class MiniPromise<Type> {
  ...

  then = (...) => {
    // `then()` 方法要返回一个 promise
    // [then must return a promise]
    // (https://promisesaplus.com/#point-40)
    return new MiniPromise((resolve, reject) => {
      const hanldePromiseOrPlain = (isFulfilled: boolean) => {
        // 注意 `executor()` 中的 try-catch 只能捕获同步代码，这里的代码可能在回调函数中调用
        // [onFulfilled or onRejected must not be called until the execution context stack contains only platform code.]
        // (https://promisesaplus.com/#point-34)
        try {
          const obj = isFulfilled ? onFulfilled(this.result) : onRejected(this.result);
          // 这里简化了判断，直接用 instanceof，实际情况应该使用 ducking type
          // `obj` 不是 promise，就返回一个 fulfilled promise，值为 x
          // [If x is not an object or function, fulfill promise with x.]
          // (https://promisesaplus.com/#point-64)
          // `obj` 如果是 promise，那么新 promise 的状态和当前的 promise 状态一样
          // 写全可能看得更清晰：`obj.then((value) => resolve(value), (reason) => reject(reason))`，
          // 这个操作就等价于复制了一份 promise，如果你成功了，那你通过 `resolve()` 将成功状态和结果传给我，
          // 如果你失败了，那你通过 `reject()` 将失败状态和结果传给我。
          // [If x is a promise, adopt its state: If/when x is fulfilled, fulfill promise with the same value.
          // If/when x is rejected, reject promise with the same reason.]
          // (https://promisesaplus.com/#point-49)
          obj instanceof MiniPromise ? obj.then(resolve, reject) : resolve(obj);
        } catch (error) {
          reject(error);
        }
      };
      if (this.status === Status.fulfilled) {
        hanldePromiseOrPlain(true);
      } else if (this.status === Status.rejected) {
        hanldePromiseOrPlain(false);
      } else {
        this.fulfilledCallbacks.push(() => hanldePromiseOrPlain(true));
        this.rejectedCallbacks.push(() => hanldePromiseOrPlain(false));
      }
    });
  };
}
```

### 测试用例

```ts
const P = MiniPromise;

console.log("\n1. 同步情况下，链式调用 `then()`");
new P((resolve) => resolve(1))
  .then((value) => {
    console.log(value);
    return 2;
  })
  .then((value) => {
    console.log(value);
    return new P((resolve, reject) => reject(-1));
  })
  .then(undefined, (reason) => {
    console.log(reason);
    return 3;
  })
  .then((value) => {
    console.log(value);
    throw -2;
  })
  .then(undefined, (reason) => {
    console.log(reason);
  });

new P((resolve) => {
  setTimeout(() => {
    console.log("\n2. 异步情况下，链式调用 `then()`");
    resolve(1);
  }, 200);
})
  .then((value) => {
    console.log(value);
    return 2;
  })
  .then((value) => {
    console.log(value);
    return new P((resolve, reject) => reject(-1));
  })
  .then(undefined, (reason) => {
    console.log(reason);
    return 3;
  })
  .then((value) => {
    console.log(value);
    throw -2;
  })
  .then(undefined, (reason) => {
    console.log(reason);
  });
```

### 结果

![](./手写`Promise()`-chain_then.png)

## 实现 catch

### 代码

```ts
...
class MiniPromise<Type> {
  ...
  // `catch()` 方法中直接调用 `then()` 方法
  // [ It behaves the same as calling Promise.prototype.then(undefined, onRejected)
  // (in fact, calling obj.catch(onRejected) internally calls obj.then(undefined, onRejected)). ]
  // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch)
  catch = (onRejected?: (reason: any) => unknown) => {
    return this.then(undefined, onRejected);
  };
}
```

### 测试用例

```ts
const P = MiniPromise;

console.log("\n1. 测试 `catch()`");
new P((resolve, reject) => reject(-1))
  .catch()
  .catch((reason) => console.log(reason));
```

### 结果

![](./手写`Promise()`-catch.png)

## 实现 Promise.resolve & Promise.reject

### 代码

```ts
class MiniPromise<Type> {
  ...
  // 如果 value 是 promise，那么就返回此 promise；如果不是，那就返回一个 fulfilled promise
  // [If the value is a promise, that promise is returned,
  // otherwise the returned promise will be fulfilled with the value.]
  // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/resolve)
  static resolve = (value: any): MiniPromise<unknown> => {
    return value instanceof MiniPromise
      ? value
      : new MiniPromise((resolve) => resolve(value));
  };

  // 返回一个 rejected promise
  // [The static Promise.reject function returns a Promise that is rejected.]
  // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/reject)
  static reject = (reason: any) => {
    return new MiniPromise((resolve, reject) => reject(reason));
  };
}
```

### 测试用例

```ts
const P = MiniPromise;

console.log("\n1. `Promise.resolve()` 一个非 promise 对象");
P.resolve(1).then((value) => console.log(value));

console.log("\n2. `Promise.resolve()` 一个 promise 对象");
P.resolve(new P<number>((resolve) => resolve(2))).then((value) =>
  console.log(value)
);
P.resolve(new P((resolve, reject) => reject(-1))).catch((reason) =>
  console.log(reason)
);

console.log("\n3. 测试 `Promise.reject()`");
P.reject(-1).catch((reason) => console.log(reason));
P.reject(new P<number>((resolve) => resolve(1))).catch((reason) =>
  console.log(reason)
);
```

### 结果

![](./手写`Promise()`-static-resolve-and-reject.png)

## 实现 finally

### 代码

```ts
class MiniPromise<Type> {
  ...
  // `finally(onFinally)` 和 `then(onFinally, onFinally)` 很相似
  // [The finally() method is very similar to calling .then(onFinally, onFinally) however there are a couple of differences:]
  // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally#description)
  // `onFinally` 不接收参数
  // [A finally callback will not receive any argument]
  // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally#description)
  finally = (onFinally: () => unknown) => {
    // `finally()` 将返回调用的 Promise
    // [Promise.resolve(2).finally(() => {}) will be resolved with 2. Promise.reject(3).finally(() => {}) will be rejected with 3.]
    // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally#description)
    // 只有 `finally()` 中抛出异常或返回 rejected Promise，才会不返回调用的 Promise
    // [A throw (or returning a rejected promise) in the finally callback will reject the new promise with the rejection reason specified when calling throw.]
    // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally#description)
    const callback = () => {
      return MiniPromise.resolve(onFinally()).then(
        () => this,
        (reason: any) => {
          throw reason;
        }
      );
    };
    return this.then(callback, callback);
  };
}
```

### 测试用例

```ts
const P = MiniPromise;

console.log("\n1. 测试 fulfilled 状态下的 `finally()`");
P.resolve(1)
  .then((value) => {
    console.log(value);
    return P.resolve(2);
  })
  .finally(() => console.log("我是 finally"))
  .then((value) => console.log(value));

console.log("\n2. 测试 rejected 状态下的 `finally()`");
P.reject(-1)
  .catch((reason) => {
    console.log(reason);
    return P.reject(-2);
  })
  .finally(() => console.log("我是 finally"))
  .catch((reason) => console.log(reason));

console.log("\n3. `finally()` 中抛出异常");
P.reject(-1)
  .catch((reason) => {
    console.log(reason);
    return P.reject(-2);
  })
  .finally(() => {
    console.log("我是 finally");
    throw -3;
  })
  .catch((reason) => console.log(reason));

console.log("\n4. `finally()` 中返回 rejected Promise");
P.reject(-1)
  .catch((reason) => {
    console.log(reason);
    return P.reject(-2);
  })
  .finally(() => {
    console.log("我是 finally");
    return P.reject(-4);
  })
  .catch((reason) => console.log(reason));
```

### 结果

![](./手写`Promise()`-finally.png)

## 最终代码

```ts
// Promise 有三种状态：pending、fulfilled、rejected
// [A promise must be in one of three states: pending, fulfilled, or rejected.]
// (https://promisesaplus.com)
enum Status {
  pending = "pending",
  rejected = "rejected",
  fulfilled = "fulfilled",
}

export class MiniPromise<Type> {
  status: Status = Status.pending;
  // 因为状态不可能共存，所以只需要一个变量存储结果
  result: any = null;
  fulfilledCallbacks: (() => void)[] = [];
  rejectedCallbacks: (() => void)[] = [];

  constructor(
    // `executor()` 接收两个函数参数，这两个函数参数又分别接收一个任意类型的参数。
    // [resolutionFunc and rejectionFunc are also functions, and you can give them whatever actual names you want.
    // Their signatures are simple: they accept a single parameter of any type.]
    // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise#parameters)
    executor: (
      resolve: (value: Type) => void,
      reject: (reason: any) => void
    ) => void
  ) {
    const resolve = (value: Type) => {
      // 只有 pending 状态下才能改变状态
      // [When fulfilled, a promise: must not transition to any other state.]
      // (https://promisesaplus.com)
      if (this.status === Status.pending) {
        this.status = Status.fulfilled;
        this.result = value;
        this.fulfilledCallbacks.forEach((func) => func());
      }
    };

    const reject = (reason: any) => {
      // 只有 pending 状态下才能改变状态
      // [When rejected, a promise: must not transition to any other state.]
      // (https://promisesaplus.com)
      if (this.status === Status.pending) {
        this.status = Status.rejected;
        this.result = reason;
        this.rejectedCallbacks.forEach((func) => func());
      }
    };

    try {
      // `executor()` 会在构造函数中执行
      // [A function to be executed by the constructor,
      // during the process of constructing the new Promise object.]
      // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise#parameters)
      executor(resolve, reject);
    } catch (error) {
      // `executor()` 中如果有异常，那状态为 `rejected`。
      // [If an error is thrown in the executor, the promise is rejected.]
      // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise#parameters)
      reject(error);
    }
  }

  then = (
    // `then()` 接收两个函数参数
    // [A promise’s then method accepts two arguments:]
    // (https://promisesaplus.com/#the-then-method)
    // 两个函数参数都是可选参数
    // [Both onFulfilled and onRejected are optional arguments:]
    // (https://promisesaplus.com/#point-23)
    // `onFulfilled()` 的默认值是返回参数
    // [If it is not a function, it is internally replaced with an
    // "Identity" function (it returns the received argument).]
    // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then#parameters)
    // `onRejected()` 的默认值是抛出异常
    // [If it is not a function, it is internally replaced with a "Thrower"
    // function (it throws an error it received as argument).]
    // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then#parameters)
    onFulfilled: (value: Type) => unknown = (value: Type) => value,
    onRejected: (reason: any) => unknown = (reason: any) => {
      throw reason;
    }
  ) => {
    // `then()` 方法要返回一个 promise
    // [then must return a promise]
    // (https://promisesaplus.com/#point-40)
    return new MiniPromise((resolve, reject) => {
      const hanldePromiseOrPlain = (isFulfilled: boolean) => {
        // 注意 `executor()` 中的 try-catch 只能捕获同步代码，这里的代码可能在回调函数中调用
        // [onFulfilled or onRejected must not be called until the execution context stack contains only platform code.]
        // (https://promisesaplus.com/#point-34)
        try {
          const obj = isFulfilled
            ? // 如果状态是 fulfilled，调用 `onFulfilled()`。
              // [If onFulfilled is a function: it must be called after promise is rejected,
              // with promise’s reason as its first argument.]
              // (https://promisesaplus.com/#point-26)
              onFulfilled(this.result)
            : // 如果状态是 rejected，调用 `onRejected()`。
              // [If onRejected is a function: it must be called after promise is rejected,
              // with promise’s reason as its first argument.]
              // (https://promisesaplus.com/#point-30)
              onRejected(this.result);
          // 这里简化了判断，直接用 instanceof，实际情况应该使用 ducking type
          // `obj` 不是 promise，就返回一个 fulfilled promise，值为 x
          // [If x is not an object or function, fulfill promise with x.]
          // (https://promisesaplus.com/#point-64)
          // `obj` 如果是 promise，那么新 promise 的状态和当前的 promise 状态一样
          // 写全可能看得更清晰：`obj.then((value) => resolve(value), (reason) => reject(reason))`，
          // 这个操作就等价于复制了一份 promise，如果你成功了，那你通过 `resolve()` 将成功状态和结果传给我，
          // 如果你失败了，那你通过 `reject()` 将失败状态和结果传给我。
          // [If x is a promise, adopt its state: If/when x is fulfilled, fulfill promise with the same value.
          // If/when x is rejected, reject promise with the same reason.]
          // (https://promisesaplus.com/#point-49)
          obj instanceof MiniPromise ? obj.then(resolve, reject) : resolve(obj);
        } catch (error) {
          reject(error);
        }
      };
      if (this.status === Status.fulfilled) {
        hanldePromiseOrPlain(true);
      } else if (this.status === Status.rejected) {
        hanldePromiseOrPlain(false);
      } else {
        // this.status === Status.pending
        // 发布 - 订阅
        // `then()` 可以被调用多次，因此需要
        // [then may be called multiple times on the same promise.]
        // (https://promisesaplus.com/#point-36)
        this.fulfilledCallbacks.push(() => hanldePromiseOrPlain(true));
        this.rejectedCallbacks.push(() => hanldePromiseOrPlain(false));
      }
    });
  };

  // `catch()` 方法中直接调用 `then()` 方法
  // [ It behaves the same as calling Promise.prototype.then(undefined, onRejected)
  // (in fact, calling obj.catch(onRejected) internally calls obj.then(undefined, onRejected)). ]
  // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch)
  catch = (onRejected?: (reason: any) => unknown) => {
    return this.then(undefined, onRejected);
  };

  // `finally(onFinally)` 和 `then(onFinally, onFinally)` 很相似
  // [The finally() method is very similar to calling .then(onFinally, onFinally) however there are a couple of differences:]
  // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally#description)
  // `onFinally` 不接收参数
  // [A finally callback will not receive any argument]
  // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally#description)
  finally = (onFinally: () => unknown) => {
    // `finally()` 将返回调用的 Promise
    // [Promise.resolve(2).finally(() => {}) will be resolved with 2. Promise.reject(3).finally(() => {}) will be rejected with 3.]
    // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally#description)
    // 只有 `finally()` 中抛出异常或返回 rejected Promise，才会不返回调用的 Promise
    // [A throw (or returning a rejected promise) in the finally callback will reject the new promise with the rejection reason specified when calling throw.]
    // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally#description)
    const callback = () => {
      return MiniPromise.resolve(onFinally()).then(
        () => this,
        (reason: any) => {
          throw reason;
        }
      );
    };
    return this.then(callback, callback);
  };

  // 如果 value 是 promise，那么就返回此 promise；如果不是，那就返回一个 fulfilled promise
  // [If the value is a promise, that promise is returned,
  // otherwise the returned promise will be fulfilled with the value.]
  // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/resolve)
  static resolve = (value: any): MiniPromise<unknown> => {
    return value instanceof MiniPromise
      ? value
      : new MiniPromise((resolve) => resolve(value));
  };

  // 返回一个 rejected promise
  // [The static Promise.reject function returns a Promise that is rejected.]
  // (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/reject)
  static reject = (reason: any) => {
    return new MiniPromise((resolve, reject) => reject(reason));
  };
}
```
