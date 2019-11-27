# React Hooks原理解析与一点体会
## Hooks原理
这里把常见的Hook方法分两类来讲，useState、useReducer、useCallback、useMemo、useRef为一类，useEffect、useImperativeHandle、useLayoutEffect为一类，前一类是状态的管理更新，后一类是副作用的实现。两类也分属不同的react执行阶段，前一类处理在渲染阶段，后一类主要处理在提交阶段

由于函数式组件的重新渲染也就是整个组件函数的再执行的过程，所以对于Hook方法来说就需要区分是首次渲染运行还是重渲染运行。react内部便将这些Hook方法分成了挂载时和更新时两类：HooksDispatcherOnMount和HooksDispatcherOnUpdate两个对象包含了所有可用的Hook方法。对于useState来说对应的就是mountState和updateState方法，对应useEffect来说对应的就是mountEffect和updateEffect方法。下面分别以useState和useEffect为例说说它们的原理
### useState
在react内部每个组件都有至少一个Fiber进行着工作。对于函数式组件的Fiber来说，如下两个属性
```ts
// effect队列
updateQueue: UpdateQueue<any> | null,

// hook队列
memoizedState: any,
```
memoizedState存放的是组件中第一个hook的信息，hook的结构如下
```ts
const hook: Hook = {
    memoizedState: null,

    baseState: null,
    queue: null,
    baseUpdate: null,

    next: null,
};
```
对于useState hook来说，memoizedState存放的是当前state值，queue存放的是该state的异步更新队列，next则存放组件中下一个hook的信息。因此可以知道，是以链表的形式存储组件中所有hooks信息。下面我们按整个渲染绘制流程来分析

函数式组件的render都是renderWithHooks方法实现的，渲染时会执行`let children = Component(props, refOrContext);`得到组件的信息。这一步useState取的就是mountState
```ts
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
    // 创建hook对象，并插入到当前hook队列尾部
    const hook = mountWorkInProgressHook();
    if (typeof initialState === 'function') {
    initialState = initialState();
    }
    // 初始化值
    hook.memoizedState = hook.baseState = initialState;
    // 上面提到的更新队列
    const queue = (hook.queue = {
        last: null,
        dispatch: null,
        lastRenderedReducer: basicStateReducer,
        lastRenderedState: (initialState: any),
    });
    const dispatch: Dispatch<
        BasicStateAction<S>,
    > = (queue.dispatch = (dispatchAction.bind(
        null,
        ((currentlyRenderingFiber: any): Fiber),
        queue,
    ): any));
    // 返回的设置方法是个dispatch
    return [hook.memoizedState, dispatch];
}
```
运行组件结束后，renderWithHooks还有如下操作
```ts
const renderedWork: Fiber = (currentlyRenderingFiber: any);

renderedWork.memoizedState = firstWorkInProgressHook;
renderedWork.expirationTime = remainingExpirationTime;
renderedWork.updateQueue = (componentUpdateQueue: any);
renderedWork.effectTag |= sideEffectTag;
```
renderedWork也就是当前组件的Fiber，可以看到就再这里把该组件的第一个hook赋给了memoizedState，把effect队列（componentUpdateQueue）赋给了updateQueue，effect下面再继续说

假设现在首次渲染结束了，提交并绘制也成功了，下面该更新状态了

