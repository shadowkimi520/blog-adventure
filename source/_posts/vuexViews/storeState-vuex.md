# state
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#state-2)

`state` 为 `store` 实例对象：
```js{8}
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {
    …………
  }
  …………
  //实例方法
  get state() {// store.state --> 根实例的state
    return this._vm._data.$$state //此行不明白可以查看 $store响应式章节
  }
  set state(v) {
    if (__DEV__) {
      assert(false, `use store.replaceState() to explicit replace store state.`)
    }
  }
  …………
}
```
因此我们在组件中可以这样使用：`this.$store.state`。注意：此 `state` 只读