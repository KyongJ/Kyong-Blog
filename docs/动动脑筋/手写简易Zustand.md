---
top: 1
title: 手写简易Zustand
description: Zustand 是一个轻量级、易于使用的状态管理库，专为 React 应用设计。它的名字来源于德语，意为“状态”（state）。Zustand 的核心理念是“简单即美”，它通过提供简洁的 API 和灵活的使用方式，帮助开发者高效地管理应用状态，而不必陷入复杂的配置和冗长的代码中。
author: Kyong
tag: 
 - 动动脑筋
---
# 手写简易Zustand

:::tip Zustand 是什么？
Zustand 是一个轻量级、易于使用的状态管理库，专为 React 应用设计。它的名字来源于德语，意为“状态”（state）。Zustand 的核心理念是“简单即美”，它通过提供简洁的 API 和灵活的使用方式，帮助开发者高效地管理应用状态，而不必陷入复杂的配置和冗长的代码中。

Github 地址：[Zustand](https://github.com/pmndrs/zustand)
:::

## 一、Zustand的使用

从概念上Zustand与Redux类似，都是基于不可变的 state 模型。但是，Redux 要求程序包装在上下文提供程序中；Zustand 并没有要求你使用上下文提供程序来包裹整个应用。相反，它提供了更细粒度的控制，让你可以在需要的地方创建和使用状态存储。

---

#### 创建Store

```typescript
import { create } from "../zustand-ky";
// import { create } from 'zustand';

interface BearState {
    bears: number;
    increase: (count?: number) => void;
    decrease: (count?: number) => void;
    reset: () => void;
}

const useBearStore = create<BearState>((set, get) => ({
    bears: 0,
    increase: (count = 1) => set(state => ({ bears: state.bears + count })),
    decrease: (count = 1) => set(state => ({ bears: Math.max(0, state.bears - count) })),
    reset: () => set({ bears: 0 }),
}));

export default useBearStore;
```

**在组件里调用store里的状态与方法**

```typescript
import { memo } from 'react';
import useBearStore from '../store/useBearStore';

const Bears = () => {
    const bearsStore = useBearStore();
    const { bears, increase, decrease, reset } = bearsStore;

    return (
        <div>
            <h3>BearsPage</h3>

            <p>Number of bears: {bears}</p>
            <button onClick={() => increase()}>Increase</button>
            <button onClick={() => decrease()}>Decrease</button>
            <button onClick={() => reset()}>Reset</button>

            <Child />
        </div>
    );
};
const Child = memo(() => {
    // const bears = useBearStore(state => state.bears);
    const { bears } = useBearStore();

    return (
        <div>
            <h3>ChildPage</h3>
            <p>{bears}</p>
        </div>
    );
});
```

**用户更改状态**

![Zustand-count](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/Zustand-count.gif)

#### 获取所有内容

可以获取到所有的状态，但是这样做会导致组件在每次状态变化时都进行重新渲染。

```typescript
const state = useBearStore();
```

#### 选择状态片段

通过传入selecor进行单个或者多个状态选择。默认情况下，Zustand 使用`Object.is`来检测变化。

```javascript
const bears = useBearStore((state) => state.bears);
```

当构造多个状态对象时，由于每次返回的都是新对象，会导致不必要的渲染。因此可以传入Zustand提供的useShallow来防止当选择器输出根据浅相等不变时不必要的重新渲染。

```javascript
import { create } from "zustand";
import { useShallow } from "zustand/react/shallow";

const { bears, increase } = useBearStore(
  useShallow((state) => ({ bears: state.bears, increase: state.increase })),
);
```

#### 覆盖状态

`set` 函数有第二个参数，默认为 `false`。当设置为`true`时，它将替换状态模型。

```javascript
const useBearStore = create<BearState>((set, get) => ({
    bears: 0,
    increase: (count = 1) => set(state => ({ bears: state.bears + count })),
    decrease: (count = 1) => set(state => ({ bears: Math.max(0, state.bears - count) })),
    reset: () => set({},true), //替换整个state，不是合并
}));
```

## 二、Zustand的实现

从上面的使用案例中可以看到，Zustand 的工作方式与 Redux 有很多相似之处。例如，**状态值不能直接修改**，都是通过 `setState` 方法来触发修改，保证了状态的不可变性，使状态管理更加可预测和易于调试。此外，Zustand 在通知状态变化时使用了**发布订阅模式**，只有订阅了相应状态的组件才会被更新，从而提高了应用的性能。

任何状态管理工具的核心都围绕着两个基本要素：

1. **单一的状态容器（简称：store）**：这个容器负责持有应用的状态或数据，并提供接口让外部代码可以对这些状态进行查询、添加、删除和修改。
2. **观察者模式**：当store中的状态发生变更时，它会通过事件通知机制告知所有监听该状态的组件或模块，使它们能够获取并响应最新的状态变化。

Zustand 也不例外，它通过一个单一的 store 来集中管理状态，并利用观察者模式确保状态的变化能够被及时且高效地传播给所有相关的监听者。

我们将分两部分来实现简易版的 Zustand：`index.ts` 和 `vanilla.ts`。`index.ts` 负责提供高层的 API，包括与React的交互，订阅状态以及暴露给开发者的API，而 `vanilla.ts` 则实现了状态管理的核心逻辑(完整代码将会在文末给出)。

### 2.1 index.ts文件解析

`index.ts` 文件主要负责定义和导出高层次的 API，其中最主要的则是 `create` 函数和 `useStore` Hook。

#### create函数

`create`函数是Zustand暴露给开发者的 API，它接受一个 `createStateFn` 函数作为参数，并返回一个自定义的 `useBoundStore` hook。这个 hook 允许组件订阅状态，并获取最新的状态。

之所以将入参设置为函数，是为了更好的支持中间件的开发，中间件只需要接收之间的函数，并返回一个新的函数就可以达到中间处理的效果，不需要改动Zustand的源码。

可以发现这个函数的内容非常简单，总共只有三行代码，而创造store的核心则是第一行的`createStore`方法。

```typescript
//实现create函数，接受一个函数作为参数，函数的作用是创建仓库对象
export const create: Creator = <T>(createStateFn: creatorState<T>) =>
    createStateImpl(createStateFn);

const createStateImpl = <T>(createStateFn: creatorState<T>) => {
    // 调用创建store方法
    const api = createStore(createStateFn);

    //创建一个自定义hook，供组件使用并返回
    const useBoundStore: any = (selector?: any) => useStore(api, selector);

    Object.assign(useBoundStore, api);

    return useBoundStore;
};
```

---

#### **useStore Hook**

useStore 负责与React组件进行连接，用于在 React 组件中订阅状态变化并获取最新状态。它使用了 `useSyncExternalStore` 来确保状态变化时组件能够及时更新。**因此使用 `zustand` 不需要给程序包裹 context provider**。

`identity` 函数是一个非常简单的通用函数，它接收一个参数并返回相同的参数，同时保留其类型信息。在 `useStore` 函数中，`identity` 作为默认的选择器函数，当没有提供选择器时，它会直接返回整个状态。

```typescript
//默认返回自身的函数
const identity = <T>(arg: T): T => arg;
// 函数重载
export function useStore<S extends ReadonlyStoreApi<unknown>>(api: S): GetState<S>;

export function useStore<S extends ReadonlyStoreApi<unknown>, U>(
    api: S,
    selector: (state: GetState<S>) => U
): U;

export function useStore<TState, StateSlice>(
    api: ReadonlyStoreApi<TState>,
    selector: (state: TState) => StateSlice = identity as any
) {
    const slice = useSyncExternalStore(
        api.subscribe,
        () => selector(api.getState()),
        () => selector(api.getInitialState())
    );
    return slice;
}
```

---

#### useSyncExternalStore

`useSyncExternalStore`是React 提供的一个 Hook，用于同步外部状态。它接收三个参数：`subscribe` 、 `getState` 和`getServerSnapshot`，分别用于订阅状态变化和获取当前状态。

- `subscribe`：一个函数，接收一个单独的 `callback` 参数并把它订阅到 store 上。当 store 发生改变时会调用提供的 `callback`，这将导致 React 重新调用 `getSnapshot` 并在需要的时候重新渲染组件。`subscribe` 函数会返回清除订阅的函数。
- `getSnapshot`：一个函数，返回组件需要的 store 中的数据快照。在 store 不变的情况下，重复调用 `getSnapshot` 必须返回同一个值。如果 store 改变，并且返回值也不同了（用 [`Object.is`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/is) 比较），React 就会重新渲染组件。
- **可选** `getServerSnapshot`：一个函数，返回 store 中数据的初始快照。它只会在服务端渲染时，以及在客户端进行服务端渲染内容的激活时被用到。快照在服务端与客户端之间必须相同，它通常是从服务端序列化并传到客户端的。如果你忽略此参数，在服务端渲染这个组件会抛出一个错误

具体用法可查看React官方文档(useSyncExternalStore用法)[https://zh-hans.react.dev/reference/react/useSyncExternalStore#extracting-the-logic-to-a-custom-hook]

---

#### 原生订阅写法

`useSyncExternalStore`的出现极大的方便了React组件对外部store的订阅，那如果不使用这个API，原生的写法该怎么写呢？

```typescript
export function useCustomStore<TState, StateSlice>(
    api: ReadonlyStoreApi<TState>,
    selector: (state: TState) => StateSlice
): StateSlice {
    const [, forceRender] = useState(0);
    useEffect(() => {
        const handleChange = (state: TState, prevState: TState) => {
            const newObj = selector(state);
            const oldObj = selector(prevState);

            if (newObj !== oldObj) {
                forceRender(Math.random());
            }
        };
        // 订阅状态变化
        const unsubscribe = api.subscribe(handleChange);

        // 立即调用 handleChange 以确保初始状态是正确的
        handleChange(api.getState(), api.getInitialState());

        // 返回一个清理函数，在组件卸载或依赖项变化时取消订阅
        return () => {
            unsubscribe();
        };
    }, [api, selector]); // 添加依赖项

    return selector(api.getState());
}
```
使用`useState`来触发组件更新，通过`useEffect`来订阅状态变化。创建`handlechange`函数来判断状态是否变化，如果变化则触发组件更新。


### 2.2 vanilla.ts文件解析

`vanilla.ts` 文件实现了状态管理的核心逻辑，包括状态的创建、更新、订阅等功能。而其中最主要的则是`createStore`函数，他接受外部传来的`createStateFn`函数，从而创建状态仓库。

#### createStore

```typescript
/**
 * const createStoreFunction = createStore();
   const store = createStoreFunction((set, get) => ({ count: 0 }));
 */
export const createStore = (createStateFn =>
    createStateFn ? createStoreImpl(createStateFn) : createStoreImpl) as creatorStore;
```

这一段代码的意义是：`createStore` 函数既可以作为一个工厂函数，也可以是一个构造器，具体行为取决于是否提供了 `createStateFn` 参数。如果提供了 `createStateFn`，它会立即使用这个函数创建并返回一个 `store`；如果没有提供 `createStateFn`，`createStore` 返回一个函数，允许在后续调用时传递 `createStateFn`。

#### **createStoreImpl**

状态存储的核心实现。它接受一个 `createStateFn` 函数，该函数用于初始化状态并定义状态更新逻辑。`createStoreImpl` 返回一个类型为StoreApi的对象，包含 `setState`、`getState`、`getInitialState` 和 `subscribe` 方法，这些方法分别用于更新状态、获取当前状态、获取初始状态和订阅状态变化。

```typescript
//类型定义
type SetStateInternal<T> = (
    partial: T | Partial<T> | { _(state: T): T | Partial<T> }['_'],
    replace?: boolean
) => void;
type Listener<T> = (state: T, prevState: T) => void;

export interface StoreApi<T> {
    setState: SetStateInternal<T>;
    getState: () => T;
    getInitialState: () => T;
    subscribe: (listener: Listener<T>) => () => void;
}

//创建状态仓库
export const createStoreImpl: creatorStoreImpl = createStateFn => {
    type TState = ReturnType<typeof createStateFn>;

    let state: TState;
    //定义监听器
    const listeners: Set<Listener<TState>> = new Set();

    const setState: StoreApi<TState>['setState'] = (partial, replace) => {
        // 类型断言 防止编译出错
        const nextState =
            typeof partial === 'function' ? (partial as (state: TState) => TState)(state) : partial;

        // 判断是否更新
        if (!Object.is(state, nextState)) {
            const previousState = state;
            state =
                replace ?? (typeof nextState !== 'object' || nextState === null)
                    ? (nextState as TState)
                    : Object.assign({}, state, nextState);
            listeners.forEach(listener => listener(state, previousState));
        }
    };
    const getState: StoreApi<TState>['getState'] = () => state;
    const getInitialState: StoreApi<TState>['getInitialState'] = () => initialState;
    const subscribe: StoreApi<TState>['subscribe'] = listener => {
        listeners.add(listener);
        //删除监听器
        return () => listeners.delete(listener);
    };
    const api = {
        setState,
        getState,
        getInitialState,
        subscribe,
    };
    state = createStateFn(setState, getState);
    const initialState = state;

    return api;
};
```

- **`setState`**：更新状态的核心方法。它接受一个 `partial` 参数，可以是部分状态对象、完整状态对象或一个返回新状态的函数。`setState` 会根据传入的参数更新状态，并在状态发生变化时遍历listeners通知所有监听者。从`setState`的实现我们可以知道zustand支持自动合并，给 `setState` 传的状态（或者 updater 更新函数返回的新状态）不需要包含状态的全部属性。
- **`listeners`**：这是一个 `Set`集合，用于存储所有的状态变化监听器。每当状态发生变化时，`setState` 会遍历 `listeners` 并调用每个监听器，通知它们状态已经更新。
- **`subscribe`**：这是订阅状态变化的方法。它接受一个监听器函数，并将其添加到 `listeners` 中。同时，它返回一个取消订阅的函数，允许用户在不再需要监听时移除监听器。
- **`initialState`**：这是状态的初始值，保存在 `createStoreImpl` 内部，以便后续可以通过 `getInitialState` 获取



## 三、总结

到这里，我们已经搭建了一个简易的Zustand，并具备了基本的全局状态管理功能。在这次Zustand源码学习中，除了学习到设计的巧思之外，我对ts的类型定义也有了更深的理解。当然，这只是一个简单的演示项目，主要目的是帮助我们深入理解 Zustand 的工作原理和实现细节。在实际生产环境中，一个成熟的状态管理器还需要应对更多复杂的挑战，比如性能优化、中间件集成、跨平台支持。我将继续深入学习其他源码，并与大家分享我的学习成果！

完整代码:https://github.com/KyongJ/Zustand-ky

