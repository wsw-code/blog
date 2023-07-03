# React-redux  源码笔记

react 自v16.8引入hooks的api后，官方推荐函数组件+hooks编写React代码,本文主要围绕react-redux的hooks来对源码进行分析，进行源码分析前首先了解一下`React.useSyncExternalStore`

## useSyncExternalStore

`useSyncExternalStore` 是一个让你订阅外部数据store的hooks。React18 带来了Concurrent 模式，在开启此模式下一些redux,mobx等第三方状态库会出现订阅同一个数据但是UI展示不一致的情况，也就是数据撕裂[(tearing)](https://github.com/reactwg/react-18/discussions/69)。针对此问题，React 提供了`useSyncExternalStore` React hooks
`useSyncExternalStore`的参数有三个：
+ subscribe,可以接受一个callback参数的函数，并返回一个清除订阅函数
+ getSnapshot，一个函数返回store中的快照
+ getServerSnapshot，可选一个供服务端使用的函数



