# Vuex version 3.1.3

> 先看下Vuex仓库的目录结构

```
- Vuex
    - src
        - modules
            - module-collection.js // 注册module，生成一个module树，建立状态树的关系
            - module.js // module实例的构造函数
        - plugins
            - devtool.js // 
            - logger.js // Vuex自带的日志插件
        - helpers.js // mapState mapGetters mapMutations mapActions 语法糖
        - index.esm.js // 入口函数
        - index.js // 入口函数
        - mixin.js 
        - store.js // Store构造函数，初始化整个Vuex仓库
        - util.js // 工具函数库
```

## Vuex 初始化

> 在项目中我们使用Vuex通常都会通过 import Vuex from 'Vuex'，对应的就是/src/index.js

```
export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}

```
### install

> 在使用Vuex之前，我们会通过Vue.use(Vuex)去注册Vuex，Vue.use方法注册在vue/src/core/global-api/use.js中

> _installedPlugins来缓存已注册的plugin，防止重复注册，然后去执行plugin的install方法

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

> Vuex提供了install方法，定义在/src/store.js中

> install方法通过把传入的_Vue赋值给全局的Vue，防止重复注册，Vue在store.js最上面定义了

```
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```

> 执行applyMixin方法，实际上是通过beforeCreate钩子，把options.store存到this.$store对象上，定义在src/mixin.js中

```
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0]) // 对vue版本做判断，2.0版本以上通过beforeCreate全局钩子混入

  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }
```

> vuexInit方法实际上是去合并配置项取new Vue时传入的store对象，子组件通过parent.$store获取

```
  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options // 在执行beforeCreate钩子之前去合并配置
    // store injection
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}

```
## Store 实例化

> new Vue的时候会传入store对象，这个就是我们在后面拿到的options.store

```
cosnt store = new Vuex.store({
    state,
    getters,
    mutations,
    actions,
    modules
})
new Vue({
    store,
    ...
})
```

> 看一下Store的构造函数，定义在src/store.js中

- _committing 在提交commit时用来锁定状态，防止手动去修改state，dev时报错提醒
- _actions 收集注册的actions
- _mutations 收集注册的mutations
- _wrappedGetters 收集注册的getters
- ModuleCollection 建立module的树，维护modules的关系
- _modulesNamespaceMap 建立命名空间的modules的映射
- _subscribers 用于订阅状态更新
- _watcherVM
- _makeLocalGettersCache // getter做了个缓存 防止namespace的重复注册
- installModule // 递归注册所有module的state，getters，mutations，actions，执行makeLocalContext做命名空间的处理
- resetStoreVM 实现Vuex的响应式
```
export class Store {
    constructor (options = {}) {
        if (!Vue && typeof window !== 'undefined' && window.Vue) {
            install(window.Vue)
        }
        ...
        // store internal state
        this._committing = false // 在提交commit时用来锁定状态，防止手动去修改state，dev时报错提醒
        this._actions = Object.create(null)
        this._actionSubscribers = []
        this._mutations = Object.create(null)
        this._wrappedGetters = Object.create(null)
        this._modules = new ModuleCollection(options)
        this._modulesNamespaceMap = Object.create(null)
        this._subscribers = []
        this._watcherVM = new Vue()
        this._makeLocalGettersCache = Object.create(null)
        ...
        // bind commit and dispatch to self
        const store = this
        const { dispatch, commit } = this
        this.dispatch = function boundDispatch (type, payload) {
            return dispatch.call(store, type, payload)
        }
        this.commit = function boundCommit (type, payload, options) {
            return commit.call(store, type, payload, options)
        }
        const state = this._modules.root.state

        // init root module.
        // this also recursively registers all sub-modules
        // and collects all module getters inside this._wrappedGetters
        installModule(this, state, [], this._modules.root)

        // initialize the store vm, which is responsible for the reactivity
        // (also registers _wrappedGetters as computed properties)
        resetStoreVM(this, state)
        ...
    }
}
```

## ModuleCollection

