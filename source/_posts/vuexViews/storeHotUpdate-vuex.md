# hotUpdate
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#hotupdate)
```js{3}
//热替换新的 action 和 mutation
hotUpdate(newOptions) {
  this._modules.update(newOptions)
  resetStore(this, true)
}
```
`newOptions` 包含着各种配置信息，如： `actions` `mutations` `modules` 等

`this._modules.update(newOptions)`
```js{2,9}
update(rawRootModule) {
  update([], this.root, rawRootModule)
}
function update(path, targetModule, newModule) {
  if (__DEV__) {//非生产环境，检测模块中 actions mutations getters 类型是否正确
    assertRawModule(path, newModule)
  }
  // update target module
  targetModule.update(newModule)
  // update nested modules
  if (newModule.modules) {//更新子模块
    for (const key in newModule.modules) {
      if (!targetModule.getChild(key)) {
        if (__DEV__) {
          console.warn(
            `[vuex] trying to add a new module '${key}' on hot reloading, ` +
            'manual reload is needed'
          )
        }
        return
      }
      update(
        path.concat(key),
        targetModule.getChild(key),
        newModule.modules[key]
      )
    }
  }
}
```
`targetModule.update`
```js
update(rawModule) {//更新模块
  this._rawModule.namespaced = rawModule.namespaced
  if (rawModule.actions) {
    this._rawModule.actions = rawModule.actions
  }
  if (rawModule.mutations) {
    this._rawModule.mutations = rawModule.mutations
  }
  if (rawModule.getters) {
    this._rawModule.getters = rawModule.getters
  }
}
```