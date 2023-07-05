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
- 改变外部数据,调用`forceUpdate`函数更新 UI，`forceUpdate`内部再次调用第二个参数返回的快照数据跟上一步的快照做浅比较`Object.is`,如果不一致就会重新渲染

## Provider 组件

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

  const previousState = useMemo(() => store.getState(), [store])

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
