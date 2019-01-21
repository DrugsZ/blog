### 						虚拟DOM探索

​	*前言：虚拟DOM的深入学习是我在学习Vue源码过程中进行的（虽然只是学了皮毛）；但是在此过程中我发现其虚拟DOM实现借鉴于`snabbdom`,所以大部分借鉴于vue,及`snabbdom`，但同时存在些许异同*

​	在说虚拟DOM之前，思考一下Jquery主要是用来干啥的，讲道理我并没有深入用过Jquery，主要得益于现代HTML，及Js的发展，使的大部分Jquery功能是完全可以用原生来实现的，但是Jquery在当时是划时代的出现，他抹平了各浏览器的差异，手动操作DOM更加快速，方便。

​	随着前端应用越来越复杂，手动操作DOM也越来越复杂，而且总的来说，整个过程还是从后端请求到数据后，根据数据决定如何去渲染操作DOM，那么能不能通过函数直接将数据渲染为DOM，而我们只需要关心数据，这就是Vue/React这类框架的思想,创建数据到UI的映射函数来实现只需要关心数据,自动生成UI,这个其实也就是早就已经有的模板引擎的功能,那么React/Vue跟早就已有的模板引擎有什么区别呢?

首先来说,我们简单的想一下,当我们要手动创建一个DOM节点时需要什么?

```
tag,attr,style,class,event,children
```

这样的话我们可以简单的通过一个Javascript对象来表示将要创建的DOM节点

```js
const willCreate = {
    tag:'div',
    attr:{},
    style:{},
    className:{},
    event:{},
    children:[]
}

const createElm = (vnode) => {
    if(!vnode) return;
    const {tag,attr,style,className,children} = vnode
    const elm = document.createElement(tag);
    applyAttr(elm,attr);
    applyStyle(elm,style);
    applyClass(elm,className);
    applyEvent(elm,event);
    let len = children.length
    if(len){
        while(len--){
            elm.appendChild(createElm(children[len]));
        }
    }
    return elm;
}
```

通过上面的`createElm`函数我们可以将一个JS对象树转为真实的DOM树,那么接下来我们要变更数据,然后将变更体现到真实的UI上,最简单的方式就是清空之前的所有DOM重新调用`createElm`进行渲染,如果如此做的话那么首先会有很严重的性能问题,重排重绘,JS与DOM通信都是耗时耗性能的,所以这个时候就需要`diff`算法了,很多文章再说到虚拟DOM的时候都说通过JS模拟DOM结构所以性能好,我觉得其实还是有一些不严谨的,应该是`diff`算法使得其避免了很多不需要的DOM操作,那么`diff`算法到底是如何?

其实DOM的更改是极少会出现跨层级的移动的，所以如果我们完全忽略这种情况，只去对比同层的树节点，就可以节省大量的时间，使得时间复杂度只有O(n)

