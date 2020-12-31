# 公共方法

## inBrowser 
```js
export const inBrowser = typeof window !== 'undefined'
```
## assert
```js
export function assert(condition, message) {
  if (!condition) {
    throw new Error(`[vue-router] ${message}`)
  }
}
```
## warn
```js
export function warn(condition, message) {
  if (process.env.NODE_ENV !== 'production' && !condition) {
    typeof console !== 'undefined' && console.warn(`[vue-router] ${message}`)
  }
}
```
## isError
```js
export function isError(err) {//是一个错误
  return Object.prototype.toString.call(err).indexOf('Error') > -1
}
```
## isRouterError
```js
export function isRouterError(err, errorType) {//是一个router错误
  return isError(err) && err._isRouter && (errorType == null || err.type === errorType)
}
```
## extend
```js
export function extend(a, b) {
  for (const key in b) {
    a[key] = b[key]
  }
  return a
}
```
## encode 和 decode
```js
const encodeReserveRE = /[!'()*]/g
const encodeReserveReplacer = c => '%' + c.charCodeAt(0).toString(16)
const commaRE = /%2C/g

// fixed encodeURIComponent which is more conformant to RFC3986:
// - escapes [!'()*]
// - preserve commas
const encode = str => //编码
  encodeURIComponent(str)
    .replace(encodeReserveRE, encodeReserveReplacer)
    .replace(commaRE, ',')

const decode = decodeURIComponent //解码
```
## stringifyQuery
```js
// obj: {//例子
//   a:1,
//   b:2,
//   c: [3,4]
// } --> 把参数字符串化后 ?a=1&b=2&c=3&c=4
export function stringifyQuery(obj) {
  const res = obj
    ? Object.keys(obj)
      .map(key => {
        const val = obj[key]
        //判断值是什么类型，然后返回相应值
        if (val === undefined) {
          return ''
        }
        if (val === null) {
          return encode(key)//encode在工具中查看
        }
        if (Array.isArray(val)) {
          const result = []
          val.forEach(val2 => {
            if (val2 === undefined) {
              return
            }
            if (val2 === null) {
              result.push(encode(key))
            } else {
              result.push(encode(key) + '=' + encode(val2))
            }
          })
          return result.join('&')
        }
        return encode(key) + '=' + encode(val)
      })
      .filter(x => x.length > 0)
      .join('&')
    : null
  return res ? `?${res}` : ''
}
```

## cleanPath
```js
//把path中 '//' --> '/'
export function cleanPath(path) {
  return path.replace(/\/\//g, '/')
}
```

