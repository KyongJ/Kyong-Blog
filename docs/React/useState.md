---
title: 你真的了解useState吗？
description: 默认支持流程图，tabs面板，待办列表，作品页面
---

# 你真的了解useState吗？

## 前言

自React引入Hooks后，、React允许你在不编写class的情况下可以使用state和其他的React特性。

`useState`就是函数组件引入状态的钩子函数。在平时开发的时候，我一直对setState的执行机制有点迷糊，`useState`方法到底是同步还是异步，如果是异步，执行的时机又在什么什么时候呢？再观看了许多博客文章和自己实践后，决定写一个总结。



## 执行顺序

请看下面的例子

```js
const Demo: FC = () => {
    const [count, setCount] = useState<number>(0);
    console.log('count', count);
    const handleClickAsync = () => {
        setCount(count+1);
        console.log(count);
    };

    return (
        <div >
            {console.log('render')}
            <div>count:{count}</div>
            <button onClick={handleClickAsync} >Increase async</button>
        </div>
    );
};

export default Demo;
```

**页面呈现：**

![image-20220404165413068](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20220404165413068.png)

**点击按钮结果：**

![image-20220404164951475](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20220404164951475.png)

可以发现在`setCount(count+1)`后，输出的count值为0。这是因为React的每一次更新都是独立的，你拿到的还是上一个 `state` ，那打印出来的值自然是上一次的，但是此时真正的 `count`其实已经被改变了。

从上述的例子可以看出`setState`在React的`onClick`事件中，存在延迟执行，也就是所谓的异步执行。熟悉浏览器事件循环的小伙伴都知道Js执行过程的异步函数执行依靠的是两个任务队列，即宏任务队列和微任务队列。

`macro-task`大概包括：

- script(整体代码)
- setTimeout
- setInterval
- setImmediate
- I/O
- UI render

`micro-task`大概包括:

- process.nextTick
- Promise
- Async/Await(实际就是promise)
- MutationObserver(html5新特性)

而setState所谓的异步与这两个异步任务执行顺序有何不同呢？为此我们补充一下`handleClickAsync`的函数体

```jsx
const handleClickAsync = () => {
        Promise.resolve()
        .then(() => {
            console.log('微任务1执行');
        })
        .then(() => {
            console.log('微任务2执行');
        });
        setCount(count + 1);
        setTimeout(() => {
            console.log('宏任务执行');
        });
        console.log(count);
    };
```

**点击按钮结果:**

![image-20220404173036582](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20220404173036582.png)

很明显可以看出`setState`的执行时机是晚于`console.log(count)`，并且早于微任务与宏任务。通俗的讲，它其实是一个伪异步的过程，即延迟执行但本身还在一个事件循环，所以它的执行顺序在同步代码后、异步代码前。

## 合成事件

为什么会有这种现象？这就要说到react的合成事件了，`react`的批处理更新(下面会说到)也得益于合成事件。在合成事件和钩子函数中是“异步”的。

在原生事件、`setTimeout`、`Promise.resolve().then` 中执行`setState`，你会得到同步的代码。

### 什么是合成事件？

React合成事件是指将原生事件合成一个React事件，之所以要封装自己的一套事件机制，目的是为了实现全浏览器的一致性，抹平不同浏览器之间的差异性。

```js
const handleClick = (e) => {e.preventDefault();}

// 原生事件
<div onclick="handleClick()"></div>

// React合成事件
<div onClick={HandleCilck}></div>
```

> 上述引自 https://juejin.cn/post/6991645668934680584#heading-0
>
> 有兴趣的小伙伴可以去学习一下



已经懂了的小伙伴可以通过下面这个例子自我练习一下

```jsx
const Demo: FC = () => {
    const [count, setCount] = useState<number>(0);
    console.log('count', count);
    const handleClickAsync = () => {
        console.log(111111);
        Promise.resolve()
            .then(() => {
                setCount(count => count + 1);
                console.log('微任务1执行');
                console.log('微任务1:count', count);
            })
            .then(() => {
                setCount(count + 1);
                console.log('微任务2执行');
                console.log('微任务2:count', count);
            });
        setCount(count => count + 1);
        console.log(222222);
        setTimeout(() => {
            setCount(count => count + 1);
            console.log('宏任务执行');
            console.log('宏任务:count', count);
        });
        console.log(333333);
    };
    return (
        <div>
            {/* {console.log('render')} */}
            <div>count:{count}</div>
            <button onClick={handleClickAsync}>Increase async</button>
        </div>
    );
};
```

**结果：**

![image-20220404195247901](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20220404195247901.png)

注意：

由于`React`闭包特点，`setCount(count+1)`里面的`count`值实际为0，所以之前的`count`会被覆盖，如果每次都想拿到新值，可以使用`setCount(count => count + 1)`



