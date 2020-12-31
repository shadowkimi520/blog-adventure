# 滚动行为
[官网介绍](https://router.vuejs.org/zh/guide/advanced/scroll-behavior.html)

完整代码分支 [stage-8](https://github.com/shengrongchun/parse-vue-router)

在解析如何实现滚动行为之前，你应该熟悉如何使用或者你已经阅读过官网介绍。滚动行为应该是路由跳转成功后需要执行的行为
## 浏览器回退执行滚动行为
如果定义了滚动行为，在浏览器回退时，我们应该做什么
+ 记住之前页面的滚动位置，当作用户自定义滚动行为的第三个参数：scrollBehavior (to, from, savedPosition)
+ 回退的新页面也要执行滚动行为

所以在 `popState` 监听事件那里要加上滚动行为事件
```js{6-11,20-22}
setupListeners() {//
  if (this.listeners.length > 0) {
    return
  }
  //
  const router = this.router
  const expectScroll = router.options.scrollBehavior//用户自定义的滚动行为
  const supportsScroll = supportsPushState && expectScroll//支持pushState且定义了scrollBehavior
  if (supportsScroll) {//支持滚动
    this.listeners.push(setupScroll())
  }
  const handleRoutingEvent = () => {
    // Avoiding first `popstate` event dispatched in some browsers but first
    // history route not updated since async guard at the same time.
    const location = getLocation(this.base)
    if (this.current === START && location === this._startLocation) {
      return
    }
    this.transitionTo(location, route => {//退回路由跳转成功后
      if (supportsScroll) {//支持滚动
        handleScroll(router, route, current, true)//执行滚动行为
      }
    })
  }
  window.addEventListener('popstate', handleRoutingEvent)
  this.listeners.push(() => {
    window.removeEventListener('popstate', handleRoutingEvent)
  })
}
``` 
`20-22` 代码是浏览器回退成功后执行滚动行为，当然 `supportsPushState` 和 `handleScroll` 方法接下来会详细介绍，这里先看第 `10` 行的 `setupScroll`

### setupScroll
`setupScroll` 最主要的功能是记住回退前页面的滚动位置
```js{4-6,18,20}
export function setupScroll() {
  //因为我们是需要自定义滚动位置的，所以应该阻止浏览器默认滚动到之前页面的滚动位置
  //如果还不是很明白，可以查看：https://jsonz.cn/2018/05/history-scroll-restoration/
  if ('scrollRestoration' in window.history) {
    window.history.scrollRestoration = 'manual'
  }
  // Fix for #1585 for Firefox
  // Fix for #2195 Add optional third attribute to workaround a bug in safari https://bugs.webkit.org/show_bug.cgi?id=182678
  // Fix for #2774 Support for apps loaded from Windows file shares not mapped to network drives: replaced location.origin with
  // window.location.protocol + '//' + window.location.host
  // location.host contains the port and location.hostname doesn't
  const protocolAndPath = window.location.protocol + '//' + window.location.host
  const absolutePath = window.location.href.replace(protocolAndPath, '')
  // preserve existing history state as it could be overriden by the user
  const stateCopy = extend({}, window.history.state)
  stateCopy.key = getStateKey()
  window.history.replaceState(stateCopy, '', absolutePath)
  window.addEventListener('popstate', handlePopState)
  return () => {
    window.removeEventListener('popstate', handlePopState)
  }
}
```
当 `popState` 发生时，执行了 `handlePopState`
```js{2,11-16}
const positionStore = Object.create(null)
export function saveScrollPosition() {//保存页面滚动位置
  const key = getStateKey() //工具方法中查看
  if (key) {
    positionStore[key] = {
      x: window.pageXOffset,
      y: window.pageYOffset
    }
  }
}
//
function handlePopState(e) {
  saveScrollPosition() //保存当前页面的偏移量
  if (e.state && e.state.key) {//还记得编程式导航章节里面 pushState({key:……})
    setStateKey(e.state.key)//工具方法中查看
  }
}
```
## 初始化部分执行滚动行为
```js{10,15}
init(app) {// app根实例
  /*…………*/
  const history = this.history
  //初始化时候去匹配更改当前路由
  const handleInitialScroll = (routeOrError) => {//滚动行为
    const from = history.current
    const expectScroll = this.options.scrollBehavior //获取用户配置滚动行为
    const supportsScroll = supportsPushState && expectScroll //是否支持滚动
    if (supportsScroll && 'fullPath' in routeOrError) {
      handleScroll(this, routeOrError, from, false)
    }
  }
  const onCompleteOrAbort = (routeOrError) => {
    history.setupListeners()
    handleInitialScroll(routeOrError)
  } //路由跳转成功或者失败我们可能需要执行的函数
  history.transitionTo(history.getCurrentLocation(), onCompleteOrAbort, onCompleteOrAbort)
  /*…………*/
}
```
从 `15` 代码开始看，不管路由跳转成功与否，都会去执行 `handleInitialScroll` 方法。在此方法中还要判断当前条件是否支持滚动行为：
+ 支持滚动行为 `supportsPushState`
+ 用户定义了 `scrollBehavior`
+ `routeOrError` 中有 `fullPath`字段，说明 `routeOrError` 应该是一个路由对象而不是一个错误
### supportsPushState
```js
export const supportsPushState =
  inBrowser &&
  (function () {
    const ua = window.navigator.userAgent
    if (
      (ua.indexOf('Android 2.') !== -1 || ua.indexOf('Android 4.0') !== -1) &&
      ua.indexOf('Mobile Safari') !== -1 &&
      ua.indexOf('Chrome') === -1 &&
      ua.indexOf('Windows Phone') === -1
    ) {
      return false
    }
    return window.history && typeof window.history.pushState === 'function'
  })()
```
这个功能只在支持 `history.pushState` 的浏览器中可用，官网已经说的很明白

我们重点看 `handleScroll`
```js{5,19-25,33,41}
export function handleScroll(// ok
  router,
  to,
  from,
  isPop //是否是popState监听退回页面成功执行的handleScroll
) {
  if (!router.app) {//没有app返回
    return
  }
  const behavior = router.options.scrollBehavior
  if (!behavior) {//没有定义scrollBehavior返回
    return
  }
  if (process.env.NODE_ENV !== 'production') {//生产环境scrollBehavior不是函数报错
    assert(typeof behavior === 'function', `scrollBehavior must be a function`)
  }
  // wait until re-render finishes before scrolling
  router.app.$nextTick(() => {//当然需要页面渲染完成有DOM时候才能滚动
    const position = getScrollPosition()//获取回退前页面的滚动位置
    const shouldScroll = behavior.call(//对用户定义的滚动行为函数执行返回: {x:……,y:……}
      router,
      to,
      from,
      isPop ? position : null //如果是退回情况，第四个参数是回退前页面的滚动位置详情看官网
    )
    if (!shouldScroll) {//用户设置behavior返回的值如 {x:0,y:0}
      return
    }
    //通过官网可知：你也可以返回一个 Promise来实现异步滚动
    if (typeof shouldScroll.then === 'function') {
      shouldScroll
        .then(shouldScroll => {// {x:0,y:0}
          scrollToPosition((shouldScroll), position)
        })
        .catch(err => {
          if (process.env.NODE_ENV !== 'production') {
            assert(false, err.toString())
          }
        })
    } else {
      scrollToPosition(shouldScroll, position)
    }
  })
}
```
代码基本都有注释很容易看懂，我们看 `19` 行代码
```js
function getScrollPosition() {
  const key = getStateKey()
  if (key) {
    return positionStore[key]
  }
}
```
获取回退前页面滚动位置，还记得我们之前是通过 `saveScrollPosition` 方法保存位置的。接下来是：`scrollToPosition`
```js{26}
const hashStartsWithNumberRE = /^#\d/ //以#和数字开头的正则表达式

function scrollToPosition(shouldScroll, position) {
  const isObject = typeof shouldScroll === 'object'
  // shouldScroll应该是一个对象才会有效
  if (isObject && typeof shouldScroll.selector === 'string') {//定义selector滚动到锚点
    //如果selector是以#加数字开头的，通过 document.getElementById取元素，否则document.querySelector取
    const el = hashStartsWithNumberRE.test(shouldScroll.selector) // $flow-disable-line
      ? document.getElementById(shouldScroll.selector.slice(1)) // slice(1): #123 --> 123
      : document.querySelector(shouldScroll.selector)

    if (el) {//如果真的取到了元素
      let offset =
        shouldScroll.offset && typeof shouldScroll.offset === 'object'
          ? shouldScroll.offset
          : {} //在定义滚动到锚点的前提下也有可能定义偏移量：offset: {x: 0,y:0}
      offset = normalizeOffset(offset)// 标准化偏移
      position = getElementPosition(el, offset)//获取目标元素的滚动偏移
    } else if (isValidPosition(shouldScroll)) {//如果是有效的滚动偏移
      position = normalizePosition(shouldScroll)
    }
  } else if (isObject && isValidPosition(shouldScroll)) {
    position = normalizePosition(shouldScroll)
  }
  if (position) {
    window.scrollTo(position.x, position.y)//最终的滚动行为代码
  }
}
```
相关工具方法
```js
function getElementPosition(el, offset) {//获取目标元素应该滚动的x,y位置
  const docEl = document.documentElement //当前文档
  const docRect = docEl.getBoundingClientRect()//当前文档的位置信息
  const elRect = el.getBoundingClientRect() //目标元素位置信息
  return {
    x: elRect.left - docRect.left - offset.x,
    y: elRect.top - docRect.top - offset.y
  }
}
function isValidPosition(obj) {
  return isNumber(obj.x) || isNumber(obj.y)
}
function normalizePosition(obj) {//标准化偏移
  return {
    x: isNumber(obj.x) ? obj.x : window.pageXOffset,
    y: isNumber(obj.y) ? obj.y : window.pageYOffset
  }
}
function normalizeOffset(obj) {//标准化offset
  return {
    x: isNumber(obj.x) ? obj.x : 0,
    y: isNumber(obj.y) ? obj.y : 0
  }
}
function isNumber(v) {
  return typeof v === 'number'
}
```
## push/replace 部分执行滚动行为
当执行 `push` 或者 `replace` 的时候也需要执行滚动行为
```js{6,16}
push(location, onComplete, onAbort) {
  const Complete = (route) => {
    const { current: fromRoute } = this
    //不刷新更改浏览器url 并且增加一条记录，浏览器可以回退
    pushState(cleanPath(this.base + route.fullPath))
    handleScroll(this.router, route, fromRoute, false) //滚动行为
    onComplete && onComplete(route)
  }
  this.transitionTo(location, Complete, onAbort)
}
replace(location, onComplete, onAbort) {
  const Complete = (route) => {
    const { current: fromRoute } = this
    //不刷新更改浏览器url 并且刷新记录
    replaceState(cleanPath(this.base + route.fullPath))
    handleScroll(this.router, route, fromRoute, false) //滚动行为
    onComplete && onComplete(route)
  }
  this.transitionTo(location, Complete, onAbort)
}
```
在执行 `push` 或者 `replace` 的时候，我们不仅需要执行滚动行为，我们还需要保存当前页面的滚动位置，因为回退的时候是需要用到相应页面他之前的滚动位置的
### push-state
```js{2}
export function pushState(url, replace) {
  saveScrollPosition()
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
```