# vue-router 3.1.6

> 先看下Vuex仓库的目录结构

```
- vue-router
    - src
        - components
            - link.js
            - view.js
        - history
            - abstract.js 
            - base.js
            - errors.js
            - hash.js
            - html5.js
        - util
            - async.js
            - dom.js
            - location.js
            - misc.js
            - params.js
            - path.js
            - push-state.js
            - query.js
            - resolve-components.js
            - route.js
            - scroll.js
            - state-key.js
            - warn.js
        - create-matcher.js
        - create-route-map.js
        - index.js
        - install.js
```

## Vue.use(Router)

> 执行vue-router中定义的install方法，Vue.use方法注册在vue/src/core/global-api/use.js中


```
    export function initUse (Vue: GlobalAPI) {
        Vue.use = function (plugin: Function | Object) {
            const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
            if (installedPlugins.indexOf(plugin) > -1) {
                return this
            }

            // additional parameters
            const args = toArray(arguments, 1)
            args.unshift(this)
            if (typeof plugin.install === 'function') {
                plugin.install.apply(plugin, args)
            } else if (typeof plugin === 'function') {
                plugin.apply(null, args)
            }
            installedPlugins.push(plugin)
            return this
        }
    }
```

## VueRouter.install

> VueRouter.install = install，静态方法install 定义在src/install.js中

> 通过Vue.mixin在执行beforeCreate钩子时把我们传入的router混入到Vue实例上，this._routerRoot表示Vue实例本身，this._router表示传入的router实例，通过this._router.init(this)初始化VueRouter，调用Vue.util.defineReactive(this, '_route', this._router.history.current)，把_route变成响应式对象，通过Object.defineProperty在Vue的原型上添加$router和$route，注册RouterView和RouterLink组件

![VueRouter实例](./VueRouter.png)

```
    export function install (Vue) {
        if (install.installed && _Vue === Vue) return
        install.installed = true

        _Vue = Vue

        const isDef = v => v !== undefined

        const registerInstance = (vm, callVal) => {
            let i = vm.$options._parentVnode
            if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
                i(vm, callVal)
            }
        }

        Vue.mixin({
            beforeCreate () {
            if (isDef(this.$options.router)) { // 我们在new Vue时传入的VueRouter对象，会在mergeOptions时把router合并配置到$options上
                this._routerRoot = this // _routerRoot即Vue实例本身
                this._router = this.$options.router // VueRouter实例赋值给_router
                this._router.init(this) // 执行VueRouter原型方法init初始化路由，添加监听
                Vue.util.defineReactive(this, '_route', this._router.history.current) // _route添加响应式，后面路由重新rander
            } else {
                this._routerRoot = (this.$parent && this.$parent._routerRoot) || this // 往最近的父实例上找
            }
            registerInstance(this, this)
            },
            destroyed () {
                registerInstance(this)
            }
        })

        Object.defineProperty(Vue.prototype, '$router', {
            get () { return this._routerRoot._router } // 访问this.$router即可以拿到router对象
        })

        Object.defineProperty(Vue.prototype, '$route', {
            get () { return this._routerRoot._route }
        })

        Vue.component('RouterView', View)
        Vue.component('RouterLink', Link)

        const strats = Vue.config.optionMergeStrategies
        // use the same hook merging strategy for route hooks
        strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
    }
```

## new VueRouter

> new VueRouter时会把我们提前约定好的路由配置传入，this.app即根Vue实例


```
    export default class VueRouter {
        static install: () => void;
        static version: string;

        app: any;
        apps: Array<any>;
        ready: boolean;
        readyCbs: Array<Function>;
        options: RouterOptions;
        mode: string;
        history: HashHistory | HTML5History | AbstractHistory;
        matcher: Matcher;
        fallback: boolean;
        beforeHooks: Array<?NavigationGuard>;
        resolveHooks: Array<?NavigationGuard>;
        afterHooks: Array<?AfterNavigationHook>;

        constructor (options: RouterOptions = {}) {
            this.app = null // Vue根实例
            this.apps = [] 
            this.options = options // 传入的路由配置
            this.beforeHooks = [] // router.beforeEach 注册的全局前置守卫
            this.resolveHooks = [] // 导航确认前的全局守卫，在组件内守卫解析后调用
            this.afterHooks = [] // 全局后置钩子
            this.matcher = createMatcher(options.routes || [], this) // 路由匹配映射 返回 match 和 addRoutes 

            let mode = options.mode || 'hash' // 默认hash路由
            this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
            if (this.fallback) { // 判断当前浏览器是否支持history路由，否则降级处理
                mode = 'hash'
            }
            if (!inBrowser) {
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
        }
        ...
    }
```