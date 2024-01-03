# 1.数据结构
### 1.1 hostRoot
页面dom根节点
### 1.2 hostComponent
原生 DOM组件 对应的 Fiber节点
### 1.3 stateNode
refers to the component instance a fiber belongs to
### 1.4 fiberRoot 和 rootFiber
fiberRoot是实例，current指向rootFiber，rootFiber.stateNode指向fiberRoot

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
- 对workInProgress Fiber的子节点进行diff算法，diff算法会创建子fiber，return 第一个子节点
- mount的时候会给rootFiber的子fiber（只给App函数）标记为Placement
```ts
function placeSingleChild(newFiber: Fiber): Fiber {
    // This is simpler for the single child case. We only need to do a
    // placement for inserting new children.
    // shouldTrackSideEffects只有在mount的时候为true
    if (shouldTrackSideEffects && newFiber.alternate === null) {
      newFiber.flags |= Placement;
    }
    return newFiber;
  }
```
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
- 创建dom节点，并将props赋值到dom[__reactProps$]
- 向里面添加子节点appendAllChildren：由于子结点向上递归执行completeWork，所以子结点已经构建好了dom tree，只需添加子结点即可
- fiber.stateNode=dom节点
- 调用finalizeInitialChildren设置dom节点的各种属性，e.g. style textContent
4. 处理subTreeFlags: 不断或（|）所有子节点的subTreeFlags
5. return null
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
通过key缓存老的在map中，开始遍历剩余新的
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
	if key相同且elementType相同则复用，并删除剩余的silbing，return existing
	if key相同且elementType不同，则删除当前和剩余的silbing，break循环
	if key不同则删除当前结点，继续循环（可能silbing有相同的key）
	child = child.silbing
end
没有可复用的且已经删完老结点，创建新fiber并return
```

## 5.Schedule and Entrance
[[React schedule and entrace.drawio.svg]]

## 6. commit flow
1. 首先会调flushPassiveEffects确保清除当前副作用（还没遇到过，一开始存在副作用情况）
2. 如果树中存在副作用，schedule设置清除副作用
```js
if (
(finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
(finishedWork.flags & PassiveMask) !== NoFlags
) {
	if (!rootDoesHavePassiveEffects) {
	rootDoesHavePassiveEffects = true;
	...
	scheduleCallback(NormalSchedulerPriority, () => {
		flushPassiveEffects();
		return null;
	});
	}
}
```
3. commitBeforeMutationEffects
- 从子->兄弟->父遍历，如果是class组组件执行instance.getSnapshotBeforeUpdate
4. commitMutationEffects 这个阶段会渲染dom树，并执行layout的destroy函数
- flow 开始
- 从父节点开始向子节点递归，只是从屏幕中删除flag标记为deleted的结点，如果删除节点有destroy layout也会在这个阶段执行（destroy effect会在2阶段回调函数中执行）
- 父节点通过subTreeFlag 是否存在 MutationMask，存在则从子节点递归重新进入flow，不存则在通过fiber.flags & placement or fiber.flags & update在屏幕上插入dom树或者，通过updateQueue更新dom属性并将props赋值到dom[__reactProps$]，并执行layout destroy函数
5. commitLayoutEffects
从子到父执行layout create函数
## 7. hooks
```js
// 通过current fiber是否存在来决定是mount还是update
ReactCurrentDispatcher.current =
      current === null || current.memoizedState === null
        ? HooksDispatcherOnMount
        : HooksDispatcherOnUpdate;
```
### 7.1 useState
- mountState
```js
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
	// 创建hook，并返回
  const hook = mountWorkInProgressHook();
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  hook.memoizedState = hook.baseState = initialState;
  const queue: UpdateQueue<S, BasicStateAction<S>> = {
    pending: null, // 表示当前正在等待处理的更新，保存着update对象
    interleaved: null, // 表示当前正在处理的更新
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  };
  hook.queue = queue;
  const dispatch: Dispatch<
    BasicStateAction<S>,
  > = (queue.dispatch = (dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));
  return [hook.memoizedState, dispatch];
}
```
- updateState/updateReducer
```js
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
	// 获取当前的hook
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
	... 拼接pendingQueue到baseQueue，注意是循环链表
	... 遍历queue，顺序计算newState，涉及到根据优先级跳过当前update
    do {
	    const action = update.action;
      newState = reducer(newState, action);
      update = update.next;
    } while (update !== null && update !== first);
    
    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;
    queue.lastRenderedState = newState;
  }
	...
  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```
- dispatchAction 
```ts
function dispatchSetState<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  const lane = requestUpdateLane(fiber);
  const update: Update<S, A> = {
    lane,
    action,
    hasEagerState: false,
    eagerState: null,
    next: (null: any),
  };

  .....
  将update添加进queue
	......
	// 会调用 ensureRootIsScheduled
  scheduleUpdateOnFiber(root, fiber, lane, eventTime);
}
```
### 7.2 useEffect
- mountEffect
```ts
function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber.flags |= fiberFlags;
  // 将新的effect加入循环链表并返回
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    undefined,
    nextDeps,
  );
}
```
```ts
function pushEffect(tag, create, destroy, deps) {
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    // Circular
    next: (null: any),
  };
  let componentUpdateQueue: null | FunctionComponentUpdateQueue = (currentlyRenderingFiber.updateQueue: any);
  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  return effect;
}
```
- updateEffect
```ts
function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
	// 获得当前hook
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      if (areHookInputsEqual(nextDeps, prevDeps)) {
	      // 注意如果依赖相同effect就没有HookHasEffect这个flag，也就不会执行
        hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }

  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    destroy,
    nextDeps,
  );
}
```
## 15.浏览器每帧执行
```
一个task(宏任务) -- 队列中全部job(微任务) -- requestAnimationFrame -- 浏览器重排/重绘 -- requestIdleCallback
```
# 参考资料
- https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react
- https://juejin.cn/post/7143134747114340382
- [diff算法](https://juejin.cn/post/6925342156135596046)
- [简单的useState](https://code.h5jun.com/woniq/1/edit?js,console)
- [useState](https://www.xiabingbao.com/post/react/react-usestate-rn5bc0.html) 
- https://xiaochen1024.com/article_item/600acb68245877002ed5defe



# Question
### 1. 为何commitWork的时候没有更新onclick，onclick是在哪里更新的
react props key: __reactProps$，赋值为mount时在completeWork创建实例的时候，更新为commitMutationEffects阶段
commitWork或completeWork设置的click事件都是空函数，主要由页面冒泡处理
[React 事件源码解析 - 掘金 (juejin.cn)](https://juejin.cn/post/7231428738914287677)
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
### 3. 为何mount的时候current rootFiber不为null
renderRootSync/renderRootConcurrent会调用prepareFreshStack
```ts
// createRoot -> createFiberRoot
const uninitializedFiber = createHostRootFiber(
    tag,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
  );