## parsePath
```js
//从path中解析出单纯的 path hash query
export function parsePath(path) {
  let hash = ''
  let query = ''
  // path: /a?b=1#c=2
  const hashIndex = path.indexOf('#')
  if (hashIndex >= 0) {
    hash = path.slice(hashIndex) // #c=2
    path = path.slice(0, hashIndex) // /a?b=1
  }

  const queryIndex = path.indexOf('?')
  if (queryIndex >= 0) {
    query = path.slice(queryIndex + 1) // b=1
    path = path.slice(0, queryIndex) // /a
  }

  return {
    path,
    query,
    hash
  }
}
```
## resolvePath
```js
export function resolvePath(
  relative,
  base,
  append, // 是否直接追加到base后面
) {
  const firstChar = relative.charAt(0)
  if (firstChar === '/') {
    return relative
  }

  if (firstChar === '?' || firstChar === '#') {
    return base + relative
  }
  //说明这个 relative 是一个相对路径
  const stack = base.split('/')

  // remove trailing segment if:
  // - not appending
  // - appending to trailing slash (last segment is empty)
  // path: abc/d/ --> ['abc','d','']
  // 清除最后的空字符串
  if (!append || !stack[stack.length - 1]) {
    stack.pop()
  }

  // resolve relative path
  const segments = relative.replace(/^\//, '').split('/') // /a/b/c --> [a,b,c]
  for (let i = 0; i < segments.length; i++) {
    const segment = segments[i]
    if (segment === '..') { // ../ 这个意思大家应该明白吧 上一级
      stack.pop()
    } else if (segment !== '.') {
      stack.push(segment)
    }
  }

  // ensure leading slash 确保开始是 /
  //如果stack[0]是空字符，那么在join('/')会确保开始是 /   ['','a','b'] --> /a/b
  if (stack[0] !== '') {
    stack.unshift('')
  }
  return stack.join('/')
}
```
## parseQuery
```js
// query: ?a=1&b=2&c=3&c=4
// --> {
//   a:1,
//   b:2,
//   c: [3,4]
// }
function parseQuery(query) {
  const res = {}
  query = query.trim().replace(/^(\?|#|&)/, '') // ？# & --> ''
  if (!query) {
    return res
  }
  query.split('&').forEach(param => { //[a=1,b=2,……]
    const parts = param.replace(/\+/g, ' ').split('=') // + --> ' '
    const key = decode(parts.shift()) // parts: [a,1] shift删除第一个元素并且返回第一个元素
    const val = parts.length > 0 ? decode(parts.join('=')) : null

    if (res[key] === undefined) {
      res[key] = val
    } else if (Array.isArray(res[key])) {
      res[key].push(val)
    } else {
      res[key] = [res[key], val]
    }
  })

  return res
}
```
## resolveQuery
```js
//query字符串 --> {key:value}形式
const castQueryParamValue = value => (value == null || typeof value === 'object' ? value : String(value))
export function resolveQuery(
  query, // ?a=1
  extraQuery, // {b:2}
  _parseQuery
) {
  const parse = _parseQuery || parseQuery
  let parsedQuery
  try {
    parsedQuery = parse(query || '')
  } catch (e) {
    process.env.NODE_ENV !== 'production' && warn(false, e.message)
    parsedQuery = {}
  }
  for (const key in extraQuery) {
    const value = extraQuery[key]
    parsedQuery[key] = Array.isArray(value)
      ? value.map(castQueryParamValue)
      : castQueryParamValue(value)
  }
  return parsedQuery
}
```
## normalizeLocation
::: tip 注意
如果还没有看过动态路由匹配章节，请无视 `34-50` 行，相关用到的方法可以在工具中查看
:::
```js
import { parsePath, resolvePath } from './path'
import { resolveQuery } from './query'
import { fillParams } from './params'
import { warn } from './warn'
import { extend } from './misc'

// raw的可能性
// 1：字符串如：/pathname?search=123#hash=111
// 2：对象如： {path: '/……', query: {}} 
// 3: {name: xxx, params: {}, query: {}}
export function normalizeLocation(
  raw,
  current,
  append,
  router
) {
  // 字符串类型直接当做path
  let next = typeof raw === 'string' ? { path: raw } : raw
  // named target
  if (next._normalized) {//有标准化后的标识直接返回,已经normalize了
    return next
  } else if (next.name) { // 如果有name
    // 不希望用户传入的raw和源码内部之间相互影响，所以用了浅copy,raw中没有像对象，数组这样类型的值，所以也就相当于深copy
    next = extend({}, raw)
    const params = next.params
    if (params && typeof params === 'object') {
      next.params = extend({}, params)
    }
    return next
  }
  // 如果还没有看过动态路由匹配章节，请无视 `34-50` 行
  // 当 push 的location没有path,同时有params和当前路由，会把params追加到当前路由，这就是相对参数
  // 如：push({ params: { name: '追加当前路由参数' } }) relative params 
  if (!next.path && next.params && current) {
    next = extend({}, next)
    next._normalized = true
    const params = extend(extend({}, current.params), next.params)
    if (current.name) {
      next.name = current.name
      next.params = params
    } else if (current.matched.length) {//如果当前路由没有name,path
      //寻找最后一个子路由的path(这里可能你还不理解，等到讲到嵌套路由的时候就明白了)
      const rawPath = current.matched[current.matched.length - 1].path
      // 把params填充到path，如record.path: /a/:username/:userid; params: {username:vue,userid:router};
      // 那么最终的path: /a/vue/router
      next.path = fillParams(rawPath, params, `path ${current.path}`)
    } else if (process.env.NODE_ENV !== 'production') {
      warn(false, `relative params navigation requires a current route.`)
    }
    return next
  }

  //解析 path hash query  主要是path里面可能会解析出hash,query，然后合并
  const parsedPath = parsePath(next.path || '')
  const basePath = (current && current.path) || '/'
  const path = parsedPath.path
    ? resolvePath(parsedPath.path, basePath, append || next.append)
    : basePath

  // ?a=1&b=2 --> {a:1,b:2}
  const query = resolveQuery(
    parsedPath.query, // 在next.path中解析出的query
    next.query, //next自带的query
    router && router.options.parseQuery // route配置中有自带的解析query的方法
  )

  let hash = next.hash || parsedPath.hash
  if (hash && hash.charAt(0) !== '#') {//hash值第一个字符必须是#
    hash = `#${hash}`
  }

  return {
    _normalized: true,
    path,
    query,
    hash
  }
}
```

## fillParams
```js
import { warn } from './warn'
import Regexp from 'path-to-regexp'

