# Redux v4+ 源码笔记

## createStore 实现

createStore 函数是 redux 的核心主要结构为,其代码主要结构为：

```javascript
function createStore(reducer) {
  let currentReducer = reducer
  // 获取数据
  function getState() {
  }
  // 订阅数据函数
  function subscribe() {
  }
	// 触发数据更新函数
	function dispatch(action) {
        ....
  }
  // ...其他函数
	return {
        dispatch,
        subscribe,
        getState,
    }

}
```

redux 中的 createStore 是一个发布订阅的设计模式，按上述结构实现一个简易版，代码如下：

```javascript
function counterReducer(state = { value: 0 }, action) {
  switch (action.type) {
    case "counter/incremented":
      return { value: state.value + 1 };
    case "counter/decremented":
      return { value: state.value - 1 };
    default:
      return state;
  }
}

function createStore(reducer, preloadedState) {
  let currentState = preloadedState;
  let currentReducer = reducer;
  let listeners = [];
  // 获取数据
  function getState() {
    return currentState;
  }

  // 订阅数据函数
  function subscribe(listenerFn) {
    listeners.push(listenerFn);

    // 返回一个解除订阅的函数
    return function () {
      const index = listeners.indexOf(listenerFn);
      listeners.splice(index, 1);
      console.log(`取消订阅第${index + 1}个订阅`);
    };
  }

  // 触发数据更新函数
  function dispatch(action) {
    //根据action更新store
    currentState = currentReducer(currentState, action);
    // 遍历订阅函数并执行
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i];
      listener();
    }
  }

  // ...其他函数
  return {
    dispatch,
    subscribe,
    getState,
  };
}

var store = createStore(counterReducer, { value: 0 });
var fn = () => console.log("我第一个订阅了数据");
var fn2 = () => console.log("我第二个订阅了数据");
var fn3 = () => console.log("我第三个订阅了数据");
var fn4 = () => console.log("我第四个订阅了数据");

var unsubscribe = store.subscribe(fn);
var unsubscribe2 = store.subscribe(fn2);
var unsubscribe3 = store.subscribe(fn3);
var unsubscribe4 = store.subscribe(fn4);

store.dispatch({ type: "counter/incremented" });

unsubscribe4();

store.dispatch({ type: "counter/incremented" });

// 打印结果：
// 我第一个订阅了数据
// 我第二个订阅了数据
// 我第三个订阅了数据
// 我第四个订阅了数据
// 取消订阅第4个订阅
// 我第一个订阅了数据
// 我第二个订阅了数据
// 我第三个订阅了数据
```

按上述代码简易的实现了一个 redux,但对比 redux 源码比较还是有区别的，简易版的 redux 维护了一个 listeners 的变量用于存放订阅函数的变量，但是 redux 源码上是用了两个变量来维护订阅函数，使用两个变量来维护源码上有注释

```
  /**
   * This makes a shallow copy of currentListeners so we can use
   * nextListeners as a temporary list while dispatching.
   *
   * This prevents any bugs around consumers calling
   * subscribe/unsubscribe in the middle of a dispatch.
   */

   function ensureCanMutateNextListeners() {
    ....
   }
```

上述意思就是防止在 dispatch 时发生订阅/取消订阅所产生的 bug,下面已一个例子来说明

```javascript
// 在简易版的redux基础上
var fn = () => console.log("我第一个订阅了数据");
var fn2 = () => console.log("我第二个订阅了数据");
var fn3 = () => console.log("我第三个订阅了数据");
var fn4 = () => console.log("我第四个订阅了数据");

var unsubscribe = store.subscribe(fn);
var unsubscribe2 = store.subscribe(fn2);
var unsubscribe3 = store.subscribe(fn3);
var unsubscribe4 = store.subscribe(fn4);

var fn5 = () => {
  console.log("我第五个订阅了数据");
  unsubscribe(); // 取消第一个订阅
};

var fn6 = () => console.log("我第六个订阅了数据");
var unsubscribe5 = store.subscribe(fn5);
var unsubscribe6 = store.subscribe(fn6);

store.dispatch({ type: "counter/incremented" });
```

从上述代码上可以看到我在第五个订阅函数里又取消了第一个订阅，打印结果

```
我第一个订阅了数据
我第二个订阅了数据
我第三个订阅了数据
我第四个订阅了数据
我第五个订阅了数据
取消订阅第1个订阅
第六个订阅没有被取消但是跳过去了，没有触发
```