> store可以看做是一个root module，在根module下的modules可以看做是子module，Vuex通过ModuleCollection把我们传入的参数对象构造成一个module树状结构，方法定义在src/module/module-collection.js

```
    export default class ModuleCollection {
        constructor (rawRootModule) {
            // register root module (Vuex.Store options)
            this.register([], rawRootModule, false)
        }

        get (path) {
            return path.reduce((module, key) => {
            return module.getChild(key)
            }, this.root)
        }

        getNamespace (path) {
            let module = this.root
            return path.reduce((namespace, key) => {
            module = module.getChild(key)
            return namespace + (module.namespaced ? key + '/' : '')
            }, '')
        }

        update (rawRootModule) {
            update([], this.root, rawRootModule)
        }

        register (path, rawModule, runtime = true) {
            if (process.env.NODE_ENV !== 'production') {
                assertRawModule(path, rawModule)
            }

            const newModule = new Module(rawModule, runtime)
            if (path.length === 0) {
                this.root = newModule
            } else {
                const parent = this.get(path.slice(0, -1))
                parent.addChild(path[path.length - 1], newModule)
            }

            // register nested modules
            if (rawModule.modules) {
                forEachValue(rawModule.modules, (rawChildModule, key) => {
                    this.register(path.concat(key), rawChildModule, runtime)
                })
            }
        }

        unregister (path) {
            const parent = this.get(path.slice(0, -1))
            const key = path[path.length - 1]
            if (!parent.getChild(key).runtime) return

            parent.removeChild(key)
        }
    }
```

> 执行new ModuleCollection会去注册module，传入的第一个参数是一个空数组用于维护module的一个层级关系，第二个参数rawModule就是我们传入的参数对象，runtime默认false

- 首次进入path数组长度为0，创建根节点root
- 判断当前module是否存在子module，循环注册子module，path去拼接module的key，即namespace
- 有子module的时候，第二次进入，path长度不为0，执行else，通过get去获取父module，获取父module的方式很简单，通过path.slice(0, -1)即数组最后一个的上一个值，然后通过reduce去获取，传入的默认是this.root，即当前根节点，显然第二次进去当前module是没有子module的，就返回this.root
- 最后通过addChild把当前module添加到父module上
- 继续判断当前module是否有子module
```
    register (path, rawModule, runtime = true) {
        if (process.env.NODE_ENV !== 'production') {
            assertRawModule(path, rawModule)
        }

        const newModule = new Module(rawModule, runtime)
        if (path.length === 0) {
            this.root = newModule
        } else {
            const parent = this.get(path.slice(0, -1))
            parent.addChild(path[path.length - 1], newModule)
        }

        // register nested modules
        if (rawModule.modules) {
            forEachValue(rawModule.modules, (rawChildModule, key) => {
                this.register(path.concat(key), rawChildModule, runtime)
            })
        }
    }
```

> 这里做了一个例子，通过path.slice(0, -1)可以取到父module
```
/** 
*   对于module a -> c ['a', 'c']
*   对于module b -> d -> e ['b', 'd', 'e']
*   对于module b -> g ['b', 'g']
*   对于module f -> g ['g', 'g']
*/
    var root = {
        modules: {
            a: {
                modules: {
                    c: {}
                }
            },
            b: {
                modules: {
                    d: {
                        modules: {
                            e: {}
                        }
                    },
                    g: {
                        
                    }
                }
            },
            f: {
                modules: {
                    g: {}
                }
            }
        }
    }
    class ModuleCollection {
        constructor (path, rawModule) {
            this.register(path, rawModule)
        }
        register (path, rawModule) {
            console.log(path);
            if (rawModule.modules) {
                Object.keys(rawModule.modules).forEach(key => {
                    this.register(path.concat(key), rawModule.modules[key])
                })
            }
        }
    }
/**
*   
*/
    const root = {
        children: {
            a: {
                state: 1,
                children: {
                    b: {
                        state: 2,
                        children: {
                            c: {
                                state: 3,
                                chilrend: {
                                    d: {
                                        state: 4
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    const path = ['a', 'b', 'c', 'd']
    const get = path => {
        return path.reduce((module, key) => module.children[key], root)
    }
    get(path.slice(0, -1)) // 可以取到d的父module
```

