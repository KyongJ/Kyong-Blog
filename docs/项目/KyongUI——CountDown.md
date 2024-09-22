# KyongUI——CountDown倒计时组件

## 一、前言

在YY直播实习的时候，负责的是YY520交友活动页面的搭建，其中有一个需求就是页面会存在一个倒计时组件，在活动结束的前一天在完成某些任务后，可以点击抽奖。由于 `UI` 库使用的是 `antd`，所以我第一反应是使用 `antd` 的 `CountDown` 组件。

## 二、antd的问题

在使用现成的组件后，测试反映倒计时不准确，我的代码中并没有依赖任何客户端时间，问题肯定是出现在 `antd` 的 `CountDown` 组件上。于是我就去看了一下 `antd` 的 `CountDown` 组件的源码

```jsx
 // 30帧
 const REFRESH_INTERVAL= 1000 / 30;

  const stopTimer = () => {
    onFinish?.();
    if (countdown.current) {
      clearInterval(countdown.current);
      countdown.current = null;
    }
  };

  const syncTimer = () => {
    const timestamp = getTime(value);
    if (timestamp >= Date.now()) {
      countdown.current = setInterval(() => {
        forceUpdate();
        onChange?.(timestamp - Date.now());
        if (timestamp < Date.now()) {
          stopTimer();
        }
      }, REFRESH_INTERVAL);
    }
  };

  React.useEffect(() => {
    syncTimer();
    return () => {
      if (countdown.current) {
        clearInterval(countdown.current);
        countdown.current = null;
      }
    };
  }, [value]);
```

核心代码就是这段，本质 `CountDown` 并不是一个倒计时，而是根据客户端时间算出来的一个时间差值，这也能解释为啥这个倒计时相对比较准确。内部通过`timestamp`与`Date.now()`进行比较来进行时间计时，从而对客户端时间产生了依赖。



## 三、计算API的对比

倒计时功能必然需要一个不断执行的 **异步过程**，这可以使用运行时环境提供的 **API**，即 **setInterval、setTimeout、requestAnimationFrame**，那么到底该选择谁更合适呢？

