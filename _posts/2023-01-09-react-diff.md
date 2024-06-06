---
layout: post
title: React-Diff
date: 2023-01-09
author: shaokang
header-img: img/react.png
catalog: true
tags:
    - React
    - 复习
---

## React Diff 是什么

React 通过引入 Virtual DOM 的概念，极大地避免无效的 Dom 操作，使我们的页面的构建效率提到了极大的提升。
而 diff 算法就是更高效地通过对比新旧 Virtual DOM 来找出真正的 Dom 变化之处。

传统 diff 算法通过循环递归对节点进行依次对比，效率低下，算法复杂度达到 O(n^3)，react 将算法进行一个优化，复杂度降为 O(n)。

## 原理

react 中 diff 算法主要遵循三个层级的策略：

-   tree 层级
-   component 层级
-   element 层级

### tree 层级

DOM 节点跨层级的操作不做优化，只会对相同层级的节点进行比较

![tree 层级](/img/react-tree.png)

只有删除、创建操作，没有移动操作，如下图：
![remove a](/img/react-tree-a.png)

react 发现新树中，R 节点下没有了 A，那么直接删除 A，在 D 节点下创建 A 以及下属节点

由于没做性能优化，所以官方建议少做这样的跨层级操作。

### component 层级

如果是同一个类的组件，则会继续往下 diff 运算; 如果不是一个类的组件，那么直接删除这个组件下的所有子节点，创建新的。

![component 层级](/img/react-component.png)

当 component D 换成了 component G 后，即使两者的结构非常类似，也会将 D 删除再重新创建 G。

这也是合情合理的，因为基本不存在两个不同类但是组件结构相同的情况，如果有，只能证明使用者的代码需要优化，为什么会存在多个相同可复用性的代码。

### element 层级

对于比较同一层级的节点们，每个节点在对应的层级用唯一的 key 作为标识。

提供了 3 种节点操作，分别为 INSERT_MARKUP(插入)、MOVE_EXISTING (移动)和 REMOVE_NODE (删除)

如下场景：
![element 层级](/img/react-element.png)

概念介绍：

-   index： 新集合的遍历下标。
-   oldIndex：当前节点在老集合中的下标。
-   maxIndex：在新集合访问过的节点在老集合中的最大下标。

如果当前节点在新集合中的位置比老集合中的位置靠前的话，是不会影响后续节点操作的，这里这时候被动字节不用动。

操作过程中只比较 oldIndex 和 maxIndex，规则如下：

-   当 oldIndex > maxIndex 时，将 oldIndex 的值赋值给 maxIndex;
-   当 oldIndex = maxIndex 时，不操作;
-   当 oldIndex < maxIndex 时，将当前节点移动到 index 的位置;

diff 过程如下：

-   节点 B：此时 maxIndex=0，oldIndex=1。满足 oldIndex > maxIndex，因此 B 节点不动，此时 maxIndex = Math.max(oldIndex, maxIndex)，就是 1;
-   节点 A：此时 maxIndex=1，oldIndex=0。不满足 oldIndex > maxIndex，因此 A 节点进行移动操作，此时 maxIndex = Math.max(oldIndex, maxIndex)，还是 1；
-   节点 D：此时 maxIndex=1，oldIndex=3。满足 oldIndex > maxIndex，因此 D 节点不动，此时 maxIndex = Math.max(oldIndex, maxIndex)，就是 3；
-   节点 C：此时 maxIndex=3，oldIndex=2。不满足 oldIndex > maxIndex，因此 C 节点进行移动操作，当前已经比较完了。

当 ABCD 节点比较完成后，diff 过程还没完，还会整体遍历老集合中节点，看有没有没用到的节点，有的话，就删除。
