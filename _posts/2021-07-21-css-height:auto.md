---
layout: post
title: css height auto 探秘
date: 2020-07-14
author: shaokang
header-img: img/css.jpg
catalog: true
tags:
    - CSS
---

### 为什么要写这篇博客？

前两天在开发时发现项目中一个 bug: `height: auto 导致动画失效` 问题。
大致的代码如下：

```css
--hidden
    height: 0
    background-color: #fff
    transform: translateY(100%)
    opacity: 0

--visible
    height: auto
    transform: translateY(0)
    opacity: 1
    border-top-width: 1px

--animation
    transition: all .3s ease
```

出现的问题是：显示元素时动画正常，而隐藏元素时动画却失效了。

为什么动画会失效呢？  
其实 css 动画是一个过渡过程，是从一个值到另一个值的过渡，但由于隐藏时 height: auto，浏览器拿不到一个明确的值，不知道该如何变化，于是就直接返回到了初始值，最终看不到过渡的效果。

并且在 [MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Transitions/Using_CSS_transitions) 也说了不建议用在动画过程中使用 auto

> The auto value is often a very complex case. The specification recommends not animating from and to auto. Some user agents, like those based on Gecko, implement this requirement and others, like those based on WebKit, are less strict. Using animations with auto may lead to unpredictable results, depending on the browser and its version, and should be avoided.

### 解决方案

方案 1：使用 max-height/min-height 来过渡

```css
--hidden
    height: 0
    min-height: 0
    background-color: #fff
    transform: translateY(100%)
    opacity: 0

--visible
    height: auto
    // 设置一个最小达到的高度
    min-height: 60px
    transform: translateY(0)
    opacity: 1
    border-top-width: 1px

--animation
    transition: all .3s ease
```

方案 2：使用 height: 100%，该值为一个明确值

```
--hidden
    height: 0
    min-height: 0
    background-color: #fff
    transform: translateY(100%)
    opacity: 0

--visible
    height: 100%
    transform: translateY(0)
    opacity: 1
    border-top-width: 1px

--animation
    transition: all .3s ease
```

解释: 该节点的父元素没有设置高度(默认 auto)，本身设置成 100% 的话(被子元素高度撑开)，会根据其内容自动计算其高度

参考：
[stackoverflow](https://stackoverflow.com/questions/3508605/how-can-i-transition-height-0-to-height-auto-using-css) [w3c](https://github.com/w3c/csswg-drafts/issues/626)

## height: auto 和 height: 100%

height:auto，是指根据块内内容自动调节高度。
height:100%，是指其相对父块高度而定义的高度。

height:auto 是随内容的高度而撑开的，而 height:100% 则是根据父级元素高度决定的(如果父级元素没有设置高度，那么就会被子元素自动撑开)

### height:100% 与绝对定位和非绝对定位

当前元素非绝对定位的情况下，height:100% 是相对于父元素的 content-box 计算的。即高度 = 父元素 content _ 100%
而绝对定位的情况下，是根据离得最近的定位祖先元素的 padding-box 来计算的。即高度 = 父元素 (content + padding) _ 100%

### inline-block 对 height 的影响

先看一个例子：

```
<div class="parent">
    <div class="child" style="display: inline-block;"></div>
</div>
```

猜一下 parent 会有高度吗？  
答案是有，那为什么 child 没有设置高度，parent 却有高度呢？

可以看到[w3c](https://www.w3.org/TR/CSS22/visudet.html#strut)上其实是有说明的：

> 1.  The height of each inline-level box in the line box is calculated. For replaced elements, inline-block elements, and inline-table elements, this is the height of their margin box; for inline boxes, this is their 'line-height'. (See "Calculating heights and margins" and the height of inline boxes in "Leading and half-leading".)
> 2.  The inline-level boxes are aligned vertically according to their 'vertical-align' property. In case they are aligned 'top' or 'bottom', they must be aligned so as to minimize the line box height. If such boxes are tall enough, there are multiple solutions and CSS 2.2 does not define the position of the line box's baseline (i.e., the position of the strut, see below).
> 3.  The line box height is the distance between the uppermost box top and the lowermost box bottom. (This includes the strut, as explained under 'line-height' below.)
>     意思是说 inline-block 默认放在 baseline 基线上，其高度是由 line-height 和 font-size 属性决定的.

如何解决(parent 高度保持 0px)：parent 设置 line-height: 0 / font-size: 0。

又一个例子

```
<div class="parent">
    <div class="child" style="display: inline-block; height: 40px">
        aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
        bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
        cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc
    </div>
</div>
```

这个时候 parent 的高度是多少？
答案是比 40px 大，还是上面说的到，inline-block 默认放在 baseline 基线，所以高度会比 child 设置的高度要大。

如何解决(parent 高度仍保持 40px)：child 设置 vertical-align: top。如果对 parent 设置 line-height: 0 / font-size: 0 会对文字产生影响
