---
title: mapMutations
author: 张凯强
date: 2020-12-31 00:00:00
index_img: https://rmt.dogedoge.com/fetch/fluid/storage/actions-deploy/cover.png?w=480&fmt=webp
category: vuex
tags:
  - vue全家桶
  - vuex
excerpt: 二春子的mapMutations
---

# mapMutations
完整代码分支 [stage-4](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#mapmutations)

你可以在组件中使用 this.$store.commit('xxx') 提交 mutation，或者使用 mapMutations 辅助函数将组件中的 methods 映射为 store.commit 调用（需要在根节点注入 store）

```js
import { mapMutations } from 'vuex'

export default {
  // ...
  methods: {
    ...mapMutations([
      'increment', // 将 `this.increment()` 映射为 `this.$store.commit('increment')`

      // `mapMutations` 也支持载荷：
      'incrementBy' // 将 `this.incrementBy(amount)` 映射为 `this.$store.commit('incrementBy', amount)`
    ]),
    ...mapMutations({
      add: 'increment' // 将 `this.add()` 映射为 `this.$store.commit('increment')`
    })
  }
}
```
```js{24,25}
export const mapMutations = normalizeNamespace((namespace, mutations) => {
  const res = {}
  if (__DEV__ && !isValidMap(mutations)) {
    console.error('[vuex] mapMutations: mapper parameter must be either an Array or an Object')
  }
  normalizeMap(mutations).forEach(({ key, val }) => {
    res[key] = function mappedMutation(...args) {
      // Get the commit method from store
      let commit = this.$store.commit
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapMutations', namespace)
        if (!module) {
          return
        }
        commit = module.context.commit
      }//获取模块对应的commit
      // 函数类型举例：
      // mapMutations({
      //   setUserName(commit, userName) {
      //     commit("setUserName", userName)
      //   }
      // })
      return typeof val === 'function'
        ? val.apply(this, [commit].concat(args))//如果是函数，函数的第一个参数是commit
        : commit.apply(this.$store, [val].concat(args))
    }
  })
  return res
})
```
