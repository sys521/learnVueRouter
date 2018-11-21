```
function addRouteRecord (
  pathList: Array<string>,
  pathMap: Dictionary<RouteRecord>,
  nameMap: Dictionary<RouteRecord>,
  route: RouteConfig,
  parent?: RouteRecord,
  matchAs?: string
) {
  // 从我们传递的route选项中结构出来 path, 和name
  const { path, name } = route
  if (process.env.NODE_ENV !== 'production') {
    // 非生产环境的警告。
    assert(path != null, `"path" is required in a route configuration.`)
    assert(
      typeof route.component !== 'string',
      `route config "component" for path: ${String(path || name)} cannot be a ` +
      `string id. Use an actual component instead.`
    )
  }
  // 中文文档只说是编译正则的选项。 是一个对象
  const pathToRegexpOptions: PathToRegexpOptions = route.pathToRegexpOptions || {}
  // 规范化path. strict 为空。
  // 返回的就是我们当前的 path. 如果父节点存在，且子节点不以'/'开头，就要加上父节点的路径。
  const normalizedPath = normalizePath(
    path,
    parent,
    pathToRegexpOptions.strict
  )
  // 判断route.caseSensitive 是不是大小写敏感的。
  if (typeof route.caseSensitive === 'boolean') {
    pathToRegexpOptions.sensitive = route.caseSensitive
  }

  const record: RouteRecord = {
    // 规范后的path
    path: normalizedPath,
    // 应该是一个正则对象。
    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
    // vue的子组件
    components: route.components || { default: route.component },

    instances: {},
    // 之前结构过。
    name,
    // 传递进来的。
    parent,
    // 传递进来的。
    matchAs,
    // 重定向选项
    redirect: route.redirect,
    // options.route
    beforeEnter: route.beforeEnter,
    // 路由元信息
    meta: route.meta || {},
    // 传递进来的props
    // 三目运算符
    props: route.props == null
      ? {}
      : route.components
        ? route.props
        : { default: route.props }
  }

  if (route.children) {
    // Warn if route is named, does not redirect and has a default child route.
    // If users navigate to this route by name, the default child will
    // not be rendered (GH Issue #629)
    if (process.env.NODE_ENV !== 'production') {
      if (route.name && !route.redirect && route.children.some(child => /^\/?$/.test(child.path))) {
        warn(
          false,
          `Named Route '${route.name}' has a default child route. ` +
          `When navigating to this named route (:to="{name: '${route.name}'"), ` +
          `the default child route will not be rendered. Remove the name from ` +
          `this route and use the name of the default child route for named ` +
          `links instead.`
        )
      }
    }
    route.children.forEach(child => {
      const childMatchAs = matchAs
        ? cleanPath(`${matchAs}/${child.path}`)
        : undefined
        // 如果有子选项, 将当前的record 作为parent, 将childMatchAs 也传递进去。
      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
    })
  }
  // 处理别名。
  if (route.alias !== undefined) {
    const aliases = Array.isArray(route.alias)
      ? route.alias
      : [route.alias]
    // 如果aliase存在。做的一件事，就是在各种字典中加入以alias为名的数据。
    aliases.forEach(alias => {
      const aliasRoute = {
        path: alias,
        children: route.children
      }
      addRouteRecord(
        pathList,
        pathMap,
        nameMap,
        aliasRoute,
        parent,
        record.path || '/' // matchAs
      )
    })
  }
  // 不存就添加一个record.path 应该是一个个路径， 最后。pathList: [path1, path2, path3] 且path唯一。
  if (!pathMap[record.path]) {
    pathList.push(record.path)
    pathMap[record.path] = record
  }
  // name存在，就用name 做一个 一一映射。
  if (name) {
    if (!nameMap[name]) {
      nameMap[name] = record
    } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
      warn(
        false,
        `Duplicate named routes definition: ` +
        `{ name: "${name}", path: "${record.path}" }`
      )
    }
  }
}
```

我们来稍微分析一下这个addRouteRecord 函数, 其中routes 有下列的选项，摘录官方文档
```
declare type RouteConfig = {
  path: string;
  component?: Component;
  name?: string; // 命名路由
  components?: { [name: string]: Component }; // 命名视图组件
  redirect?: string | Location | Function;
  props?: boolean | Object | Function;
  alias?: string | Array<string>;
  children?: Array<RouteConfig>; // 嵌套路由
  beforeEnter?: (to: Route, from: Route, next: Function) => void;
  meta?: any;

  // 2.6.0+
  caseSensitive?: boolean; // 匹配规则是否大小写敏感？(默认值：false)
  pathToRegexpOptions?: Object; // 编译正则的选项
}
```

1. 从我们传递的route选项中结构出来 path, 和name
2. 根据环境，看是否有path。 来发出警告。
3. 确定 route 编译正则的选项。// pathToRegexpOptions
4. 规范化path
  ```
  function normalizePath (path: string, parent?: RouteRecord, strict?: boolean): string {
   // 如果不是严格模式， 去掉结尾的 '/'
   if (!strict) path = path.replace(/\/$/, '')
   // 判断path 是不是以'/'开头， 如果是的话，就直接返回 path
   if (path[0] === '/') return path
   if (parent == null) return path
   return cleanPath(`${parent.path}/${path}`)
  }
  ```
  所以，这个就是返回一个符合内部标准的path 其中cleanPath,是一个比较简单的函数，是去掉path中的'//'
5. 确定大小写是否敏感。
6. 生成一个路由记录。
7. 然后判断是否有子选项，如果有子选项的的话，将现有的pathList, pathMap, nameMap, 还有当前子选项，父选项分别再递归调用addRouteRecord
8. 别名处理，只要是有别名的，别名处理的策略实际上就是 在各种字典中加入别名的路由记录，或者路径。
9. 由最后的代码，我们知道，pathList里面存放的是所有构造的路由字符串。 nameMap 放着的是所有 能以name为属     性的路由记录，pathMap 放着的就是以 path 为属性的路由记录

  ```
   if (!pathMap[record.path]) {
      pathList.push(record.path)
      pathMap[record.path] = record
    }
    if (name) {
      if (!nameMap[name]) {
        nameMap[name] = record
      } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
        warn(
          false,
          `Duplicate named routes definition: ` +
          `{ name: "${name}", path: "${record.path}" }`
        )
      }
    }
  ```

当然上面recored里面的值很多，我们也大概描述了一下，包括其中的动态路由部分，我们就不再分析了，我们所有的字典其实都是扁平化的。

