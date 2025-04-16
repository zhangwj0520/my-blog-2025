---
sidebar_position: 5
---

# React 19新特性

## Actions

使用异步过渡的函数被称为 “Actions”

## 新的 Hook: useActionState

为了使 Actions 的常见情况更加简单，我们添加了一个名为 useActionState 的新 Hook：

```js
const [error, submitAction, isPending] = useActionState(
  async (previousState, newName) => {
    const error = await updateName(newName);
    if (error) {
      // 你可以返回操作的任何结果。
      // 这里，我们只返回错误。
      return error;
    }

    // 处理成功的情况。
    return null;
  },
  null,
);
```
useActionState 接受一个函数（“Action”），并返回一个被包装的用于调用的 Action。这是因为 Actions 是可以组合的。当调用被包装的 Action 时，useActionState 将返回 Action 的最后结果作为 data，以及 Action 的待定状态作为 pending。

## React DOM: <form> Actions

```js
<form action={actionFunction}>
```

## React DOM: 新 Hook: useFormStatus

```js
const { pending, data, method, action } = useFormStatus();
```

```js
import { useFormStatus } from "react-dom";
import action from './actions';

function Submit() {
  const status = useFormStatus();
  return <button disabled={status.pending}>提交</button>
}

export default function App() {
  return (
    <form action={action}>
      <Submit />
    </form>
  );
}
```

## 新 Hook: useOptimistic

useOptimistic 是一个 React Hook，它可以帮助你更乐观地更新用户界面
立即向用户呈现执行操作的结果，即使实际上操作需要一些时间来完成。

```js
import { useOptimistic } from 'react';

function AppContainer() {
  const [optimisticState, addOptimistic] = useOptimistic(
    state,
    // 更新函数
    (currentState, optimisticValue) => {
      // 使用乐观值
      // 合并并返回新 state
    }
  );
}
```

## 新的 API: use

在 React 19 中，我们引入了一个新的 API 来在渲染中读取资源：use。

例如，你可以使用 use 读取一个 promise，React 将挂起，直到 promise 解析完成

```js
import {use} from 'react';

function Comments({commentsPromise}) {
  // `use` 将被暂停直到 promise 被解决.
  const comments = use(commentsPromise);
  return comments.map(comment => <p key={comment.id}>{comment}</p>);
}

function Page({commentsPromise}) {
  // 当“use”在注释中暂停时,
  // 将显示此悬念边界。
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  )
}
```

## ref作为一个属性

## <Context> 作为提供者 
你可以将 <Context> 渲染为提供者，就无需再使用 <Context.Provider> 了

## refs 支持清理函数

```js
<input
  ref={(ref) => {
    // ref 创建

    // 新特性: 当元素从 DOM 中被移除时
    // 返回一个清理函数来重置 ref
    return () => {
      // ref cleanup
    };
  }}
/>
```

## useDeferredValue 初始化 value

## 支持文档元数据

当 React 渲染这个组件时，它会看到 <title>、<link> 和 <meta> 标签，并自动将它们提升到文档的 <head> 部分。通过原生支持这些元数据标签，我们能够确保它们与仅客户端应用、流式 SSR 和服务器组件一起工作。

## 支持样式表

样式表的 precedence，它将管理样式表在 DOM 中的插入顺序，并确保在显示依赖于这些样式规则的内容之前加载样式表（如果是外部的）。

```js
function ComponentOne() {
  return (
    <Suspense fallback="loading...">
      <link rel="stylesheet" href="foo" precedence="default" />
      <link rel="stylesheet" href="bar" precedence="high" />
      <article class="foo-class bar-class">
        {...}
      </article>
    </Suspense>
  )
}

function ComponentTwo() {
  return (
    <div>
      <p>{...}</p>
      <link rel="stylesheet" href="baz" precedence="default" />  <-- will be inserted between foo & bar
    </div>
  )
}
```

## 支持异步脚本

通过允许你在组件树的任何位置，即实际依赖脚本的组件内部，渲染它们，从而为异步脚本提供了更好的支持，无需管理脚本实例的重新定位和去重。

```js
function MyComponent() {
  return (
    <div>
      <script async={true} src="..." />
      Hello World
    </div>
  )
}

function App() {
  <html>
    <body>
      <MyComponent>
      ...
      <MyComponent> // won't lead to duplicate script in the DOM
    </body>
  </html>
}
```

## 支持预加载资源 

在初始文档加载和客户端更新时，尽早告诉浏览器它可能需要加载的资源，可以显著提高页面性能
React 19 包含了一些新的 API，用于加载和预加载浏览器资源，使得构建不受资源加载效率影响的优秀体验变得尽可能容易。
```js
import { prefetchDNS, preconnect, preload, preinit } from 'react-dom'
function MyComponent() {
  preinit('https://.../path/to/some/script.js', {as: 'script' }) // loads and executes this script eagerly
  preload('https://.../path/to/font.woff', { as: 'font' }) // preloads this font
  preload('https://.../path/to/stylesheet.css', { as: 'style' }) // preloads this stylesheet
  prefetchDNS('https://...') // when you may not actually request anything from this host
  preconnect('https://...') // when you will request something but aren't sure what
}
```

```js
<!-- the above would result in the following DOM/HTML -->
<html>
  <head>
    <!-- links/scripts are prioritized by their utility to early loading, not call order -->
    <link rel="prefetch-dns" href="https://...">
    <link rel="preconnect" href="https://...">
    <link rel="preload" as="font" href="https://.../path/to/font.woff">
    <link rel="preload" as="style" href="https://.../path/to/stylesheet.css">
    <script async="" src="https://.../path/to/some/script.js"></script>
  </head>
  <body>
    ...
  </body>
</html>
```

## 支持自定义元素
