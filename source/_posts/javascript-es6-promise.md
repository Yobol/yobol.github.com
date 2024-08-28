---
title: JavaScript ES6 Promise 特性
date: 2024-08-12 21:08:06
tags:
- JavaScript
- ES6
- Promise
- Asynchronous Programming
---
## 基本介绍

在 `Promise` 出现之前，处理异步请求通常使用 `Ajax` 回调函数中实现。显然，这样存在一个棘手问题：当业务逻辑比较复杂时，如果在一个操作中需要执行多个异步请求，而每个请求又依赖上个请求的结果，就会导致代码嵌套层数过多，即“回调地狱”的问题。当出现这种情况时，就意味着我们的代码可读性和可维护性都会变差。

那么是否存在一种解决方案能够简化异步编程模式，降低异步编程难度，提高编码效率和质量呢？答案就是 `Promise`。

---

`Promise` 是一种在 `JavaScript` 的 `ES6` 提案中引入的一种异步编程解决方案。

## 核心特性

## 基本原理

### 生命周期

每个 `Promise` 对象都有三种状态：

- `pending`（进行中）；
- `fulfilled`（已成功）；
- `rejected`（已失败）。

`Promise` 在创建时处于 `pending` 状态，而状态转移只有两种形式：

- 执行成功：从 `pending` 状态转移到 `fulfilled` 状态；
- 执行失败：从 `pending` 状态转移到 `rejected` 状态。

并且状态一旦改变，就不能再改变。

## 基本用法

### 原生用法

#### `Promise`

```javascript
const promise = new Promise((resolve, reject) => {
  // 异步请求处理逻辑
  if (/* 请求处理成功标识 */) {
    resolve(); // 将状态从 pending 转移到 fulfilled
  } else {
    reject(); // 将状态从 pending 转移到 rejected
  }
});
```

需要注意，因为 `Promise` 对象就是一个构造函数，所以一旦创建，就会立即调用，然后等待 `resolve` 或者 `reject` 函数来确定最终状态。这两个函数可以传递参数，作为后续 `then` 或者 `catch` 函数执行时的数据源。

```javascript
const promise = new Promise(function(resolve, reject) {
  console.log('Promise');
  resolve();
});
promise.then(function() {
  console.log('resolved');
});
console.log('Hello');
```

上述代码会先后输出 `Promise`、`Hello` 和 `resolved`。

为什么是上述执行顺序呢？因为，异步函数在提交后，会进入一个等待队列中排队，当执行结束后才会通过 `then` 触发回调函数输出 `resloved`。

#### `then()` 函数

`Promise` 在原型属性上增加了一个 `then()` 函数，表示在 `Promise` 实例状态发生改变时执行的回调函数。接收两个函数作为参数：

- 第一个参数必选，表示 `Promise` 在执行成功（即调用 `resolve`）后，所要执行的回调函数；
- 第二个参数可选，表示 `Promise` 在执行失败（即调用 `reject`）后，所要执行的回调函数。

`then` 函数返回的是一个**新的** `Promise` 实例，上一轮 `then()` 函数内部的返回值会作为下一轮 `then()` 函数的参数值，因此可以使用链式调用处理业务逻辑：

```javascript
const promise = new Promise((resolve, reject) => {
  resolve(1);
});

promise.then((result) => {
  console.log(result); // 1
  return 2;
}).then((result) => {
  console.log(result); // 2
  return 3;
}).then((result) => {
  console.log(result); // 3
});
```

这样是不是就可以解决“回调地狱”的问题啦？代码确实优雅很多。

需要注意的是，在 `then()` 函数中不能返回 `Promise` 实例本身，否则会出现 `Promise` 循环引用的问题，抛出异常：

```javascript
const promise = Promise.resolve().then(() => {
  return promise; // TypeError: Chaining cycle detected for promise #<Promise>
});
```

由上可知，`then()` 函数能够处理 `rejected` 状态的回调函数，但是并不推荐这么做，而是应该使用 `catch()` 函数来处理。

#### `catch()` 函数



### 封装用法



### 用法示例

#### 基于 `Ajax` 封装 `Get` 方法

在引入 `Promise` 解决方案后，实现原生 `Get` 方法的实现如下：

```javascript
function ajaxGetPromise(url) {
  const promise = new Promise(function(resolve, reject) {
    const handler = function() {
      if (this.readyState != 4) {
        return;
      }

      if (this.status != 200) {
        resolve(this.response);
      } else {
        reject(new Error(this.statusText));
      }
    };

    // 原生 Ajax 操作
    const client = new XMLHttpRequest();
    client.open('GET', url);
    client.onreadystatechange = handler;
    client.responseType = 'json';
    client.setRequestHeader('Accept', 'application/json');
    client.send();
  });

  return promise;
}
```

调用如下：

```javascript
ajaxGetPromise('/products').then((response) => {
  console.log(response);
});
```

## 操作进阶

## 总结

## 参考

1. [Promise - JavaScript - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
