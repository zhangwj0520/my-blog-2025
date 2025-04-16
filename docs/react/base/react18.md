---
sidebar_position: 4
---

# React18新特性

18 之前，批处理只限于 React 原生事件内部的更新。

18 中，批处理支持处理的操作范围扩大了：Promise，setTimout，native event handler 等这些非 React 原生事件。

## 1.自动批处理（Automatic Batching）

### 1.1. 什么是批处理

在 React 中，批处理是指将`多个状态更新合并为一次重新渲染的过程`。在 React 18 之前，React 只在 `React 事件处理函数`中进行批处理。例如，在一个按钮的点击事件处理函数中多次更新状态，React 会将这些更新合并，只进行一次重新渲染。

优势:自动批处理`减少了不必要的重新渲染，提高了应用的性能`。特别是在处理复杂的状态更新时，减少重新渲染的次数可以显著提升应用的响应速度和流畅度，为用户带来更好的体验。


### 2. React 18 之前的批处理限制

批处理只限于 React 原生事件内部的更新。

```js
import React, { useState } from 'react';
 
function App() {
    const [count, setCount] = useState(0);
    const [flag, setFlag] = useState(false);

     const handleClick1 = () => {
        setCount(count + 1);
        setFlag(!flag);
        // 在 React 18 之前，这里只会进行一次重新渲染
    };
 
    const handleClick2 = () => {
        setTimeout(() => {
            setCount(count + 1);
            setFlag(!flag);
            // 在 React 18 之前，这里会触发两次重新渲染
        }, 1000);
    };
 
    return (
        <div>
            <button onClick={handleClick1}>Click me</button>
            <p>Count: {count}</p>
            <p>Flag: {flag ? 'True' : 'False'}</p>
        </div>
    );
}
 
export default App;
```

### 3. React 18 的自动批处理

React 18 引入了自动批处理的概念，现在无论在 React 事件处理函数、异步操作（如 setTimeout、Promise 等）还是原生事件处理函数中，React 都会自动进行批处理，将多个状态更新合并为一次重新渲染。

```js
import React, { useState } from 'react';
 
function App() {
    const [count, setCount] = useState(0);
    const [flag, setFlag] = useState(false);
 
    const handleClick = () => {
        setTimeout(() => {
            setCount(count + 1);
            setFlag(!flag);
            // 在 React 18 中，这里只会进行一次重新渲染
        }, 1000);
    };
 
    return (
        <div>
            <button onClick={handleClick}>Click me</button>
            <p>Count: {count}</p>
            <p>Flag: {flag ? 'True' : 'False'}</p>
        </div>
    );
}
 
export default App;
```

### 4. 深入理解自动批处理的机制

react也有相应的更新主要包含

- render函数初始化时,
- 调用setState时

每一次更新都存在优先级，对于有相同优先级的多次更新，只要实际调度第一个更新，而在后续的更新请求中提前返回函数就能实现批处理

如果我们看setState的源码，主要通过`[dispatchSetStateInternal](https://github.com/facebook/react/blob/v19.1.0/packages/react-reconciler/src/ReactFiberHooks.js#L3765)`会发现它们主要做了两件事：

1. 记录一次hook更新（enqueueConcurrentHookUpdate）
   1. 将更新记录到`concurrentQueues`队列中
2. 调度一次react更新（`scheduleUpdateOnFiber`）
   1. `markRootUpdated`标记根节点有一个 pendingLanes，即待处理的更新
   2. 执行 `[ensureRootIsScheduled](https://github.com/facebook/react/blob/v19.1.0/packages/react-reconciler/src/ReactFiberRootScheduler.js#L103)` 把root添加到调度中,`root === lastScheduledRoot`,设置mightHavePendingSyncWork为true
   3. 判断`didScheduleMicrotask`锁,为false时,执行scheduleImmediateRootScheduleTask,同时设置为true;true时跳过(开启批处理调度,确保只有一个任务),调用`scheduleImmediateRootScheduleTask`
   4. scheduleImmediateRootScheduleTask中 在`[queueMicrotask](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/queueMicrotask)` 执行中`processRootScheduleInMicrotask`,保证执行完不会被打断.
   5. processRootScheduleInMicrotask 中调用`scheduleCallback(schedulerPriorityLevel, performWorkOnRootViaSchedulerTask.bind(null, root))`
   6. 开始执行performWorkOnRootViaSchedulerTask,调用`performWorkOnRoot`,->`renderRootSync`->`workLoopSync`->`performUnitOfWork`,开始调用`beginWork`
   7. `beginWork`
   8. todo
   

react 大致流程

1. setState
2. scheduleUpdateOnFiber
3. ensureRootIsScheduled（有部分逻辑判断是否批处理，如需要，提前return）
4. 如果第三步没有提前中断，调度react更新的回调函数performSyncWorkOnRoot或者performConcurrentWorkOnRoot
5. 异步地执行performXXXWorkOnRoot（包含了render阶段）

## 2.并发模式（Concurrent Mode）