// $flow-disable-line
const regexpCompileCache = Object.create(null)
// ## pathMatch 通配符匹配的字符串
export function fillParams(
  path,
  params,
  routeMsg
) {
  params = params || {}
  try {
    // 例子
    // path: /user/:id/:name
    // demoParams: {id:111, name: hello}
    // Regexp.compile(path)(demoParams) --> /user/111/hello
    const filler =
      regexpCompileCache[path] ||
      (regexpCompileCache[path] = Regexp.compile(path))

    // Fix #2505 resolving asterisk routes { name: 'not-found', params: { pathMatch: '/not-found' }}
    // and fix #3106 so that you can work with location descriptor object having params.pathMatch equal to empty string
    if (typeof params.pathMatch === 'string') params[0] = params.pathMatch

    return filler(params, { pretty: true })
  } catch (e) {
    if (process.env.NODE_ENV !== 'production') {
      // Fix #3072 no warn if `pathMatch` is string
      warn(typeof params.pathMatch === 'string', `missing param for ${routeMsg}: ${e.message}`)
    }
    return ''
  } finally {
    // delete the 0 if it was added
    delete params[0]
  }
}
```

## genStateKey
```js
// use User Timing api (if present) for more accurate key precision
const Time =
  inBrowser && window.performance && window.performance.now
    ? window.performance
    : Date
export function genStateKey() {
  return Time.now().toFixed(3)
}
```
## _key
```js
let _key = genStateKey()
```
## getStateKey
```js
export function getStateKey() {//获取唯一key
  return _key
}
```
## setStateKey
```js
export function setStateKey(key) {//设置唯一key
  return (_key = key)
}
```
## getElementPosition
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
```
## isValidPosition
```js
function isValidPosition(obj) {
  return isNumber(obj.x) || isNumber(obj.y)
}
```
## normalizePosition
```js
function normalizePosition(obj) {//标准化偏移
  return {
    x: isNumber(obj.x) ? obj.x : window.pageXOffset,
    y: isNumber(obj.y) ? obj.y : window.pageYOffset
  }
}
```
## normalizeOffset
```js
function normalizeOffset(obj) {//标准化offset
  return {
    x: isNumber(obj.x) ? obj.x : 0,
    y: isNumber(obj.y) ? obj.y : 0
  }
}
```
## isNumber
```js
function isNumber(v) {
  return typeof v === 'number'
}
```
## isSameRoute
```js
//是否相同route
export function isSameRoute(a, b) {
  if (b === START) {
    return a === b
  } else if (!b) {
    return false
  } else if (a.path && b.path) {
    return (
      a.path.replace(trailingSlashRE, '') === b.path.replace(trailingSlashRE, '') &&
      a.hash === b.hash &&
      isObjectEqual(a.query, b.query)
    )
  } else if (a.name && b.name) {
    return (
      a.name === b.name &&
      a.hash === b.hash &&
      isObjectEqual(a.query, b.query) &&
      isObjectEqual(a.params, b.params)
    )
  } else {
    return false
  }
}
```
## isObjectEqual
```js
function isObjectEqual(a = {}, b = {}) {
  // handle null value #1566
  if (!a || !b) return a === b
  const aKeys = Object.keys(a)
  const bKeys = Object.keys(b)
  if (aKeys.length !== bKeys.length) {
    return false
  }
  return aKeys.every(key => {
    const aVal = a[key]
    const bVal = b[key]
    // query values can be null and undefined
    if (aVal == null || bVal == null) return aVal === bVal
    // check nested equality
    if (typeof aVal === 'object' && typeof bVal === 'object') {
      return isObjectEqual(aVal, bVal)
    }
    return String(aVal) === String(bVal)
  })
}
```
## isIncludedRoute
```js
//
export function isIncludedRoute(current, target) {
  return (
    current.path.replace(trailingSlashRE, '/').indexOf(
      target.path.replace(trailingSlashRE, '/')
    ) === 0 &&
    (!target.hash || current.hash === target.hash) &&
    queryIncludes(current.query, target.query)
  )
}
```
## queryIncludes
```js
function queryIncludes(current, target) {
  for (const key in target) {
    if (!(key in current)) {
      return false
    }
  }
  return true
}
```
## resolveQueue
::: tip 例子
比如：current：[a,b,c], next: [a,b,d] --> i=2

