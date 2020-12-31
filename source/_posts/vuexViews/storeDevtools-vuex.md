# devtools
完整代码分支 [stage-2](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#devtools)
```js{6,17,26,31,37}
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {
    …………
    const useDevtools = options.devtools !== undefined ? options.devtools : Vue.config.devtools
    if (useDevtools) {
      devtoolPlugin(this)
    }
  }
}
const target = typeof window !== 'undefined'
  ? window
  : typeof global !== 'undefined'
    ? global
    : {}
const devtoolHook = target.__VUE_DEVTOOLS_GLOBAL_HOOK__ //装了devtools就有这个属性

export default function devtoolPlugin(store) {
  if (!devtoolHook) return

  store._devtoolHook = devtoolHook
  //devtool需要开始初始化vuex
  devtoolHook.emit('vuex:init', store)
  //触发时光旅行执行 store.replaceState
  devtoolHook.on('vuex:travel-to-state', targetState => {
    //通过传来的state,展示state的状态，达到时光旅行的效果
    store.replaceState(targetState) //可以看实例方法中replaceState
  })
  //订阅mutation,当有mutation执行后，会执行下面定义的函数
  store.subscribe((mutation, state) => {//可以看实例方法中subscribe
    //当mutation执行后，会通过devtools,保存当前快照（state）
    devtoolHook.emit('vuex:mutation', mutation, state)
  }, { prepend: true })//第一参数为函数， prepend: true，向数组的头部装入函数

  //订阅action,当有action执行后，会执行下面定义的函数
  store.subscribeAction((action, state) => {
    //当action执行后，会通过devtools,保存当前快照（state）
    devtoolHook.emit('vuex:action', action, state)
  }, { prepend: true })
}
```