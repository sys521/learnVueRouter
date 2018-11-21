# router-view

我们已经大概了解了，vue-router大部分做的最重要的事情，就是更新当前路由的，并构造了一个当前的路由对象，这个对象中，就放置了我们对应的模板。然后用```router-view``` 来更新我们的视图。我们可以稍微看一下```router-view```.


```router-view``` 这个组件，放在```/src/component/view.js```中，整个```view.js```代码量并不大，也就100多行。

```
import { warn } from '../util/warn'
import { extend } from '../util/misc'

export default {
  name: 'RouterView',
  // 函数式组件
  functional: true,
  props: {
    name: {
      type: String,
      default: 'default'
    }
  },
  render (_, { props, children, parent, data }) {
    ...
  }
}
```
我们可以看到 router-view 用的是jxs写的函数式组件。具体用法和写法，vue官方文档上有较为详细的说明。
我们主要看这个render函数好了。

```
 render (_, { props, children, parent, data }) {
    // used by devtools to display a router-view badge
    data.routerView = true

    // directly use parent context's createElement() function
    // so that components rendered by router-view can resolve named slots
    const h = parent.$createElement
    const name = props.name
    // $route 指向当前路由对象
    const route = parent.$route
    const cache = parent._routerViewCache || (parent._routerViewCache = {})

    // determine current view depth, also check to see if the tree
    // has been toggled inactive but kept-alive.
    let depth = 0
    let inactive = false
    while (parent && parent._routerRoot !== parent) {
      if (parent.$vnode && parent.$vnode.data.routerView) {
        depth++
      }
      if (parent._inactive) {
        inactive = true
      }
      parent = parent.$parent
    }
    data.routerViewDepth = depth

    // render previous view if the tree is inactive and kept-alive
    if (inactive) {
      return h(cache[name], data, children)
    }

    const matched = route.matched[depth]
    // render empty node if no matched route
    if (!matched) {
      cache[name] = null
      return h()
    }

    const component = cache[name] = matched.components[name]

    // attach instance registration hook
    // this will be called in the instance's injected lifecycle hooks
    data.registerRouteInstance = (vm, val) => {
      // val could be undefined for unregistration
      const current = matched.instances[name]
      if (
        (val && current !== vm) ||
        (!val && current === vm)
      ) {
        matched.instances[name] = val
      }
    }

    // also register instance in prepatch hook
    // in case the same component instance is reused across different routes
    ;(data.hook || (data.hook = {})).prepatch = (_, vnode) => {
      matched.instances[name] = vnode.componentInstance
    }

    // resolve props
    let propsToPass = data.props = resolveProps(route, matched.props && matched.props[name])
    if (propsToPass) {
      // clone to prevent mutation
      propsToPass = data.props = extend({}, propsToPass)
      // pass non-declared props as attrs
      const attrs = data.attrs = data.attrs || {}
      for (const key in propsToPass) {
        if (!component.props || !(key in component.props)) {
          attrs[key] = propsToPass[key]
          delete propsToPass[key]
        }
      }
    }

    return h(component, data, children)
  }
  ```

  我们看代码
  ```
  const h = parent.$createElement

  ```
  这个为什么不用 前面的```_```二用父元素的 crateElement 上面的英文注释也有说明，就是为了解决具名插槽的问题。

  ```
  const route = parent.$route
  ```
  还记得$route指向谁么
  ```
  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })
  Vue.util.defineReactive(this, '_route', this._router.history.current)
  ```
 所以，我们的_route就是当前路由对象。也就是我们之前的vueRouter.history.current对象

```
const cache = parent._routerViewCache || (parent._routerViewCache = {})
```
从命名上来看，就是与缓存相关的。这个暂时我们先不用管。

```
let depth = 0
let inactive = false
```
```inactive```从注释上看,应该是和```keep-alive```相关，暂时也不用考虑。

```
while (parent && parent._routerRoot !== parent) {
  if (parent.$vnode && parent.$vnode.data.routerView) {
    depth++
  }
  if (parent._inactive) {
    inactive = true
  }
  parent = parent.$parent
}
```
这个循环，其实，就是为了确定当前 的router-view的深度的。 也就是，我们当前的router-view 的嵌套深度是多少。

```
const matched = route.matched[depth]
```
然后，我们用当前的嵌套深度，匹配到当前路由的matched。然后，就能拿到相应的mathced中的模板了。