updated: next.slice(0, i), --> a,b

activated: next.slice(i),  --> d

deactivated: current.slice(i) --> c
:::
```js
// 获取更新 激活 停用 record 数组
function resolveQueue(
  current,
  next
) {
  let i
  const max = Math.max(current.length, next.length)
  for (i = 0; i < max; i++) {
    if (current[i] !== next[i]) {
      break
    }
  }
  return {
    updated: next.slice(0, i), //更新 record 数组
    activated: next.slice(i), //激活 record 数组
    deactivated: current.slice(i)//停用 record数组
  }
}
```
## flatMapComponents
```js
export function flatMapComponents(
  matched,
  fn
) {
  return flatten(matched.map(m => {
    return Object.keys(m.components).map(key => fn( //key是定义的组件名称： default name1 name2
      m.components[key],// 组件 component
      //对应组件实例 前提是 record上应该有个instances对象
      m.instances[key], 
      m, key    // record 和 如default
    ))
  }))
}
```
## flatten
``` js
export function flatten(arr) {//数组扁平处理 [].concat([1,2],3)--> [1,2,3]
  return Array.prototype.concat.apply([], arr)
}
```
## runQueue
::: tip 解析
轮询处理 `queue` 中的每一个`item`。如果`item`是`undefined`则执行下一个`item`。如果不是`undefined`则执行`fn`。如果一切顺利`queue`中的所有`item`都处理了，执行`cb`
::: 
queue: [undefined,……,function(to,from,next),……]
```js
export function runQueue(queue, fn, cb) {
  const step = index => {
    if (index >= queue.length) {
      cb()
    } else {
      if (queue[index]) {
        fn(queue[index], () => {
          step(index + 1)
        })
      } else {
        step(index + 1)
      }
    }
  }
  step(0)
}
```
## NavigationFailureType
```js
export const NavigationFailureType = {
  redirected: 1, //重定向错误类型码
  aborted: 2, // 终止路由错误
  cancelled: 3, // 取消路由跳转错误
  duplicated: 4 //跳转相同路由错误码 （冗余导航）
}
```
## createNavigationRedirectedError
```js
export function createNavigationRedirectedError(from, to) {
  return createRouterError(
    from,
    to,
    NavigationFailureType.redirected,
    `Redirected when going from "${from.fullPath}" to "${stringifyRoute(
      to
    )}" via a navigation guard.`
  )
}
```
## createNavigationDuplicatedError
```js
//冗余导航错误
export function createNavigationDuplicatedError(from, to) {
  return createRouterError(
    from,
    to,
    NavigationFailureType.duplicated,
    `Avoided redundant navigation to current location: "${from.fullPath}".`
  )
}
```
## createNavigationCancelledError
```js
export function createNavigationCancelledError(from, to) {
  return createRouterError(
    from,
    to,
    NavigationFailureType.cancelled,
    `Navigation cancelled from "${from.fullPath}" to "${to.fullPath
    }" with a new navigation.`
  )
}
```
## createNavigationAbortedError
```js
export function createNavigationAbortedError(from, to) {
  return createRouterError(
    from,
    to,
    NavigationFailureType.aborted,
    `Navigation aborted from "${from.fullPath}" to "${to.fullPath
    }" via a navigation guard.`
  )
}
```
## createRouterError
```js
//创建一个路由类型错误
function createRouterError(from, to, type, message) {
  const error = new Error(message)
  error._isRouter = true
  error.from = from
  error.to = to
  error.type = type

  return error
}
```
## stringifyRoute
```js
const propertiesToLog = ['params', 'query', 'hash']
function stringifyRoute(to) {
  if (typeof to === 'string') return to
  if ('path' in to) return to.path
  const location = {}
  propertiesToLog.forEach(key => {
    if (key in to) location[key] = to[key]
  })
  return JSON.stringify(location, null, 2)
}
```
