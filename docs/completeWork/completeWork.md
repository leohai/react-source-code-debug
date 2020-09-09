# 概述
每个WIP节点都会经历两个阶段：beginWork和completeWork。节点进入complete的前提是已经完成了beginWork。这个时候拿到的WIP节点都是
经过diff算法调和过的，也就意味着对于某个WIP节点来说它的fiber类型的形态已经基本确定了。此时有两点需要注意：
* 目前只有fiber形态变了，对于原生DOM组件（HostComponent）和文本节点（HostText）的fiber来说，对应的DOM节点（fiber.stateNode）并未变化。
* 经过Diff生成的新的WIP节点持有了effectTag

基于这两个特点，completeWork的工作主要有：
* 构建或更新DOM节点，
     - 构建过程中，会自下而上将子节点的第一层第一层插入到当前节点。
     - 更新过程中，会计算DOM节点的属性，一旦属性需要更新，会为DOM节点对应的WIP节点标记Update的effectTag。
* 自下而上收集effectList，最终收集到root上

对于正常执行工作的WIP节点来说，会执行以上的任务。但由于是WIP节点的完成阶段，免不了之前的工作会出错，所以也会对出错的节点采取措施，
这涉及到错误边界以及Suspense的概念，相关思想会在对应的文章里专门介绍。

# 流程
completeUnitOfWork是complete阶段的入口。它内部有一个循环，会自下而上地遍历workInProgress节点，依次处理节点。

对于正常的WIP节点，会执行completeWork。这其中会对HostComponent组件完成更新props、绑定事件等DOM相关的工作。

```javascript
function completeUnitOfWork(unitOfWork: Fiber): void {
  let completedWork = unitOfWork;
  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    if ((completedWork.effectTag & Incomplete) === NoEffect) {
      // 如果WIP节点没有出错，走正常的complete流程
      ...

      let next;

      // 省略了判断逻辑
      // 对节点进行completeWork，生成DOM，更新props，绑定事件
      next = completeWork(current, completedWork, subtreeRenderLanes);

      if (next !== null) {
        // 任务被挂起的情况，
        workInProgress = next;
        return;
      }

      // 收集WIP节点的lanes，不漏掉被跳过的update的lanes，便于再次发起调度
      resetChildLanes(completedWork);

      // 将当前节点的effectList并入父级节点
       ...

      // 如果当前节点他自己也有effectTag，将它自己
      // 也并入到父级节点的effectList
    } else {
      // 执行到这个分支说明之前的更新有错误
      // 进入unwindWork
      const next = unwindWork(completedWork, subtreeRenderLanes);
      ...

    }

    // 查找兄弟节点，若有则进行beginWork -> completeWork
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {

      workInProgress = siblingFiber;
      return;
    }
    // 若没有兄弟节点，那么向上回到父级节点
    // 父节点进入complete
    completedWork = returnFiber;
    // 将WIP节点指向父级节点
    workInProgress = completedWork;
  } while (completedWork !== null);

  // 到达了root，整棵树完成了工作，标记完成状态
  if (workInProgressRootExitStatus === RootIncomplete) {
    workInProgressRootExitStatus = RootCompleted;
  }
}

```

由于React的大部分类型的fiber节点最终都要体现为DOM，所以该阶段对于HostComponent（原生DOM组件）的处理需要着重理解。
```javascript
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {

  ...

  switch (workInProgress.tag) {
    ...
    case HostComponent: {
      ...
      if (current !== null && workInProgress.stateNode != null) {
        // 更新
      } else {
        // 创建
      }
      return null;
    }
    case HostText: {
      const newText = newProps;
      if (current && workInProgress.stateNode != null) {
        // 更新
      } else {
        // 创建
      }
      return null;
    }
    case SuspenseComponent:
    ...
  }
}
```
由completeWork的结构可以看出，就是依据fiber的tag做不同处理。对于HostComponent 和 HostText的处理是类似的，都是视情况来决定是更新还是创建DOM。
若current存在并且workInProgress.stateNode（WIP节点对应的DOM实例）存在，说明此WIP节点的DOM节点已经存在，走更新逻辑，否则进行创建。

