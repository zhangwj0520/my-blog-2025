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

## 2.过渡更新

`过渡（transition）更新`是 React 中一个新的概念，用于区分紧急和非紧急的更新。

- `紧急更新` 对应直接的交互，如输入，点击，按压等。
- `过渡更新` 将 UI 从一个视图过渡到另一个

并发渲染中将会加入过渡更新，允许更新被中断。如果更新内容被重新挂起，过渡机制也会告诉 React 在后台渲染过渡内容时继续展示当前内容

- useTransition： 一个用于开启过渡更新的 Hook，用于跟踪待定转场状态。
- startTransition： 当 Hook 不能使用时，用于开启过渡的方法。

### 并发特性(Concurrent React)

React 18 添加了期待已久的并发渲染器和对 Suspense 的更新。应用程序可以升级到 React 18 并开始逐步采用并发功能.
`这意味着没有并发模式，只有并发功能`


并发模式的一个关键特性是`渲染可中断`。当首次升级到 React 18，在加入任何并发功能之前，更新内容渲染的方式和 React 之前的版本一样——通过一个单一的且不可中断的同步事务进行处理。同步渲染意味着，一旦开始渲染就无法中断，直到用户可以在屏幕上看到渲染结果。

在并发渲染中，情况并不总是如此。React 可能开始渲染一个更新，然后中途挂起，稍后又继续。它甚至可能完全放弃一个正在进行的渲染。React 保证即使渲染被中断，UI 也会保持一致。为了实现这一点，它会在整个 DOM 树被计算完毕前一直等待，完毕后再执行 DOM 变更。这样做，React 就可以在后台提前准备新的屏幕内容，而不阻塞主线程。这意味着用户输入可以被立即响应，即使存在大量渲染任务，也能有流畅的用户体验。

另一个例子是`可重用状态`。并发 React 可以从屏幕中移除部分 UI，然后在稍后将它们再添加回来，并重用之前的状态。例如，当用户来回切换标签页，React 应该能够立即将屏幕恢复到它先前的状态。在即将到来的次要版本中，我们计划添加一个新的名为 <Offscreen> 的组件，它实现了这种模式。同样地，你将能够使用 Offscreen 在后台准备新的 UI，在显示前就准备完毕以便快速响应。

并发渲染是一个 React 中非常强大的工具，并且我们大多数新功能都是利用了它的优势来创建的，包括 Suspense，transition 和流式服务端渲染。但是在并发渲染这个方向，React 18 也仅仅只是实现我们最终目标的第一步。

### 1.React18支持并发特性的三个API

- startTransition()
- useDeferredValue()
- useTransition()

#### 1.startTransition

startTransition 可以让你在后台渲染 UI 的一部分。

startTransition的作用就是：被startTransition包裹的setState触发的渲染被标记为不紧急渲染，意味着它们可以被其他紧急渲染所抢占，这种渲染优先级的调整手段可以帮助我们解决各种性能伪瓶颈，提升用户体验。

- 只有当你能访问某个 state 的 set 函数时，你才能将它的更新包裹到 Transition 中。
- 一个被标记为 Transition 的 state 更新时将会被其他 state 更新打断
- 传递给 startTransition 的函数会立即被调用，并将其执行时发生的所有状态更新标记为 Transitions。 `setTimeout 中进行状态更新不会被标记为 Transitions`
- 有多个正在进行的 transition，目前 React 会将它们集中在一起处理

```js
import { startTransition } from 'react';
// 紧急更新
setInputValue(input)
// 标记回调函数内的更新为 非紧急更新
startTransition(() => {
  setSearchQuery(input)
})
```

#### 2.useDeferredValue

useDeferredValue 是一个 React Hook，可以让你延迟更新 UI 的某些部分.
在组件的顶层调用 useDeferredValue 来延迟更新 UI 的某些部分。

将 useDeferredValue 作为性能优化的手段。当你的 UI 某个部分重新渲染很慢,你有一个文本框和一个组件（例如图表或长列表），在每次按键时都会重新渲染

```js
import { useState, useDeferredValue } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  // ...
}
```

#### 3.useTransition

```js
function TabContainer() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState('about');

  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab);
    });
  }
  // ……
}
```

## 3. Suspense 特性

Suspense 允许你声明式地为一部分还没有准备好被展示的组件树指定加载状态：

```js
<Suspense fallback={<Spinner />}>
  <Comments />
</Suspense>
```

支持了服务端 Suspense,并且使用并发渲染特性扩展了其功能


## 4.新的客户端和服务端渲染 APIs (New Render API)

```js
import ReactDOM from 'react-dom/client';
ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

## 5.新的 Hook

1. useId 
   用于生成在客户端和服务端两侧都独一无二的 id
2. useTransition 过度更新
3. useDeferredValue 允许推迟渲染树的非紧急更新
4. useSyncExternalStore 是一个让你订阅外部 store 的 React Hook
5. useInsertionEffec


[面试题](https://juejin.cn/post/7348651815759282226#heading-12)