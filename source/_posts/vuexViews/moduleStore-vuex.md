---
title: module 与模块收集
author: 张凯强
date: 2020-12-31 00:00:00
index_img: https://rmt.dogedoge.com/fetch/fluid/storage/actions-deploy/cover.png?w=480&fmt=webp
category: vuex
tags:
  - vue全家桶
  - vuex
excerpt: 二春子的module 与模块收集
---


# module 与模块收集
[官网介绍](https://vuex.vuejs.org/zh/guide/modules.html)

完整代码分支 [stage-1](https://github.com/shengrongchun/parse-vue-vuex)

## module
由于使用单一状态树，应用的所有状态会集中到一个比较大的对象。当应用变得非常复杂时，`store` 对象就有可能变得相当臃肿。

为了解决以上问题，`Vuex` 允许我们将 `store` 分割成 `模块（module）`。每个模块拥有自己的 `state、mutation、action、getter`、甚至是嵌套子模块——从上至下进行同样方式的分割。我们可以把最外层当作根模块，内部可以定义子模块，子模块里也可以定义子模块……
```js
const moduleA = {
  state: () => ({ ... }),
  mutations: { ... },
  actions: { ... },
  getters: { ... }
}
const moduleB = {
  state: () => ({ ... }),
  mutations: { ... },
  actions: { ... }
}
const store = new Vuex.Store({//options
  modules: {
    a: moduleA,
    b: moduleB
  }
})
store.state.a // -> moduleA 的状态
store.state.b // -> moduleB 的状态
```
## 模块收集
我们需要梳理并收集用户定义的模块，以一种树状结构存入根模块中供后续方便使用，大致结构应该像这样：
```js{3-6}
const rootModule = {//根模块
  _children: {
    moduleA: {
      _children: {……}
    },
    moduleB: {……}
  }
}
```

`store.js`
```js{16}
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue) //window上有Vue,并且没有Vue(没有Vue.use)会自动安装
    }
    if (__DEV__) {//开发环境会有一些警告。比如创建store实例前需要执行Vue.use(Vuex)，
    // vuex需要浏览器支持Promise, 必须 new Store等
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
      assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
      assert(this instanceof Store, `store must be called with the new operator.`)
    }
    //
    this._modules = new ModuleCollection(options) //收集配置文件中定义的模块，并且返回树状模块数据结构
  }
}
```
`options` 是用户定义的配置信息，通过 `new ModuleCollection` 把定义的模块信息收集起来
::: tip 提示
`this._modules = new ModuleCollection(options)` 直接到 `辅助` 目录下查看
:::
## `actions/mutations/getters` 收集
如果你看完 `ModuleCollection` 会发现 `this._modules` 中多了一个 `root` 属性。接下来我们需要根据此 `root` 来收集 `actions mutations getters`

```js{2,3,4,6,7,13}
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
```
收集相关类型的对象已经定义好，接下来就是 `installModule` 方法做收集的工作了，请在 `辅助` 目录下查看