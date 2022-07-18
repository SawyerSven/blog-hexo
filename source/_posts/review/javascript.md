---
title: 2022春季复习之Javascript
date: 2022-03-14 14:57:12
tags: 
  - 复习
  - javascript
  - 手写代码
categories:
  - javascript
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

```js

const clone = (target, map = new WeakMap()) => {
  if (typeof target === 'object') {
    let cloneTarget = Array.isArray(target) ? [] : {}
    if (map.get(target)) {
      return map.get(target)
    }
    map.set(target, cloneTarget)
    for (const key in target) {
      cloneTarget[key] = clone(target[key], map)
    }
    return cloneTarget
  } else {
    return target
  }
}


const data = {
  field1: 1,
  field2: undefined,
  field3: 'ConardLi',
  field4: {
    child: 'child',
    child2: {
      child2: 'child2'
    }
  },
  field5: [1, 2],
  field6: [{
    name:'Siyuan'
  },{
    name:'Mingyang'
  }]
}

data.reference = data

```

使用上边的用例和代码执行clone操作，打印出clone后的结果

![](http://47.95.214.156:3001/uploads/big/9062fbaad26f319e98bf9dd11c72b0a5.png)


其中reference的值是一个Circular,指代循环引用

使用WeakMap代替Map起到画龙点睛的作用，即：

```js

function clone(target,map = new WeakMap()){

}

```

##### WeakMap和Map的区别

先看MDN的描述:

> Map对象保存键值对，并且能够记住键的原始插入顺序。任何值(对象或原始值)都可以作为一个键或一个值

> WeakMap对象是一组键值对的集合，其中的键是`弱引用`的。其键`必须是对象`,而值可以是任意的


弱引用：

> 在计算机程序设计中,弱引用与强引用相对，是指不能确保其引用对象不会被垃圾回收器回收的引用。一个对象若只被弱引用所引用，则认为是不可访问(或弱可访问)的，
> 并因此可能在任何时候被回收。

我们默认创建一个对象:`const obj = {}`就默认创建了一个强引用对象，我们只有手动将`obj = null`,才会被垃圾回收机制进行回收，如果是弱引用对象，
垃圾回收机制会自动帮我们回收。

使用`Map`时，对象间存在强引用关系，即使手动释放obj,target依然对`obj`存在强引用关系,所以这部分内存依然无法被释放。

如果使用`WeakMap`，`target`和`obj`存在的就是弱引用关系,当下一次垃圾回收机制执行时,这块内存就会被释放掉。

如果我们要拷贝的对象非常庞大时，使用`Map`会对内存造成非常大的额外消耗，而我们需要手动清除`Map`的属性才能释放这块内存，而
`WeakMap`会帮我们巧妙化解这个问题。

#### 性能优化

使用while代替for..in以获得更好的性能表现。 详情见参考3


#### 其他数据类型

在目前的实现中，只考虑了普通`object`和`array`两种数据类型,实际上所有的引用类型还有很多。

可以使用`toString`来获取准确的引用类型的:

> 每一个引用类型都有toString方法，默认情况下,`toString`被每个`Object`对象继承。如果此方法在自定义对象中未被覆盖，`toString`返回
> `"[object type]"`，其中type是对象的类型。

大部分引用类型比如`Array、Date、RegExp`等都重写了`toString`方法

可以直接调用`Object`原型上未被覆盖的`toString()`方法，使用`call`来改变`this`指向来达到我们想要的效果

```js

Object.prototype.toString.call(target)

```

抽离一些常用的数据类型以便后续使用:

```js
const mapTag = '[object Map]';
const setTag = '[object Set]';
const arrayTag = '[object Array]';
const objectTag = '[object Object]';

const boolTag = '[object Boolean]';
const dateTag = '[object Date]';
const errorTag = '[object Error]';
const numberTag = '[object Number]';
const regexpTag = '[object RegExp]';
const stringTag = '[object String]';
const symbolTag = '[object Symbol]';

```

在上面的类型中，简单分为两类：

- 可以继续遍历的类型
- 不可以继续遍历的类型

分别为他们做不同的拷贝。

#### 可持续遍历的类型

上文的`object、array`都属于可以继续遍历的类型，因为它们内存都还可以存储其他数据类型的数据，另外还有`Map,Set`等都是可以继续遍历的类型。这里只考虑这四种。

这几种类型还需要继续进行递归，首先要获取他们的初始化数据，例如`[]`和`{}`,我们可以通过拿到`constructor`的方式来通用的获取

例如:`const target = {}`就是`const target = new Object()`的语法糖。并且这种方法因为使用了原对象的构造方法，所以它可以保留对象原型上的数据。


```js

function getInit(target){
  const Ctor = target.constructor;
  return new Ctor()
}

```

改写`clone`函数:

```js


function forEach(array, iteratee) {
  let index = -1
  const length = array.length
  while (++index < length) {
    iteratee(array[index], index)
  }
  return array
}

function getType(target) {
  return Object.prototype.toString.call(target)
}

function isObject(target) {
  const type = typeof target
  return target !== null && (type === 'object' || type === 'function')
}

function getInit(target) {
  const Ctor = target.constructor
  return new Ctor()
}

const clone = (target, map = new WeakMap()) => {
  if (!isObject(target)) {
    return target
  }

  const type = getType(target)
  let cloneTarget

  if (deepTag.includes(type)) {
    cloneTarget = getInit(target, type)
  }
  // 防止循环引用
  if (map.get(target)) {
    return map.get(target)
  }
  map.set(target, cloneTarget)

  //克隆set
  if (type === deepTagMap.setTag) {
    target.forEach((value) => {
      cloneTarget.add(clone(value, map))
    })
    return cloneTarget
  }

  // 克隆map
  if (type === deepTagMap.mapTag) {
    target.forEach((value, key) => {
      cloneTarget.set(key, clone(value, map))
    })
    return cloneTarget
  }

  // 克隆对象和数组
  const keys = type === deepTagMap.arrayTag ? undefined : Object.keys(target)

  forEach(keys || target, (value, key) => {
    if (keys) {
      key = value
    }
    cloneTarget[key] = clone(target[key], map)
  })
  return cloneTarget
}

```

#### 不可继续遍历类型

其他剩余类型统一归类成不可处理的数据类型，依次进行处理：

`Bool`,`Number`,`String`,`Date`,`Error`这几种类型都可以直接用构造函数和原始数据创建一个新对象

```js
function cloneOtherType(target, type) {
    const Ctor = target.constructor;
    switch (type) {
        case boolTag:
        case numberTag:
        case stringTag:
        case errorTag:
        case dateTag:
            return new Ctor(target);
        case regexpTag:
            return cloneReg(target);
        case symbolTag:
            return cloneSymbol(target);
        default:
            return null;
    }
}

```

克隆`symbol`类型:

```js

function cloneSymbol(target){
  return Object(Symbol.prototype.valueOf.call(target))
}


```

克隆`正则`:

```js

function cloneReg(target){
  const reFlags = /\w*$/;
  const result = new target.constructor(target.source,reFlags.exec(target));
  result.lastIndex = target.lastIndex;
  return result
}

```

实际上还有很多数据类型没有写到.可以继续探索实现。

#### 克隆函数

这里不考虑克隆函数的实现。


### 参考

- 1 [WeakMap - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)
- 2 [Map - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Map)
- 3 [如何写出一个惊艳面试官的深拷贝?](https://juejin.cn/post/6844903929705136141#comment)

