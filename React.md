# 1.数据结构
### 1.1 hostRoot
页面dom根节点
### 1.2 hostComponent
原生 DOM组件 对应的 Fiber节点
### 1.3 stateNode
refers to the component instance a fiber belongs to

# 2. workloop
workloop的主要结构
```js
const a1 = { name: 'a1', child: null, sibling: null, return: null };
const b1 = { name: 'b1', child: null, sibling: null, return: null };
const b2 = { name: 'b2', child: null, sibling: null, return: null };
const b3 = { name: 'b3', child: null, sibling: null, return: null };
const c1 = { name: 'c1', child: null, sibling: null, return: null };
const c2 = { name: 'c2', child: null, sibling: null, return: null };
const d1 = { name: 'd1', child: null, sibling: null, return: null };
const d2 = { name: 'd2', child: null, sibling: null, return: null };

a1.child = b1;
b1.sibling = b2;
b2.sibling = b3;
b2.child = c1;
b3.child = c2;
c1.child = d1;
d1.sibling = d2;

b1.return = b2.return = b3.return = a1;
c1.return = b2;
d1.return = d2.return = c1;
c2.return = b3;

let nextUnitOfWork = a1;
workLoop();

function workLoop () {
  while (nextUnitOfWork !== null) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }
}

function performUnitOfWork (workInProgress) {
  let next = beginWork(workInProgress);
  if (next === null) {
    next = completeUnitOfWork(workInProgress);
  }
  return next;
}

function beginWork (workInProgress) {
  log('work performed for ' + workInProgress.name);
  return workInProgress.child;
}

function completeUnitOfWork (workInProgress) {
  while (true) {
    let returnFiber = workInProgress.return;
    let siblingFiber = workInProgress.sibling;

    nextUnitOfWork = completeWork(workInProgress);

    if (siblingFiber !== null) {
      // If there is a sibling, return it 
      // to perform work for this sibling
      return siblingFiber;
    } else if (returnFiber !== null) {
      // If there's no more work in this returnFiber, 
      // continue the loop to complete the returnFiber.
      workInProgress = returnFiber;
      continue;
    } else {
      // We've reached the root.
      return null;
    }
  }
}

function completeWork (workInProgress) {
  log('work completed for ' + workInProgress.name);
  return null;
}
```
## 3. render workflow
FIXME 流程图
### 3.1beginWork
对workInProgress Fiber的子节点进行diff算法，diff算法会创建子fiber，return 第一个子节点
### 3.2 performUnitOfWork
1. 执行beginWork 
2. 设置属性 ```  unitOfWork.memoizedProps = unitOfWork.pendingProps;``` 
3. ```if beginWork return null -> 工作fiber执行completeUnitOfWork``` 
4. ```else 子节点进入workflow：workloop -> performUnitOfWork```
### 3.3 completeUnitOfWork
1. 工作fiber执行 completeWork
2. ```if 有兄弟节点 -> 兄弟节点进入workflow```
3. ```else 通过fiber.returnFiber找到父节点进入completeUnitOfWork流程```
4. 父节点为空，代表当前为root节点，标记为root complete

### 3.4 completeWork
1. 根据fiber的tag执行相应操作，以tag === HostComponent为例
2. if 已经有dom节点 -> updateHostComponent
-  调用diffProperties，找出不同的dom属性e.g. style、children...返回数组偶数为key，奇数为value
- 赋值到workInProgress.updateQueue
- 如果有不同的属性，则workInProgress.flag 加上update标记
3. else mount操作
- 创建dom节点，
- 向里面添加子节点appendAllChildren：由于子结点向上递归执行completeWork，所以子结点已经构建好了dom tree，只需添加子结点即可
- fiber.stateNode=dom节点
- 调用finalizeInitialChildren设置dom节点的各种属性和事件，e.g. style textContent、click
- 处理subTreeFlags
- return null
## 4. diff 算法
### 4.1 reconcileChildrenArray
#### 4.1.1 flow
写法是共有3个for循环（实际是两轮，2和3是互斥的），newChildren为react element（jsx结构），next节点通过oldfiber.silbing
- 第一个：
```
key 不同停止循环 break
elementType 相同则复用；不同则创建新的，且标记老的为删除
更新lastPlacedIndex
```
- 如果新的遍历完，删除当前剩余老的（加到父节点.deletions数组中，FIXME 怎么具体怎么删的），返回newFirstChildFiber
- 第二个：新的还有，如果老的已经遍历完，则直接创建新的，返回newFirstChildFiber
- 第三个：新老都有
```
缓存老的在map中，开始第三轮
begin
	通过key取出老的，对比key和elementType，相同复用；不同则创建新的
	更新lastPlacedIndex
end
第三轮循环结束，删除当前剩余老的
return newFirstChildFiber
```

#### 4.1.2 lastPlacedIndex
代表老的，当前最后一个可复用的fiber的index
```
// 更新逻辑
const current = newFiber.alternate;
if (current !== null) {
	const oldIndex = current.index;
	if (oldIndex < lastPlacedIndex) {
    // This is a move. 该节点需要向右移动
		newFiber.flags |= Placement;
		return lastPlacedIndex;
	} else {
		// This item can stay in place.
		return oldIndex;
	}
} else {
	// This is an insertion.
	newFiber.flags |= Placement;
	return lastPlacedIndex;
}
```
### 4.2 reconcileSingleElement
删除代表标记删除
```
const key = element.key;
let child = currentFirstChild;
begin child != null
	if key相同且elementType相同则复用，并删除剩余的child，return existing
	if key相同且elementType不同，则删除剩余的child，break循环
	if key不同则删除当前结点，继续循环（可能silbing有相同的key）
	child = child.silbing
end
没有可复用的且已经删完老结点，创建新fiber并return
```

## 5.Schedule and Entrance
[[React schedule and entrace.drawio.svg]]

## 15.浏览器每帧执行
```
一个task(宏任务) -- 队列中全部job(微任务) -- requestAnimationFrame -- 浏览器重排/重绘 -- requestIdleCallback
```
# 参考资料
- https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react
- https://juejin.cn/post/7143134747114340382
- [diff算法](|https://juejin.cn/post/6925342156135596046)
- [简单的useState](https://code.h5jun.com/woniq/1/edit?js,console)
- [useState](https://www.xiabingbao.com/post/react/react-usestate-rn5bc0.html) 
- https://xiaochen1024.com/article_item/600acb68245877002ed5defe



# Question
### 1. 为何commitWork的时候没有更新onclick，onclick是在哪里更新的
### 2. root fiber 的 updateQueue的作用
![[initialFlow.png]]
- rootFiber, updateQueue是在updateContainer里创建的
- 在beginWork updateHostRoot中会处理updateQueue 得到 rootState的element（就是App组件函数）
- 然后进行diff算法，创建子fiber
```js
const nextState: RootState = workInProgress.memoizedState;
const nextChildren = nextState.element;
reconcileChildren(current, workInProgress, nextChildren, renderLanes);

// diff 算法
const created = createFiberFromElement(element, returnFiber.mode, lanes);
...
...
return created;
```
