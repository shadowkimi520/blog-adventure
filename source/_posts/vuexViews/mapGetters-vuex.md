# mapGetters
完整代码分支 [stage-4](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#mapgetters)

```js{17}
export const mapGetters = normalizeNamespace((namespace, getters) => {
  const res = {}
  if (__DEV__ && !isValidMap(getters)) {
    console.error('[vuex] mapGetters: mapper parameter must be either an Array or an Object')
  }
  normalizeMap(getters).forEach(({ key, val }) => {
    // The namespace has been mutated by normalizeNamespace
    val = namespace + val
    res[key] = function mappedGetter() {
      if (namespace && !getModuleByNamespace(this.$store, 'mapGetters', namespace)) {
        return //不存在这样的命名空间模块
      }
      if (__DEV__ && !(val in this.$store.getters)) {
        console.error(`[vuex] unknown getter: ${val}`)
        return
      }
      return this.$store.getters[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```
mapGetters 辅助函数仅仅是将 store 中的 getter 映射到局部计算属性中，关键代码：
```js
return this.$store.getters[val]
```