# Redux 源码笔记

createStore函数是redux的核心主要结构为,其代码主要结构为：
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

redux是一个发布订阅模式，按上述结构实现一个简易版，代码如下：
```javascript
function counterReducer(state = { value: 0 }, action) {
  switch (action.type) {
    case 'counter/incremented':
      return { value: state.value + 1, }
    case 'counter/decremented':
      return { value: state.value - 1 }
    default:
      return state
  }
}

function createStore (reducer,preloadedState) {
  let currentState = preloadedState;
  let currentReducer = reducer;
  let listeners = [];
  // 获取数据
  function getState() {
     
    return currentState
  }

  // 订阅数据函数
  function subscribe(listenerFn) {
    listeners.push(listenerFn);

    return function() {
      const index = listeners.indexOf(listenerFn)
      listeners.splice(index, 1);
      console.log( `取消订阅第${index+1}个订阅`)
    }
  }

  // 触发数据更新函数
  function dispatch(action) {
    currentState = currentReducer(currentState, action)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }
  }
    
  // ...其他函数
  return {
      dispatch,
      subscribe,
      getState,
  }
}

var store = createStore(counterReducer,{ value: 0 });
var fn = () => console.log('我第一个订阅了数据')
var fn2 = () => console.log('我第二个订阅了数据')
var fn3 = () => console.log('我第三个订阅了数据')
var fn4 = () => console.log('我第四个订阅了数据')

var unsubscribe = store.subscribe(fn)
var unsubscribe2 = store.subscribe(fn2)
var unsubscribe3 = store.subscribe(fn3)
var unsubscribe4=  store.subscribe(fn4)

store.dispatch({ type: 'counter/incremented' })

unsubscribe4();

store.dispatch({ type: 'counter/incremented' })

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

按上述代码简易的实现了一个redux,但对比redux源码比较还是有区别的，简易版的redux维护了一个listeners的变量用于存放订阅函数的变量，但是redux源码上是用了两个变量来维护订阅者函数，使用两个变量来维护源码上有注释
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
上述意思就是防止在dispatch时发生订阅/解除订阅所产生的bug,下面已一个例子来说明

````javascript

// 在简易版的redux基础上
var fn = () => console.log('我第一个订阅了数据')
var fn2 = () => console.log('我第二个订阅了数据')
var fn3 = () => console.log('我第三个订阅了数据')
var fn4 = () => console.log('我第四个订阅了数据')

var unsubscribe = store.subscribe(fn)
var unsubscribe2 = store.subscribe(fn2)
var unsubscribe3 = store.subscribe(fn3)
var unsubscribe4=  store.subscribe(fn4)

var fn5 = () => {
    console.log('我第五个订阅了数据')
    unsubscribe() // 取消第一个订阅
}

var fn6 = () => console.log('我第六个订阅了数据')
var unsubscribe5=  store.subscribe(fn5)
var unsubscribe6=  store.subscribe(fn6)

store.dispatch({ type: 'counter/incremented' })

````
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

redux中每次dispath中所做的订阅/解除订阅是不会改变当前正在遍历的订阅函数数组，而是放到下一次的dispath中，我们在上一个例子上的代码后面再加一行代码`store.dispatch({ type: 'counter/incremented' })`,也就是重复两次dispatch,打印结果如下：
```javascript
//第一次dispath
我第一个订阅了数据
我第二个订阅了数据
我第三个订阅了数据
我第四个订阅了数据
我第五个订阅了数据
我第六个订阅了数据
取消订阅第1个订阅

//第二次dispath
我第二个订阅了数据
我第三个订阅了数据
我第四个订阅了数据
我第五个订阅了数据
我第六个订阅了数据

```


