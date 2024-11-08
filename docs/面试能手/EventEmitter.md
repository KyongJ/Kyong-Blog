---
title: 发布订阅者模式——EventMitter
description: 发布-订阅模式其实是一种对象间一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都将得到状态改变的通知。
author: Kyong
date: 2023-02-13
tag: 
  - 面试能手
---
# 发布订阅者模式——EventMitter

## 一、前言

**发布-订阅模式**其实是一种对象间一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都将得到状态改变的通知。

- **订阅者**（Subscriber）把自己想订阅的事件 **注册**（Subscribe）到调度中心（Event Channel）;
- 当**发布者**（Publisher）**发布该事件**（Publish Event）到调度中心，也就是该事件触发时，由 **调度中心** 统一调度（Fire Event）订阅者注册到调度中心的处理代码。

举个简单的例子

上面一个看似简单的操作，其实是一个典型的发布订阅模式，`公众号`属于发布者，`用户`属于订阅者；用户将订阅公众号的事件注册到调度中心，公众号作为发布者，当有新文章发布时，公众号发布该事件到调度中心，调度中心会及时发消息告知用户。

## 二、手写订阅发布者模式

> 题目

```js
class EventEmitter {
    constructor() {
    }
    on() {
    }
    once() {        
    }
    emit() {
    }
    off() {
    }
  }
  
  // 运行示例
let ev = new EventEmitter();

const say = (v) => {
  console.log(v);
}

ev.on('say', say);
ev.emit('say', 'Kyonglok');
ev.off('say', say);
ev.once('say', say)
```

> 分析

我们要对`EventEmitter类`中的四个方法有个了解

- **on()**:  类似于订阅微信公众号的订阅方法，但不会触发事件
- **emit()**:  类似于订阅公众号之后，公众号一有消息就会通知给我们
- **once()**:  代表我订阅公众号之后，它只通知我一次
- **off()**:  就是我们常见的取消关注该公众号了

### 实现on方法

`on` 方法用来存储该事件类型的回调函数，建立一个数组在存储回调函数

```js
on(type, fn) {
    this.event[type] = this.event[type] || [];
    this.event[type].push(fn);
}
```

### 实现emit方法

`emit` 方法用来执行订阅事件的回调函数

```js
emit(type, ...args) {
    this.event[type] = this.event[type] || [];
    this.event[type].forEach(fn => fn(...args));
}
```

### 实现off方法

`off` 方法用来删除事件队列里的回调函数

```js
off(type, fn) {
    this.event[type] = this.event[type] || [];
    this.event[type] = this.event[type].filter(item => item !== fn);
}
```

### 实现once方法

```js
once(type, fn) {
    let newFn = (...args)=>{
    	fn(...args);
    	this.off(type, newFn);
    };
    this.on(type, newFn);
}
```



### 运行结果

> 使用on方法订阅

```js
// 运行示例
let ev = new EventEmitter();

const say = v => {
    console.log(v);
};
 
ev.on('say', say);
ev.emit('say', 'Kyonglok');  //Kyonglok
ev.emit('say', 'Kyonglok');  //Kyonglok
```

> 使用once方法订阅

```js
// 运行示例
let ev = new EventEmitter();

const say = v => {
    console.log(v);
};
 
ev.once('say', say);
ev.emit('say', 'Kyonglok');  //Kyonglok
ev.emit('say', 'Kyonglok');  //undefined
```

## 三、完整代码

```js
class EventEmitter {
    constructor() {
        this.event = {};
    }
    on(type, fn) {
        this.event[type] = this.event[type] || [];
        this.event[type].push(fn);
    }
    emit(type, ...args) {
        this.event[type] = this.event[type] || [];
        this.event[type].forEach(fn => fn(...args));
    }
    off(type, fn) {
        this.event[type] = this.event[type] || [];
        this.event[type] = this.event[type].filter(item => item !== fn);
    }
    once(type, fn) {
        let newFn = (...args)=>{
            fn(...args);
            this.off(type, newFn);
        };
        this.on(type, newFn);
    }
}

// 运行示例
let ev = new EventEmitter();

const say = v => {
    console.log(v);
};
 
// ev.on('say', say);
ev.on('say', say);
ev.emit('say', 'Kyonglok');
ev.emit('say', 'Kyonglok');
// ev.off('say', say);

```