## 批量更新

在做我的博客项目时，我时常在一个方法里面写两个或者多个`setState`方法来控制状态。我一直疑惑，`setState`方法是写N次就会更新N次吗？带着疑问，我们来看下面这个代码。

### 同步事件

#### 连续执行多个State

```jsx
const Demo: FC = () => {
    const [count, setCount] = useState<number>(0);
    const [name, setName] = useState<string>('')
    console.log('count', count);
    console.log('name', name);
    const handleClickAsync = () => {
        setCount(count=>count+1)
        setName('kyong')
    };
    return (
        <div>
            {console.log('render')}
            <div>count:{count}</div>
            <button onClick={handleClickAsync}>Increase async</button>
        </div>
    );
};

export default Demo;
```

**结果：**

![image-20220404182946593](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20220404182946593.png)

可以发现在React合成事件和钩子函数中，React会通过`batchUpdate `机制进行合并更新，减少`render`次数，提高渲染效率。



#### 连续执行一个State

```jsx
const Demo: FC = () => {
    const [count, setCount] = useState<number>(0);
    const [name, setName] = useState<string>('')
    console.log('count', count);
    console.log('name', name);
    const handleClickAsync = () => {
        setCount(count+1)
        setCount(count+2)
        setCount(count+1)
        setCount(count=>count+1)
        setCount(count=>count+1)
        setName('kyong')
        setName('Genie')
    };
    return (
        <div>
            {console.log('render')}
            <div>count:{count}</div>
            <button onClick={handleClickAsync}>Increase async</button>
        </div>
    );
};

export default Demo;
```

**结果：**

![image-20220404183732452](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20220404183732452.png)

因为`setState`有两种更新方法，直接赋值更新和**函数赋值更新**。很明显，多次直接赋值，最后一次的值会覆盖先前的值。而使用函数赋值更新每次都会拿到最新的值。所以在平时开发过程中，如果**当前值与先前的值有关**的话，建议使用函数赋值。



### 异步事件

#### 连续执行多个State

```jsx
const Demo: FC = () => {
    const [count, setCount] = useState<number>(0);
    const [name, setName] = useState<string>('');
    console.log('count', count);
    console.log('name', name);
    const handleClickAsync = () => {
        console.log('异步执行');
        Promise.resolve().then(() => {
            setCount(count => count + 1);
            setName('kyong');
        });
    };
    return (
        <div>
            {console.log('render')}
            <div>count:{count}</div>
            <button onClick={handleClickAsync}>Increase async</button>
            {/* <button onClick={handleClickSync}>Increase sync</button> */}
        </div>
    );
};
```

**结果：**

![image-20220404184919765](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20220404184919765.png)

在异步执行过程中，`setState`执行机制是同步的，N个State会更新N次



#### 连续执行一个State

```jsx
const Demo: FC = () => {
    const [count, setCount] = useState<number>(0);
    const [name, setName] = useState<string>('');
    console.log('count', count);
    console.log('name', name);
    const handleClickAsync = () => {
        console.log('异步执行');
        Promise.resolve().then(() => {
            setCount(count + 1);
            setCount(count + 2);
            setCount(count => count + 1);
            setCount(count => count + 1);
            setName('kyong');
            setName('Genie');
        });
    };
    return (
        <div>
            {console.log('render')}
            <div>count:{count}</div>
            <button onClick={handleClickAsync}>Increase async</button>
            {/* <button onClick={handleClickSync}>Increase sync</button> */}
        </div>
    );
};
```

**结果:**

![image-20220404192009050](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20220404192009050.png)

浏览器依然渲染了多次，而且在`setState`同步执行过程中，依然有闭包问题的存在



**注：**

异步事件中`setTimeout`和`setInterval`两个函数，自身因为可以设置延迟执行时间和闭包的存在，所以在多次点击时，表现的与promise.then()方法有些许不同。

延迟时间为0ms

```jsx
const handleClickAsync = () => {
        console.log('异步执行');
        setTimeout(() => {
            setCount(count => count + 1);
            console.log(count);
        },0);
    };
```

![image-20220404201529495](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20220404201529495.png)

延迟时间为1000ms

```js
const handleClickAsync = () => {
        console.log('异步执行');
        setTimeout(() => {
            setCount(count => count + 1);
            console.log(count);
        },1000);
    };
```

![image-20220404201657069](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20220404201657069.png)

## 总结

1. 在正常的react的事件流里（如onClick等）

   - useState是‘异步’执行的（不会立即更新state的结果）

   - 多次执行useState，只会调用一次重新渲染render

2. 在setTimeout，Promise.then等异步事件中

   - useState是同步执行的（立即更新state的结果）

   - 多次执行useState，每一次的执行useState，都会调用一次render



