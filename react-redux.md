# React-redux 源码笔记

react 自 v16.8 引入 hooks 的 api 后，官方推荐函数组件+hooks 编写 React 代码,本文主要围绕 react-redux 的 hooks 来对源码进行分析，进行源码分析前首先了解一下`React.useSyncExternalStore`

## useSyncExternalStore

`useSyncExternalStore` 是一个让你订阅外部数据 store 的 hooks。React18 带来了 Concurrent 模式，在开启此模式下一些 redux,mobx 等第三方状态库会出现订阅同一个数据但是 UI 展示不一致的情况，也就是数据撕裂[(tearing)](https://github.com/reactwg/react-18/discussions/69)。针对此问题，React 提供了`useSyncExternalStore` hooks

下面用官方 demo 展示改 hooks 用法：

```
import { useSyncExternalStore } from "react";

let nextId = 0;
/**外部数据 */
let todos = [{ id: nextId++, text: "Todo #1" }];
/**subscribe函数中的forceUpdate集合  */
let listeners = [];

const todosStore = {
  addTodo() {
    /**改变外部数据 */
    todos = [...todos, { id: nextId++, text: "Todo #" + nextId }];
    emitChange();
  },
  subscribe(forceUpdate) {
    /**收集listener */
    listeners = [...listeners, forceUpdate];
    /**返回解除订阅函数 */
    return () => {
      listeners = listeners.filter((f) => f !== forceUpdate);
      console.log(listeners);
    };
  },
  /**返回数据快照 */
  getSnapshot() {
    return todos;
  },
};

function emitChange() {
  /**触发更新 */
  for (let forceUpdate of listeners) {
    forceUpdate();
  }
}

export default function TodosApp() {
  const todos = useSyncExternalStore(
    todosStore.subscribe,
    todosStore.getSnapshot
  );
  return (
    <>
      <button onClick={() => todosStore.addTodo()}>Add todo</button>
      <hr />
      <ul>
        {todos.map((todo) => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
    </>
  );
}

```

- 第一个参数：一个接受单独参数`forceUpdate`的函数,`forceUpdate`可以使组件重新渲染，我们使用 listeners 数组变量保存`forceUpdate`函数，并返回一个函数用于把`forceUpdate`从 listeners 数组中删除的函数，类似 redux 的 subscribe,返回一个解除订阅的函数
- 第二个参数：一个根据外部数据返回对应快照的函数

一般使用流程大概为：

- 保存`forceUpdate`函数，用于外部调用
- `useSyncExternalStore`内部记录第二个参数返回的快照数据
- 改变外部数据,调用`forceUpdate`函数更新渲染,渲染前后调用第二个参数返回的快照数据做浅比较`Object.is`,如果不一致更新值

## React-redux 使用

首先来个简单的使用案例

```
/// main.ts 入口文件
import { legacy_createStore } from 'redux';
import reducer from './reducer'
import './index.css'

/**创建store */
const store = legacy_createStore(reducer);

ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(
  /**传递store */
  <Provider store={store}>
    <App />
  </Provider>
)

/// reducer
const initState = {
  count:0,
}

export default (state=initState, action:{type:string})=> {
  switch (action.type) {
    case "incremented":
      return { ...state,count:state.count + 1 };
    case "decremented":
      return { ...state,count:state.count - 1 };
    default:
      return state;
  }
}

```

reducer.ts

```
const initState = {
  count:0,
}

export default (state=initState, action:{type:string})=> {
  switch (action.type) {
    case "incremented":
      return { ...state,count:state.count + 1 };
    case "decremented":
      return { ...state,count:state.count - 1 };
    default:
      return state;
  }
}

```

App.tsx

```
import { useSelector, useDispatch } from "react-redux";

function App() {
  const dispatch = useDispatch();
  /**获取store中对应数据 */
  const { count } = useSelector((state: any) => ({ count: state.count }));

  return (
    <div>
      <div>
        <button
          type="primary"
          onClick={() => {
            /**触发更新 */
            dispatch({ type: "incremented" });
          }}
        >
          新增
        </button>
        <button
          onClick={() => {
            /**触发更新 */
            dispatch({ type: "decremented" });
          }}
        >
          减少
        </button>
      </div>
      <div>我是一个页面{count}</div>
    </div>
  );
}
export default App;

```

## Provider.tsx

```typescript

.....
function Provider<A extends Action = AnyAction, S = unknown>({
  /**外部数据store 例如 redux const store = legacy_createStore(reducer); */
  store,
  context,
  children,
  serverState,
  stabilityCheck = 'once',
  noopCheck = 'once',
}: ProviderProps<A, S>) {
  const contextValue = useMemo(() => {
    /**创建订阅类收集订阅函数 */
    const subscription = createSubscription(store)
    return {
      store,
      subscription,
      getServerState: serverState ? () => serverState : undefined,
      stabilityCheck,
      noopCheck,
    }
  }, [store, serverState, stabilityCheck, noopCheck])

  const previousState = useMemo(() => store.getState(), [store]);


/**
 * export const useIsomorphicLayoutEffect = canUseDOM ? useLayoutEffect : useEffect
 * 兼容服务端环境
 */
  useIsomorphicLayoutEffect(() => {
    const { subscription } = contextValue
    subscription.onStateChange = subscription.notifyNestedSubs
    subscription.trySubscribe()

    if (previousState !== store.getState()) {
      subscription.notifyNestedSubs()
    }
    /**解除订阅 */
    return () => {
      subscription.tryUnsubscribe()
      subscription.onStateChange = undefined
    }
  }, [contextValue, previousState])

  const Context = context || ReactReduxContext

  // @ts-ignore 'AnyAction' is assignable to the constraint of type 'A', but 'A' could be instantiated with a different subtype
  return <Context.Provider value={contextValue}>{children}</Context.Provider>
}

export default Provider

```

Provider 组件代码主要就是创建`Context.Provider`,传递 `subscription` 和 `store`等属性给子组件

## useSelector.ts

```typescript
/**
 * 默认对比函数
 * 
 */
const refEquality: EqualityFn<any> = (a, b) => a === b;

export function createSelectorHook(context = ReactReduxContext): UseSelector {
  const useReduxContext =
    context === ReactReduxContext
      ? useDefaultReduxContext
      : createReduxContextHook(context);

  return function useSelector<TState, Selected extends unknown>(
    selector: (state: TState) => Selected,
    equalityFnOrOptions:
      | EqualityFn<NoInfer<Selected>>
      | UseSelectorOptions<NoInfer<Selected>> = {}
  ): Selected {
    const {
      equalityFn = refEquality,
      stabilityCheck = undefined,
      noopCheck = undefined,
    } = typeof equalityFnOrOptions === "function"
      ? { equalityFn: equalityFnOrOptions }
      : equalityFnOrOptions;
    if (process.env.NODE_ENV !== "production") {
      if (!selector) {
        throw new Error(`You must pass a selector to useSelector`);
      }
      if (typeof selector !== "function") {
        throw new Error(
          `You must pass a function as a selector to useSelector`
        );
      }
      if (typeof equalityFn !== "function") {
        throw new Error(
          `You must pass a function as an equality function to useSelector`
        );
      }
    }

    /**非空断言 */
    const {
      store,
      subscription,
      getServerState,
      stabilityCheck: globalStabilityCheck,
      noopCheck: globalNoopCheck,
    } = useReduxContext()!;

    const firstRun = useRef(true);

    /**这里在selector函数上包裹了一层检验逻辑，
     */
    const wrappedSelector = useCallback<typeof selector>(
      {
        [selector.name](state: TState) {
          const selected = selector(state);
          if (process.env.NODE_ENV !== "production") {
            const finalStabilityCheck =
              typeof stabilityCheck === "undefined"
                ? globalStabilityCheck
                : stabilityCheck;
            if (
              finalStabilityCheck === "always" ||
              (finalStabilityCheck === "once" && firstRun.current)
            ) {
              const toCompare = selector(state);
              /**调用两次 selector函数对比
               *
               */
              if (!equalityFn(selected, toCompare)) {
                console.warn(
                  "Selector " +
                    (selector.name || "unknown") +
                    " returned a different result when called with the same parameters. This can lead to unnecessary rerenders." +
                    "\nSelectors that return a new reference (such as an object or an array) should be memoized: https://redux.js.org/usage/deriving-data-selectors#optimizing-selectors-with-memoization",
                  {
                    state,
                    selected,
                    selected2: toCompare,
                  }
                );
              }
            }
            const finalNoopCheck =
              typeof noopCheck === "undefined" ? globalNoopCheck : noopCheck;
            if (
              finalNoopCheck === "always" ||
              (finalNoopCheck === "once" && firstRun.current)
            ) {
              // @ts-ignore
              if (selected === state) {
                console.warn(
                  "Selector " +
                    (selector.name || "unknown") +
                    " returned the root state when called. This can lead to unnecessary rerenders." +
                    "\nSelectors that return the entire state are almost certainly a mistake, as they will cause a rerender whenever *anything* in state changes."
                );
              }
            }
            if (firstRun.current) firstRun.current = false;
          }
          return selected;
        },
      }[selector.name],
      [selector, globalStabilityCheck, stabilityCheck]
    );

    /**强化版的useSyncExternalStore */
    const selectedState = useSyncExternalStoreWithSelector(
      subscription.addNestedSub,
      store.getState,
      getServerState || store.getState,
      wrappedSelector,
      equalityFn
    );

    useDebugValue(selectedState);

    return selectedState;
  };
}

export const useSelector = /*#__PURE__*/ createSelectorHook();
```

useSelector主要是获取store中我们所需要的属性，通过`useSyncExternalStoreWithSelector`监听数据变化，控制渲染更新


## useSyncExternalStoreWithSelector（use-sync-external-store/cjs/use-sync-external-store-shim/with-selector.development.js）

```javascript

var useRef = React.useRef,
    useEffect = React.useEffect,
    useMemo = React.useMemo,
    useDebugValue = React.useDebugValue; // Same as useSyncExternalStore, but supports selector and isEqual arguments.

/**
 * subscribe 用于 useSyncExternalStore的subscribe参数
 * getSnapshot 获取整个store外部数据
 * getServerSnapshot 获取整个store外部数据（服务端）
 * selector 在原有基础上强化功能 memoizedSelector作为useSelector 第二个参数
 */
function useSyncExternalStoreWithSelector(subscribe, getSnapshot, getServerSnapshot, selector, isEqual) {
  // Use this to track the rendered snapshot.

  var instRef = useRef(null);
  var inst;

  if (instRef.current === null) {
    inst = {
      hasValue: false,
      value: null
    };
    instRef.current = inst;
  } else {
    inst = instRef.current;
  }

  var _useMemo = useMemo(function () {

    var hasMemo = false;
    //使用闭包记录外部数据
    var memoizedSnapshot;
    // 使用闭包记录selector函数返回快照
    var memoizedSelection;
    
    /**增强版的selector */
    var memoizedSelector = function (nextSnapshot) {
      if (!hasMemo) {
        // The first time the hook is called, there is no memoized result.
        hasMemo = true;
        memoizedSnapshot = nextSnapshot;

        var _nextSelection = selector(nextSnapshot);

        if (isEqual !== undefined) {
          if (inst.hasValue) {
            var currentSelection = inst.value;

            if (isEqual(currentSelection, _nextSelection)) {
              memoizedSelection = currentSelection;
              return currentSelection;
            }
          }
        }

        memoizedSelection = _nextSelection;
        return _nextSelection;
      } 

      var prevSnapshot = memoizedSnapshot;
      var prevSelection = memoizedSelection;

      /** 条件一  渲染前后对比store数据一样就返回 prevSelection*/ 
      if (objectIs(prevSnapshot, nextSnapshot)) {

        return prevSelection;
      }


      var nextSelection = selector(nextSnapshot);
      /** 条件二  渲染前后对比快照数据一样就返回 prevSelection*/ 
      if (isEqual !== undefined && isEqual(prevSelection, nextSelection)) {
        return prevSelection;
      }

      memoizedSnapshot = nextSnapshot;
      memoizedSelection = nextSelection;
      return nextSelection;
    }; 

    var maybeGetServerSnapshot = getServerSnapshot === undefined ? null : getServerSnapshot;

    var getSnapshotWithSelector = function () {
      return memoizedSelector(getSnapshot());
    };

    var getServerSnapshotWithSelector = maybeGetServerSnapshot === null ? undefined : function () {
      return memoizedSelector(maybeGetServerSnapshot());
    };
    return [getSnapshotWithSelector, getServerSnapshotWithSelector];
  }, [getSnapshot, getServerSnapshot, selector, isEqual]),
      getSelection = _useMemo[0],
      getServerSelection = _useMemo[1];

  //使用 getSelection（功能增强后的selector） 
  var value = useSyncExternalStore(subscribe, getSelection, getServerSelection);
  useEffect(function () {
    inst.hasValue = true;
    inst.value = value;
  }, [value]);
  useDebugValue(value);
  return value;
}


```

上述代码从原理来说就是一个增强版的`useSyncExternalStore`主要是对其第二个参数进行增强，`useSelector`第一个参数经过处理变成`getSelection = _useMemo[0]`然后传递给`useSyncExternalStore`的第二个参数

前面说过`useSyncExternalStore`每次渲染前后会调用第二个参数返回的快照来对比，如果不一致就重新渲染，
例如`const {a,b} = useSyncExternalStore((state)=>({a:state.a,b:state.b}))`,像这样使用进入页面就会报错的，因为每次都是返回一个新对象

`useSelector`的做法是劫持了传进来的参数`selector`,使用闭包保存一份快照备份`prevSelection`并返回给`useSyncExternalStore`内部,当我的数据满足上面两个条件就返回`prevSelection`,而不是返回一个新的对象，用此方法避免重复渲染,所以使用 `const {a,b} = useSelector((state)=>({a:state.a,b:state.b}))`进入页面时是不会报错的，因为这里满足条件一，此外有了`isEqual`函数我们可以对返回的数据快照做自定义的对比，比如返回的是对象，可以对渲染前后的两个对象进行深对比（react-redux提供的对应函数[shallowEqual](https://github.com/reduxjs/react-redux/blob/master/src/utils/shallowEqual.ts)），一样的话也返回`prevSelection`这样就可以避免store外部数据其他key值更新时引起不必要的更新

 





