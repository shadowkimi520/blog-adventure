# 路由组件传参
[官网详解](https://router.vuejs.org/zh/guide/essentials/passing-props.html)

完整代码分支 [stage-4](https://github.com/shengrongchun/parse-vue-router)

在组件中使用 `$route` 会使之与其对应路由形成高度耦合，从而使组件只能在某些特定的 URL 上使用，限制了其灵活性。如
```js{2}
const User = {
  template: '<div>User {{ $route.params.id }}</div>'
}
const router = new VueRouter({
  routes: [
    { path: '/user/:id', component: User }
  ]
})
```
`User` 组件使用非常的受限制，当前路由的 `params` 中必须要有 `id` 才行

使用 `props` 将组件和路由解耦
```js{4}
const User = {
  template: '<div>User {{ $route.params.id }}</div>',
  props: {
    id: {
      type: String,
      default: ''
    }
  }
}
```
`User` 组件只要传 `id` 参数就可以使用他，那么怎么在路由中传 `id ` 呢？
```js{4,6,9}
const router = new VueRouter({
  routes: [
    { path: '/user/:id', component: User,
      props: true, //如果 props 被设置为 true，route.params 将会被设置为组件属性
      //如果 props 是一个对象，它会被按原样设置为组件属性。当 props 是静态的时候有用
      props: { id: 123 },
      //你可以创建一个函数返回 props。这样你便可以将参数转换成另一种类型，
      // 将静态值与基于路由的值结合等等。route为跳转成功后的route对象
      props: (route) => ({ query: route.query.id })
    }
  ]
})
```
在路由配置中，我们有布尔模式，对象模式，函数模式三种。话不多说，直接上代码看如何实现功能

`record` 中添加 `props` 属性
### createRouteMap
```js{21-26}
//
export function createRouteMap(
  routes
) {
  …………
function addRouteRecord(
  pathList,
  pathMap,
  nameMap,
  route,
  parent,
) {
  ……
  const record = {
    path: normalizedPath,
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    components: route.components || { default: route.component },
    name,
    parent,
    meta: route.meta || {},
    props: //路由组件传参 有了这个功能，组件可以不再和$route耦合
      route.props == null
        ? {}
        : route.components
          ? route.props
          : { default: route.props } //非多组件的情况下，就是对默认组件的设置
  }
  ………………
```
因为有[命名视图](https://router.vuejs.org/zh/guide/essentials/named-views.html)的概念，所以一个 `record` 可能会对应多个组件：`components` 。`props` 的设置也要考虑多组件的情况，如果非多组件的情况下，就是对默认组件的设置。
接下来就是在 `router-view` 中解析他

### router-view
```js{16-20,25-62}
import { warn } from '../util/warn'
import { extend } from '../util/misc'

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
    …………
    // 从record中获取此组件name的props
    const configProps = matched.props && matched.props[name]
    // save route and configProps in cache
    if (configProps) {//把props信息注入到组件的props中
      fillPropsinData(component, data, route, configProps)
    }
    return h(component, data, children)
  }
}

function fillPropsinData(component, data, route, configProps) {
  // data.props在组件render过程中，会给相同key的component.props赋值
  // 如：data.props = {name:111},组件的props如果定义了name,name的值会赋值为111
  let propsToPass = data.props = resolveProps(route, configProps)
  if (propsToPass) {
    // clone to prevent mutation
    propsToPass = data.props = extend({}, propsToPass)
    // props传的值没有在组件中的props里定义，就降级存储在attrs中
    const attrs = data.attrs = data.attrs || {}
    for (const key in propsToPass) {
      if (!component.props || !(key in component.props)) {
        attrs[key] = propsToPass[key]
        delete propsToPass[key]
      }
    }
  }
}

function resolveProps(route, config) {//三种类型的判断（布尔，对象，函数）
  switch (typeof config) {
    case 'undefined':
      return
    case 'object':
      return config
    case 'function':
      return config(route) // 函数的参数传的是当前的route
    case 'boolean':
      return config ? route.params : undefined  // 比如 params: {a:1,b:2} --> 在组件的props: {a:1,b:2}
    default:
      if (process.env.NODE_ENV !== 'production') {
        warn(
          false,
          `props in "${route.path}" is a ${typeof config}, ` +
          `expecting an object, function or boolean.`
        )
      }
  }
}
```
