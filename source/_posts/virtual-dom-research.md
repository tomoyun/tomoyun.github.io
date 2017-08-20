---
title: Web技术研究 01 - Virtual Dom
date: 2017-08-13 13:44:09
categories:
- Web技术研究
tags:
- Web
- js
---

最近工作中需要一直和Web前端打交道，对于之前完全没有Web相关经验的我来说，真是一片空白。每天的工作中都会接触不少的新技术、新名词，无奈半吊子属性太过明显，很多知识都是一知半解，也常常会遇到自己难以解决的问题，需要麻烦同事帮忙。为了改变现状，提升自己的Web实力，搞清那剩下的一半，决定开始进行Web技术研究，总结知识的同时改变自己。Virtual Dom 就是其中最常被提起的名词之一，首先就来对它进行一番研究。

## 源起
长久以来，Web前端工作可以认为是维护状态和更新页面，其大致的流程可以理解为：
用户Action => 维护状态Model => 操作更新Dom => 渲染浏览器
用户打开网页，初始化状态Model，构建Dom，浏览器渲染页面得以展示在用户面前。而后用户操作页面内容(点击、输入等等)，触发事件回调，改变Model状态，继而操作Dom更新页面，浏览器也重新渲染，用户在电脑前看到了更新后的页面内容。如此不断循环，实现用户与网站之间的信息交互。

乍一看一切都很美好。事实上在PC端，浏览器性能已足够强大，过去大部分网站似乎确实没啥问题。但是在移动端，浏览器性能还十分受限，Web App应用的复杂性却是一点不输PC端，甚至由于手势操作还要更加复杂。这一模式的问题也暴露出来，由于频繁操作Dom，导致页面性能不够用了。 这也是过去Web App以及复杂网页经常被用户说卡的主要原因。Virtual Dom正是针对这一情况所进行的优化方案。

### Dom为什么慢
懂一点Web前端的人都会说"Dom很慢"，"操作Dom是一件成本很高的事情"，然而进一步追问"Dom操作为什么慢"，却有一大半的人支支吾吾，说不清楚。

首先，Dom是复杂的。"那为什么Dom是复杂的呢？" 真正的Dom元素非常庞大。可以调用操作Dom的Api创建一个空的Dom元素，并将其属性打印出来, 从打印结果就不难发现Dom元素远比我们想象的要庞大复杂的多。另一方面，Dom元素庞大，导致操作Dom结构的方法实现也很复杂。

其次，操作Dom会触发浏览器渲染页面，重新定位元素的位置。Dom结构复杂，浏览器解析Dom结构，渲染页面的过程势必更加的复杂，更加消耗性能。频繁操作Dom，很可能导致页面多次重新渲染，那可就是杀死性能的罪魁祸首。

## 核心思想
既然知道Dom很慢，那在Web中，什么比较快呢？js! js对象本身和页面并没有什么关系，操作js对象就好像操作一般的内存对象，处理起来更快，也更简单。***在计算机领域，在操作慢介质之前利用快介质做缓冲，是提升性能的常用手段。*** 那么，是否可以利用较快的js对象来对Dom进行缓冲呢？Vitual Dom正是基于这个想法来实现的。

***在状态Model和真实Dom之间建立一层缓冲，它是真实Dom的一种抽象表示，可以根据它来构建真实的Dom。当状态Model数据改变时，首先更新缓冲，对比前后差异，计算出最小变更，然后再将变更应用到真实Dom。这个缓冲就是Virtual Dom。***

这里关键的地方有2点：
1. Virtual Dom是js对真实Dom的一种表达。在每次数据发生变化时，不需要关心是数据的那部分发生了变化，只要重新整体生成一次Virtual Dom即可。这样的简单性有助于开发。
2. Virtual Dom重新生成之后，并不会马上构建并替换Dom。新的Virtual Dom树会对比之前得到的旧Virtual Dom树，计算得到需要在Dom上进行的最小改变。最小化的Dom操作保证了页面的执行效率。
---

{% asset_img diagram.png Virtual Dom示意图 %}

初始渲染时，首先由状态Model生成Virtual Dom，然后由Virtual Dom构建真实Dom。

{% asset_img diagram_update.png 更新virtual dom %}

状态Modle更新时，重新生成新的Virtual Dom，与上一次得到的Virtual Dom进行对比Diff，得到需要在Dom上进行的最小化操作，然后在patch过程中应用到真实Dom上实现页面更新。

## virtual dom实现

Vue2.x、React中的使用的Virtual Dom算法比较复杂，存在较多细节。本文并不对它们的源码进行分析，而是从原理上探讨Virtual Dom的实现。

Virtual Dom的实现大致可分为一下几个阶段:  
{% asset_img phase.jpg Virtual Dom阶段 %}

