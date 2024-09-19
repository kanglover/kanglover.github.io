---
layout: post
title: react hooks原理
date: 2024-08-06
author: shaokang
header-img:
catalog: true
tags:
    - react
---

## 背景

React Hooks 解决的痛点问题是组件复用状态逻辑很难、复杂的组件难以理解、难以理解的 class 等原因。

详见[React Hooks 简介](https://zh-hans.legacy.reactjs.org/docs/hooks-intro.html#motivation)

## 设计目标和原则

-   无 Class 的复杂性
-   无生命周期的困扰
-   优雅地复用
-   对齐 React Class 组件已经具备的能力

## 概览

React Hooks 提供了以下 API：

-   useState: 给组件添加一些内部 state
-   useEffect: 给函数组件增加了操作副作用的能力
-   useReducer: 通过 reducer 来管理组件本地的复杂 state
-   useContext: 共享数据
-   useMemo & useCallback : 缓存函数避免非必要渲染

更多的见[React Hooks API 参考](https://zh-hans.react.dev/reference/react/hooks)

## 实现原理

### fiber 节点

Hook 对象是相对于组件存在的，所以 Hook 对象是挂在 fiberNode 上的。

fiber 的主要属性如下：

```js
var fiber = {
    alternate,
    child,
    elementType: () => {},
    memoizedProps: null,
    memoizedState: null, // 在函数组件中，memoizedState 用于保存 hook 链表
    pendingProps: {},
    return,
    sibling,
    stateNode,
    tag,
    type: () => {}
    updateQueue: null,
}
```

在函数组件的 fiber 中，有两个属性和 hook 有关：memoizedState 和 updateQueue 属性。

-   memoizedState 属性用于保存 hook 链表，hook 链表是单向链表。
-   updateQueue 属性用于收集 hook 的副作用信息，保存 useEffect、useLayoutEffect、useImperativeHandle 这三个 hook 的 effect 信息，是一个环状链表，其中 updateQueue.lastEffect 指向最后一个 effect 对象。effect 描述了 hook 的信息，比如 useLayoutEffect 的 effect 对象保存了监听函数，清除函数，依赖等。

### hooks 数据结构

```ts
type Hook = {
    memoizedState: any; // 上一次完整更新之后的最终状态值
    baseState: any; // 当前更新的状态值
    baseQueue: Update<any, any> | null; // 当前更新队列
    queue: UpdateQueue<any, any> | null; // 更新队列
    next: Hook | null; // 下一个hook
};
```

React Hooks 是一个`链表结构`，每个 Hook 对象都包含一个 next 指针，指向下一个 Hook 对象。假如函数组件中定义了多个 useState，那么就会有多个 Hook 对象，对象之间通过 next 指针连接起来。

### Hook 对象属性

### memoizedState

hook 对象中的 memoizedState 属性和 fiber 的 memoizedState 属性含义是不同的。

**useState**

memoizedState 保存的是 useState 的 state 值。

对于 const [state, updateState] = useState(initialState)，memoizedState 保存的就是 state 的值。

**useReducer**

memoizedState 保存的是 state 值。

对于 const [state, dispatch] = useReducer(reducer, {}); memoizedState 保存 state 的值。

> useState 其实是 useReducer 的语法糖，useState 内部实现就是 useReducer。

**useEffect**

memoizedState 保存的是一个 effect 对象，effect 对象保存的是 hook 的状态信息，比如监听回调函数，依赖项，清除函数等。

```ts
type Effect = {
    tag: any; // effect 的类型，useEffect 对应的 tag 为 5，useLayoutEffect 对应的 tag为 3
    create: any; // useEffect 或者 useLayoutEffect 的监听函数，即第一个参数
    destroy: any; // useEffect 或者 useLayoutEffect 的清除函数，即监听函数的返回值
    deps: Array; // useEffect 或者 useLayoutEffect 的依赖，第二个参数
    next: Effect; // 在 updateQueue 中使用，将所有的 effect 连成一个链表
};

type EffectQueue = {
    lastEffect: Effect;
};

type FiberNode = {
    memoizedState: any; // 用来存放某个组件内所有的Hook状态
    updateQueue: any; // effect 对象组成的环状链表，updateQueue.lastEffect 指向最后一个 effect 对象
};
```

effect 链表同时会保存在 fiber.updateQueue 中。在 hook 执行的过程中需要给 fiber 添加对应的副作用标记。然后在 commit 阶段执行对应的操作，比如调用 useEffect 的监听函数，清除函数等。因此，React 需要将这三个 hook 函数的 effect 对象存到 fiber.updateQueue 中，以便在 commit 阶段遍历 updateQueue，执行对应的操作。updateQueue 也是一个环状链表，lastEffect 指向最后一个 effect 对象。effect 和 effect 之间通过 next 相连。

```js
const effect = {
    create: () => { console.log("useEffect", count); },
    deps: [0]
    destroy: undefined,
    tag: 5,
};
effect.next = effect;
fiber.updateQueue = {
    lastEffect: effect,
};
```

**useRef**

memoizedState 保存的是 ref 的值。比如 const myRef = useRef(null)，memoizedState 保存的是 myRef 的值。

```js
hook.memoizedState = {
    current,
};
```

对于 useRef(1)，memoizedState 保存 { current: 1 }。

**useMemo**

memoizedState 保存的是 useMemo 的返回值和依赖。比如 useMemo(callback, [depA])，memoizedState 保存[callback(), depA]

```js
const res = useMemo(() => {
    return count * count;
}, [count]);
```

对于这个例子，hook.memoizedState = [count \* count, [count]];

**useCallback**

memoizedState 保存的是 useCallback 的回调函数和依赖.

对于 useCallback(callback, [depA])，memoizedState 保存[callback, depA]。

与 useMemo 的区别是，useCallback 保存的是 callback 函数本身，而 useMemo 保存的是 callback 函数的执行结果。

> [!NOTE]
> useContext 是唯一一个不需要添加到 hook 链表的 hook 函数

### queue 数据结构

hook.queue 保存的是更新队列，是个环状单向链表。

queue 的属性如下：

```js
hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: initialState,
};
```

queue.pending 指向最后一个更新对象，update 对象属性如下：

```js
var update = {
    lane: lane,
    action: action, // setCount的参数
    eagerReducer: null,
    eagerState: null,
    next: null,
};
```

比如我们在 onClick 中调用 setCount，每次调用 setCount，都会创建一个新的 update 对象，并添加进 hook.queue 中。

queue 队列：

```js
queue.pending = update4 ---> update1 ---> update2 ---> update3
                ^                                       |
                |                                       |
                -----------------------------------------
```

queue.pending 始终指向最后一个插入的 update。当我们要遍历 update 时，queue.pending.next 指向第一个插入的 update。

### dispatcher

组件 mount 时的 hook 与 update 时的 hook 来源于不同的对象，这类对象在源码中被称为 dispatcher。

初次渲染和更新这两个过程，构建 hook 链表的算法不一样，因此 React 对这两个过程是分开处理的。

```js
// mount 时的 Dispatcher
const HooksDispatcherOnMount: Dispatcher = {
    useCallback: mountCallback,
    useContext: readContext,
    useEffect: mountEffect,
    useImperativeHandle: mountImperativeHandle,
    useLayoutEffect: mountLayoutEffect,
    useMemo: mountMemo,
    useReducer: mountReducer,
    useRef: mountRef,
    useState: mountState,
    // ...省略
};

// update 时的 Dispatcher
const HooksDispatcherOnUpdate: Dispatcher = {
    useCallback: updateCallback,
    useContext: readContext,
    useEffect: updateEffect,
    useImperativeHandle: updateImperativeHandle,
    useLayoutEffect: updateLayoutEffect,
    useMemo: updateMemo,
    useReducer: updateReducer,
    useRef: updateRef,
    useState: updateState,
    // ...省略
};
```

在 FunctionComponent render 前，会根据 FunctionComponent 对应 fiber 的以下条件区分 mount 与 update。

```js
current === null || current.memoizedState === null;
```

并将不同情况对应的 dispatcher 赋值给全局变量 ReactCurrentDispatcher 的 current 属性。

```js
ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
        ? HooksDispatcherOnMount
        : HooksDispatcherOnUpdate;
```

### 构建 hook 链表

上面我们已经说过了创建 hook 分为两种情况，如果是初次渲染，则使用 HooksDispatcherOnMount，此时如果我们调用 useState，实际上调用的是 HooksDispatcherOnMount.useState，执行的是 mountState 方法。如果是更新阶段，则使用 HooksDispatcherOnUpdate，此时如果我们调用 useState，实际上调用的是 HooksDispatcherOnUpdate.useState，执行的是 updateState。

初次渲染和更新渲染执行 hook 函数的区别在于：

-   构建 hook 链表的算法不同。初次渲染只是简单的构建 hook 链表，而更新渲染会遍历上一次的 hook 链表，构建新的 hook 链表，并复用上一次的 hook 状态
-   依赖的判断。初次渲染不需要判断依赖，更新渲染需要判断依赖是否变化。
-   对于 useState 来说，更新阶段还需要遍历 queue 链表，计算最新的状态。

函数组件的执行都是从 [renderWithHooks](https://github.com/acdlite/react/blob/master/packages/react-reconciler/src/ReactFiberHooks.new.js#L352) 函数开始执行，这个时候会去判断组件是渲染还是更新。

**renderWithHooks**

```js
function renderWithHooks(current, workInProgress, Component, props) {
    currentlyRenderingFiber = workInProgress;
    workInProgress.memoizedState = null;
    workInProgress.updateQueue = null;

    ReactCurrentDispatcher.current =
        current === null || current.memoizedState === null
            ? HooksDispatcherOnMount
            : HooksDispatcherOnUpdate;

    var children = Component(props, secondArg);

    currentlyRenderingFiber = null;
    currentHook = null;
    workInProgressHook = null;

    return children;
}
```

以 useState 为例：

```js
function useState(initialState) {
    var dispatcher = resolveDispatcher();
    return dispatcher.useState(initialState);
}
function resolveDispatcher() {
    var dispatcher = ReactCurrentDispatcher.current;
    return dispatcher;
}
```

每一个 hook 函数在执行时，都会调用 resolveDispatcher 方法获取当前的 dispatcher，然后调用 dispatcher 中对应的方法处理 mount 或者 update 逻辑。

useState mount 会执行 [mountReducer](https://github.com/acdlite/react/blob/master/packages/react-reconciler/src/ReactFiberHooks.new.js#L685) 方法，update 会执行 [updateReducer](https://github.com/acdlite/react/blob/master/packages/react-reconciler/src/ReactFiberHooks.new.js#L714)。对应的其他 hook 函数都有对一个的 mount 和 update 方法。

我们能从这些方法中看到创建 hook 对象是通过一个 [mountWorkInProgressHook](https://github.com/acdlite/react/blob/master/packages/react-reconciler/src/ReactFiberHooks.new.js#L592)/[updateWorkInProgressHook](https://github.com/acdlite/react/blob/master/packages/react-reconciler/src/ReactFiberHooks.new.js#L613) 函数生成的，并且会把 hook 对象绑定到 fiber 节点上。

**mountWorkInProgressHook**

```js
function mountWorkInProgressHook() {
    var hook = {
        memoizedState: null,
        baseState: null,
        baseQueue: null,
        queue: null,
        next: null,
    };

    if (workInProgressHook === null) {
        // hook链表中的第一个hook
        currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
    } else {
        // 添加到hook链表末尾
        workInProgressHook = workInProgressHook.next = hook;
    }

    return workInProgressHook;
}
```

初次渲染创建的第一个 hook 对象添加到 memoizedState，后面的的 hook 添加到 hook 链表末尾。

**updateWorkInProgressHook**

```js
function updateWorkInProgressHook(): Hook {
    let nextCurrentHook: null | Hook;
    if (currentHook === null) {
        const current = currentlyRenderingFiber.alternate;
        if (current !== null) {
            nextCurrentHook = current.memoizedState;
        } else {
            nextCurrentHook = null;
        }
    } else {
        nextCurrentHook = currentHook.next;
    }

    let nextWorkInProgressHook: null | Hook;
    if (workInProgressHook === null) {
        nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
    } else {
        nextWorkInProgressHook = workInProgressHook.next;
    }

    if (nextWorkInProgressHook !== null) {
        workInProgressHook = nextWorkInProgressHook;
        nextWorkInProgressHook = workInProgressHook.next;

        currentHook = nextCurrentHook;
    } else {
        invariant(
            nextCurrentHook !== null,
            'Rendered more hooks than during the previous render.'
        );
        currentHook = nextCurrentHook;

        const newHook: Hook = {
            memoizedState: currentHook.memoizedState,

            baseState: currentHook.baseState,
            baseQueue: currentHook.baseQueue,
            queue: currentHook.queue,

            next: null,
        };

        if (workInProgressHook === null) {
            currentlyRenderingFiber.memoizedState = workInProgressHook =
                newHook;
        } else {
            workInProgressHook = workInProgressHook.next = newHook;
        }
    }
    return workInProgressHook;
}
```

更新渲染，renderWithHooks 开始调用，workInProgressHook、currentHook 会设置为 null，memoizedState、updateQueue 都被重置为 null。

假如有两个 hook。

第一次初始化它指向旧的 hook 链表的第一个 hook 对象。

```js
if (currentHook === null) {
    var current = currentlyRenderingFiber.alternate;

    if (current !== null) {
        nextCurrentHook = current.memoizedState;
    } else {
        nextCurrentHook = null;
    }
}

currentHook = nextCurrentHook;
```

创建一个新的 hook 对象，复用上一次的 hook 对象的状态信息，并初始化 hook 链表

```js
var newHook = {
    memoizedState: currentHook.memoizedState,
    baseState: currentHook.baseState,
    baseQueue: currentHook.baseQueue,
    queue: currentHook.queue,
    next: null,
};

if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
}
```

第二个 hook，此时 currentHook 已经有值，指向第一个 hook 对象。因此将 currentHook 指向它的下一个 hook 对象，即第二个

```js
if (currentHook === null) {
    ...
} else {
    nextCurrentHook = currentHook.next;
}
currentHook = nextCurrentHook;
```

同样也会创建一个新的 hook 对象，并复用上一次的 hook 状态。

**从更新渲染的过程也可以看出，hook 函数的执行是会遍历旧的 hook 链表并复用旧的 hook 对象的状态信息。这也是为什么我们不能将 hook 函数写在条件语句或者循环中的根本原因，我们必须保证 hook 函数的顺序在任何时候都要一致**

## 常见问题

Q: **为什么 hook 函数不能写在条件语句或者循环中**  
A: hook 对象是链表结构，依赖定义时的顺序。如果写在了条件语句或者循环中，hooks 链表结构被破坏，可能会导致意想不到的错误。

Q: **hook 为什么要用链表结构**  
A：链表结构确保 Hook 调用的顺序一致、简化状态管理、支持嵌套 Hooks 以及高效的内存管理，不需要管理 key（map 结构）。

## 参考资料

-   [彻底搞懂 React-hook 链表构建原理](https://cloud.tencent.com/developer/article/2227386)
-   [React 技术揭秘](https://react.iamkasong.com/hooks/prepare.html)
