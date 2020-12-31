---
title: mapState
author: 张凯强
date: 2020-12-31 00:00:00
index_img: https://rmt.dogedoge.com/fetch/fluid/storage/actions-deploy/cover.png?w=480&fmt=webp
category: vuex
tags:
  - vue全家桶
  - vuex
excerpt: 二春子的mapState
---

# mapState
完整代码分支 [stage-4](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#mapstate)

当一个组件需要获取多个状态的时候，将这些状态都声明为计算属性会有些重复和冗余。为了解决这个问题，我们可以使用 mapState 辅助函数帮助我们生成计算属性，让你少按几次键：
```js
// 在单独构建的版本中辅助函数为 Vuex.mapState
import { mapState } from 'vuex'

export default {
  // ...
  computed: mapState({
    // 箭头函数可使代码更简练
    count: state => state.count,

    // 传字符串参数 'count' 等同于 `state => state.count`
    countAlias: 'count',

    // 为了能够使用 `this` 获取局部状态，必须使用常规函数
    countPlusLocalState (state) {
      return state.count + this.localCount
    }
  })
}
```
从代码可以看出：`mapState` 通过传入的参数解析后返回的是一个计算属性对象，引入的方式有两种：
+ Vuex.mapState
+ import { mapState } from 'vuex'

源码如下：
```js{7,13}
import { mapState } from './helpers'
// 这种模式 vuex.Store
export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
}
// 这种模式 import {Store} from vuex
export {
  Store,
  install,
  mapState
}
```
`helpers.js`

`normalizeNamespace` 执行返回一个函数赋值给mapState，此函数的执行直接执行 `fn`。也就是：
```js
mapState(namespace,map) --> fn(namespace,map)
```
```js{1,12}
function normalizeNamespace(fn) {
  return (namespace, map) => {
    if (typeof namespace !== 'string') {//可能大多数在使用mapState的时候不会传namespace   mapState({……})
      map = namespace
      namespace = '' //默认没有命名空间
    } else if (namespace.charAt(namespace.length - 1) !== '/') {
      namespace += '/'
    }
    return fn(namespace, map)
  }
}
export const mapState = normalizeNamespace(fn)
```
`mapState` 是有两个参数的：
+ 参数一，命名空间
+ 参数二，states

`normalizeNamespace` 方法对传参的各种情况做了处理，最终实现 fn 的第一个参数是命名空间，第二个参数是 state

`fn 函数`
```js{1,4,12,20,25-40,44}
function isValidMap(map) {//是否是数组或者对象类型
  return Array.isArray(map) || isObject(map)
}
function normalizeMap(map) {//标准化map最终返回 ｛key, val｝的形式
  if (!isValidMap(map)) {
    return []
  }
  return Array.isArray(map)
    ? map.map(key => ({ key, val: key }))
    : Object.keys(map).map(key => ({ key, val: map[key] }))
}
function getModuleByNamespace(store, helper, namespace) {//通过命名空间获取模块
  const module = store._modulesNamespaceMap[namespace]
  if (__DEV__ && !module) {
    console.error(`[vuex] module namespace not found in ${helper}(): ${namespace}`)
  }
  return module
}
export const mapState = normalizeNamespace((namespace, states) => {//fn
  const res = {}
  if (__DEV__ && !isValidMap(states)) {// mapState 只支持数组或者对象
    console.error('[vuex] mapState: mapper parameter must be either an Array or an Object')
  }
  normalizeMap(states).forEach(({ key, val }) => {
    res[key] = function mappedState() {
      //默认是找根模块的
      let state = this.$store.state
      let getters = this.$store.getters
      if (namespace) {//如果有命名空间的话，找相应的模块
        const module = getModuleByNamespace(this.$store, 'mapState', namespace)
        if (!module) {
          return
        }
        state = module.context.state
        getters = module.context.getters
      }
      return typeof val === 'function'
        ? val.call(this, state, getters) //函数直接参数传state,getters
        : state[val] //非函数直接在state中获取
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```
最终返回 `res` 计算属性对象，`25-40` 行是定义每个 `key` 的计算属性的主体。参数定义了命名空间，就会找相应模块的 `state,getters` 进行处理，未定义命名空间则默认为根模块。

+ 定义的主体是函数就执行此函数，参数为 `state getters`
+ 非函数就当做属性从 `state` 中取值返回
```js{9,12,15}
return typeof val === 'function'
        ? val.call(this, state, getters) //函数直接参数传state,getters
        : state[val] //非函数直接在state中获取
//例子
export default {
  // ...
  computed: mapState({
    // 箭头函数可使代码更简练
    count: state => state.count,

    // 传字符串参数 'count' 等同于 `state => state.count`
    countAlias: 'count',

    // 为了能够使用 `this` 获取局部状态，必须使用常规函数
    countPlusLocalState (state) {
      return state.count + this.localCount
    }
  })
}
```

