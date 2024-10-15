---
layout: post
title: react requestIdleCallback
date: 2024-09-19
author: shaokang
header-img:
catalog: true
tags:
    - react
---

## 前言

React 为优化页面性能，会在浏览器空闲时巧妙地执行代码，这其中涉及到了 requestIdleCallback API 的运用。本文将深入探讨 requestIdleCallback API 的工作原理，并剖析 React 是如何模拟实现这一 API 的。

## requestIdleCallback API 详解

requestIdleCallback 是一个允许开发者在浏览器空闲时段执行非紧急、低优先级任务的 API。通过这一机制，我们能够确保关键事件的响应不受影响，同时有效利用浏览器的空闲资源。更多关于此 API 的信息，可参阅 [MDN 官方文档](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback)。

以下是 requestIdleCallback 的基本使用示例：

```js
requestIdleCallback(
    deadline => {
        console.log(deadline.timeRemaining()); // 输出当前闲置周期的预估剩余毫秒数
    },
    { timeout: 1000 } // 设置超时时间，单位毫秒
);
```

使用方法相当直观，回调函数中提供的 deadline 对象包含了 timeRemaining() 方法，该方法返回当前闲置周期内预估的剩余时间（以毫秒为单位）。我们可以根据这个剩余时间来判断是否有足够的时间来完成我们的任务。

接下来，让我们通过 MDN 上的一个例子来进一步了解这个 API 的实际应用：

```JS
function runTaskQueue(deadline) {
  while (
    (deadline.timeRemaining() > 0 || deadline.didTimeout) &&
    taskList.length
  ) {
    const task = taskList.shift();
    currentTaskNumber++;

    task.handler(task.data);
    scheduleStatusRefresh();
  }

  if (taskList.length) {
    taskHandle = requestIdleCallback(runTaskQueue, { timeout: 1000 });
  } else {
    taskHandle = 0;
  }
}
```

## 兼容性问题

