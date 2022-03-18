---
title: 2022春季复习之Javascript
date: 2022-03-14 14:57:12
tags:
---

# 手写常用工具函数

## 防抖(debounce)和节流(throttle)

**目的**：为了解决DOM事件如onresize,mousemove或input的change等事件频繁触发导致的事件在短时间内频繁调用增加的浏览器负担及性能影响。

**区别**：
  - 防抖： 单位时间内多次触发事件，最后一次执行事件处理函数
  - 节流： 固定间隔执行时间处理函数

### 防抖函数(debounce)

**思路**: 第一次触发事件的时候，设置定时器。如果在定时器结束前重复触发事件，则重置定时器。如果定时器结束前没有新的事件触发，则定时器结束后执行事件处理函数

#### 实现

```javascript

const debounce = (fn, delay) => {
  let timer = null
  return function (...args) {
    if (timer) clearTimeout(timer)
    timer = setTimeout(() => {
      fn.apply(this, ...args)
    }, delay)
  }
}

```

### 节流函数(throttle)

**时间戳实现思路**: 事件触发时候，判断上次执行的时间戳lastInvoke，如果存在则判断上次执行距现在是否大于指定的delay，是则再次执行事件处理函数，并且将当前时间戳设定为lastInvoke。
反之则不作操作。如果lastInvoke不存在，则设定当前时间戳为lastInvoke,并且执行时间处理函数


#### 时间戳实现:

```js
const throttle = (fn, delay) => {
  let lastInvoke = Date.now()
  function throttled(...args) {
    let time = Date.now()
    if (lastInvoke + delay < time) {
      fn.apply(this, args)
      lastInvoke = time
    }
  }
  return throttled
}
```

缺点: 最后一次触发回调与前一次的触发回调时间差小于delay,则最后一次触发事件不会执行


#### 定时器实现

```js
const throttle = (fn, delay) => {
  let flag = false
  function throttled(...args) {
    if (flag) return
    flag = true
    setTimeout(() => {
      flag = false
      fn.apply(this,args)
    }, delay)
  }
  return throttled
}
```
缺点： 第一次触发的事件不会立即执行，也会等待delay时间后才会执行

### rAF实现

```js 

let throttle = (fn, delay) => {
  let flag
  return function (...args) {
    if (!flag) {
      let that = this
      requestAnimationFrame(function () {
        fn.apply(that, args)
        flag = false
      })
    }
    flag = true
  }
}


```



## 深浅拷贝

### 基本类型 & 引用类型

**区别**:

  - *基本类型*在内存中占据固定大小，保存在内存栈当中
  - *引用类型*的值是对象，保存在堆内存，而栈内存存储的是*对象的标识符*以及*对象在堆内存中的十六进制存储地址*

**复制方式**:
  - *基本类型*：从一个变量向另一个新变量复制基本类型的值，会创建一个这个值的副本，并将该副本复制给新变量
  - *引用类型的*：从一个变量向另一个新变量复制引用类型的的值，其实复制的是指针，最终两个变量都指向同一个对象


### 深浅拷贝定义

![浅拷贝](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/9/1/16ce894a1f1b5c32~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

> 创建一个新对象，这个对象有着原始对象属性值的一份精确拷贝。如果属性是基本类型，拷贝的就是基本类型的值，如果属性是引用类型的，拷贝的就是内存地址，所以如果其中一个对象改变了这个地址，就会影响到另一个对象。

![深拷贝](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/9/1/16ce893a54f6c13d~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

> 将一个对象从内存中完整的拷贝一份出来，从堆内存中开辟一个新的区域存放新对象，且修改新对象不会影响原始对象。


### 深拷贝的几种实现方式

#### JSON大法

```js

JSON.parse(JSON.stringify());

```

优点: 写法简单，适用于大部分应用场景
缺点：部分场景不适用，如循环引用、拷贝其他引用类型、面试只写这一种卷王肯定不会让你过


#### 基础版

```js

const clone = (target) => {
  if (typeof target === 'object') {
    let cloneTarget = {}
    for (const key in target) {
      cloneTarget[key] = clone(target[key])
    }
    return cloneTarget
  } else {
    return target
  }
}

```

做深拷贝时，我们不知道要拷贝的对象有多少层深度，所以用递归来解决问题

整体逻辑：

  - 如果target是原始类型,无需继续拷贝,直接返回
  - 如果是引用类型，创建一个新的对象，遍历需要克隆的对象，将要克隆的对象的属性递归执行clone后依次添加到新对象上


#### 处理数组

上边的代码并没有考虑到数组应该如何处理，只处理了普通的Object，所以需要对数组做兼容处理:

```js
const clone = (target) => {
  if (typeof target === 'object') {
    // 这一行判断target是否是数组并进行不同的初始化
    let cloneTarget = Array.isArray(target) ? [] : {}
    for (const key in target) {
      cloneTarget[key] = clone(target[key])
    } 
    return cloneTarget
  } else {
    return target
  }
}

```

#### 循环引用

上边的代码如果遇倒循环引用则会导致栈内存溢出

如: 

```js
target.target = target
```
> 会返回*Maximum call stack size exceeded*


因为递归进入死循环导致栈内存溢出。

原因就是上面的对象存在循环引用的情况，即对象的属性间接或直接引用了自身的情况

解决循环引用问题，我们可以额外开辟一个存储空间，来存储当前对象和拷贝对象的对应关系，当需要拷贝当前对象时，先去存储空间找，有没有拷贝过这个对象，
如果有的话直接返回，没有的话继续拷贝，这样就巧妙化解循环引用的问题

这个存储空间需要可以存储`key-value` 形式的数据，而且key可以是一个引用类型，我们可以选择`Map`数据结构:

- 检查`map`中有无克隆过的对象
  - 有 - 直接返回
  - 没有 - 将当前对象作为`key`,克隆对象作为`value`进行存储
- 继续克隆

### 参考

[如何写出一个惊艳面试官的深拷贝?](https://juejin.cn/post/6844903929705136141#comment)