DOM节点的更新实则是属性的更新，会在下面的`DOM属性的处理 -> 属性的更新`中讲到，先来看一下DOM节点的创建和插入。

## DOM节点的创建和插入
我们知道，此时的completeWork针对的是经过diff算法产生的新fiber。对于HostComponent类型的新fiber来说，它可能有DOM节点，也可能没有。没有的话，
就需要执行先创建，再插入的操作，由此引入DOM的插入算法。
```
if (current !== null && workInProgress.stateNode != null) {
    // 表明fiber有dom节点，需要执行更新过程
} else {
    // fiber不存在DOM节点
    // 先创建DOM节点
    const instance = createInstance(
      type,
      newProps,
      rootContainerInstance,
      currentHostContext,
      workInProgress,
    );

    //DOM节点插入
    appendAllChildren(instance, workInProgress, false, false);

    // 将DOM节点挂载到fiber的stateNode上
    workInProgress.stateNode = instance;

    ...

}

```

**需要注意的是，DOM的插入并不是将当前DOM插入它的父节点，而是将当前这个DOM节点的第一层子节点插入到它自己的下面。**

### 图解算法
此时处于completeWork阶段，会自下而上遍历WIP树到root，每经过一层都会按照上面的规则插入DOM。下边用一个例子来理解一下这个过程。

这是一棵fiber树的结构，workInProgress树最终要成为这个形态。
```
  1              App
                  |
                  |
  2              div
                /   \
               /     \
  3        <List/>--->span
            /   \
           /     \
  4       p ----> 'text node'
         /
        /
  5    h1
```
构建WIP树的DFS遍历对沿途节点一路beginWork，此时已经遍历到最深的h1节点，它的beginWork已经结束，开始进入completeWork阶段，此时所在的层级深度为第5层。
**第5层**

```
  1              App
                  |
                  |
  2              div
                /
               /
  3        <List/>
            /
           /
  4       p
         /
        /
  5--->h1
```

此时WIP节点指向h1的fiber，它对应的dom节点为h1，dom标签创建出来以后进入`appendAllChildren`，因为当前的workInProgress节点为h1，此时它的child为null，无需插入，所以退出。
h1节点完成工作往上返回到第4层的p节点

此时的dom树为
```
      h1
```

**第4层**

```
  1              App
                  |
                  |
  2              div
                /
               /
  3        <List/>
            /   \
           /     \
  4 --->  p ----> 'text node'
         /
        /
  5    h1
```


此时WIP节点指向p的fiber，它对应的dom节点为p，进入`appendAllChildren`，发现 p 的child为 h1，并且是HostComponent组件，将 h1 插入 p，然后寻找h1是否有同级的sibling节点。
发现没有，退出。

p节点的所有工作完成，它的兄弟节点：HostText类型的组件'text'会作为下一个工作单元，执行beginWork再进入completeWork。现在需要对它执行`appendAllChildren`，发现没有child，
不执行插入操作。它的工作也完成，return到父节点<List/>，进入第3层

此时的dom树为
```
        p      'text'
       /
      /
     h1
```

**第3层**

```
  1              App
                  |
                  |
  2              div
                /   \
               /     \
  3 --->   <List/>--->span
            /   \
           /     \
  4       p ----> 'text'
         /
        /
  5    h1
```


此时WIP节点指向List的fiber，对它进行completeWork，由于此时它是自定义组件，不属于HostComponent，所以不会对它进行子节点的插入操作。寻找它的兄弟节点span，beginWork再completeWork，执行子节点的插入操作，
发现它没有child，退出。return到父节点div，进入第二层。

此时的dom树为
```
                span

        p      'text'
       /
      /
     h1
```
**第2层**

