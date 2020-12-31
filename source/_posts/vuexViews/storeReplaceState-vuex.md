# replaceState
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#replacestate)
```js
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {}
  ……
  //时光旅行，他可能会返回之前某个时间点的state,然后赋值此state,达到当前展示之前某个时间的快照
  replaceState(state) {
    this._withCommit(() => {
      this._vm._data.$$state = state
    })
  }
}
```
有时候我们可能需要时光旅行，希望回到之前或者之后的某个状态点。如果我们保存了状态点的 `state`，就可以通过 `this.$store.replaceState(state)` 来实现，非常方便