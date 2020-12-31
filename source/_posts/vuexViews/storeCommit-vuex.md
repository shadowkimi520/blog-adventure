# commit
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#commit)

`store` 的实例方法 `commit`
```js{6,13,23,27,36-38}
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {
    …………
    //装入用户自定义的一些函数，当有mutation执行后，会执行此数组中的函数
    // 实例对象的 this.$store.subscribe 方法是装入功能
    this._subscribers = [] 

    // bind commit and dispatch to self
    const store = this //this.$store 实例
    const {  commit } = this 
    //中转下是保证方法里的this是store
    this.commit = function boundCommit(type, payload, options) {// this.$store.commit
      return commit.call(store, type, payload, options) 
    }
  }
  …………
  commit(_type, _payload, _options) {
    // check object-style commit type通过unifyObjectStyle解析起到支持对象风格的提交方式
    const {
      type,
      payload,
      options
    } = unifyObjectStyle(_type, _payload, _options)

    const mutation = { type, payload }
    //获取 type的mutation，之前说过它可能是有多个函数的数组，所以要遍历执行
    const entry = this._mutations[type]
    if (!entry) {
      if (__DEV__) {
        console.error(`[vuex] unknown mutation type: ${type}`)
      }
      return
    }
    //_withCommit很重要，作用是fn里面改变state,不会触发警告
    this._withCommit(() => {
      entry.forEach(function commitIterator(handler) {
        handler(payload)
      })
    })
    //mutation执行完后，执行订阅的回调函数
    this._subscribers
      .slice() // shallow copy to prevent iterator invalidation if subscriber synchronously calls unsubscribe
      .forEach(sub => sub(mutation, this.state))

    if (
      __DEV__ &&
      options && options.silent
    ) {
      console.warn(
        `[vuex] mutation type: ${type}. Silent option has been removed. ` +
        'Use the filter functionality in the vue-devtools'
      )
    }
  }
  …………
  function unifyObjectStyle(type, payload, options) {
    if (isObject(type) && type.type) {
      options = payload
      payload = type
      type = type.type
    }
    if (__DEV__) {
      assert(typeof type === 'string', `expects string as the type, but found ${typeof type}.`)
    }
    return { type, payload, options }
  }
}
```
对于 `this._subscribers` 中订阅函数执行不清楚可直接看实例方法中的 `subscribe` 章节