```
  1              App
                  |
                  |
  2 --------->   div
                /   \
               /     \
  3        <List/>--->span
            /   \
           /     \
  4       p ---->'text'
         /
        /
  5    h1
```
此时WIP节点指向div的fiber，对它进行completeWork，执行子节点插入操作。由于它的child是<List/>，不满足`node.tag === HostComponent || node.tag === HostText`的条件，所以
不会将它插入到div中。继续向下找<List/>的child，发现是p，将P插入div，寻找p的sibling，发现了'text'，将它也插入div。之后再也找不到同级节点，此时回到第三层的<List/>节点。

<List/>有sibling节点span，将span插入到div。由于span没有子节点，所以退出。

此时的dom树为
```
             div
          /   |   \
         /    |    \
       p   'text'  span
      /
     /
    h1
```

**第1层**
此时WIP节点指向App的fiber，由于它是自定义节点，所以不会对它进行子节点的插入操作。

到此为止，dom树基本构建完成。在这个过程中我们可以总结出几个规律：
1. 向节点中插入dom节点时，只插入它子节点中第一层的dom。可以把这个插入可以看成是一个自下而上收集dom节点的过程。第一层之下的dom，在该dom节点执行插入时已经被插入了，类似于累加的
概念。
2. 总是优先看本身可否插入，再往下找，之后才是sibling节点。

这是由于fiber树和dom树的差异导致，每个fiber节点不一定对应一个dom节点，但一个dom节点一定对应一个fiber节点。
```
   fiber树      DOM树

   <App/>       div
     |           |
    div        input
     |
  <Input/>
     |
   input
```
由于一个原生DOM组件的子组件有可能是类组件或函数组件，优先检查自身，但它们不是原生DOM组件，不能被插入到父级的DOM组件对应的DOM节点中，所以下一步要往下找，直到找到原生DOM组件，执行插入，
最后再从这一层找同级的fiber节点，同级节点也会执行`先自检，再检查下级，再检查下级的同级`的操作。

可以看出，节点的插入也是深度优先。

## DOM属性的处理
上面的插入过程完成了DOM树的构建，这之后要做的就是为每个DOM节点计算它自己的属性（props）。由于节点存在创建和更新两种情况，所以对属性的处理也会区别对待。

### 属性的创建
属性的创建相对更新来说比较简单，这个过程发生在DOM节点构建的最后，调用`finalizeInitialChildren`函数完成新节点的属性设置。
```
if (current !== null && workInProgress.stateNode != null) {
    // 更新
} else {
    ...
    // 创建、插入DOM节点的过程
    ...

    // DOM节点属性的初始化
    if (
      finalizeInitialChildren(
        instance,
        type,
        newProps,
        rootContainerInstance,
        currentHostContext,
      )
     ) {
       // 最终会依据textarea的autoFocus属性
       // 来决定是否更新fiber
       markUpdate(workInProgress);
     }
}

```
`finalizeInitialChildren`最终会调用`setInitialProperties`，来完成属性的设置。
这个过程相对简单。主要就是调用`setInitialDOMProperties`将属性直接设置进DOM节点（事件在这个阶段绑定）
```
function setInitialDOMProperties(
  tag: string,
  domElement: Element,
  rootContainerElement: Element | Document,
  nextProps: Object,
  isCustomComponentTag: boolean,
): void {
  for (const propKey in nextProps) {
    const nextProp = nextProps[propKey];
    if (propKey === STYLE) {
      // 设置行内样式
      setValueForStyles(domElement, nextProp);
    } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
      // 设置innerHTML
      const nextHtml = nextProp ? nextProp[HTML] : undefined;
      if (nextHtml != null) {
        setInnerHTML(domElement, nextHtml);
      }
    }
     ...
     else if (registrationNameDependencies.hasOwnProperty(propKey)) {
      // 绑定事件
      if (nextProp != null) {
        ensureListeningTo(rootContainerElement, propKey);
      }
    } else if (nextProp != null) {
      // 设置其余属性
      setValueForProperty(domElement, propKey, nextProp, isCustomComponentTag);
    }
  }
}
```
### 属性的更新
若对已有DOM节点进行更新，说明只对属性进行更新即可，因为节点已经存在，不存在删除和新增的情况。`updateHostComponent`函数
负责HostComponent的更新，代码不多很好理解。
```
  updateHostComponent = function(
    current: Fiber,
    workInProgress: Fiber,
    type: Type,
    newProps: Props,
    rootContainerInstance: Container,
  ) {
    const oldProps = current.memoizedProps;
    // 新旧props相同，不更新
    if (oldProps === newProps) {
      return;
    }

    const instance: Instance = workInProgress.stateNode;
    const currentHostContext = getHostContext();

    // prepareUpdate计算新属性
    const updatePayload = prepareUpdate(
      instance,
      type,
      oldProps,
      newProps,
      rootContainerInstance,
      currentHostContext,
    );

    // 最终新属性被挂载到updateQueue中，供commit阶段使用
    workInProgress.updateQueue = (updatePayload: any);

    if (updatePayload) {
      // 标记WIP节点有更新
      markUpdate(workInProgress);
    }
  };
```

