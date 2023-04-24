---
title: 如何实现 call、apply、bind？
---

## 介绍

> `call`：改变函数作用域中的 `this` 指向且立即执行。

```
参数一：`this` 指向的对象
参数二等：序列。调用函数需要传递的参数列表。 

示例：`fn.call(obj, arg1, arg2, ...args)`
```

> `apply`：改变函数作用域中的 `this` 指向且立即执行。

```
参数一：`this` 指向的对象
参数二：数组。调用函数需要传递的参数。 

示例：`fn.apply(obj, [arg1, arg2, ...args])`
```

> `bind`：改变函数作用域中的 `this` 指向，不立即执行，但是会返回一个新的函数

```
参数一：`this` 指向的对象
参数二：序列。给函数传递的参数序列。
        如果这个序列的长度小于命名参数的长度，则调用返回的新函数可以补充剩余的未传递的参数
        如果这个序列的长度等于命名参数的长度，则调用返回的新函数传递的参数无效
        如果这个序列的长度大于命名参数的长度，则只截取需要的个数，多余的被忽略
    
示例：`const newFn = fn.bind(obj, arg1, arg2);`
     `newFn(arg3, arg4);`
```

## 手动实现

> **call**

```js
Function.prototype.myCall = function (context) {
  /** 
   * 防止原型对象上的该方法被直接调用
   * const myCall = Function.prototype.myCall;
   * myCall();
   * 这是 this === window，下面执行 context[fn]() 的时候会报错
  */
  if (typeof this !== 'function') {
    return undefined;
  }
  // 如果 context 为 null 或者 undefined，则默认为 window
  if (context === null || context === undefined) {
    context = window;
  } else {
    // 否则就包装成 Object 类型（这里主要是针对原始值做处理）
    context = Object(context);
  } 
  // 这个 context 对象将会作为调用 myCall 方法的函数体内部的 this 指向

  // 获取除了第一个以外的所有参数
  const args = [...arguments].slice(1);
  // 定义一个 Symbol 类型的变量
  const fn = Symbol();
  // 让 Symbol 类型的变量作为 context 的 key，值为被执行的函数体 
  // (这里的 this 指向的是调用 myCall 方法的函数体，context 代表的是调用该方法的函数体内部的 this)
  context[fn] = this;
  /** 
   * context: {
   *   props: ...,
   *   Symbol(): this
   * }
  */
  // 执行对象中的方法，因此方法内部的 this 就会指向该对象
  const result = context[fn](...args);
  // 得到结果就在对象中删除该方法，防止污染 context 对象
  delete context[fn];
  // 返回结果
  return result;
}
```

> apply

`apply` 和 `call` 的实现方式很相似。唯一不同的是，前者接收传递给函数的参数是 ==数组类型==，后者接收传递给函数的参数是 ==序列==。

```js
Function.prototype.myApply = function (context) {
  if (typeof this !== 'function') {
    return undefined;
  }
  if (context === null || context === undefined) {
    context = window;
  } else {
    context = Object(context);
  }

  const fn = Symbol();
  context[fn] = this;
  let result;
  // call 和 apply 的区别是除了第一个参数以外的其他参数的传递方式
  if (arguments[1] instanceof Array) {
    result = context[fn](...arguments[1]); // 如果第二个参数是数组，则作为参数传递给函数
  } else {
    result = context[fn](); // 不是数组 什么也不传
  }
  delete context[fn];
  return result;
}
```

> bind

```js
Function.prototype.myBind = function (context) {
  /** 
   * 这里判断 this 是否为 function 与 myCall 和 myApply 不同的是，
   * myCall 和 myApply 是立即执行的，当调用错误时返回 undefined 也没问题
   * 而 myBind 返回的是一个新的函数，因此直接给报错更合理
  */
  if (typeof this !== 'function') {
    throw new TypeError('error');
  }

  // 缓存调用 bind 方法的函数体，以及调用 bind 方法时传递的参数
  const _this = this;
  const args = [...arguments].slice(1);

  // bind 方法不是立即执行的，而是会返回一个新的函数
  return function F() {
    // 如果返回的新的函数是通过 new 调用的，则返回调用 bind 方法的函数体的实例对象
    if (this instanceof F) {
      // 这里 args 是调用 bind 方法传递给函数体的参数，arguments 是调用返回的新的函数时传递的参数
      return new _this(...args, ...arguments);
    } else {
      // 正常调用返回的新的函数，已经缓存了调用方法的函数体，和函数体内部的 this 指向，因此相当于调用 myApply，或者 myCall 方法
      return _this.myApply(context, [...args, ...arguments]);
    }
  }
}
```

## 无注释版本

```js
// call
Function.prototype.myCall = function (context) {
  if (typeof this !== 'function') {
    return undefined;
  }
  if (context === null || context === undefined) {
    context = window;
  } else {
    context = Object(context);
  }
  const args = [...arguments].slice(1);
  const fn = Symbol();
  context[fn] = this;
  const result = context[fn](...args);
  delete context[fn];
  return result;
}

// apply
Function.prototype.myApply = function (context) {
  if (typeof this !== 'function') {
    return undefined;
  }
  if (context === null || context === undefined) {
    context = window;
  } else {
    context = Object(context);
  }

  const fn = Symbol();
  context[fn] = this;
  let result;
  if (arguments[1] instanceof Array) {
    result = context[fn](...arguments[1]);
  } else {
    result = context[fn]();
  }
  delete context[fn];
  return result;
}

// bind
Function.prototype.myBind = function (context) {
  if (typeof this !== 'function') {
    throw new TypeError('error');
  }
  const _this = this;
  const args = [...arguments].slice(1);
  return function F() {
    if (this instanceof F) {
      return new _this(...args, ...arguments);
    } else {
      return _this.myApply(context, [...args, ...arguments]);
    }
  }
}
```