其实useState的实现是useReducer实现上的一层封装，看看更新的updateState的实现
```ts
function updateState<S>(
    initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
    return updateReducer(basicStateReducer, (initialState: any));
}

function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
    return typeof action === 'function' ? action(state) : action;
}
```
更新state时，新的state即为dispatch的action。那下面就看看dispatch的实现
```ts
function dispatchAction<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
    // dispatchAction部分实现
    const update: Update<S, A> = {
        expirationTime,
        suspenseConfig,
        action,
        eagerReducer: null,
        eagerState: null,
        next: null,
    };

    // Append the update to the end of the list.
    const last = queue.last;
    if (last === null) {
        // This is the first update. Create a circular list.
        update.next = update;
    } else {
        const first = last.next;
        if (first !== null) {
        // Still circular.
        update.next = first;
        }
        last.next = update;
    }
    queue.last = update;

    // 省略其他细节

    scheduleWork(fiber, expirationTime);
}
```
dispatch时会创建一个update，其中action就是设置的state值，然后插入到更新队列中，队列形如
![hook-queue](https://charles-hang.github.io/images/reactHooksTheoryPractice/hook-queue.jpg)

最后一步`scheduleWork(fiber, expirationTime);`开始调度工作，即重新render、commit，然后又会再次执行renderWithHooks。
这时就到了state的更新阶段了，useState取的是updateState，由上面可知是对updateReducer的封装，所以看updateReducer
```ts
// updateReducer内省略其他细节的更新部分代码
// 根据hook链取的这次的hook，
const hook = updateWorkInProgressHook();
const queue = hook.queue;
queue.lastRenderedReducer = reducer;

// The last update in the entire queue
const last = queue.last;
// The last update that is part of the base state.
const baseUpdate = hook.baseUpdate;
const baseState = hook.baseState;

// Find the first unprocessed update.
let first;
// 根据上面的更新队列图，last.next就是更新的开始节点
first = last !== null ? last.next : null;
if (first !== null) {
    let newState = baseState;
    let newBaseState = null;
    let newBaseUpdate = null;
    let prevUpdate = baseUpdate;
    let update = first;
    // 执行更新队列
    do {
        // Process this update.
        if (update.eagerReducer === reducer) {
            // If this update was processed eagerly, and its reducer matches the
            // current reducer, we can use the eagerly computed state.
            newState = ((update.eagerState: any): S);
        } else {
            // action就是set时传入的值
            const action = update.action;
            // reducer就是上面提到的basicStateReducer
            newState = reducer(newState, action);
        }
        prevUpdate = update;
        update = update.next;
    } while (update !== null && update !== first);

    newBaseUpdate = prevUpdate;
    newBaseState = newState;

    // 使用Object.is对比新旧state，不一样才会重渲染
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }

    hook.memoizedState = newState;
    hook.baseUpdate = newBaseUpdate;
    hook.baseState = newBaseState;

    queue.lastRenderedState = newState;
}

const dispatch: Dispatch<A> = (queue.dispatch: any);
return [hook.memoizedState, dispatch];
```
从这里就能知道官方定义的下面这条规则的原因
> 不要在循环，条件或嵌套函数中调用 Hook

执行更新的时候，在hook链表中按顺序取得当前hook的状态数据，在执行state计算时`newState = reducer(newState, action);`参数newState、action便是取自hook。设想在首次渲染时用了两个useState挂载了两个hook，第一个useState在一个条件里，在更新时如果第一个useState不满足条件而跳过了，那么第二个useState的执行取的hook数据便会是第一个useState的，它们之间的对应关系就被破坏了，算出的值自然就是错的

从这里还能知道自定义Hook并没有什么特别的地方，它就是一个包含Hook方法的函数，所以入参和返回可以随意定义，只要按照规则来不破坏hook链的对应关系就好，用`use`开头也只是让React能检测你的自定义Hook是否违反了规则
### useEffect
上面说了Fiber中memoizedState存放hook队列，同样Fiber中的updateQueue存放着effect队列，effect的结构如下
```ts
const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    // Circular
    next: (null: any),
};
```
tag表示该effect的类型，不同的effect执行的时间及细节不同（比如useEffect和useLayoutEffect，前者与DOM绘制异步，后者在DOM绘制之前同步进行，会阻塞绘制）就是靠这个tag来区分的，next存放下一个effect的信息，effect也是以链表的形式存储组件中的effect信息的。下面我们依然按渲染绘制流程来分析

上面提到renderWithHooks方法处理函数式组件的渲染，首次渲染时useEffect取的是mountEffect
```ts
// 省略细节代码
const hook = mountWorkInProgressHook();
const nextDeps = deps === undefined ? null : deps;
sideEffectTag |= fiberEffectTag;
hook.memoizedState = pushEffect(UnmountPassive | MountPassive, create, undefined, nextDeps);
```
从代码可以知道，effect同样是一个hook，也就是说hook队列里同时存着state hook和effect hook。不同的是，state的hook.memoizedState存放的是当前state值，而effect的hook.memoizedState存放的是上面代码中pushEffect运行的结果，pushEffect的第一个参数是effect的tag类型UnmountPassive | MountPassive。先看看pushEffect做了什么
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
    if (componentUpdateQueue === null) {
        componentUpdateQueue = createFunctionComponentUpdateQueue();
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
可以看到先获取了这个组件的componentUpdateQueue，这里存放的是这个组件的effect链表，然后将effect插入了链表中，它的结构跟上面说的state的更新队列的结构是一样的，返回的是这个effect的信息，那么这个effect的hook.memoizedState存放的就是这个effect的信息

还记得上面讲useState时，渲染运行函数结束后renderWithHooks还做了什么吗，再来复习一下
```ts
const renderedWork: Fiber = (currentlyRenderingFiber: any);

renderedWork.memoizedState = firstWorkInProgressHook;
renderedWork.expirationTime = remainingExpirationTime;
renderedWork.updateQueue = (componentUpdateQueue: any);
renderedWork.effectTag |= sideEffectTag;
```
好这里假设渲染阶段结束了，到提交阶段了，然后会执行commitPassiveHookEffects的方法
```ts
function commitPassiveHookEffects(finishedWork: Fiber): void {
    if ((finishedWork.effectTag & Passive) !== NoEffect) {
        switch (finishedWork.tag) {
            case FunctionComponent:
            case ForwardRef:
            case SimpleMemoComponent: {
                commitHookEffectList(UnmountPassive, NoHookEffect, finishedWork);
                commitHookEffectList(NoHookEffect, MountPassive, finishedWork);
                break;
            }
            default:
            break;
        }
    }
}

function commitHookEffectList(
    unmountTag: number,
    mountTag: number,
    finishedWork: Fiber,
) {
    const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
    let lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
    if (lastEffect !== null) {
        const firstEffect = lastEffect.next;
        let effect = firstEffect;
        do {
            if ((effect.tag & unmountTag) !== NoHookEffect) {
                // Unmount
                const destroy = effect.destroy;
                effect.destroy = undefined;
                if (destroy !== undefined) {
                    destroy();
                }
            }
            if ((effect.tag & mountTag) !== NoHookEffect) {
                // Mount
                const create = effect.create;
                effect.destroy = create();
            }
            effect = effect.next;
        } while (effect !== firstEffect);
    }
}
```
commitPassiveHookEffects里执行了两次commitHookEffectList，区别在于前两个参数。先看看commitHookEffectList方法，会从updateQueue中依次取得effect，看effect.tag的类型是否符合第一个参数unmountTag，是的话执行清除方法effect.destroy，再看effect.tag的类型是否符合第二个参数mountTag，是的话执行effect.create，并将create创建的destroy赋值给effect。（这里mount和unmount的条件判断用的是二进制位运算来实现的。）React里是先执行所有的清除方法再执行所有的副作用方法，所以这里执行了两次commitHookEffectList，第一次专门执行清除，第二次专门执行副作用。

useEffect的第二个参数可以设置跳过该effect的执行，那接下来看看是如何做到的，首次渲染时按刚才的分析是不会跳过的，都会执行，那么跳过的逻辑就在更新时了，useEffect取的是updateEffect
```ts
// 省略细节代码
const hook = updateWorkInProgressHook();
const nextDeps = deps === undefined ? null : deps;
let destroy = undefined;

if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
        const prevDeps = prevEffect.deps;
        if (areHookInputsEqual(nextDeps, prevDeps)) {
            pushEffect(NoHookEffect, create, destroy, nextDeps);
            return;
        }
    }
}

sideEffectTag |= fiberEffectTag;
hook.memoizedState = pushEffect(UnmountPassive | MountPassive, create, destroy, nextDeps);
```
上面的deps就是useEffect的第二个参数，通过`areHookInputsEqual(nextDeps, prevDeps)`判断依赖数组是否变化，内部通过`Object.is`来依次判断依赖是否相同。如果相同，这里注意传入pushEffect的第一个参数effect的tag类型是NoHookEffect，而不是UnmountPassive | MountPassive。渲染阶段结束，又到了提交阶段，再次运行commitPassiveHookEffects，commitHookEffectList里控制执行副作用的逻辑就不会生效，这样就跳过了该effect的执行，如下
```ts
// commitHookEffectList详情请回翻
if ((effect.tag & mountTag) !== NoHookEffect) {
    // Mount
    const create = effect.create;
    effect.destroy = create();
}
```
useEffect的原理到此也分析的差不多了。
### 原理小结
React除了在Fiber对象里存了hooks的信息外并没有提供额外的数据保持与记录处理，组件里对state（包括useCallback、useMemo返回的值）的一切操作都符合js原生函数的特性，因此对state的使用就要考虑闭包的影响。每次渲染后state都是最新的值，那么上一轮渲染时进行的异步操作中state如果没有额外处理根据闭包特性就会是上一轮渲染的值。其他的Hook方法：useCallback、useMemo、useRef、useEffect、useImperativeHandle、useLayoutEffect实现上与useState、useEffect类似，如果感兴趣可以到源码`packages/react-reconciler/src/ReactFiberHooks.js`中查看

最后放一张简易的组件对应Fiber的结构图方便进行记忆与梳理
![hook-queue](https://charles-hang.github.io/images/reactHooksTheoryPractice/FC-Fiber.jpg)
## 一点体会
React Hooks给我的感觉是向函数式编程做的一点点试探与铺垫，跟着这股感觉走，我认为在写函数式组件时我们应该抛弃类组件的思维，尤其是生命周期这个概念，任何像是用自定义Hooks模拟生命周期的方式不是说不行，而是与React Hooks设计的思想有悖。函数式组件的执行就两步：渲染 -> 执行副作用并绘制，然后往复。没有副作用的函数式组件就是一个纯函数，意思就是给定相同的props一定会渲染出相同的视图。那副作用是什么呢？任何破坏了这种一一对应的关系的作用就是副作用，比如发送了一个http请求并渲染出结果，根据返回的不同得到了不同的视图。用这个函数式的思维来写React Hooks其实就比类组件要简单多了，简洁明了。

React Hooks能更好的实现数据驱动，用一个例子来说明：一个含有筛选、查询和分页的列表页，对比两种不同的处理方法
```ts
// 类组件思维
state = {
    pageNum: 1,
    pageSize: 10,
    filter1: '',
    filter2: '',
    searchStr: ''
};

componentDidMount() {
    getList();
}

getList => () => {
    // 省略查询代码
}

onFilterOrPageChange = (params: {
    pageNum?: number;
    pageSize?: number;
    filter1?: string;
    filter2?: string;
    searchStr?: string;
}) => {
    const { pageNum = 1, ...other } = params;

    this.setState({
        pageNum,
        ...other
    }, this.getList);
}

onPageChange = (pageNum) => {
    this.onFilterOrPageChange({ pageNum });
}

onShowSizeChange = (currentPageNum, pageSize) => {
    this.onFilterOrPageChange({ pageSize });
}

onFilter1Change = (filter1) => {
    this.onFilterOrPageChange({ filter1 });
}

onFilter2Change = (filter2) => {
    this.onFilterOrPageChange({ filter2 });
}

onSearchStrChange = (searchStr) => {
    this.setState({
        searchStr
    });
}

onSearch = () => {
    this.onFilterOrPageChange({});
}
```
可以看到参数改变后，通过setState的回调重新查询
```ts
// React Hooks思维
const [pageNum, setPageNum] = useState(1);
const [pageSize, setPageSize] = useState(10);
const [filter1, setFilter1] = useState('');
const [filter2, setFilter2] = useState('');
const [searchStr, setSearchStr] = useState('');

useEffect(() => {
    getList();
    // eslint-disable-next-line
}, [pageNum]);

useEffect(() => {
    search();
    // eslint-disable-next-line
}, [filter1, filter2, pageSize]);

function getList() {
    // 省略查询代码
}

function search() {
    if (pageNum === 1) {
        getList();
    } else {
        setPageNum(1);
    }
}

const onPageChange = (pageNum) => {
    setPageNum(pageNum);
};

const onShowSizeChange = (currentPageNum, pageSize) => {
    setPageSize(pageSize);
};

const onFilter1Change = (filter1) => {
    setFilter1(filter1);
};

const onFilter2Change = (filter2) => {
    setFilter2(filter2);
};

const onSearchStrChange = (searchStr) => {
    setSearchStr(searchStr);
};

const onSearch = () => search();
```
可以看到参数的改变回调方法里就仅仅是改变参数值，不关心其它部分。若想重新查询则添加副作用对参数的改变作响应即可，这也是我理解的为什么useState返回的设置方法没有提供回调支持的原因。这么看是不是更有数据驱动的味道，也将原本在类组件中耦合在一起的两步（参数改变和重新查询）解耦开来。因此我也更倾向于后一种方式。

React Hooks官方介绍里提到动机时说了原本组件间状态逻辑的复用很难，为了复用虽然可以使用类似高阶组件的方式解决，但这就产生了需要依附于其他组件的组件，且使用时可能形成“嵌套地狱”。而React Hooks将原本只能在组件层面实现的状态逻辑复用拆成了更小粒度的自定义Hook，并且使用时是往组件里组合而不是组件的嵌套，组合自然地优于嵌套。说到这，上面对查询做的处理可以更进一步
```ts
// 将查询的副作用提取出来，实现可复用
function usePaginationFetch(
    fetchFn: () => any,
    pageNumState: [number, React.Dispatch<React.SetStateAction<number>],
    pageNumResetDeps?: any[]
) {
    const [pageNum, setPageNum] = pageNumState;

    useEffect(() => {
        fetchFn();
        // eslint-disable-next-line
    }, [pageNum]);

    useEffect(() => {
        onFetch();
        // eslint-disable-next-line
    }, pageNumResetDeps);

    function onFetch() {
        if (pageNum === 1) {
            fetchFn();
        } else {
            setPageNum(1);
        }
    }

    return onFetch;
}
```
```ts
// 列表组件
const [pageNum, setPageNum] = useState(1);
const [pageSize, setPageSize] = useState(10);
const [filter1, setFilter1] = useState('');
const [filter2, setFilter2] = useState('');
const [searchStr, setSearchStr] = useState('');
const onFetch = usePaginationFetch(
    getList,
    [pageNum, setPageNum],
    [pageSize, filter1, filter2]
);

function getList() {
    // 省略查询代码
}

const onPageChange = (pageNum) => {
    setPageNum(pageNum);
};

const onShowSizeChange = (currentPageNum, pageSize) => {
    setPageSize(pageSize);
};

const onFilter1Change = (filter1) => {
    setFilter1(filter1);
};

const onFilter2Change = (filter2) => {
    setFilter2(filter2);
};

const onSearchStrChange = (searchStr) => {
    setSearchStr(searchStr);
};

const onSearch = () => onFetch();
```
我看到了React Hooks的无限可能，也似乎窥见了函数式编程的大门
