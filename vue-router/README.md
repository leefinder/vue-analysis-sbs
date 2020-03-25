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
            if (installedPlugins.indexOf(plugin) > -1) { // 防止重复注册
                return this
            }

            // additional parameters
            const args = toArray(arguments, 1)
            args.unshift(this)
            if (typeof plugin.install === 'function') {
                plugin.install.apply(plugin, args) // 调用plugin定义好的install方法
            } else if (typeof plugin === 'function') {
                plugin.apply(null, args)
            }
            installedPlugins.push(plugin) // 把plugin存入队列防止重复注册
            return this
        }
    }
```

## VueRouter.install

> VueRouter.install = install，静态方法install 定义在src/install.js中

> 通过Vue.mixin在执行beforeCreate钩子时把我们传入的router混入到Vue实例上，this._routerRoot表示Vue实例本身，this._router表示传入的router实例，通过this._router.init(this)初始化VueRouter，调用Vue.util.defineReactive(this, '_route', this._router.history.current)，把_route变成响应式对象，通过Object.defineProperty在Vue的原型上添加$router和$route，注册RouterView和RouterLink组件，初始化路由的全局钩子mergeHook，把路由钩子存放在一个数组中

![VueRouter实例](./VueRouter.png)

```
    export function install (Vue) {
        if (install.installed && _Vue === Vue) return // 防止重复installed
        install.installed = true // 第一次执行后installed赋值true

        _Vue = Vue // 把传入的Vue赋值给全局的_Vue

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
        // 路由全局钩子mergeHook 在vue中全局生命周期钩子初始化时created = mergeHook
        strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
    }
```

> Vue生命周期钩子初始化，我们在[合并配置mergeOptions](https://github.com/leefinder/vue-analysis-sbs/tree/master/%E5%90%88%E5%B9%B6%E9%85%8D%E7%BD%AEmergeOptions)中讲到过

```
    LIFECYCLE_HOOKS.forEach(function (hook) {
        strats[hook] = mergeHook;
    });
    function mergeHook (
        parentVal,
        childVal
    ) {
        var res = childVal
        ? parentVal
            ? parentVal.concat(childVal)
            : Array.isArray(childVal)
            ? childVal
            : [childVal]
        : parentVal;
        return res
        ? dedupeHooks(res)
        : res
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

## createMatcher

> createMatcher实际上是返回了match和addRoutes两个方法

```
function createMatcher (routes, router) {
    const { pathList, pathMap, nameMap } = createRouteMap(routes) // 方法定义在src/create-route-map.js中
    function addRoutes () {
        createRouteMap(routes, pathList, pathMap, nameMap)
    }
    function match () { ... }
    function redirect () { ... }
    function alias () { ... }
    function _createRoute () { ... }
    return {
        match,
        addRoutes
    }
}
```

> match

> addRoutes实际上是调用createRouteMap方法,createRouteMap会去遍历我们传入的路由配置调用addRouteRecord，返回pathList，pathMap，nameMap这3个对象

```
/**
* routes 就是我们传入的路由配置
* oldPathList 第一次执行的时候是传入undefined， 第二次执行addRoutes 会把上一次的路径保存下来，传入
* oldPathMap 第一次执行的时候是传入undefined
* oldNameMap 第一次执行的时候是传入undefined
*/
function createRouteMap (routes, oldPathList, oldPathMap, oldNameMap) {
    const pathList: Array<string> = oldPathList || [] // 初始化pathList，数组
    // $flow-disable-line
    const pathMap: Dictionary<RouteRecord> = oldPathMap || Object.create(null) // 初始化的pathMap，初始化一个新对象，没有继承Object的任何原型
    // $flow-disable-line
    const nameMap: Dictionary<RouteRecord> = oldNameMap || Object.create(null)

    routes.forEach(route => {
        addRouteRecord(pathList, pathMap, nameMap, route)
    })
}
```

> addRouteRecord，通过把route配置转化成record对象，维护一个route的树状关系

> normalizePath拼接一个完整的路由地址，record.path = normalizedPath，如果path的第一个字符是'/'就返回这个path，否则拼接上父路径，如果没有父路径也直接返回，这个path存在pathList数组中；path到record的映射关系存放在pathMap中；vue的路由配置支持name字段配置，这里通过nameMap记录name到record的映射关系

```example

父路径 /parent
子路径 /child
最终路径 /child

父路径 /parent
子路径 child
最终路径 /parent/child

```

```
function addRouteRecord (pathList, pathMap, nameMap, route, parent, matchAs) {
    const pathToRegexpOptions: PathToRegexpOptions =
    route.pathToRegexpOptions || {}
    const normalizedPath = normalizePath(path, parent, pathToRegexpOptions.strict)

    if (typeof route.caseSensitive === 'boolean') {
        pathToRegexpOptions.sensitive = route.caseSensitive
    }

    const record: RouteRecord = {
        path: normalizedPath, // 一个完整的路由地址
        regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
        components: route.components || { default: route.component }, // 路由配置传入的组件对象
        instances: {}, // 组件实例
        name, // 通过name匹配路由
        parent, // 当前record的父record
        matchAs,
        redirect: route.redirect, // 重定向
        beforeEnter: route.beforeEnter, // 路由配置的钩子
        meta: route.meta || {}, // 路由的meta对象
        props:
        route.props == null
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
            if (
                route.name &&
                !route.redirect &&
                route.children.some(child => /^\/?$/.test(child.path))
            ) {
                warn(
                false,
                `Named Route '${route.name}' has a default child route. ` +
                    `When navigating to this named route (:to="{name: '${
                    route.name
                    }'"), ` +
                    `the default child route will not be rendered. Remove the name from ` +
                    `this route and use the name of the default child route for named ` +
                    `links instead.`
                )
            }
        }
        route.children.forEach(child => { // 递归创建children的record
            const childMatchAs = matchAs
                ? cleanPath(`${matchAs}/${child.path}`)
                : undefined
            addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
        })
        if (!pathMap[record.path]) {
            pathList.push(record.path) // 记录完整的路由地址
            pathMap[record.path] = record // 记录path到record的映射关系
        }
        if (route.alias !== undefined) { // 路由别名这个先不看了
            ...
        }
        if (name) {
            if (!nameMap[name]) {
                nameMap[name] = record // 记录name到record的映射关系
            } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
                warn(
                    false,
                    `Duplicate named routes definition: ` +
                    `{ name: "${name}", path: "${record.path}" }`
                )
            }
        }
    }
}
```

