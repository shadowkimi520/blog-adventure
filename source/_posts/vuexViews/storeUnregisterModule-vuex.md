# unregisterModule
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#unregistermodule)
```js{9,11-12,14}
//卸载一个动态模块,初始化定义的模块是无法卸载的
unregisterModule(path) {
  if (typeof path === 'string') path = [path]

  if (__DEV__) {
    assert(Array.isArray(path), `module path must be a string or an Array.`)
  }

  this._modules.unregister(path)
  this._withCommit(() => {//删除模块的state
    const parentState = getNestedState(this.state, path.slice(0, -1))
    Vue.delete(parentState, path[path.length - 1])
  })
  resetStore(this)
}
```
`this._modules.unregister`
```js{17}
//卸载模块
unregister(path) {
  const parent = this.get(path.slice(0, -1))
  const key = path[path.length - 1]
  const child = parent.getChild(key)

  if (!child) {
    if (__DEV__) {
      console.warn(
        `[vuex] trying to unregister module '${key}', which is ` +
        `not registered`
      )
    }
    return
  }

  if (!child.runtime) {
    return
  }

  parent.removeChild(key)
}
```
获取此模块，然后在其父模块的 `_children` 中移除，`17` 行代码已经限制只有动态模块才可以移除
`resetStore(this)`
```js
function resetStore(store, hot) {
  store._actions = Object.create(null)
  store._mutations = Object.create(null)
  store._wrappedGetters = Object.create(null)
  store._modulesNamespaceMap = Object.create(null)
  const state = store.state
  // init all modules
  installModule(store, state, [], store._modules.root, true)
  // reset vm
  resetStoreVM(store, state, hot)
}
```
当我们删除模块的时候，需要再一次初始化所有模块以达到删除此模块的相关信息