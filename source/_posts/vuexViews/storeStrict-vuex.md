# 严格模式 strict
完整代码分支 [stage-2](https://github.com/shengrongchun/parse-vue-vuex)

首先需要知道严格模式是什么？[官网介绍](https://vuex.vuejs.org/zh/api/#strict)
```js{4,7,16-18,20,25}
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {
    const {// 用户定义是否严格模式，默认非严格
      strict = false//在严格模式下，任何 mutation 处理函数以外修改 Vuex state 都会抛出错误
    } = options
    // store internal state
    this._committing = false //state改变是否触发警告标识(严格模式下在mutations以外处修改state会触发警告)
    // initialize the store vm, which is responsible for the reactivity
    // (also registers _wrappedGetters as computed properties)
    resetStoreVM(this, state)
  }
}
//
function resetStoreVM(store, state, hot) {
  // enable strict mode for new vm
  if (store.strict) {//严格模式
    enableStrictMode(store)
  }
}
function enableStrictMode(store) {//监听 state 的变化
  store._vm.$watch(function () { return this._data.$$state }, () => {
    if (__DEV__) {
      //当store._committing为false,并且 state变化了，才会警告
      //
      assert(store._committing, `do not mutate vuex store state outside mutation handlers.`)
    }
  }, { deep: true, sync: true })
}
```
相关代码都展示了，当用户通过配置信息定义严格模式的时候，会执行 `enableStrictMode` 方法。当 `state` 发生变化时并且 `_committing` 是 `false` 的情况下会发出警告。其实 `vuex` 内部在多处都在改变 `state` 。为了不发生警告，所有改变的地方包括 `mutations` 都用 `_withCommit` 方法包裹
```js{3}
_withCommit(fn) {
  const committing = this._committing
  this._committing = true //state改变不要警告标识
  fn()
  this._committing = committing
}
```
看代码一目了然，在执行 `fn` 期间，`_committing=true` 所以当用户定义了严格模式，改变 `state` 没有用此方法包裹都会发出警告