可以看出它只做了一件事就是计算新的属性，并挂载到WIP节点的updateQueue中，它的形式为：
```
[ 'style', { color: 'blue' }, title, '测试标题' ]
```
这个结果由`diffProperties`计算产生，它对比lastProps和nextProps，计算出updatePayload。内部的过程可以概括为：

若有某个属性（propKey）它在

* lastProps存在，nextProps不存在，将propKey的value标记为null表示删除
* lastProps不存在，nextProps存在，将nextProps中的propKey和对应的value添加到updatePayload
* lastProps存在，nextProps也存在，将nextProps中的propKey和对应的value添加到updatePayload

举个例子来说，有如下组件，div上绑定的点击事件会改变它的props。
```
class PropsDiff extends React.Component {
    state = {
        title: '更新前的标题',
        color: 'red',
        fontSize: 18
    }
    onClickDiv = () => {
        this.setState({
            title: '更新后的标题',
            color: 'blue'
        })
    }
    render() {
        const { color, fontSize, title } = this.state
        return <div
            className="test"
            onClick={this.onClickDiv}
            title={title}
            style={{color, fontSize}}
            {...this.state.color === 'red' && { props: '自定义旧属性' }}
        >
            测试div的Props变化
        </div>
    }
}
```
lastProps和nextProps分别为
```
lastProps
{
  "className": "test",
  "title": "更新前的标题",
  "style": { "color": "red", "fontSize": 18},
  "props": "自定义旧属性",
  "children": "测试div的Props变化",
  "onClick": () => {...}
}

nextProps
{
  "className": "test",
  "title": "更新后的标题",
  "style": { "color":"blue", "fontSize":18 },
  "children": "测试div的Props变化",
  "onClick": () => {...}
}
```
它们有变化的是propsKey是`style、title、props`，经过diff，最终打印出来的updatePayload为
```
[
   "props", null,
   "title", "更新后的标题",
   "style", {"color":"blue"}
]
```