### render - 创建虚拟节点
Dom节点用js来表示非常简单，只需要记录它的节点类型、属性、子节点。
``` javascript
function h(tagName, props, ...children) {
    return { tagName, props, children }
}
```
假设我们有这样一颗Dom树: 
``` html
<ul id='list'>
  <li class='item'>Item 1</li>
  <li class='item'>Item 2</li>
  <li class='item'>Item 3</li>
</ul>
```
可以将这颗Dom树表示为如下javascript代码: 
``` javascript
let vTree = h('ul', {id: 'list'}, [
    h('li', {class:'item'}, ['Item 1']),
    h('li', {class:'item'}, ['Item 2']),
    h('li', {class:'item'}, ['Item 3'])
])
```

### createElement - 创建Dom节点
初始化的时候，我们需要根据Virtual Dom树构建真是的Dom树。  
下面的createElement函数，正是用来负责完成这一转化过程：
``` javascript
function createElement(node) {
    if(typeof node === 'string') {
        return document.createTextNode(node);
    }

    let el = document.createElement(node.tagName);

    for (let propName in node.props) { // 设置节点的DOM属性
        let propValue = node.props[propName];
        el.setAttribute(propName, propValue);
    }

    let children = node.children || [];
    children.map(createElement)
        .forEach(el.appendChild.bind(el));

    return el;
}
```

根据Virtual Node创建真实的Dom节点，并把它插入到文档中。
``` javascript
let root = createElement(vTree);
document.body.appendChild(root);
```

### diff - 新旧VTree差异对比
diff算法是Virtual Dom最核心的部分。完全比较两颗树差异的算法是一个复杂度O(n^3)的问题，代价非常高。考虑到在前端中很少会跨级的移动Dom元素，diff算法只会在同层级进行比较，不会进行跨级比较，这样算法的复杂度可以降低到O(n)。

{% asset_img diff.png diff 分层对比 %}

``` javascript
function diff(oldTree, newTree) {
    let index = 0;
    let patches = {};
    dfsWalk(oldTree, newTree, index, patches);

    return patches;
}

function dfsWalk(oldNode, newNode, index, patches) {
    let apply = [{difference} ...]

    //遍历子节点
    diffChildren(oldNode.children, newNode.children, index, patches, apply);

    patches[index] = apply;
}

function diffChildren(oldChildren, newChildren, index, patches, apply) {
    ...

    let orderedSet = reorder(oldChildren, newChildren);
    newChildren = orderedSet.children;

    let oLen = oldChildren.length;
    let nLen = newChildren.length;
    let len = oLen > nLen ? oLen : nLen

    for(let i=0; i<len; i++) {
        let leftNode = oldChildren[i];
        let rightNode = newChildren[i];
        index++;

        ...
        if(leftNode) {
            dfsWalk(leftNode, rightNode, index, patches);

            index += leftNode.count;
        }

        ...
    }
}
```

diff算法对所有的子节点进行递归遍历，每个节点都有一个对应的编号(index)。它将每个子节点与新的树进行对比，如果存在变化，就将差异记录在patches[index]中。  
diff得到的patches数据结构是这样的:
``` javascript
let pathches = {
    0: [{type: REPALCE, node: newNode}, {type: PROPS, props: {id: 'id'}}],
    1: [{type: REORDER, moves: changObjs}],
    ...
}
```

diffChildren代码中的reorder方法特别值得关注。在子节点重新排序的情况下，如果按照同层级顺序对比的话，所有的子节点都将被替换掉，这样Dom开销就很大。reorder方法就是用来处理这个问题的，实际上不需要替换节点，只用移动节点就可以完成。reorder的具体实现比较复杂，可以理解为是最小编辑距离问题，通过动态规划求解，这里不再展开。

此外，列表对比中tagName是可重复的, 不能用它来进行对比，所以需要给子节点加上唯一标识key。这样在列表对比的时候，使用key进行比较，才能复用老的Dom树的节点。


### patch - 补丁更新Dom节点
通过diff差异对比，得到了patches对象。接下来，只要遍历patches对象，根据不同类型的差异操作对应的Dom节点。

``` javascript
function applyPatches(domNode, currentPatches) {
    currentPatches.forEach(function (currentPatch) {
        switch (currentPatch.type) {
            case REPLACE:
                replaceNode(domNode, currentPatch.node);
            break;
            case REORDER:
                reorderChildren(domNode, currentPatch.moves);
            break;
            ...
        }
    }
}

```

## 后续
上述内容还只是对Virtual Dom的一个粗略研究，Virtual Dom算法中还包括许多细节，如Hooks(注册事件)、Thunk(开发者定制diff过程)等。

学习不止，记录不断，本文记录于2017年8月20日

