# hasModule
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#hasmodule)
```js{8}
//检查该模块的名字是否已经被注册
hasModule(path) {
  if (typeof path === 'string') path = [path]

  if (__DEV__) {
    assert(Array.isArray(path), `module path must be a string or an Array.`)
  }
  return this._modules.isRegistered(path)
}
```
`this._modules.isRegistered(path)`
```js{3-6}
//是否是已经注册过的模块
isRegistered(path) {
  const parent = this.get(path.slice(0, -1))
  const key = path[path.length - 1]

  return parent.hasChild(key)
}
```