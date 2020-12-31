# 完善 `mode`
[官网介绍](https://router.vuejs.org/zh/guide/essentials/history-mode.html)

完整代码分支 [stage-11](https://github.com/shengrongchun/parse-vue-router)

`mode` 的类型有三种：`hash history abstract`，我们之前一直默认讨论的都是 `history` 模式。`mode` 用户可以通过路由配置信息设置
### index.js
```js{14,17,20}
let mode = options.mode || 'hash' //默认是hash模式，如果没定义的话
// 设置 history默认，在不支持的情况下会回退到hash模式
// this.fallback 表示在浏览器不支持 history.pushState 的情况下，根据传入的 fallback 配置参数，决定是否回退到hash模式
this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
if (this.fallback) {
  mode = 'hash'
}
if (!inBrowser) {//非浏览器环境下的模式 abstract
  mode = 'abstract'
}
this.mode = mode
switch (mode) {
  case 'history':
    this.history = new HTML5History(this, options.base)
    break
  case 'hash':
    this.history = new HashHistory(this, options.base, this.fallback)
    break
  case 'abstract':
    this.history = new AbstractHistory(this, options.base)
    break
  default:
    if (process.env.NODE_ENV !== 'production') {
      assert(false, `invalid mode: ${mode}`)
    }
}
```
根据代码注释很容易看懂，我们之前一直使用的是 `HTML5History`。现在我们要分别解析 `HashHistory` `AbstractHistory`

### HashHistory
```js{49,146-160}

import { History } from './base'
import { cleanPath } from '../util/path'
import { getLocation } from './html5'
import { setupScroll, handleScroll } from '../util/scroll'
import { pushState, replaceState, supportsPushState } from '../util/push-state'

export class HashHistory extends History {
  constructor(router, base, fallback) {
    super(router, base)
    // check history fallback deeplinking
    // fallback为true说明是从 history模式 降级到 hash模式
    if (fallback && checkFallback(this.base)) {//history模式url转为hash模式url
      return
    }
    ensureSlash()
  }

  // this is delayed until the app mounts
  // to avoid the hashchange listener being fired too early
  setupListeners() {
    if (this.listeners.length > 0) {
      return
    }

    const router = this.router
    const expectScroll = router.options.scrollBehavior
    const supportsScroll = supportsPushState && expectScroll

    if (supportsScroll) {
      this.listeners.push(setupScroll())
    }

    const handleRoutingEvent = () => {
      const current = this.current
      if (!ensureSlash()) {
        return
      }
      this.transitionTo(getHash(), route => {
        if (supportsScroll) {
          handleScroll(this.router, route, current, true)
        }
        if (!supportsPushState) {
          replaceHash(route.fullPath)//不支持pushState url要改成hash模式的url
        }
      })
    }
    //降级处理：popstate 不支持就监听 hashchange 事件
    const eventType = supportsPushState ? 'popstate' : 'hashchange'
    window.addEventListener(
      eventType,
      handleRoutingEvent
    )
    this.listeners.push(() => {
      window.removeEventListener(eventType, handleRoutingEvent)
    })
  }

  push(location, onComplete, onAbort) {
    const { current: fromRoute } = this
    this.transitionTo(
      location,
      route => {
        pushHash(route.fullPath)
        handleScroll(this.router, route, fromRoute, false)
        onComplete && onComplete(route)
      },
      onAbort
    )
  }

  replace(location, onComplete, onAbort) {
    const { current: fromRoute } = this
    this.transitionTo(
      location,
      route => {
        replaceHash(route.fullPath)
        handleScroll(this.router, route, fromRoute, false)
        onComplete && onComplete(route)
      },
      onAbort
    )
  }

  ensureURL(push) {
    const current = this.current.fullPath
    if (getHash() !== current) {
      push ? pushHash(current) : replaceHash(current)
    }
  }

  getCurrentLocation() {
    return getHash()
  }
}

function checkFallback(base) {//把history模式的url改成hash模式的url
  const location = getLocation(base)
  if (!/^\/#/.test(location)) {//不是以/#开头
    window.location.replace(cleanPath(base + '/#' + location))//必须加上/#
    return true
  }
}

function ensureSlash() {//确保hash第一个字符是 /
  const path = getHash()
  if (path.charAt(0) === '/') {
    return true
  }
  replaceHash('/' + path)
  return false
}

export function getHash() {//获取hash值
  // We can't use window.location.hash here because it's not
  // consistent across browsers - Firefox will pre-decode it!
  let href = window.location.href
  const index = href.indexOf('#')
  // empty path
  if (index < 0) return ''

  href = href.slice(index + 1)
  // decode the hash but not the search or hash
  // as search(query) is already decoded
  // https://github.com/vuejs/vue-router/issues/2708
  const searchIndex = href.indexOf('?')
  if (searchIndex < 0) {
    const hashIndex = href.indexOf('#')
    if (hashIndex > -1) {
      href = decodeURI(href.slice(0, hashIndex)) + href.slice(hashIndex)
    } else href = decodeURI(href)
  } else {
    href = decodeURI(href.slice(0, searchIndex)) + href.slice(searchIndex)
  }

  return href
}

function getUrl(path) {//确保是有 #
  const href = window.location.href
  const i = href.indexOf('#')
  const base = i >= 0 ? href.slice(0, i) : href
  return `${base}#${path}`
}

function pushHash(path) {
  if (supportsPushState) {
    pushState(getUrl(path))
  } else {
    window.location.hash = path
  }
}

function replaceHash(path) {
  if (supportsPushState) {
    replaceState(getUrl(path))
  } else {
    window.location.replace(getUrl(path))
  }
}
```
代码需要同 `HTML5History` 一起看，毕竟功能都基本相同。`49` 行代码是在不支持 `popstate` 的情况下，使用了 `hashchange`。
我们先看 `popstate` 的作用：
::: tip popstate
当活动历史记录条目更改时，将触发`popstate`事件。如果被激活的历史记录条目是通过对`history.pushState（）`的调用创建的，或者受到对`history.replaceState（）`的调用的影响，`popstate`事件的`state`属性包含历史条目的状态对象的副本。

需要注意的是调用`history.pushState()`或`history.replaceState()`不会触发`popstate`事件。只有在做出浏览器动作时，才会触发该事件，如用户点击浏览器的`回退按钮`（或者在`Javascript`代码中调用`history.back()`或者`history.forward()`方法）
:::

::: tip hashchange
当`URL`的片段标识符更改时，将触发`hashchange`事件 (跟在＃符号后面的URL部分，包括＃符号)
:::
我们知道当浏览器前进回退等操作时，必然会导致 `hash` 值的改变从而触发 `hashchange` 事件

`pushHash` 和 `replaceHash` 方法就是如果支持 `supportsPushState` 就执行相应的 `pushState` 和 `replaceState`。否则就降级通过 `window.location.hash` 和 `window.location.replace` 来替代。

### AbstractHistory
```js{10}

import { History } from './base'
import { isRouterError } from '../util/warn'
import { NavigationFailureType } from './errors'

export class AbstractHistory extends History {

  constructor(router, base) {
    super(router, base)
    this.stack = []
    this.index = -1
  }

  push(location, onComplete, onAbort) {
    this.transitionTo(
      location,
      route => {
        this.stack = this.stack.slice(0, this.index + 1).concat(route)
        this.index++
        onComplete && onComplete(route)
      },
      onAbort
    )
  }

  replace(location, onComplete, onAbort) {
    this.transitionTo(
      location,
      route => {
        this.stack = this.stack.slice(0, this.index).concat(route)
        onComplete && onComplete(route)
      },
      onAbort
    )
  }

  getCurrentLocation() {
    const current = this.stack[this.stack.length - 1]
    return current ? current.fullPath : '/'
  }

  ensureURL() {
    // noop
  }
}

```
在非浏览器模式下，会执行 `AbstractHistory`。他通过定义 `stack` 数组来记住路由的改变，比如装路由，替换数组中的路由等