我们来看一下源码结构：
```javascript
export function diffProperties(
  domElement: Element,
  tag: string,
  lastRawProps: Object,
  nextRawProps: Object,
  rootContainerElement: Element | Document,
): null | Array<mixed> {

  let updatePayload: null | Array<any> = null;

  let lastProps: Object;
  let nextProps: Object;
  
  ...

  let propKey;
  let styleName;
  let styleUpdates = null;

  for (propKey in lastProps) {
    // 循环lastProps，找出需要标记删除的propKey
    if (
      nextProps.hasOwnProperty(propKey) ||
      !lastProps.hasOwnProperty(propKey) ||
      lastProps[propKey] == null
    ) {
      // 对propKey来说，如果nextProps也有，或者lastProps没有，那么
      // 就不需要标记为删除，跳出本次循环继续判断下一个propKey
      continue;
    }
    if (propKey === STYLE) {
      // 删除style
      const lastStyle = lastProps[propKey];
      for (styleName in lastStyle) {
        if (lastStyle.hasOwnProperty(styleName)) {
          if (!styleUpdates) {
            styleUpdates = {};
          }
          styleUpdates[styleName] = '';
        }
      }
    } else if(/*...*/) {
      ...
      // 一些特定种类的propKey的删除
    } else {
      // 将其他种类的propKey标记为删除
      (updatePayload = updatePayload || []).push(propKey, null);
    }
  }
  for (propKey in nextProps) {
    // 将新prop添加到updatePayload
    const nextProp = nextProps[propKey];
    const lastProp = lastProps != null ? lastProps[propKey] : undefined;
    if (
      !nextProps.hasOwnProperty(propKey) ||
      nextProp === lastProp ||
      (nextProp == null && lastProp == null)
    ) {
      // 如果nextProps不存在propKey，或者前后的value相同，或者前后的value都为null
      // 那么不需要添加进去，跳出本次循环继续处理下一个prop
      continue;
    }
    if (propKey === STYLE) {
      /*
      * lastProp: { color: 'red' }
      * nextProp: { color: 'blue' }
      * */
      // 如果style在lastProps和nextProps中都有
      // 那么需要删除lastProps中style的样式
      if (lastProp) {
        // 如果lastProps中也有style
        // 将style内的样式属性设置为空
        // styleUpdates = { color: '' }
        for (styleName in lastProp) {
          if (
            lastProp.hasOwnProperty(styleName) &&
            (!nextProp || !nextProp.hasOwnProperty(styleName))
          ) {
            if (!styleUpdates) {
              styleUpdates = {};
            }
            styleUpdates[styleName] = '';
          }
        }
        // 以nextProp的属性名为key设置新的style的value
        // styleUpdates = { color: 'blue' }
        for (styleName in nextProp) {
          if (
            nextProp.hasOwnProperty(styleName) &&
            lastProp[styleName] !== nextProp[styleName]
          ) {
            if (!styleUpdates) {
              styleUpdates = {};
            }
            styleUpdates[styleName] = nextProp[styleName];
          }
        }
      } else {
        // 如果lastProps中没有style，说明新增的
        // 属性全部可放入updatePayload
        if (!styleUpdates) {
          if (!updatePayload) {
            updatePayload = [];
          }
          updatePayload.push(propKey, styleUpdates);
          // updatePayload: [ style, null ]
        }
        styleUpdates = nextProp;
        // styleUpdates = { color: 'blue' }
      }
    } else if (/*...*/) {
      ...
      // 一些特定种类的propKey的处理
    } else if (registrationNameDependencies.hasOwnProperty(propKey)) {
      if (nextProp != null) {
        // 重新绑定事件
        ensureListeningTo(rootContainerElement, propKey);
      }
      if (!updatePayload && lastProp !== nextProp) {
        // 事件重新绑定后，需要赋值updatePayload，使这个节点得以被更新
        updatePayload = [];
      }
    } else if (
      typeof nextProp === 'object' &&
      nextProp !== null &&
      nextProp.$$typeof === REACT_OPAQUE_ID_TYPE
    ) {
      // 服务端渲染相关
      nextProp.toString();
    } else {
       // 将计算好的属性push到updatePayload
      (updatePayload = updatePayload || []).push(propKey, nextProp);
    }
  }
  if (styleUpdates) {
    // 将style和值push进updatePayload
    (updatePayload = updatePayload || []).push(STYLE, styleUpdates);
  }
  console.log('updatePayload', JSON.stringify(updatePayload));
  // [ 'style', { color: 'blue' }, title, '测试标题' ]
  return updatePayload;
}
```