requestIdleCallback API 的[兼容性](https://caniuse.com/?search=requestIdleCallback)并不理想，在某些浏览器中可能无法使用。需要借助 polyfill 来兼容不支持该 API 的浏览器。

以下是一个来自 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Background_Tasks_API#%E5%9B%9E%E9%80%80%E5%88%B0_settimeout) 的 polyfill 实现示例：

```js
window.requestIdleCallback =
    window.requestIdleCallback ||
    function (handler) {
        let startTime = Date.now();

        return setTimeout(function () {
            handler({
                didTimeout: false,
                timeRemaining: function () {
                    return Math.max(0, 50.0 - (Date.now() - startTime));
                },
            });
        }, 1);
    };
```

## React 中的 requestIdleCallback 模拟实现

React 并未直接采用 requestIdleCallback API，主要原因有两点。其一，该 API 的兼容性存在问题，并非所有浏览器都支持；其二，该 API 本身具有一定的局限性，其每秒最多仅能执行 20 次，这样的频率无法满足 React 对性能优化的高要求，特别是在渲染方面。因此，React 选择了其他更适合其性能需求的方案。

> requestIdleCallback is called only 20 times per second - Chrome on my 6x2 core Linux machine, it's not really useful for UI work.
> requestAnimationFrame is called more often, but specific for the task which name suggests.
> —— from [Releasing Suspense](https://github.com/facebook/react/issues/13206#issuecomment-418923831)

### React 16 实现（rAF + postMessage）

在 [React 16.0.0](https://github.com/facebook/react/blob/v16.0.0/src/renderers/shared/ReactDOMFrameScheduling.js) 版本中采用 rAF + postMessage 实现 requestIdleCallback 的模拟。

```js
var scheduledRICCallback = null;
var frameDeadline = 0;
// 假设 30fps，一秒就是 33ms
var activeFrameTime = 33;

var frameDeadlineObject = {
    timeRemaining: function () {
        return frameDeadline - performance.now();
    },
};

var idleTick = function (event) {
    scheduledRICCallback(frameDeadlineObject);
};

window.addEventListener('message', idleTick, false);

var animationTick = function (rafTime) {
    frameDeadline = rafTime + activeFrameTime;
    window.postMessage('__reactIdleCallback$1', '*');
};

var rIC = function (callback) {
    scheduledRICCallback = callback;
    requestAnimationFrame(animationTick);
    return 0;
};
```

`rIC` 函数实际会调用 `requestAnimationFrame(animationTick)`，其中 `animationTick` 会接收当前帧的执行时间 rafTime 作为参数。为了确保至少维持 30fps 的帧率，这里设定每帧的时长为 33ms。基于此，我们可以计算出这一帧应在 `frameDeadline = rafTime + activeFrameTime` 之前完成。

随后，通过 `postMessage` 进行通信，利用 frameDeadline - performance.now() 来计算出当前帧的剩余时间，并将包含这一信息的对象（frameDeadlineObject）传递给 callback 函数。通过这种方式，成功模拟了 requestIdleCallback 的功能。

为了确保在浏览器处理动画和用户输入时不受阻碍，这里并未直接执行 callback 回调，而是使用 postMessage（宏任务）将其推送到任务队列中。这样，浏览器就能优先处理更高优先级的动画或用户输入，待这些任务完成后，再执行推送的任务。

> 为什么不用 setTimeout 呢？
> setTimeout 也可以实现类似的效果。然而，当你设置 setTimeout(fn, 0) 时，尽管意图是立即执行，但浏览器在实现时会引入至少 4ms 的延迟。如果你想在浏览器中实现真正意义上的 0ms 延时定时器，应使用 postMessage 方法。

### 取消 rAF

在 React 的 v16.2.0 版本中，团队决定弃用 requestAnimationFrame，并转而采用 message event loop。这一改动可以在相关的 [Pull Request](https://github.com/facebook/react/pull/16214) 中查看详细信息。

> Adds experimental flag to yield many times per frame using a message event loop, instead of the current approach of guessing the next vsync and yielding at the end of the frame.
> This new approach forgoes a requestAnimationFrame entirely. It posts a message event and performs a small amount of work (5ms) before yielding to the browser, regardless of where it might be in the vsync cycle. At the end of the event, if there's work left over, it posts another message event.
> This should keep the main thread responsive even for really high frame rates. It also shouldn't matter if the hardware frame rate changes after page load (our current heuristic only detects if the frame rate increases, not decreases).
> The main risk is that yielding more often will exacerbate main thread contention with other browser tasks.
> I'm also not sure to what extent message events are throttled when the tab is backgrounded, relative to requestAnimationFrame or setTimeout. I'm starting with the assumption that message events fire with at least the same priority as timers, but I'll need to confirm.

### Message Channel

MessageChannel 接口允许我们创建一个新的消息通道，并通过它的两个 MessagePort 属性发送数据。

```js
var channel = new MessageChannel();

channel.port1.onmessage = () => {
    /* 处理接收到的消息 */
};

channel.port2.postMessage(null);
```

关于为何选择使用 messageChannel：

1. `宏任务的优先性`：宏任务能够及时让出主线程，确保页面的流畅性。相比之下，微任务虽然执行迅速，但它们并不会让出主线程，而是在更新页面前执行，这可能导致页面响应的延迟。

2. `宏任务中的最佳选择`：在众多的宏任务中，如 setTimeout、requestAnimationFrame、MessageChannel。setTimeout 存在一个最小延迟问题，通常会浪费大约 4ms 的时间。而 requestAnimationFrame 的触发时间则受到浏览器页面更新机制的影响，因此其稳定性无法保证。综合考量，MessageChannel 是最理想的选择，它既能够及时让出主线程，又能够确保消息的稳定传递。

### React 18 中 message event loop 的实现

在 [React v18.3.1](https://github.com/facebook/react/blob/v18.3.1/packages/scheduler/src/forks/Scheduler.js) 的版本中，实现方式如下：

```js
// 代码片段
let scheduledHostCallback;
let isMessageLoopRunning = false;
let getCurrentTime = () => performance.now();
let startTime;

function requestHostCallback(callback) {
    scheduledHostCallback = callback;
    if (!isMessageLoopRunning) {
        isMessageLoopRunning = true;
        schedulePerformWorkUntilDeadline();
    }
}

const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;

let schedulePerformWorkUntilDeadline = () => {
    port.postMessage(null);
};

function performWorkUntilDeadline() {
    if (scheduledHostCallback !== null) {
        const currentTime = getCurrentTime();
        startTime = currentTime;
        const hasTimeRemaining = true;

        let hasMoreWork = true;
        try {
            hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
        } finally {
            if (hasMoreWork) {
                schedulePerformWorkUntilDeadline();
            } else {
                isMessageLoopRunning = false;
                scheduledHostCallback = null;
            }
        }
    } else {
        isMessageLoopRunning = false;
    }
}
```

在当前的实现中，可以观察到原先的 `rIC` 已经更名为 `requestHostCallback`。当 `requestHostCallback` 函数被调用时，它会进一步触发 `schedulePerformWorkUntilDeadline` 函数的执行。值得注意的是，`schedulePerformWorkUntilDeadline` 的核心操作是运用 `postMessage` 方法，这一举动的目的在于通知浏览器：在完成了当前的所有紧急任务后，请执行 `performWorkUntilDeadline` 函数。

在 `performWorkUntilDeadline` 的执行过程中，它会调用 `scheduledHostCallback` 函数，并传入两个参数：`hasTimeRemaining `和 `currentTime`。这里的 `scheduledHostCallback` 实际上就是传递给 `requestHostCallback` 的回调函数。此回调函数会返回一个布尔值，用以告知 `performWorkUntilDeadline` 是否仍有待处理的任务。如果存在剩余任务，`performWorkUntilDeadline` 将会再次调用 `schedulePerformWorkUntilDeadline`，从而向浏览器传达继续工作的指令。这一机制确保了任务能够以小批量的方式执行，不断释放线程资源，进而保证浏览器能够及时响应动画或用户输入，维持流畅的用户体验。

接下来我们看一下 `requestHostCallback` 回调函数在做什么事情。

```js
// 传入的回调
requestHostCallback(flushWork);

function flushWork(hasTimeRemaining, initialTime) {
    return workLoop(hasTimeRemaining, initialTime);
}

var currentTask;

function workLoop(hasTimeRemaining, initialTime) {
    currentTask = taskQueue[0];
    while (currentTask != null) {
        if (
            currentTask.expirationTime > initialTime &&
            (!hasTimeRemaining || shouldYieldToHost())
        ) {
            break;
        }

        const callback = currentTask.callback;
        callback();

        taskQueue.shift();

        currentTask = taskQueue[0];
    }
    if (currentTask !== null) {
        return true;
    } else {
        return false;
    }
}

// 默认的时间切片
const frameInterval = 5;

function shouldYieldToHost() {
    const timeElapsed = getCurrentTime() - startTime;
    if (timeElapsed < frameInterval) {
        return false;
    }

    return true;
}
```

在 `flushWork` 函数中，调用了 `workLoop` 函数，该函数的核心职责是从任务列表中取出首个任务，并对其是否过期以及是否应该让出线程给宿主环境进行判断，这一判断通过 `shouldYieldToHost` 函数实现。

`shouldYieldToHost` 函数内部通过比较 `getCurrentTime()`（当前时间）和 `startTime`（接收到 `postMessage` 消息并开始执行 `performWorkUntilDeadline` 函数的时间）来决定是否让出线程。值得注意的是，从 `performWorkUntilDeadline` 到 `shouldYieldToHost` 是一系列同步操作，因此在处理第一个任务时，`getCurrentTime()` 和 `startTime` 之间的时间差确实很小。

真正的耗时操作在于执行任务队列中的任务，即 `const callback = currentTask.callback; callback()`。每完成一个任务后，都会再次调用 `shouldYieldToHost` 进行判断。如果自开始以来尚未超过 5 毫秒（这是默认的时间切片阈值），将继续执行任务，并在每次任务后重复此判断，直至任务列表为空或时间超过 5 毫秒。

一旦超过 5 毫秒且仍有未完成的任务（`hasMoreWork` 为 `true`），我们将执行 `schedulePerformWorkUntilDeadline`，这实质上是通过 `port.postMessage(null)` 来让出线程。这一举措允许浏览器有机会处理动画和用户输入。当浏览器完成这些处理后，它会通过 `port1.onmessage` 事件回调继续执行 `performWorkUntilDeadline`，从而继续遍历并执行任务列表中的剩余任务。这种每执行 5 毫秒任务就让出线程的模式将持续进行，直至所有任务均被完成。

## 参考

https://juejin.cn/post/7166547963517337614

## 总结

React `requestHostCallback` 的实现方式是利用 `MessageChannel` API 实现异步任务调度。它通过时间分片的方式每执行 5 毫秒任务就让出线程的方式，保证了任务能够以小批量的方式执行，不断释放线程资源，进而保证浏览器能够及时响应动画或用户输入，维持流畅的用户体验。



