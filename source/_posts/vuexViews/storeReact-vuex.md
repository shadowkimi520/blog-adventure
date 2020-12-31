# $Store 响应式
完整代码分支 [stage-2](https://github.com/shengrongchun/parse-vue-vuex)

## $store.state
我们知道 `$store` 中的 `state` 包括所有模块的 `state` 都是响应式的，`vuex` 对于它的实现放在了 `resetStoreVM` 方法中
```js{9,16}
//
this._actions = Object.create(null) //存储定义的所有actions
this._mutations = Object.create(null)//存储定义的所有mutations
this._wrappedGetters = Object.create(null)//存储定义的所有getters
this._modules = new ModuleCollection(options) //收集配置文件中定义的模块，并且返回树状模块数据结构
this._modulesNamespaceMap = Object.create(null) //命名空间与模块映射
this._makeLocalGettersCache = Object.create(null)//存储非根模块getters
//
const state = this._modules.root.state //根实例的state 此时还未是响应式的
// init root module.
// this also recursively registers all sub-modules 递归
// and collects all module getters inside this._wrappedGetters
installModule(this, state, [], this._modules.root)
// initialize the store vm, which is responsible for the reactivity
// (also registers _wrappedGetters as computed properties)
resetStoreVM(this, state)
```
直接看 `resetStoreVM`
```js{7-11}
function resetStoreVM(store, state, hot) {
  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true //取消 Vue 所有的日志与警告
  store._vm = new Vue({
    data: {
      $$state: state //state响应式
    }
  })
  Vue.config.silent = silent
}
```
`$store` 对象的实例方法
```js{3}
//实例方法
get state() {// $store.state --> 根实例的state
  return this._vm._data.$$state
}
set state(v) {
  if (__DEV__) {
    assert(false, `use store.replaceState() to explicit replace store state.`)
  }
}
```
由上面的代码可以获知 `$store.state` 之所以是响应式的是通过 `new Vue` 实例，把 `state` 放入 `data` 中。而根模块的 `state` 是包括所有模块 `state` 的树状结构，因此所有模块的 `state` 都是响应式的

## $store.getters
如果要把所有模块定义的 `getters` 定义为计算属性该怎么办呢？
```js{12,14,27}
function resetStoreVM(store, state, hot) {
  // bind store public getters
  store.getters = {}
  // reset local getters cache
  store._makeLocalGettersCache = Object.create(null)
  const wrappedGetters = store._wrappedGetters //存着所有模块的 getters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    // direct inline function use will lead to closure preserving oldVm.
    // using partial to return function with only arguments preserved in closure environment.
    computed[key] = partial(fn, store)//partial 到工具-方法中查看
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })
  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true //取消 Vue 所有的日志与警告
  store._vm = new Vue({
    data: {
      $$state: state //state响应式
    },
    computed
  })
  Vue.config.silent = silent
}
```
是不是已经明白了，收集的 `getters` 又重新在新创建的 `vue` 实例中定义成计算属性

## 重新设置 vm
`resetStoreVM` 方法不仅初始化的时候用，运行时也会触发，因此需要有销毁之前 `vm` 的机制
```js{2,33-42}
function resetStoreVM(store, state, hot) {
  const oldVm = store._vm

  // bind store public getters
  store.getters = {}
  // reset local getters cache
  store._makeLocalGettersCache = Object.create(null)
  const wrappedGetters = store._wrappedGetters //存着所有模块的 getters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    // direct inline function use will lead to closure preserving oldVm.
    // using partial to return function with only arguments preserved in closure environment.
    computed[key] = partial(fn, store)//partial 到工具-方法中查看
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })
  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true //取消 Vue 所有的日志与警告
  store._vm = new Vue({
    data: {
      $$state: state //state响应式
    },
    computed
  })
  Vue.config.silent = silent

  if (oldVm) {//销毁
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```