属性的diff为WIP节点挂载了带有新属性的updateQueue，一旦节点的updateQueue不为空，它就会被标记上Update的
effectTag
```javascript
if (updatePayload) {
  markUpdate(workInProgress);
}
```

## effect链的收集
经过beginWork和上面对于DOM的操作，有变化的WIP节点已经被打上了effectTag。

一旦WIP节点持有了effectTag，说明它需要在commit阶段被处理。每个WIP节点都有一个firstEffect和lastEffect，是一个单向链表，来表
示它自身以及它的子节点上所有持有effectTag的WIP节点。completeWork阶段在向上遍历的过程中也会逐层收集effect链，最终收集到root上，
供接下来的commit阶段使用。

实现上相对简单，对于某个WIP节点来说，先将它已有的effectList并入到父级节点，再判断它自己有没有effectTag，有的话也并入到父级节点。

```javascript
 /*
* effectList是一条单向链表，每完成一个工作单元上的任务，
* 都要将它产生的effect链表并入
* 上级工作单元。
* */
// 将当前节点的effectList并入到父节点的effectList
if (returnFiber.firstEffect === null) {
  returnFiber.firstEffect = completedWork.firstEffect;
}
if (completedWork.lastEffect !== null) {
  if (returnFiber.lastEffect !== null) {
    returnFiber.lastEffect.nextEffect = completedWork.firstEffect;
  }
  returnFiber.lastEffect = completedWork.lastEffect;
}

// 将自身添加到effect链，添加时跳过NoWork 和
// PerformedWork的effectTag，因为真正
// 的commit用不到
const effectTag = completedWork.effectTag;

if (effectTag > PerformedWork) {
  if (returnFiber.lastEffect !== null) {
    returnFiber.lastEffect.nextEffect = completedWork;
  } else {
    returnFiber.firstEffect = completedWork;
  }
  returnFiber.lastEffect = completedWork;
}
```
每个节点都会执行这样的操作，最终当回到root的时候，root上会有一条完整的effectList，包含了所有需要处理的fiber节点。

# 错误处理
completeUnitWork中的错误处理是错误边界机制的组成部分。

错误边界是一种React组件，一旦类组件中使用了`getDerivedStateFromError`或`componentDidCatch`，可以捕获发生在其子组件树的错误，那么它就是错误边界。

回到源码中，节点如果在completeWork之前的处理过程报错，它就会被打上Incomplete的effectTag，此时说明节点的更新工作未完成，因此不能执行正常的completeWork，
要走另一个判断分支进行处理。
```javascript
if ((completedWork.effectTag & Incomplete) === NoEffect) {
  
} else {
  // 有Incomplete的节点会进入到这个判断分支进行错误处理
}

```

## Incomplete从何而来

什么情况下节点会被标记上Incomplete呢？这还要从最外层的工作循环说起。

concurrent模式的渲染函数：renderRootConcurrent之中在构建workInProgress树时，使用了try...catch来包裹执行函数，这对处理报错节点提供了机会。
```javascript
do {
    try {
      workLoopConcurrent();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);
```
一旦某个节点执行出错，会进入`handleError`函数处理。该函数中可以获取到当前出错的WIP节点，除此之外我们暂且不关注其他功能，只需清楚它调用了`throwException`。

`throwException`会为这个出错的WIP节点打上`Incomplete 的 effectTag`，表明未完成，然后向上找到可以捕获这个错误的节点（即错误边界），添加上ShouldCapture 的 effectTag。
然后创建代表错误的update，`getDerivedStateFromError`放入payload，`componentDidCatch`放入callback。然后这个update入队节点的updateQueue。