redux 中每次 dispatch 中所做的订阅/取消订阅是不会改变当前正在遍历的订阅函数数组，而是放到下一次的 dispatch 中，每次订阅/取消订阅前都会执行一个函数,对订阅数组进行拷贝

```javascript
function ensureCanMutateNextListeners() {
  if (nextListeners === currentListeners) {
    nextListeners = currentListeners.slice();
  }
}
```

我们在上一个例子上的代码改为使用 redux,然后再加一行代码`store.dispatch({ type: 'counter/incremented' })`,也就是重复两次 dispatch,打印结果如下：

```javascript
//第一次dispatch
我第一个订阅了数据;
我第二个订阅了数据;
我第三个订阅了数据;
我第四个订阅了数据;
我第五个订阅了数据;
取消订阅第1个订阅;
我第六个订阅了数据;

//第二次dispatch,第一个订阅已经取消了
我第二个订阅了数据;
我第三个订阅了数据;
我第四个订阅了数据;
我第五个订阅了数据;
我第六个订阅了数据;
```

react-redux v7.1.3 中的`createListenerCollection`函数跟 redux 中的`createStore`函数从实现方式上来看是是差不多的，都是使用数组储存订阅函数,[react-redux v7.1.3 部分源码链接](https://github.com/reduxjs/react-redux/blob/v7.1.3/src/utils/Subscription.js),但是往后迭代 react-redux 就改为用链表的数据结构来代替数组来存储订阅函数[react-redux v7.2.0 部分源码链接](https://github.com/reduxjs/react-redux/blob/v7.2.0/src/utils/Subscription.js),原因是数组删除性能较差,看下面性能对比
||数组|链表|  
| ---- | ---- | ----|
|查询|O(1)|O(n)|
|插入|O(n)|O(1)|
|删除|O(n)|O(1)|

可以看到链表的插入和删除性能都要比数组好，而就应用场景来说，react-redux 更多的交互的插入/删除，所以改为链表形式，react-redux 该性能问题对应的[issue](https://github.com/reduxjs/react-redux/pull/1523)

## 中间件

中间件的功能其实就是增强 dispatch 的功能，我们通过中间件可以做日志记录、调用异步接口等功能，下面简单举个例子

```javascript
//.....省略部分代码

var logger1 =
  ({ getState }) =>
  (next) =>
  (action) => {
    console.log("before dispatch1", action);
    const returnValue = next(action);
    console.log("after dispatch1", "getState()获取store");
    return returnValue;
  };
const store = createStore(reducer, initState, applyMiddleware(logger1));

store.dispatch({ type: "INCREMENT" });

//打印结果

// before dispatch1 {type:'INCREMENT'}
// after dispatch1  'getState()获取store'
```

上述的中间件可以记录每次 dispatch 的 action 和 reducer 更改后的 state 值，再上述例子上再加一个中间件

```javascript
//.....省略部分代码

var logger1 =
  ({ getState }) =>
  (next) =>
  (action) => {
    console.log("before dispatch1", action);
    const returnValue = next(action);
    console.log("after dispatch1", "getState()获取store");
    return returnValue;
  };

const logger2 =
  ({ getState }) =>
  (next) =>
  (action) => {
    console.log("before dispatch2", action);
    const returnValue = next(action);
    console.log("after dispatch2", "getState()获取store");
    return returnValue;
  };

const store = createStore(
  reducer,
  initState,
  applyMiddleware(logger1, logger1)
);

store.dispatch({ type: "INCREMENT" });

//打印结果

// before dispatch1 {type:'INCREMENT'}
// before dispatch2 {type:'INCREMENT'}
// after dispatch1  'getState()获取store'
// after dispatch2  'getState()获取store'
```

从打印结果可以看到此打印顺序是先从外到里再由里到外，也就是洋葱模型机制，下面是`applyMiddleware`函数源码分析

```javascript
//applyMiddleware会返回一个参数为createStore的函数
// 在createStore中最终实现的效果为applyMiddleware(middlewares1,middlewares2)(createStore)(createStore函数原来的参数)
//返回增强后的dispatch

export default function applyMiddleware(...middlewares) {
  return (createStore) =>
    (...args) => {
      const store = createStore(...args);
      let dispatch = () => {
        throw new Error(
          "Dispatching while constructing your middleware is not allowed. " +
            "Other middleware would not be applied to this dispatch."
        );
      };

      const middlewareAPI = {
        getState: store.getState,
        dispatch: (...args) => dispatch(...args),
      };
      // 给每个中间件注入{dispatch，getState}

      const chain = middlewares.map((middleware) => middleware(middlewareAPI));

      // 这里通过compose这个组合函数把中间件组合起来
      const fn = compose(...chain);

      /**注入dispatch */
      dispatch = fn(store.dispatch);

      return {
        ...store,
        dispatch,
      };
    };
}
```

现在我们把 applyMiddleware 的参数定为 logger1,logger2 来分析源码
经过`const chain = middlewares.map((middleware) => middleware(middlewareAPI))`现在 chain 等于

```javascript
[
  (next) => (action) => {
    // logger1
    console.log("before dispatch2", action);
    const returnValue = next(action);
    console.log("after dispatch2", "getState()获取store");
    return returnValue;
  },
  (next) => (action) => {
    // logger2
    console.log("before dispatch2", action);
    const returnValue = next(action);
    console.log("after dispatch2", "getState()获取store");
    return returnValue;
  },
];
```

compose 函数是一个组合函数，参数为函数，其效果就是把`compose(a,b,c)=>a(b(c(args)))`,举个例子

```javascript
var fn1 = (n) => {
  return n + 1;
};
var fn2 = (n) => {
  return n * 2;
};

var fn3 = (n) => {
  return n - 3;
};

var fn4 = compose(fn1, fn2, fn3);
//=>

var fn4 = (n) => fn1(fn2(fn3(n)));

fn4(10);
// fn3(10)=>7=>fn2(7)=>14=>fn(14)=>15
```

其实就是把函数按顺序执行，执行返回值当作下一个执行函数的参数再执行，一直执行到最后一个函数,compose 函数源码如下

```javascript
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return (arg) => arg;
  }

  if (funcs.length === 1) {
    return funcs[0];
  }

  return funcs.reduce(
    (a, b) =>
      (...args) =>
        a(b(...args))
  );
}
```

回来上面 applyMiddleware 函数分析上，但执行到`const fn = compose(...chain);`此时 fn 等于

```javascript
(...args) => logger1(logger2(args));

//注意这里的logger1是(next) => (action)=>{},这里是简单的替代一下，方便理解
```

然后执行`dispatch = fn(store.dispatch)`,返回一个强化后的 dispatch,这里先执行`logger2(store.dispatch)`返回`(action)=>{...}`然后返回的函数作为参数`next`传到 logger1 上`(next) => (action)=>{}`执行 logger1 函数再返回`(action)=>{}`
这里可以看到函数从里开始执行一直在返回函数并作为参数给下一个函数，所以最后形成的代码是

```javascript
(action) => {
  console.log("before dispatch1", action);

  //const returnValue = next(action);
  //这里的next是logger2返回的函数也就等于
  const returnValue = ((action) => {
    console.log("before dispatch2", action);
    //这里的next就等于执行"dispatch = fn(store.dispatch)"传递进来的store.dispatch
    const returnValue = next(action);
    console.log("after dispatch2", "getState()获取store");
    return returnValue;
  })(action);

  console.log("after dispatch1", "getState()获取store");
  return returnValue;
};
```

然后试着多加一个中间件 logger3

```javascript
(action) => {
  console.log("before dispatch1", action);

  //const returnValue = next(action);
  //这里的next是logger2返回的函数也就等于
  const returnValue = ((action) => {
    console.log("before dispatch2", action);

    //这里的next是logger3返回的函数也就等于
    const returnValue = ((action) => {
      console.log("before dispatch3", action);
      //这里的next就等于执行"dispatch = fn(store.dispatch)"传递进来的store.dispatch
      const returnValue = next(action);
      console.log("after dispatch3", "getState()获取store");
      return returnValue;
    })(action);

    console.log("after dispatch2", "getState()获取store");
    return returnValue;
  })(action);

  console.log("after dispatch1", "getState()获取store");
  return returnValue;
};
```

从上面的代码可以看出最终返回的代码结构是`(action)=>{...next()...}`这里的 next 指的是下一个中间件返回的函数也是同样的结构`(action)=>{...next()...}`,所以添加中间件就等于嵌套返回的 next 的函数，而最后一个函数中`next=store.dispatch`
