# 嵌套路由
完整代码分支 [stage-3](https://github.com/shengrongchun/parse-vue-router)

我们目前的 `router-view` 组件渲染只支持一层，而实际生活中的应用界面，通常由多层嵌套的组件组合而成。同样地，URL 中各段动态路径也按某种结构对应嵌套的各层组件，例如：
```
/user/foo/profile                     /user/foo/posts
+------------------+                  +-----------------+
| User             |                  | User            |
| +--------------+ |                  | +-------------+ |
| | Profile      | |  +------------>  | | Posts       | |
| |              | |                  | |             | |
| +--------------+ |                  | +-------------+ |
+------------------+                  +-----------------+
```
这里是[官网介绍](https://router.vuejs.org/zh/guide/essentials/nested-routes.html)，现在我们来看如何用代码实现它。话不多说，先上例子

Bar.vue
```html
<p>我是Bar组件</p>
<router-view></router-view>
```
Foo.vue
```html
<p>我是Foo组件</p>
<router-view></router-view>
```
Sub.vue
```html
<p>我是Sub组件</p>
```
路由配置信息
```js
const router = new VueRouter({
  routes: [
    { path: '/bar', component: Bar,
      children: [{
        path: 'foo', component: Foo,
        children: [{
          path: 'sub', component: Sub
        }]
      }]
    }
  ]
})
```
::: tip
寻找规律，`Bar` 组件中的 `router-view`（第一层） 应该渲染配置信息的第一层的 `component`;
`Foo` 组件中的 `router-view`（第二层） 应该渲染配置信息的第二层的 `component`;
`Sub` 组件中的 `router-view`（第三层） 应该渲染配置信息的第三层的 `component`;
:::
我们回到 `router-view` 组件
::: tip
要想知道自己是第几层的 `router-view` 并不困难，只要层层往上级查找 `router-view` 直到根实例结束 `19-27` 行。
而 `matched` 不应该只是一层的 `record` 信息。他应该是多层信息的数组，如 `29-30` 行
:::
```js{12,19-27,29-30}
export default {
  name: 'RouterView',
  functional: true, // vue的函数式组件
  props: {
    name: {
      type: String,
      default: 'default'
    },
  },
  render(_, { props, children, parent, data }) {
    // used by devtools to display a router-view badge
    data.routerView = true //标示自己是一个router-view
    //
    const h = parent.$createElement
    const name = props.name
    const route = parent.$route //组件依赖了$route
    // determine current view depth, also check to see if the tree
    // has been toggled inactive but kept-alive.
    let depth = 0 //自己是第几层的 router-view
    while (parent && parent._routerRoot !== parent) {//有parent并且parent不是根实例
      const vnodeData = parent.$vnode ? parent.$vnode.data : {}
      if (vnodeData.routerView) {//router-view组件标识
        depth++//一直向上级父层查找,找到 depth加一
      }
      parent = parent.$parent
    }
    data.routerViewDepth = depth
    //matched：[……,record.parent.parent,record.parent,record]
    const matched = route.matched[depth] //record
    const component = matched && matched.components[name]
    if (!matched || !component) {
      return h()
    }
    return h(component, data, children)
  }
}
```
那么现在的问题在于如何把相关 `record` 收集到 `matched` 数组中
### createRoute
```js{15}
const route = {
  name: location.name || (record && record.name),//当前路由的名称，如果有的话
  meta: (record && record.meta) || {},//meta元数据，如果有的话
  //字符串，对应当前路由的路径，总是解析为绝对路径，如 "/foo/bar"
  path: location.path || '/',
  hash: location.hash || '',//当前路由的 hash 值 (带 #) ，如果没有 hash 值，则为空字符串
  //一个 key/value 对象，表示 URL 查询参数。例如，对于路径 /foo?user=1，
  //则有 $route.query.user == 1，如果没有查询参数，则是个空对象。
  query,
  //一个 key/value 对象，包含了动态片段和全匹配片段，如果没有路由参数，就是一个空对象
  params: location.params || {},
  //包含查询参数和 hash 的完整路径
  fullPath: getFullPath(location, stringifyQuery),//getFullPath方法在下面有定义
  //一个数组，包含当前路由的所有嵌套路径片段的路由记录
  matched: record ? formatMatch(record) : []
}
```
我们希望 `matched` 应该是这样的 `[……,record.parent.parent,record.parent,record]`
### formatMatch
```js
//嵌套路由视图
// [父, 子, 子的子……]
function formatMatch(record) {
  const res = []
  while (record) {
    res.unshift(record)
    record = record.parent
  }
  return res
}
```
`record` 记录片段上需要有 `parent` 字段，直接看代码
::: tip 解析
只加了几行深色的代码，递归遍历，赋值 `parent`
:::
```js{49,74,78-82}
import Regexp from 'path-to-regexp'
import { cleanPath } from './util/path'
import { assert, warn } from './util/warn'
//
export function createRouteMap(
  routes
) {
  const pathList = [] //创建空数组
  const pathMap = Object.create(null)//创建空对象
  const nameMap = Object.create(null)//创建空对象
  //遍历 routes 把 route 相关信息放入 pathList, pathMap, nameMap
  routes.forEach(route => {
    addRouteRecord(pathList, pathMap, nameMap, route)
  })
  // https://router.vuejs.org/zh/guide/essentials/dynamic-matching.html#%E6%8D%95%E8%8E%B7%E6%89%80%E6%9C%89%E8%B7%AF%E7%94%B1%E6%88%96-404-not-found-%E8%B7%AF%E7%94%B1
  // ensure wildcard routes are always at the end
  // 如果定义了path=*确保*的path放在pathList的最后,当通过path没有匹配到任何record，则在正则匹配的时候，*是永远匹配上的
  // https://router.vuejs.org/zh/guide/essentials/dynamic-matching.html#%E5%8C%B9%E9%85%8D%E4%BC%98%E5%85%88%E7%BA%A7
  for (let i = 0, l = pathList.length; i < l; i++) {
    if (pathList[i] === '*') {
      pathList.push(pathList.splice(i, 1)[0])
      l--
      i--
    }
  }
  if (process.env.NODE_ENV === 'development') {//非生产环境，path不是已 *或者/开头会警告
    // warn if routes do not include leading slashes
    const found = pathList
      // check for missing leading slash
      .filter(path => path && path.charAt(0) !== '*' && path.charAt(0) !== '/')

    if (found.length > 0) {
      const pathNames = found.map(path => `- ${path}`).join('\n')
      warn(false, `Non-nested routes must include a leading slash character. Fix the following routes: \n${pathNames}`)
    }
  }
  //
  return {
    pathList,
    pathMap,
    nameMap
  }
}
function addRouteRecord(
  pathList,
  pathMap,
  nameMap,
  route,
  parent,
) {
  const { path, name } = route
  if (process.env.NODE_ENV !== 'production') {//非生产环境警告，配置信息path是必须的
    assert(path != null, `"path" is required in a route configuration.`)
    assert(//非生产环境警告，component不能是字符串，必须是一个真实的组件
      typeof route.component !== 'string',
      `route config "component" for path: ${String(
        path || name
      )} cannot be a ` + `string id. Use an actual component instead.`
    )
  }
  //https://router.vuejs.org/zh/api/#router-%E6%9E%84%E5%BB%BA%E9%80%89%E9%A1%B9
  const pathToRegexpOptions = //正则配置 options
    route.pathToRegexpOptions || {}
  //标准化path
  const normalizedPath = normalizePath(path, parent, pathToRegexpOptions.strict)
  if (typeof route.caseSensitive === 'boolean') {// 正则匹配规则是否大小写敏感？(默认值：false)
    pathToRegexpOptions.sensitive = route.caseSensitive
  }
  const record = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    name,
    parent,
    meta: route.meta || {},
  }
  // **嵌套路由**
  if (route.children) {
    route.children.forEach(child => {//这里当前的record就是下一层的parent
      addRouteRecord(pathList, pathMap, nameMap, child, record)
    })
  }
  //去掉重复的path定义
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }
  //
  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else if (process.env.NODE_ENV !== 'production') {
      warn(//非生产环境警告，配置信息name不能重复
        false,
        `Duplicate named routes definition: ` +
        `{ name: "${name}", path: "${record.path}" }`
      )
    }
  }
}
//path: /man/:id
//regex.keys: [{name: id,optional: false……}]
function compileRouteRegex(
  path,
  pathToRegexpOptions
) {
  const regex = Regexp(path, [], pathToRegexpOptions)
  if (process.env.NODE_ENV !== 'production') {
    const keys = Object.create(null)
    regex.keys.forEach(key => {
      warn(// 比如这样的/man/:id/:id会发出警告
        !keys[key.name],
        `Duplicate param keys in route with path: "${path}"`
      )
      keys[key.name] = true
    })
  }
  return regex
}
// 标准化path
function normalizePath(
  path,
  parent,
  strict
) {
  if (!strict) path = path.replace(/\/$/, '') // 非严格模式会去掉path最后的 /
  if (path[0] === '/') return path
  if (parent == null) return path
  return cleanPath(`${parent.path}/${path}`)
}
```