`throwException`执行完毕，回到出错的WIP节点，执行`completeUnitOfWork`，目的是将错误终止到当前的节点，因为它本身都出错了，再向下渲染没有意义。
```javascript
function handleError(root, thrownValue):void {
  ...

  // 给当前出错的WIP节点添加上 Incomplete 的effectTag
  throwException(
    root,
    erroredWork.return,
    erroredWork,
    thrownValue,
    workInProgressRootRenderLanes,
  );

  // 开始对错误节点执行completeWork阶段
  completeUnitOfWork(erroredWork);
  
  ...

}
```

当这个错误节点进入completeUnitOfWork时，因为持有了`Incomplete`，所以不会进入正常的complete流程，而是会进入错误处理的逻辑。

错误处理逻辑做的事情：
* 对出错节点执行`unwindWork`。
* 将出错节点的父节点（returnFiber）标记上`Incomplete`，目的是在父节点执行到completeUnitOfWork的时候，也能被执行unwindWork，进而验证它是否是错误边界。
* 清空出错节点父节点上的effect链。

这里的重点是`unwindWork`会验证节点是否是错误边界，来看一下unwindWork的关键代码：
```javascript
function unwindWork(workInProgress: Fiber, renderLanes: Lanes) {
  switch (workInProgress.tag) {
    case ClassComponent: {

      ...

      const effectTag = workInProgress.effectTag;
      if (effectTag & ShouldCapture) {
        // 删它上面的ShouldCapture，再打上DidCapture
        workInProgress.effectTag = (effectTag & ~ShouldCapture) | DidCapture;
        
        return workInProgress;
      }
      return null;
    }
    ...
    default:
      return null;
  }
}
```
`unwindWork`会验证节点是错误边界的依据就是节点上是否有ShouldCapture的effectTag。而这个ShouldCapture就是在当时出错
时`throwException`找到错误边界并标记上的。如果节点是错误边界，最终会被return出去。


接下来我们看一下错误处理的整体逻辑：

```javascript
if ((completedWork.effectTag & Incomplete) === NoEffect) {

    // 正常流程
    ... 

} else {
  // 验证节点是否是错误边界
  const next = unwindWork(completedWork, subtreeRenderLanes);

  if (next !== null) {
    // 如果找到了错误边界，删除与错误处理有关的effectTag，
    // 例如ShouldCapture、Incomplete，
    // 并将workInProgress指针指向next
    next.effectTag &= HostEffectMask;
    workInProgress = next;
    return;
  }

  // ...省略了React性能分析相关的代码

  if (returnFiber !== null) {
    // 将父Fiber的effect list清除，effectTag标记为Incomplete，
    // 便于它的父节点再completeWork的时候被unwindWork
    returnFiber.firstEffect = returnFiber.lastEffect = null;
    returnFiber.effectTag |= Incomplete;
  }
}

...
// 继续向上completeWork的过程
completedWork = returnFiber;

```

现在我们要有个认知，一旦unwindWork识别当前的WIP节点为错误边界，那么现在的workInProgress节点就是这个错误边界。
然后会删除掉与错误处理有关的effectTag，DidCapture会被保留下来。
```javascript
  if (next !== null) {
    next.effectTag &= HostEffectMask;
    workInProgress = next;
    return;
  }
```
WIP节点有值，并且跳出了completeUnitOfWork，那么继续最外层的工作循环：
```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```
此时，workInProgress节点，也就是错误边界，会再被performUnitOfWork处理，然后进入beginWork、completeWork！
也就是说明报错的节点会被重新渲染一次，不同的是，这次节点上持有了DidCapture的effectTag。所以这次的流程是不一样的。
还记得`throwException`阶段入队错误边界更新队列的表示错误的update吗？它在beginWork调用processUpdateQueue的时候，会被处理，
首先保证了`getDerivedStateFromError`和`componentDidCatch`的调用，其次是产生新的state，这个state表示这次错误的状态。

之后代码执行到完成类组件更新的函数`finishClassComponent`，对于错误边界，会卸载掉它所有的子节点，重新渲染新的子节点，这个子节点有可能是
经过错误处理渲染的备用UI。
```javascript

```