[你不知道的 setTimeout、setInterval、requestAnimationFrame](https://juejin.cn/post/7045113363625410590#heading-6)

**requestAnimationFrame实现简单定时器**

```js
let rafId; // 标识动画的id
 const endTime = Date.now() + 20 * 1000 // 加上20秒  // 结束时间
     function step(timestamp) {
     const diff = endTime - Date.now()
     if(diff > 0) {
     console.log(Math.floor(diff / 1000))
     rafId = window.requestAnimationFrame(step)
 }
window.requestAnimationFrame(step)

```

满足时间间隔执行回调函数

```js
function setIntervalPrecision(callback, delay) {
  // 生成并记录定时器ID
  let obj = window.interValPrecisionObj || (window.interValPrecisionObj = { num: 0 })
  obj.num++
  obj['n' + obj.num] = true
  var intervalId = obj.num
  // 开始时间
  var startTime = +new Date()
  // 已执行次数
  var count = 0
  // 延迟时间
  delay = delay || 0

  (function loop() {
    // 定时器被清除，则终止
    if (!obj['n' + intervalId]) return
    // 满足条件执行回调
    if (+new Date() > startTime + delay * (count + 1)) {
      count++
      callback(count)
    }
    requestAnimationFrame(loop)
  })()

  return intervalId
}
```

但是`requestAnimationFram`e是由浏览器专门为动画提供的API，在运行时浏览器会自动优化方法的调用，并且如果**页面不是激活状态下的话，动画会自动暂停**,这一点要值得注意

## 四、使用web Worker消除定时器误差

​		因为`JavaScript`是单线程的，在事件循环过程中，当前宏观任务队列中的微观任务会阻塞下一个宏观任务队列中任务的执行。所以会造成一种现象，定时器中的真实执行时间并不会精准的按照第2个参数所设定的数值执行。比如设置1000毫秒，如果到了1000毫秒，主线程被其他任务所占用了，那么就会等待其它任务的执行，等其它任务执行完毕后，才会执行定时器的回调函数。

```js
setTimeout(() => {console.log('我是定时器！')},1000);
```

你可以尝试执行如下代码，会发现定时器的执行时间应该超过了1秒钟，如果正常执行，你可以从循环条件后面加个0。

```js
setTimeout(() => {console.log('我是定时器！')}, 1000);
for (let i = 0; i<1000000000; i++) {}
```

​		碰到这种循环或者递归代码时，回调函数的执行时间会根据不同的电脑运算速度决定。如果你的电脑配置够强，比如小型机，高性能服务器等，能够在1秒以内执行完逻辑，那么就不会影响定时器的正常执行。

​		要想做到时间相对准确，就必须解决这个问题，办法有很多种，最常见也最有效的办法，是在当前定时器的回调函数中校验误差并调整下一次定时器的发生时间，达到平均1秒的效果。

掘金上面有一篇介绍这种做法的文章，可供参考： [JavaScript 前端倒计时纠偏实现](https://juejin.cn/post/6844903685458231303)



**但是，如果在浏览器中单独打开一个空白页面，在控制台中运行如下代码，观察每次的输出，发现还是足够准确的，误差都在1毫秒以内。**

这是不是就意味着我们可以直接这么写代码呢？如果页面足够简单，没有其它的监听事件，不会发生频繁的交互操作，这么写仍然会出问题，当页面休眠时，定时器就会停止。如果页面存在很多监听事件或者交互操作，就可能会发生跳秒的现象。特别是在单页面应用中更应该注意，像`react`和`vue`框架中，`diff`算法和`DOM`渲染都在一个主线程中执行。

为了最大程度的避免这个问题，可以采用`web worker`来开启一个后台线程单独运行定时器，但是这样也只是能够保证计时器的运行间隔是精准的，并不能保证UI渲染是精准的。

> web Worker

​		使用`web worker`的唯一方式就是通过`new Worker('../xx.js')`的方式使用。构造参数是独立线程js文件的路径。在`react`框架中，只能引用`public`目录下的文件，才能保证打包后路径是正确的。或者修改webpack配置，但这样做并不是很优雅。

​		虽然使用`web worker`的方式只有一种，但是我们可以在遵循正常使用规则下，用一种更优雅的方式来实现。通过`Blob`对象和`URL.createObjectURL`方法来创建一个虚拟的js文件。

具体实现代码如下:

```ts
/* eslint-disable @typescript-eslint/ban-types */
class WebWorker {
  worker: Worker
    
  constructor(worker: Function) {
    const code = worker.toString()
    const blob = new Blob(['(' + code + ')()'])
    this.worker = new Worker(URL.createObjectURL(blob))
  }
  postMessage(message: any) {
    this.worker.postMessage(message)
  }
  onmessage(callback: (event: MessageEvent) => void) {
    this.worker.onmessage = callback
  }
}

export default WebWorker
```

这个类接受一个构造参数，这个构造参数是一个函数，通过`Blob`创建这个虚拟的js文件。再通过`URL.createObjectURL`方法为`Blob`对象创建一个链接。最终作为`Worker`的构造参数，来创建一个`worker`实例。

使用它也比较简单。

```tsx
  const work = function (this: Worker) {
    let timer: number | null = null
    let time = 0
    this.onmessage = (e: MessageEvent) => {
      const {restTime, state} = e.data
      time = restTime
      if (state === 'stop') {
        if (timer) {
          clearInterval(timer)
          timer = null
        }
        return
      } else if (state === 'start') {
        const interval = 1000
        if (!timer) {
          timer = setInterval(() => {
            time = time - interval
            this.postMessage(time)
          }, interval)
        }
      }
    }
  }
```

> 主线程处理指令

```ts
  const workerHandler = () => {
    worker.onmessage((e: MessageEvent) => {
      if (e.data <= 0) {
        worker.postMessage({state: 'stop'})
        onEnd()
      }
      restTime.current = e.data
      setTime(relativeTime(e.data))
    })

    worker.postMessage({
      state: 'start',
      restTime: Number.parseInt(restTime.current.toString(), 10),
    })
  }
```

**这里仍然是一个无法解决的问题。**由于`DOM`的绘制是在主线程内完成的，`web worker`不能处理`DOM`，虽然可以保证定时器的间隔精准度，但无法保证主线程更新UI的精准度。如果主线程在处理其它事情，`onmessage`不能及时响应，UI仍然会发生卡顿。



与直接在主线程执行的最大的区别在于：`setInterval`在被阻塞一次后，后面的所有执行时间间隔都会被打乱，如果被阻塞N次，时间间隔就会越来越乱。`web worker`的作用就是即使被阻塞N次，也能保证定时器中的函数执行次数是按照预期执行的。

为了避免这种情况可以按照上面提到的那种不断进行时间纠偏、重新创建`setTimeout`的方式来实现。`web worker`的方式是一种新的实现思路，其优势在于无论主线程如何阻塞，定时器的回调函数执行次数和频率是不会受到影响的。

## 五、倒计时组件设计

设计一个倒计时组件，需要考虑一下三点

- 组件封装能力
  - **组件输入**，即对应组件 **内部 Props** 的 设计 和 考量
  - **组件输出**，即对应组件 **对外提供** 的 属性 或 方法
  - **逻辑复用**，即指组件内部逻辑的 **可组合性**
- 时间相关的敏感度
  - 倒计时的实现方式有多种，例如 **`setInterval、setTimeout、requestAnimationFrame`** 等等，那么哪种更合适？
- 不对客户端时间产生依赖
  - 不对客户端时间产生依赖的方法有多种，例如WebWorker或者使用服务端校验的方法 
- 

针对一个 **CountDown 计时器组件** 的 **props** 至少要包含如下两个内容：

- **`endTime`**，结束时间
- **currentTime**，由用户控制的当前时间，默认为Date.now()
- **`format`**，即输出的时间格式，支持 **DD:HH:mm:ss** 格式和**D 天 H 时 m 分 s 秒**的格式
- **`onEnd` 事件**，即倒计时结束时会被执行的事件
- **`slot` 默认插槽**，即需要展示的组件内容视图，可接收到内部的倒计时格式输出

其中时间我们可以直接限制为 **时间戳**，数值类型，当然如果你想支持更多格式，可以自己在写一个方法处理允许外部传入的各种格式，但实际在组件内部使用时必定是保持是同一种类型，因此在这里我们直接限定类型，让外部去进行转换。

完整代码：

```tsx
import React, {useEffect, useRef, useState} from 'react'
import WebWorker from './worker'
import {formatTime} from './utils/compute'

interface CountDownTimerProps {
  endTime: number
  currentTime: number
  format?: string
  onEnd?: () => void
}

const CountDownTimer: React.FC<CountDownTimerProps> = ({
  endTime,
  currentTime = Date.now(),
  format = 'hh:mm:ss',
  onEnd = Function.prototype,
}) => {
  const restTime = useRef<number>(endTime - currentTime)

  const [time, setTime] = useState<string>(formatTime(format, restTime.current))

  // work.js
  const work = function (this: Worker) {
    let timer: number | null = null
    let time = 0
    this.onmessage = (e: MessageEvent) => {
      const {restTime, state} = e.data
      time = restTime
      if (state === 'stop') {
        if (timer) {
          clearInterval(timer)
          timer = null
        }
        return
      } else if (state === 'start') {
        const interval = 1000
        if (!timer) {
          timer = setInterval(() => {
            time = time - interval
            this.postMessage(time)
          }, interval)
        }
      }
    }
  }

  const worker = new WebWorker(work)

  const workerHandler = () => {
    worker.onmessage((e: MessageEvent) => {
      if (e.data <= 0) {
        worker.postMessage({state: 'stop'})
        onEnd()
      }
      restTime.current = e.data
      setTime(formatTime(format, e.data))
    })

    worker.postMessage({
      state: 'start',
      restTime: Number.parseInt(restTime.current.toString(), 10),
    })
  }

  useEffect(() => {
    workerHandler()
    return () => {
      worker.postMessage({state: 'stop'})
    }
  }, [endTime])

  return (
    <div>
      倒计时：<div>{time}</div>
    </div>
  )
}

export default CountDownTimer

```

