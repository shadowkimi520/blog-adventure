# watch
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#watch)

```js{4,9}
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {
    ……
    this._watcherVM = new Vue()//vue实例
    ……
  }
  ……
  //响应式地侦听 fn 的返回值，当值改变时调用回调函数
  watch(getter, cb, options) {
    if (__DEV__) {
      assert(typeof getter === 'function', `store.watch only accepts a function.`)
    }
    return this._watcherVM.$watch(() => getter(this.state, this.getters), cb, options)
  }
}
```
`getter` 必须是函数类型，监听到 `getter` 返回值变化会立即执行 `cb` 。可查看 `vue $watch` 了解更多