root.current = uninitializedFiber;

// function prepareFreshStack(root,....)
// 此时root为fiberRoot，root.current在createRoot的时候就赋值
const rootWorkInProgress = createWorkInProgress(root.current, null);
workInProgress = rootWorkInProgress; // 之后走workloop workInProgress就不为null了

function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;
  if (workInProgress === null) {
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode,
    );
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;

    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {....}
```
### 4.组件什么时候会跳过diff
只要有调用函数组件去生成JSX（renderWithHooks父组件调用自身函数），那么子组件的oldProps和newProps肯定是不同的对象（父函数调用重新生成了不同对象，即便没传东西给子组件也会有个空对象），那么肯定会执行diff
```ts
// 如果父函数调用自身，props肯定不同，即便是空对象
const oldProps = current.memoizedProps;
const newProps = workInProgress.pendingProps;
if (oldProps !== newProps } {...需要去diff }
else {
	// check wip.lanes, lans不包括更新则调用
	return attemptEarlyBailoutIfNoScheduledUpdate
}

// attemptEarlyBailoutIfNoScheduledUpdate
// 子树是否有更新
if wip.childLans 不包括更新, return null
else {
	cloneChildFibers(current, workInProgress);
	return workInProgress.child;
}

// cloneChildFibers
// 注意这个pendingProps取的是cureent
// 即当前组件复用，那么传给子组件的props也是相同的对象
let newChild = createWorkInProgress(currentChild, currentChild.pendingProps);
workInProgress.child = newChild;
```
### 5.useLayout和useEffect
- useEffect 在渲染时是异步执行，并且要等到浏览器将所有变化渲染到屏幕后才会被执行。
- useLayoutEffect 在渲染时是同步执行，在渲染前完成（废话，渲染是异步的）
##### 5.1之前以为layout是在dom渲染完成后执行
- 之所以有这样的误解，从源码上看,commitMutationEffects先操作dom，接下来执行commitLayoutEffects调用layout create函数。
- 从debug的表现也是这样的
##### 5.2 实际上
- 首先页面实际执行，是layout先执行，执行完后才渲染dom。debug时候js线程可能处于空闲状态，导致dom先完成渲染
- 注意事件循环
```
一个task(宏任务) -- 队列中全部job(微任务) -- requestAnimationFrame -- 浏览器重排/重绘 -- requestIdleCallback
```
浏览器重排/重绘就是dom渲染，此时需要该宏任务中的同步代码执行完，接着微任务执行完，才到dom渲染。
所以，commitMutationEffects执行后立马执行commitLayoutEffects，等所有同步任务执行完，js线程空闲了才会去渲染dom
##### 5.3 useEffect的执行时机
- 是异步的吗？yes
commit flow 在调用 commitMutationEffects之前会根据树中是否存在effect副作用来设置schdule
```ts
if (
    (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
    (finishedWork.flags & PassiveMask) !== NoFlags
  ) {
    if (!rootDoesHavePassiveEffects) {
      rootDoesHavePassiveEffects = true;
      pendingPassiveEffectsRemainingLanes = remainingLanes;
      pendingPassiveTransitions = transitions;
      // 这里设置了回调，通过postMessage触发（宏任务）
      scheduleCallback(NormalSchedulerPriority, () => {
        flushPassiveEffects();
        return null;
      });
    }
  }
```
这个schedule的到期时间为5000
var NORMAL_PRIORITY_TIMEOUT = 5000;
- 是怎么处理lan的？
设置为NormalSchedulerPriority
https://juejin.cn/post/7008802041602506765
