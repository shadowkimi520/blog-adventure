# registerModule
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#registermodule)
```js
//注册一个动态模块，有时候我们想加一个模块，此模块配置文件中没有定义，所以可以注册一个动态的模块
registerModule(path, rawModule, options = {}) {
  if (typeof path === 'string') path = [path]

  if (__DEV__) {
    assert(Array.isArray(path), `module path must be a string or an Array.`)
    assert(path.length > 0, 'cannot register the root module by using registerModule.')
  }

  this._modules.register(path, rawModule)
  installModule(this, this.state, path, this._modules.get(path), options.preserveState)
  // reset store to update getters...
  resetStoreVM(this, this.state)
}
```
我们注册模块一般在初始化阶段，写在配置文件中，如：
```js
new vuex.Store({
  modules: {
    模块A: {},
    模块B: {},
    ……
  }
})
```
在某些场景下，我们需要动态添加注册模块C。可以通过 `this.$store.registerModule` 完成