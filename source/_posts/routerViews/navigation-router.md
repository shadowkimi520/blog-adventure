# 编程式导航
[官网介绍](https://router.vuejs.org/zh/guide/essentials/navigation.html)

完整代码分支 [stage-7](https://github.com/shengrongchun/parse-vue-router)

编程式导航最主要的是 `push` 和 `replace` 方法。`push` 方法在之前的章节有提到过，不过功能非常简单。现在我们需要完善他们。
首先看官网介绍我们知道：
## push
+ 非必填参数会有 `onComplete, onAbort` 跳转成功与失败执行的回调函数，如果定义的话
+ 如果支持 `Promise`，`router.push` 将返回一个 `Promise`
### VueRouter
```js{3,8}
/*…………*/
push(location, onComplete, onAbort) {
  if (!onComplete && !onAbort && typeof Promise !== 'undefined') {
    return new Promise((resolve, reject) => {
      this.history.push(location, resolve, reject)
    })
  } else {
    this.history.push(location, onComplete, onAbort)
  }
}
/*…………*/
```
可以看出，当没有定义 `onComplete`, `onAbort` 并且支持 `Promise` 的时候，直接返回一个 `Promise`。`Promise` 可以让我们在路由跳转成功或者失败后执行相应的操作。

接下来是 `history` 中的 `push`
```js{4,5,7}
push(location, onComplete, onAbort) {
  const Complete = (route) => {
    //不刷新更改浏览器url 并且增加一条记录，浏览器可以回退
    pushState(cleanPath(this.base + route.fullPath))
    onComplete && onComplete(route)
  }
  this.transitionTo(location, Complete, onAbort)
}
```
从第 `7` 行代码可以看出，当路由跳转成功时执行 `Complete` 方法，失败执行 `onAbort` 方法。在 `Complete` 方法中也就是路由跳转成功后，我们需要做哪里事情：
+ 不刷新页面更改浏览器的 `url`, 并且增加一条记录，浏览器可以回退
+ 执行传过来的成功回调函数，如果 `promise` 就是 `resolve`。参数是 `route`

上述代码中 `cleanPath` 方法可以在工具中查看

### pushState
直接从 `22` 行代码开始看
```js{22}
import { inBrowser } from './dom'

// use User Timing api (if present) for more accurate key precision
const Time =
  inBrowser && window.performance && window.performance.now
    ? window.performance
    : Date

export function genStateKey() {
  return Time.now().toFixed(3)
}

let _key = genStateKey()

export function getStateKey() {//获取唯一key
  return _key
}

export function setStateKey(key) {//设置唯一key
  return (_key = key)
}
export function pushState(url) {
  // try...catch the pushState call to get around Safari
  // DOM Exception 18 where it limits to 100 pushState calls
  const history = window.history
  try {
    history.pushState({ key: setStateKey(genStateKey()) }, '', url)
  } catch (e) {
    window.location['assign'](url) //无兼容问题
  }
}
```
其实内部执行的就是 `window.history.pushState` 方法。这个方法可以去自行查看它的用处[MDN](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState)。为什么用 `try catch`，从英文注解中可以看出是对个别浏览器的降解操作

## replace
`push` 方法介绍完毕，现在可以再介绍 `replace` 方法。跟 `push` 很像，唯一的不同就是，它不会向 `history` 添加新记录，而是跟它的方法名一样 —— 替换掉当前的 `history` 记录

### VueRouter
```js{3,8}
/*…………*/
replace(location, onComplete, onAbort) {
  if (!onComplete && !onAbort && typeof Promise !== 'undefined') {
    return new Promise((resolve, reject) => {
      this.history.replace(location, resolve, reject)
    })
  } else {
    this.history.replace(location, onComplete, onAbort)
  }
}
/*…………*/
```
这里的代码没必要再解释，直接看 `this.history.replace`
```js
replace(location, onComplete, onAbort) {
  const Complete = (route) => {
    //不刷新更改浏览器url 并且刷新记录
    replaceState(cleanPath(this.base + route.fullPath))
    onComplete && onComplete(route)
  }
  this.transitionTo(location, Complete, onAbort)
}
```
和 `push` 很像，不过多解释，直接定位 `replaceState` 
```js{19-21}
export function pushState(url, replace) {
  // try...catch the pushState call to get around Safari
  // DOM Exception 18 where it limits to 100 pushState calls
  const history = window.history
  try {
    if (replace) {
      // preserve existing history state as it could be overriden by the user
      const stateCopy = extend({}, history.state)
      stateCopy.key = getStateKey()
      history.replaceState(stateCopy, '', url)
    } else {
      history.pushState({ key: setStateKey(genStateKey()) }, '', url)
    }
  } catch (e) {
    window.location[replace ? 'replace' : 'assign'](url)
  }
}
//
export function replaceState(url) {
  pushState(url, true)
}
```
`replaceState` 方法最终执行的是 `window.history.replaceState`，重写当前的 `history` 状态。

### transitionTo
别忘了对 `transitionTo` 的修改，增加出错处理
```js{1,6,8}
transitionTo(location, onComplete, onAbort) {
  //新匹配创建的route
  let route
  try {
    route = this.router.match(location, this.current)
    onComplete && onComplete(route)
  } catch (e) {
    onAbort && onAbort(e)
    // Exception should still be thrown
    throw e
  }
  this.updateRoute(route)
}
```
## 浏览器回退保持页面一致
如果你调试 `stage-7` 分支的代码会发现，虽然点击浏览器的退回按钮，`url` 是正确的回退了，但页面没有改变。所以还需回退到正确的页面，回到初始化时候的 `transitionTo`
```js{5,6,7}
init(app) {// app根实例
/*……*/
  const history = this.history
  //初始化时候去匹配更改当前路由
  const onCompleteOrAbort = () => {
    history.setupListeners()
  } //路由跳转成功或者失败我们可能需要执行的函数
  history.transitionTo(history.getCurrentLocation(), onCompleteOrAbort, onCompleteOrAbort)
  /*……*/
}
```
初始化的时候，不管路由跳转成功与否，都会执行回调函数 `onCompleteOrAbort`。此函数中会去执行 `history.setupListeners`
### history
```js{16,18}
setupListeners() {//
  // this.listeners 是一个数组定义在 base.js中。它装载着各种监听事件对应的销毁监听事件
  // 当需要销毁这些事件时，可以直接遍历销毁
  if (this.listeners.length > 0) {//如果有事件，说明不是初始化阶段，直接返回
    return
  }
  const handleRoutingEvent = () => {
    // Avoiding first `popstate` event dispatched in some browsers but first
    // history route not updated since async guard at the same time.
    const location = getLocation(this.base)
    if (this.current === START && location === this._startLocation) {
      return //默认首页面就无需再次 transitionTo
    }
    this.transitionTo(location, () => { })
  }
  window.addEventListener('popstate', handleRoutingEvent)
  this.listeners.push(() => {
    window.removeEventListener('popstate', handleRoutingEvent)
  })
}
```
`16` 行代码非常的重要，他是监听浏览器回退，`history.back` `history.forword` 等事件的，具体可自行查阅。当这些事件被触发时，我们应该重新执行 `transitionTo`。`17-19` 代码是向 `listeners` 装入监听事件的销毁事件。我们在回到 `base.js`
```js
/*……*/
teardownListeners() {
  this.listeners.forEach(cleanupListener => {
    cleanupListener()
  })
  this.listeners = []
}
/*……*/
```
增加了销毁事件的方法 `teardownListeners`