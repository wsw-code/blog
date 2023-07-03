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
/**subscribe函数中的listener集合  */
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