## Module

> 方法定义在src/module/module.js中，主要是创建module实例

- runtime
- _children 存储子module，维护父子module的关系
- _rawModule 缓存当前module
- state 当前module的state
- namespaced 当前是否为命名空间的module
- addChild 添加子module
- removeChild 移除子module
- getChild 获取子module
- update 更新_rawModule
```

    export default class Module {
        constructor (rawModule, runtime) {
            this.runtime = runtime
            // Store some children item
            this._children = Object.create(null)
            // Store the origin module object which passed by programmer
            this._rawModule = rawModule
            const rawState = rawModule.state

            // Store the origin module's state
            this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
        }

        get namespaced () {
            return !!this._rawModule.namespaced
        }

        addChild (key, module) {
            this._children[key] = module
        }

        removeChild (key) {
            delete this._children[key]
        }

        getChild (key) {
            return this._children[key]
        }

        update (rawModule) {
            this._rawModule.namespaced = rawModule.namespaced
            if (rawModule.actions) {
                this._rawModule.actions = rawModule.actions
            }
            if (rawModule.mutations) {
                this._rawModule.mutations = rawModule.mutations
            }
            if (rawModule.getters) {
                this._rawModule.getters = rawModule.getters
            }
        }

        forEachChild (fn) {
            forEachValue(this._children, fn)
        }

        forEachGetter (fn) {
            if (this._rawModule.getters) {
                forEachValue(this._rawModule.getters, fn)
            }
        }

        forEachAction (fn) {
            if (this._rawModule.actions) {
                forEachValue(this._rawModule.actions, fn)
            }
        }

        forEachMutation (fn) {
            if (this._rawModule.mutations) {
                forEachValue(this._rawModule.mutations, fn)
            }
        }
    }
```

## installModule

> 递归注册所有module，初始化state，getters，mutations，actions

```
/**
* store 当前store实例
* rootState 根节点的state
* path 维护父子module关系的key数组
* module 根module
* hot 
*/
    function installModule (store, rootState, path, module, hot) {
        const isRoot = !path.length
        const namespace = store._modules.getNamespace(path)

        // register in namespace map
        if (module.namespaced) {
            if (store._modulesNamespaceMap[namespace] && process.env.NODE_ENV !== 'production') {
            console.error(`[vuex] duplicate namespace ${namespace} for the namespaced module ${path.join('/')}`)
            }
            store._modulesNamespaceMap[namespace] = module
        }

        // set state
        if (!isRoot && !hot) {
            const parentState = getNestedState(rootState, path.slice(0, -1))
            const moduleName = path[path.length - 1]
            store._withCommit(() => {
            if (process.env.NODE_ENV !== 'production') {
                if (moduleName in parentState) {
                console.warn(
                    `[vuex] state field "${moduleName}" was overridden by a module with the same name at "${path.join('.')}"`
                )
                }
            }
            Vue.set(parentState, moduleName, module.state)
            })
        }

        const local = module.context = makeLocalContext(store, namespace, path)

        module.forEachMutation((mutation, key) => {
            const namespacedType = namespace + key
            registerMutation(store, namespacedType, mutation, local)
        })

        module.forEachAction((action, key) => {
            const type = action.root ? key : namespace + key
            const handler = action.handler || action
            registerAction(store, type, handler, local)
        })

        module.forEachGetter((getter, key) => {
            const namespacedType = namespace + key
            registerGetter(store, namespacedType, getter, local)
        })

        module.forEachChild((child, key) => {
            installModule(store, rootState, path.concat(key), child, hot)
        })
    }
```

## makeLocalContext

> 返回ModuleCollection中看一下namaspace是怎么获取的

```
    getNamespace (path) {
        let module = this.root
        return path.reduce((namespace, key) => {
        module = module.getChild(key)
        return namespace + (module.namespaced ? key + '/' : '')
        }, '')
    }
```

> 还是老样子举个例子

