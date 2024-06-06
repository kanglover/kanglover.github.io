---
layout: post
title: React-Fiber
date: 2022-12-07
author: shaokang
header-img: img/react.png
catalog: true
tags:
    - React
    - 复习
---

## 前言

React 16 采用了一个全新的架构 --- Fiber，其最大的使命是解决大型 React 项目的性能问题。React 早期的优化都是停留于 JS 层面（vdom 的 create/diff），诸如减少组件的复杂度（Stateless），减少向下 diff 的规模(SCU)，减少 diff 的成本(immutable.js)。React16 则升级到浏览器渲染机制层面, 在 patch 上取得了突破。

## 是什么？

Fiber 是 React 16 中采用的新协调（reconciliation）引擎，主要目标是支持虚拟 DOM 的渐进式渲染。

Fiber 将原有的 Stack Reconciler 替换为 Fiber Reconciler，提高了复杂应用的可响应性和性能。主要通过以下方式达成目标：

-   对大型复杂任务的分片。
-   对任务划分优先级，优先调度高优先级的任务。
-   调度过程中，可以对任务进行挂起、恢复、终止等操作。

## 为什么推出 fiber

React 16 之前协调算法是 Stack Reconciler（栈调和），即递归遍历所有的 Virtual DOM 节点执行 Diff 算法，一旦开始便无法中断，直到整颗虚拟 dom 树构建完成后才会释放主线程。如果组件树的层级很深，递归更新组件的时间超过 16ms，用户交互就会感觉到卡顿。

大家都知道，React 中调用 setState 方法后，框架会自顶向下重新渲染组件，组件以及它的子组件全部需要渲染。  
所以当一个数据改变，React 的组件渲染是很消耗性能的——父组件的状态更新了，所有的子组件得跟着一起渲染。  
React 为了让开发者优化其性能，提供了 shouldComponentUpdate、PureComponent、useMemo、useCallback 方法来保证父组件更新不引起子组件的渲染。

而 React 框架层基于性能考虑同样也做了重大地更新，下面让我们来看看是什么。
众所周知，浏览器是单线程。想象一下，如果有两个线程，一个线程要对这节点进行移除，一个要对它进行样式操作。线程是并发的，无法决定顺序，这样页面的效果是不可控的。换单线程则简单可控，但 JS 执行与视图渲染与资原加载与事件回调是如何调度呢，于是有了 EventLoop。  
EventLoop 是非常复杂的，但有一个要点，你一下子分配它许多任务，它的处理速度就下降。如果你把相同的任务放在一起，它的速度就会上去。一下子创建 10000 个 DIV，并设置随机 innerHTML，随机背景，它在 chrome 都会卡好久。如果打散，每隔 60ms 处理当中的 100 个，分 10 次处理，则页面很流畅。

React16 的优化思想就是基于这点。Fiber 调度算法分成两个阶段，第一个阶段创建虚拟 Dom 树，并且标记各种可能任务（sadeEffect）。第二个阶段执行任务，优先进行 DOM 插入或移动操作，然后才是属性样式操作。
其中先操作 DOM 再设置属性这是同步模式的优化，而异步模式的时间分片，React 通过 requestIdleCallback 方法，在浏览器空闲时再执行视图渲染/事件回调。

## Fiber Reconciler 如何工作

在 Fiber 中，会把一个耗时很长的任务分成很多小的任务片，每一个任务片的运行时间很短。虽然总的任务执行时间依然很长，但是在每个任务小片执行完之后，都会给其他任务一个执行机会。 这样，唯一的线程就不会被独占，其他任务也能够得到执行机会。

为了实现渐进渲染的目的，Fiber 架构中引入了新的数据结构：Fiber Node，Fiber Node Tree 根据 React Element Tree 生成，并用来驱动真实 DOM 的渲染。

Fiber 节点的大致结构：

```js
{
    tag: TypeOfWork, // 标识 fiber 类型
    type: 'div', // 和 fiber 相关的组件类型
    return: Fiber | null, // 父节点
    child: Fiber | null, // 子节点
    sibling: Fiber | null, // 同级节点
    // fiber的版本池，即记录fiber更新过程，便于恢复
    alternate: Fiber | null, // diff 的变化记录在这个节点上
    ...
}
```

Fiber 节点正式有了 return, child, sibling 这三个属性能让一棵树变成一个链表，从而实现深度优化遍历.只要保留下中断的节点索引，就可以恢复之前的工作进度，保证代码能够断开重连。  
而 stack 结构当递归遍历时，如果打断我们将无法再重新遍历剩下的所有节点，即使我们能记录上次的节点，那我们只能获取父子节点，而兄弟节点就没办法被访问到。

Fiber 的主要工作流程：

1. ReactDOM.render() 引导 React 启动或调用 setState() 的时候开始创建或更新 Fiber 树。
2. 从根节点开始遍历 Fiber Node Tree， 并且构建 WokeInProgress Tree（reconciliation 阶段）。

-   本阶段可以暂停、终止、和重启，会导致 react 相关生命周期重复执行。
-   React 会生成两棵树，一棵是代表当前状态的 current tree，一棵是待更新的 workInProgress tree。
-   遍历 current tree，重用或更新 Fiber Node 到 workInProgress tree，workInProgress tree 完成后会替换 current tree。
-   每更新一个节点，同时生成该节点对应的 Effect List。
-   为每个节点创建更新任务。

3. 将创建的更新任务加入任务队列，等待调度。

-   调度由 scheduler 模块完成，其核心职责是执行回调。
-   scheduler 模块实现了跨平台兼容的 requestIdleCallback。
-   每处理完一个 Fiber Node 的更新，可以中断、挂起，或恢复。

4. 根据 Effect List 更新 DOM （commit 阶段）。

-   React 会遍历 Effect List 将所有变更一次性更新到 DOM 上。
-   这一阶段的工作会导致用户可见的变化。因此该过程不可中断，必须一直执行直到更新完成。

reconciler 阶段可能执行多次，这样也会导致 willXXX 钩子执行多次，违反它们的语义，它们的废弃是不可逆转的。在进入 commit 阶段时，组件多了一个新钩子叫 getSnapshotBeforeUpdate，它与 commit 阶段的钩子一样只执行一次。

废弃：

-   componentWillMount
-   componentWillUpdate
-   componentWillReceiveProps

新增：

-   static getDerivedStateFromProps(props, state)
-   getSnapshotBeforeUpdate(prevProps, prevState)
-   componentDidCatch
-   static getDerivedStateFromError
    > getDerivedStateFromError 是在 reconciliation 阶段触发，所以 getDerivedStateFromError 进行捕获错误后进行组件的状态变更，不允许出现副作用。componentDidCatch 因为在 commit 阶段，因此允许执行副作用。 它应该用于记录错误之类的情况：

## 总结

React Fiber 是 React 框架重大的一次更新，解决了 React 项目严重依赖于手工优化的痛点，通过系统级别的时间调度，实现划时代的性能优化。

参考资源：

-   https://zhuanlan.zhihu.com/p/37095662
-   https://juejin.cn/post/6844904019660537869#heading-36

## Vue 为什么不需要 fiber

Vue 使用 Object.defineProperty（vue@3 迁移到了 Proxy）对数据的设置（setter）和获取（getter）做了劫持，通过 nextTick 实现异步任务地更新。因此 Vue 能准确知道视图模版中哪一块更新了数据，因此能够准确地渲染视图。  
而 React 因为先天的不足——无法精确更新，所以需要 React fiber 把组件渲染工作切片来减轻性能的负担；而 Vue 基于数据劫持，更新粒度很小，没有这个压力；
