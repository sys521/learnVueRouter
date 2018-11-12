# vue-router 源码概览

## clone 一份vue-router源码。

当我们开始阅读一份源码时候，我们首先要做的是，看源码的```package.json``` ，从中，我们可以看到一些构建运行命令和一些依赖包。

```
"scripts": {
    "dev": "node examples/server.js",
    "dev:dist": "rollup -wm -c build/rollup.dev.config.js",
    "build": "node build/build.js",
    "lint": "eslint src examples",
    "test": "npm run lint && npm run flow && npm run test:unit && npm run test:e2e && npm run test:types",
    "flow": "flow check",
    "test:unit": "jasmine JASMINE_CONFIG_PATH=test/unit/jasmine.json",
    "test:e2e": "node test/e2e/runner.js",
    "test:types": "tsc -p types/test",
    "docs": "vuepress dev docs",
    "docs:build": "vuepress build docs",
    "release": "bash build/release.sh"
  },
```
从```script```脚本，我们大致可以看到,vue-router的```dev: node examples/servers.js``` 也可以看到。dev:dist大致依赖于 ```rollup``` 打包工具。

然后，我们打开```server.js``` 不难发现，这是一个```express``` 加 ```webpack-dev-middleware```来搭建的开发环境。大致就是帮我们把examples整个目录，当成一个静态目录，访问相应的html,就是访问相应的```index.html```,然后我们发现，每个具体目录下的```index.html``` 都是一个个vue-router的例子。

每个页面的js 都写了这样一行代码
```
import VueRouter from 'vue-router'
```
奇怪，我们当前的项目就是vue-router 。那么这个vue-router 到底是从哪里引过来的呢？

打开```example/webpack.config.js``` 不难找到
```
resolve: {
    alias: {
      vue: 'vue/dist/vue.esm.js',
      'vue-router': path.join(__dirname, '..', 'src')
    }
  },
```
所以，vue-router 是项目的一个别名。就是我们项目根目录下的```src/index```

搞清楚这些，我们就可以正式来看vue-router的源码了。

## 阅读前准备

在```src/index.js``` 文件，我们看到了一大堆 ```import xxx from ...```语句。这些，我们暂时不用关心，我们只用关心一个,导出了什么？
```
export default class VueRouter {
  ...
}
```
所以，我们知道，当我们在写 引用的就是这个```class VueRouter```。 然后，我们就可以看是阅读了，但是，我们这次的开始阅读，并不是要做到逐行阅读，清楚源码每一行到底来做什么。我们这一遍阅读，大概是为了知道整个项目结构 和 大概的一个流程。 对于一时不好看懂，不清楚作者为什么这样写的，我们先不急于去弄清楚，因为，这样可能只会让我们阅读的思路更混乱。

## 开始阅读

首先，我们把重点放在```VueRouter```这个构造函数上。 因为，我们写vue-router。一般情况下，我们要写的东西。一句代码就是
```
const router = new VueRouter({...})
```

### VueRouter的构造器

从VueRouter的构造器上，我们可以看到，首先，VueRouter初始化了一些实例属性。包括
```
// router挂载的根Vue实例
  this.app = null
  // router 文档中并没有被提到,后面我们可以了解到
    this.apps = []
    // 透传的options
    this.options = options
    // 三个钩子函数数组
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []

    // 根据名字好像是匹配的规则
    this.matcher = createMatcher(options.routes || [], this)

    // 模式
    let mode = options.mode || 'hash'
    // 是否回退标志, 浏览器支持 pushState
    this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false

    if (this.fallback) {
      mode = 'hash'
    }

    // 不是在浏览器中
    if (!inBrowser) {
      mode = 'abstract'
    }

    // 重新确定mode.
    this.mode = mode

    switch (mode) {
      case 'history':
        // 构造了最重要的history对象
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
          // 断言, 无效的模式
          assert(false, `invalid mode: ${mode}`)
        }
    }
```

首先，我们大概可以看到，```this.app```官方文档中有讲，这个将来会指向 Vue的跟根组件。然后初始话的其他的东西，我们可以大概猜测一下。

然后，我们大概确定这里面，比较重要的部分就是this.matcher。 和 this.history了。具体干了什么，我们先不关心。

### VueRouter的实例上的方法

我们大致看到有这么一些方法

```
// match 方法
  match
  // 获取当前路径
  get currentRoute (): ?Route {
    return this.history && this.history.current
  }
  // init 方法
  init (app: any /* Vue component instance */) {
  }
  // 全局的beforeEach
  beforeEach

  beforeResolve

  afterEach

  onReady

  onError

  push

  replace
  go

  back

  forward

  getMatchedComponents

  resolve

  addRoutes
```
大致，有这么些方法。

到这里，我们还是一头雾水。直到，我们看到最后还有

```
// 扩展VueRouter， 加上install方法
VueRouter.install = install
VueRouter.version = '__VERSION__'

if (inBrowser && window.Vue) {
  window.Vue.use(VueRouter)
}
```
### intall 方法

终于 VueRouter 有了install 毕竟 他是一个Vue的插件，我们知道Vue的插件，必须是一个有install方法的对象。

所以，我们可以看到install 来自同目录下的install.js

然后，再去看install.js


整个install.js 的内容不多。而且，我们可以很清晰的大致了解install 内容都干了什么
```

首先，有一个全局的混入，在Vue实例beforeCreated 和 destoryed 阶段调用。

```
 Vue.mixin({
    beforeCreate () {
      // this 其实指的就是Vue的实例，根组件 或者 组件， this.options对象，会获取到我们传过去的 new VueRouter()
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router

        this._router.init(this)
        // $router 响应式的面纱
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })
```
我们可以大致了解到。 首先给Vue实例上增加了几个属性 ```_router, _routerRoot``` 然后，通过this.$options 选项，拿到我们最初传给Vue的 options,
```
new Vue({
  el: '#app',
  router,
  ...
})
```
其中router, 我们在构造完Vue后，可以通过this.$options拿到，这些，vue官方文档上也有说明。
然后，用 vue.util.defineReactive。。。。

```
  // 劫持Vue.$router属性 返回的是this._routerRoot_router, 实际上这个对象指的是 VueRouter实例的history对象
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })
  // 劫持Vue.$route 属性。 返回的是
  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })
```
这段代码确保，我们可以在每一个组件中访问$router, 都指向的是this._rooterRoot, this_rotterRoot._router 又指向的是this._router 也就是 this.$options.router 就是我们使用```new Vue({router})```中的router. 访问this._routerRoot._route 其实，就是访问的 ```this.$options.router.history.current```


```
 Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)
```
最后挂在了两个通用的组建

```
const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
```
 设定了合并策略。。这个合并策略和created的合并策略保持了一样




