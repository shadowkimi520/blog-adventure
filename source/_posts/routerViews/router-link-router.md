# 完善 `router-link`
[官网介绍](https://router.vuejs.org/zh/api/#router-link)

完整代码分支 [stage-10](https://github.com/shengrongchun/parse-vue-router)

`router-link` 在之前的章节中说的很少，为了和官网功能统一，现在对其完善

```js{16,25-29}
// work around weird flow bug
const toTypes = [String, Object]
const eventTypes = [String, Array]

export default {
  name: 'RouterLink',
  props: {
    to: {
      type: toTypes,
      required: true
    },
    tag: {// 希望渲染的 tag 默认是 a
      type: String,
      default: 'a'
    },
    append: Boolean,//是否在当前 (相对) 路径前添加基路径
    event: {//激活事件
      type: eventTypes,
      default: 'click'
    }
  },
  render(h) {
    const router = this.$router
    const current = this.$route
    const { location, route, href } = router.resolve(
      this.to,
      current,
      this.append
    )
    const data = {
      on: {
        click: () => {
          router.push(location)
        }
      }
    }
    return h(this.tag, data, this.$slots.default)
  }
}
```
我们增加了 `append` 配置项，在 `router.resolve` 获取了 `route` 和 `href` 共后续使用

### 解析 `resolve`
```js{16,19,20}
resolve(
  to,
  current,
  append
) {
  current = current || this.history.current
  const location = normalizeLocation(
    to,
    current,
    append,
    this
  )
  const route = this.match(location, current)//返回匹配的路由对象
  const fullPath = route.redirectedFrom || route.fullPath //获取fullPath
  const base = this.history.base
  const href = createHref(base, fullPath, this.mode) //创建 href
  return {
    location,
    route,
    href
  }
}
```
createHref
```js
function createHref(base, fullPath, mode) {
  var path = mode === 'hash' ? '#' + fullPath : fullPath
  return base ? cleanPath(base + '/' + path) : path
}
```
### 解析添加激活类
激活类有两种：
+ 默认激活类 `active-class`
+ 精确匹配激活类  `exact-active-class`

什么是精确匹配激活呢？

`是否激活`默认类名的依据是包含匹配。 举个例子，如果当前的路径是 `/a` 开头的，那么 `<router-link to="/a">` 也会被设置 `CSS` 类名。
按照这个规则，每个路由都会激活 `<router-link to="/">`！想要链接使用“精确匹配模式”，则使用 `exact` 属性

两种类可以通过全局配置，但自定义优先级更高
```js{16,18,19,34-49,55}
// work around weird flow bug
const toTypes = [String, Object]
const eventTypes = [String, Array]

export default {
  name: 'RouterLink',
  props: {
    to: {
      type: toTypes,
      required: true
    },
    tag: {// 希望渲染的 tag 默认是 a
      type: String,
      default: 'a'
    },
    exact: Boolean,//是否精确激活class
    append: Boolean,//是否在当前 (相对) 路径前添加基路径
    activeClass: String,//设置链接激活时使用的 CSS 类名
    exactActiveClass: String,//配置当链接被精确匹配的时候应该激活的 class
    event: {//激活事件
      type: eventTypes,
      default: 'click'
    }
  },
  render(h) {
    const router = this.$router
    const current = this.$route
    const { location, route, href } = router.resolve(
      this.to,
      current,
      this.append
    )
    //
    const classes = {}
    const globalActiveClass = router.options.linkActiveClass
    const globalExactActiveClass = router.options.linkExactActiveClass
    // Support global empty active class
    const activeClassFallback =
      globalActiveClass == null ? 'router-link-active' : globalActiveClass
    const exactActiveClassFallback =
      globalExactActiveClass == null
        ? 'router-link-exact-active'
        : globalExactActiveClass
    const activeClass =
      this.activeClass == null ? activeClassFallback : this.activeClass
    const exactActiveClass =
      this.exactActiveClass == null
        ? exactActiveClassFallback
        : this.exactActiveClass

    const compareTarget = route.redirectedFrom
      ? createRoute(null, normalizeLocation(route.redirectedFrom), null, router)
      : route

    classes[exactActiveClass] = isSameRoute(current, compareTarget)
    classes[activeClass] = this.exact
      ? classes[exactActiveClass]
      : isIncludedRoute(current, compareTarget)

    const data = {
      class: classes,
      on: {
        click: () => {
          router.push(location)
        }
      }
    }
    return h(this.tag, data, this.$slots.default)
  }
}
```
+ `34-49` 行代码就是解析最后 `activeClass ` 和 `exactActiveClass` 到底是什么，他们受全局和自定义配置的影响
+ 精确匹配是当前路由和 `router-link` 的 `to` 创建的路由相同，那么 `router-link` 是会有 `exactActiveClass`
+ 默认情况下只要 `to` 创建的路由包含在当前路由，`router-link` 就会有 `activeClass`。如果定义了 `exact`，则必须精确匹配才行

### 解析事件 `event`
`event` 默认值是 `click` ，声明可以用来触发导航的事件。可以是一个字符串或是一个包含字符串的数组
```js{42-58}
import { createRoute, isSameRoute, isIncludedRoute } from '../util/route'
import { extend } from '../util/misc'
import { normalizeLocation } from '../util/location'
import { warn } from '../util/warn'
// work around weird flow bug
const toTypes = [String, Object]
const eventTypes = [String, Array]
const noop = () => { }

export default {
  name: 'RouterLink',
  props: {
    to: {
      type: toTypes,
      required: true
    },
    tag: {// 希望渲染的 tag 默认是 a
      type: String,
      default: 'a'
    },
    exact: Boolean,//是否精确激活class
    append: Boolean,//是否在当前 (相对) 路径前添加基路径
    activeClass: String,//设置链接激活时使用的 CSS 类名
    exactActiveClass: String,//配置当链接被精确匹配的时候应该激活的 class
    event: {//激活事件
      type: eventTypes,
      default: 'click'
    }
  },
  render(h) {
    const router = this.$router
    const current = this.$route
    const { location, route, href } = router.resolve(
      this.to,
      current,
      this.append
    )
    //
    const classes = {}
    …………
    //
    const handler = e => {
      if (guardEvent(e)) {
        if (this.replace) {
          router.replace(location, noop)
        } else {
          router.push(location, noop)
        }
      }
    }
    const on = { click: guardEvent }
    if (Array.isArray(this.event)) {
      this.event.forEach(e => {
        on[e] = handler
      })
    } else {
      on[this.event] = handler
    }

    const data = {
      class: classes,
      on
    }
    return h(this.tag, data, this.$slots.default)
  }
}
function guardEvent(e) {
  // don't redirect with control keys
  if (e.metaKey || e.altKey || e.ctrlKey || e.shiftKey) return
  // don't redirect when preventDefault called
  if (e.defaultPrevented) return
  // don't redirect on right click
  if (e.button !== undefined && e.button !== 0) return
  // don't redirect if `target="_blank"`
  if (e.currentTarget && e.currentTarget.getAttribute) {
    const target = e.currentTarget.getAttribute('target')
    if (/\b_blank\b/i.test(target)) return
  }
  // this may be a Weex event which doesn't have this method
  if (e.preventDefault) {
    e.preventDefault()
  }
  return true
}
```
从 `51` 行开始看，当用户定义了 `event` 事件并且没有 `click` 事件的条件下，会自动加上一个 `click` 事件
```js
const on = { click: guardEvent }
```
此时的 `click` 事件没有多大用处，大多数情况下，这个 `click` 事件会被用户定义的或默认的`click`覆盖。最终执行 `handler`
```js
const handler = e => {
  if (guardEvent(e)) {//过滤掉一些无效的事件行为
    if (this.replace) {
      router.replace(location, noop)
    } else {
      router.push(location, noop)
    }
  }
}
```
可知，最终根据 `replace` 的值来判断是执行 `replace` 还是 `push`