```
    let root = {
        namespaced: false,
        state: 1,
        children: {
            a: {
                namespaced: true,
                state: 2,
                children: {
                    b: {
                        namespaced: true,
                        state: 3,
                        children: {
                            c: {
                                namespaced: true,
                                children: {
                                    d: {
                                        namespaced: true,
                                        state: 4
                                    }
                                }
                            }
                        }
                    }
                }
            },
            b: {
                namespaced: true,
                state: 5,
                children: {
                    e: {
                        namespaced: true,
                        state: 6
                    }
                }
            }
        }
    }
    const path = ['a', 'b', 'c', 'd']
    let module = root
    const getNamespace = path => path.reduce((namespace, key) => 
    (module = module.children[key], namespace + (module.namespaced ? key + '/' : '')), '')

    getNamespace(path) // a/b/c/d/
```

> 拼接namespace，返回一个local对象，如果不存在命名空间则返回store.dispath,否则把namespace拼接进去

```
    function makeLocalContext (store, namespace, path) {
        const noNamespace = namespace === ''

        const local = {
            dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
            const args = unifyObjectStyle(_type, _payload, _options)
            const { payload, options } = args
            let { type } = args

            if (!options || !options.root) {
                type = namespace + type
                if (process.env.NODE_ENV !== 'production' && !store._actions[type]) {
                console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
                return
                }
            }

                return store.dispatch(type, payload)
            },

            commit: noNamespace ? store.commit : (_type, _payload, _options) => {
            const args = unifyObjectStyle(_type, _payload, _options)
            const { payload, options } = args
            let { type } = args

            if (!options || !options.root) {
                type = namespace + type
                if (process.env.NODE_ENV !== 'production' && !store._mutations[type]) {
                    console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
                    return
                }
            }

                store.commit(type, payload, options)
            }
        }

    // getters and state object must be gotten lazily
    // because they will be changed by vm update
        Object.defineProperties(local, {
            getters: {
                get: noNamespace
                    ? () => store.getters
                    : () => makeLocalGetters(store, namespace)
                },
            state: {
                get: () => getNestedState(store.state, path)
            }
        })
        return local
    }
```

> makeLocalGetters，先通过_makeLocalGettersCache缓存防止重复监听，

```
    function makeLocalGetters (store, namespace) {
        if (!store._makeLocalGettersCache[namespace]) {
            const gettersProxy = {}
            const splitPos = namespace.length
            Object.keys(store.getters).forEach(type => {
            // skip if the target getter is not match this namespace
            if (type.slice(0, splitPos) !== namespace) return

            // extract local getter type
            const localType = type.slice(splitPos)

            // Add a port to the getters proxy.
            // Define as getter property because
            // we do not want to evaluate the getters in this time.
            Object.defineProperty(gettersProxy, localType, {
                get: () => store.getters[type],
                enumerable: true
            })
            })
            store._makeLocalGettersCache[namespace] = gettersProxy
        }

        return store._makeLocalGettersCache[namespace]
    }
```

> getNestedState通过reduce逐级获取，state的定义跟getters不同，getters是通过 'a/b/c' 去读取key，而state是对象形式，因此只需要根据path去获取即可

```
    function getNestedState (state, path) {
        return path.reduce((state, key) => state[key], state)
    }
```

## registerMutation 和 registerAction

> 注册mutation和action时会把store实例传入，type即带命名空间的属性，handler即执行函数，local通过makeLocalContext转换过得对象，分别存到_mutations和_actions中

```
    function registerMutation (store, type, handler, local) {
        const entry = store._mutations[type] || (store._mutations[type] = [])
        entry.push(function wrappedMutationHandler (payload) {
            handler.call(store, local.state, payload)
        })
    }

    function registerAction (store, type, handler, local) {
        const entry = store._actions[type] || (store._actions[type] = [])
        entry.push(function wrappedActionHandler (payload) {
            let res = handler.call(store, {
                dispatch: local.dispatch, // 我们在执行action函数式可以拿到转换后的dispatch
                commit: local.commit,
                getters: local.getters,
                state: local.state, // 命名空间的state
                rootGetters: store.getters, // 根module的getters
                rootState: store.state // 根module的state
            }, payload)
            if (!isPromise(res)) { // 判断是否支持promise
                res = Promise.resolve(res) // 返回promise
            }
            if (store._devtoolHook) {
                return res.catch(err => {
                    store._devtoolHook.emit('vuex:error', err)
                    throw err
                })
            } else {
                return res
            }
        })
    }
```

