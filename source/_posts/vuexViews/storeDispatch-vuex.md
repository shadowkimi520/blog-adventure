# dispatch
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-vuex)

[官网介绍](https://vuex.vuejs.org/zh/api/#dispatch)

`store` 的实例方法 `dispatch`
```js{5,14-16,22,30,50-52,54}
export class Store { // this--> vue实例中的 this.$store
  constructor(options = {}) {
    …………
    //订阅action的回调函数，当有action执行，会执行此数组中的函数
    this._actionSubscribers = [] 
    //装入用户自定义的一些函数，当有mutation执行后，会执行此数组中的函数
    // 实例对象的 this.$store.subscribe 方法是装入功能
    this._subscribers = [] 

    // bind commit and dispatch to self
    const store = this //this.$store 实例
    const { dispatch, commit } = this
    //中转下是保证方法里的this是store
    this.dispatch = function boundDispatch(type, payload) {
      return dispatch.call(store, type, payload) // this.$store.dispatch
    }
    this.commit = function boundCommit(type, payload, options) {
      return commit.call(store, type, payload, options) // this.$store.commit
    }
  }
  …………
  dispatch(_type, _payload) {
    // check object-style dispatch
    const {
      type,
      payload
    } = unifyObjectStyle(_type, _payload)

    const action = { type, payload }
    const entry = this._actions[type]
    if (!entry) {
      if (__DEV__) {
        console.error(`[vuex] unknown action type: ${type}`)
      }
      return
    }

    try {//如果有before说明是希望在action分发之前调用
      this._actionSubscribers
        .slice() // shallow copy to prevent iterator invalidation if subscriber synchronously calls unsubscribe
        .filter(sub => sub.before)
        .forEach(sub => sub.before(action, this.state))
    } catch (e) {
      if (__DEV__) {
        console.warn(`[vuex] error in before action subscribers: `)
        console.error(e)
      }
    }

    const result = entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)

    return new Promise((resolve, reject) => {
      result.then(res => {
        try {
          this._actionSubscribers//分发之后调用
            .filter(sub => sub.after)
            .forEach(sub => sub.after(action, this.state))
        } catch (e) {
          if (__DEV__) {
            console.warn(`[vuex] error in after action subscribers: `)
            console.error(e)
          }
        }
        resolve(res)
      }, error => {
        try {
          this._actionSubscribers
            .filter(sub => sub.error)
            .forEach(sub => sub.error(action, this.state, error))
        } catch (e) {
          if (__DEV__) {
            console.warn(`[vuex] error in error action subscribers: `)
            console.error(e)
          }
        }
        reject(error)
      })
    })
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
+ `this.$store.dispatch` 返回的是一个 `promise` 。触发的所有 `action` 执行完异步后会触发此 `promise` 的 `then` 方法。这对于某些场景下是很有用的
+ 对于 `this._actionSubscribers` 中的 `sub.before sub.after sub.error` 建议直接看实例方法 `subscribeAction` 章节