![](https://raw.githubusercontent.com/DrugsZ/blog/master/%E8%99%9A%E6%8B%9FDOM%E4%B8%8Ediff/images/beforediff.png)

![](https://raw.githubusercontent.com/DrugsZ/blog/master/%E8%99%9A%E6%8B%9FDOM%E4%B8%8Ediff/images/diffing.png)

> 图片引用自染陌同学[blog](https://github.com/answershuto/Blog/blob/master/blogs/VirtualDOM%E4%B8%8Ediff(Vue%E5%AE%9E%E7%8E%B0).MarkDown)

看一下`snabbdom`的`patch`

```js
//以下代码有一些删减
function sameVnode(vnode1: VNode, vnode2: VNode): boolean {
  return vnode1.key === vnode2.key;
}

function patch(oldVnode: VNode | Element, vnode: VNode): VNode {
    let i: number, elm: Node, parent: Node;
    const insertedVnodeQueue: VNodeQueue = [];
    for (i = 0; i < cbs.pre.length; ++i) cbs.pre[i]();

    if (!isVnode(oldVnode)) {
      oldVnode = emptyNodeAt(oldVnode);
    }

    if (sameVnode(oldVnode, vnode)) {
      patchVnode(oldVnode, vnode, insertedVnodeQueue);
    } else {
      elm = oldVnode.elm as Node;
      parent = api.parentNode(elm);

      createElm(vnode, insertedVnodeQueue);

      if (parent !== null) {
        api.insertBefore(parent, vnode.elm as Node, api.nextSibling(elm));
        removeVnodes(parent, [oldVnode], 0, 0);
      }
    }
    return vnode;
  };
```

首先`patch`会对新旧两个`VNode`节点对比`key`在`snabbdom`及`Vue`中还会判断其他的属性,如果通过则判定当前的旧节点可以复用,进去`patchVnode`函数进行具体的复用对比,如果不通过则删除旧节点,调用`createElm`函数渲染新`Vnode`节点,然后插入到旧节点,

看一下`patchVnode`的代码

```typescript
function patchVnode(oldVnode: VNode, vnode: VNode, insertedVnodeQueue: VNodeQueue) {
    let i: any, hook: any;
    if (isDef(i = vnode.data) && isDef(hook = i.hook) && isDef(i = hook.prepatch)) {
      i(oldVnode, vnode);
    }
    const elm = vnode.elm = (oldVnode.elm as Node);
    let oldCh = oldVnode.children;
    let ch = vnode.children;
    if (oldVnode === vnode) return;
    if (vnode.data !== undefined) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode);
      i = vnode.data.hook;
      if (isDef(i) && isDef(i = i.update)) i(oldVnode, vnode);
    }
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh as Array<VNode>, ch as Array<VNode>, insertedVnodeQueue);
      } else if (isDef(ch)) {
        if (isDef(oldVnode.text)) api.setTextContent(elm, '');
        addVnodes(elm, null, ch as Array<VNode>, 0, (ch as Array<VNode>).length - 1, insertedVnodeQueue);
      } else if (isDef(oldCh)) {
        removeVnodes(elm, oldCh as Array<VNode>, 0, (oldCh as Array<VNode>).length - 1);
      } else if (isDef(oldVnode.text)) {
        api.setTextContent(elm, '');
      }
    } else if (oldVnode.text !== vnode.text) {
      if (isDef(oldCh)) {
        removeVnodes(elm, oldCh as Array<VNode>, 0, (oldCh as Array<VNode>).length - 1);
      }
      api.setTextContent(elm, vnode.text as string);
    }
    if (isDef(hook) && isDef(i = hook.postpatch)) {
      i(oldVnode, vnode);
    }
  }
```

从代码中来看`patchVnode`核心规则主要如下

- 如果新旧节点同时存在`children`则调用`updateChildren`来对子节点进行`diff`
- 如果老节点不存在`children`而新节点存在,则清空挂载DOM,然后渲染`children`
- 如果老节点存在`children`而新节点不存在,则调用`removeVnodes`清空老节点所有子节点
- 当新老节点都不存在`children`时
  - 如果新节点不存在text属性,怎清空挂载DOM的textContent
  - 如果新节点存在text值,则设置挂载DOM的textContent为新节点的text值

从上面的代码中可以看出,`updateChildren`是整个`patch`最核心的函数,就是这个函数中所运用的`diff`算法,使得整个节点复用效率大大提高,看一下`updateChildren`
```typescript
function updateChildren(parentElm: Node,
                          oldCh: Array<VNode>,
                          newCh: Array<VNode>,
                          insertedVnodeQueue: VNodeQueue) {
    let oldStartIdx = 0, newStartIdx = 0;
    let oldEndIdx = oldCh.length - 1;
    let oldStartVnode = oldCh[0];
    let oldEndVnode = oldCh[oldEndIdx];
    let newEndIdx = newCh.length - 1;
    let newStartVnode = newCh[0];
    let newEndVnode = newCh[newEndIdx];
    let oldKeyToIdx: any;
    let idxInOld: number;
    let elmToMove: VNode;
    let before: any;

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (oldStartVnode == null) {
        oldStartVnode = oldCh[++oldStartIdx]; // Vnode might have been moved left
      } else if (oldEndVnode == null) {
        oldEndVnode = oldCh[--oldEndIdx];
      } else if (newStartVnode == null) {
        newStartVnode = newCh[++newStartIdx];
      } else if (newEndVnode == null) {
        newEndVnode = newCh[--newEndIdx];
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
        oldStartVnode = oldCh[++oldStartIdx];
        newStartVnode = newCh[++newStartIdx];
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue);
        oldEndVnode = oldCh[--oldEndIdx];
        newEndVnode = newCh[--newEndIdx];
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
        api.insertBefore(parentElm, oldStartVnode.elm as Node, api.nextSibling(oldEndVnode.elm as Node));
        oldStartVnode = oldCh[++oldStartIdx];
        newEndVnode = newCh[--newEndIdx];
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue);
        api.insertBefore(parentElm, oldEndVnode.elm as Node, oldStartVnode.elm as Node);
        oldEndVnode = oldCh[--oldEndIdx];
        newStartVnode = newCh[++newStartIdx];
      } else {
        if (oldKeyToIdx === undefined) {
          oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);
        }
        idxInOld = oldKeyToIdx[newStartVnode.key as string];
        if (isUndef(idxInOld)) { // New element
          api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          newStartVnode = newCh[++newStartIdx];
        } else {
          elmToMove = oldCh[idxInOld];
          if (elmToMove.sel !== newStartVnode.sel) {
            api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          } else {
            patchVnode(elmToMove, newStartVnode, insertedVnodeQueue);
            oldCh[idxInOld] = undefined as any;
            api.insertBefore(parentElm, (elmToMove.elm as Node), oldStartVnode.elm as Node);
          }
          newStartVnode = newCh[++newStartIdx];
        }
      }
    }
    if (oldStartIdx <= oldEndIdx || newStartIdx <= newEndIdx) {
      if (oldStartIdx > oldEndIdx) {
        before = newCh[newEndIdx+1] == null ? null : newCh[newEndIdx+1].elm;
        addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx, insertedVnodeQueue);
      } else {
        removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
      }
    }
  }
```
在`updateChildren`中`snabbdom`使用了双索引对比,新旧节点同时维护前后两个索引,然后像中间进发,逐个对比,
在每次对比时会存在四个节点`newStartVnode`,`newEndVnode`,`oldStartVnode`,`oldEndVnode`两两对比