## registerGetter

> 先判断是getter是否重复，然后getters存在_wrappedGetters上，rawGetter即我们定义的回调函数

```
    function registerGetter (store, type, rawGetter, local) {
        if (store._wrappedGetters[type]) {
            if (process.env.NODE_ENV !== 'production') {
                console.error(`[vuex] duplicate getter key: ${type}`)
            }
            return
        }
        store._wrappedGetters[type] = function wrappedGetter (store) {
            return rawGetter(
                local.state, // local state
                local.getters, // local getters
                store.state, // root state
                store.getters // root getters
            )
        }
    }
```

## resetStoreVM

> 主要通过store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  }) 实现数据的响应式更新，我们在读取state时其实就是访问store._vm.$$state

```
    get state () {
        return this._vm._data.$$state
    }

    function resetStoreVM (store, state, hot) {
        const oldVm = store._vm

        // bind store public getters
        store.getters = {}
        // reset local getters cache
        store._makeLocalGettersCache = Object.create(null)
        const wrappedGetters = store._wrappedGetters
        const computed = {}
        forEachValue(wrappedGetters, (fn, key) => {
            // use computed to leverage its lazy-caching mechanism
            // direct inline function use will lead to closure preserving oldVm.
            // using partial to return function with only arguments preserved in closure environment.
            computed[key] = partial(fn, store)
            Object.defineProperty(store.getters, key, {
            get: () => store._vm[key],
            enumerable: true // for local getters
            })
        })

        // use a Vue instance to store the state tree
        // suppress warnings just in case the user has added
        // some funky global mixins
        const silent = Vue.config.silent
        Vue.config.silent = true
        store._vm = new Vue({
            data: {
            $$state: state
            },
            computed
        })
        Vue.config.silent = silent

        // enable strict mode for new vm
        if (store.strict) {
            enableStrictMode(store)
        }

        if (oldVm) {
            if (hot) {
                // dispatch changes in all subscribed watchers
                // to force getter re-evaluation for hot reloading.
                store._withCommit(() => {
                    oldVm._data.$$state = null
                })
            }
            Vue.nextTick(() => oldVm.$destroy())
        }
    }
```

## Vuex语法糖

> 我们在使用mapState等方法时可以支持命名空间定义，也可以直接通过/定义，定义在src/helper.js中

> 我们可以传数组，也可以传对象

```
    Vuex.mapState('a', {
        acount: state => state.count,
        f: state => state.f
    })
    Vuex.mapState('a/acount')

    Vuex.mapState(['a'])
    Vuex.mapState({a: state => state.count})
```

> 主要是通过normalizeNamespace和normalizeMap两个方法做了转换

```
    function normalizeNamespace (fn) {
        return (namespace, map) => {
            if (typeof namespace !== 'string') {
                map = namespace
                namespace = ''
            } else if (namespace.charAt(namespace.length - 1) !== '/') {
                namespace += '/'
            }
            return fn(namespace, map)
        }
    }
    function normalizeMap (map) {
        if (!isValidMap(map)) {
            return []
        }
        return Array.isArray(map)
            ? map.map(key => ({ key, val: key }))
            : Object.keys(map).map(key => ({ key, val: map[key] }))
    }
```

> 之前在store初始化时定义的_modulesNamespaceMap，通过获取命名空间的映射返回

```
    function getModuleByNamespace (store, helper, namespace) {
        const module = store._modulesNamespaceMap[namespace]
        if (process.env.NODE_ENV !== 'production' && !module) {
            console.error(`[vuex] module namespace not found in ${helper}(): ${namespace}`)
        }
        return module
    }
```