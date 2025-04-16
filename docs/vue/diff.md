---
sidebar_position: 1
---

# vue3中的diff算法

Vue3中的Diff算法优化
1
2
3
在Vue3中，Diff算法经历了显著的优化，这些优化主要集中在减少不必要的DOM操作和提高渲染性能上。Vue3的Diff算法优化包括静态标记和PatchFlag的引入，以及使用最长递增子序列（LIS）来优化节点的移动。

静态标记和PatchFlag

Vue3在编译阶段会对模板中的静态内容打上标记，这些标记被称为PatchFlag。在更新过程中，Vue会检查这些标记，只对那些有可能发生变化的部分进行Diff比较。这样，静态内容就可以在首次渲染后被忽略，从而减少了Diff算法的工作量。

例如，对于以下模板：

```vue
<div>
<p>静态内容</p>
<p>{{ dynamicContent }}</p>
</div>
```

Vue3编译器会在动态内容的`<p>`标签上添加PatchFlag，而静态内容则不会被标记。在更新时，只有带有PatchFlag的节点会被检查和更新。

最长递增子序列（LIS）

Vue3的Diff算法还利用了最长递增子序列（LIS）来优化节点的移动。在对比新旧虚拟DOM时，Vue3会尝试找出一组不需要移动的节点，这组节点就是最长递增子序列。通过保留这些节点，Vue可以减少DOM的移动操作，从而减少渲染成本。

在实际操作中，Vue3会先处理新旧虚拟DOM树的前置和后置节点，这些节点通常不需要移动。然后，对于中间可能需要移动的节点，Vue3会计算出最长递增子序列，并根据这个序列来移动剩余的节点。

代码示例

以下是Vue3中Diff算法的一个简化示例：

```js
function diff(oldVNode, newVNode) {
// 静态标记和PatchFlag的处理
//

// 计算最长递增子序列
const lis = calculateLIS(oldVNode.children);
oldVNode.children.forEach((child, index) => {
if (!lis.includes(index)) {
// 如果节点不在最长递增子序列中，需要移动
moveNode(child, newVNode.children[index]);
}
});
}
```
在这个示例中，calculateLIS函数用于计算最长递增子序列，而moveNode函数负责移动那些不在序列中的节点。

总结

Vue3的Diff算法优化通过静态标记和最长递增子序列的使用，显著提高了渲染性能。这些优化使得Vue3在处理大型应用时更加高效，尤其是在列表渲染等场景下，能够减少不必要的DOM操作，提升用户体验。


