# 合并配置

> new Vue时会去执行mergeOptions去合并options,实际上是对vm.constructor和传入的options做合并

```
// src/core/instance/init.js
    vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
    )
```

## initGlobalAPI

> initGlobalAPI,这里用_base缓存了Vue,这个我们在讲createComponent是提到过vm.$options._base,就是在这里定义的,Vue.options = Object.create(null),通过创建空对象,初始化options配置,挂上components,directives,filters等,代码定义在src/core/global-api/...

```
    Vue.options = Object.create(null) // 初始化options对象
    ASSET_TYPES.forEach(type => { // 全局的 component directive filter初始化
        Vue.options[type + 's'] = Object.create(null)
    })

    // this is used to identify the "base" constructor to extend all plain-object
    // components with in Weex's multi-instance scenarios.
    Vue.options._base = Vue // 把 Vue缓存到options._base

    extend(Vue.options.components, builtInComponents) // 这里就只有keep-alive静态组件

    initUse(Vue) // 注册Vue.use
    initMixin(Vue) // 注册Vue.mixin 其实就是mergeOptions方法
    initExtend(Vue) // 注册Vue.extend
    initAssetRegisters(Vue) // 注册全局 Vue.component Vue.filter Vue.directive方法

```

## mergeOptions

> mergeOptions主要是对parent和child根据合并策略返回新的对象,通过递归调用把extends和mixins合并到parent上,然后遍历parent执行mergeField,再遍历child,如果发现新的key，parent上不存在的则调用mergeField.这里定义了strats就是Vue的全局策略对象

```
export function mergeOptions(
    parent: Object,
    child: Object,
    vm?: Component
): Object {
    if (process.env.NODE_ENV !== 'production') {
        checkComponents(child)
    }

    if (typeof child === 'function') {
        child = child.options
    }

    normalizeProps(child, vm) // 传入的props可以为数组，也可以为对象，这里把这两种类型统一处理成 key-val的对象
    normalizeInject(child, vm)
    normalizeDirectives(child) // 把传入的directives 转成 { bind: def, update: def } 的形式

    if (!child._base) { // 递归调用mergeOptions把extends 和 mixins 合并到parent上
        if (child.extends) {
            parent = mergeOptions(parent, child.extends, vm)
        }
        if (child.mixins) {
            for (let i = 0, l = child.mixins.length; i < l; i++) {
                parent = mergeOptions(parent, child.mixins[i], vm)
            }
        }
    }

    const options = {}
    let key
    for (key in parent) {
        mergeField(key)
    }
    for (key in child) {
        if (!hasOwn(parent, key)) { // 不存在parent上的属性调用mergeField合并
            mergeField(key)
        }
    }
    function mergeField(key) {
        const strat = strats[key] || defaultStrat // 策略模式的应用
        options[key] = strat(parent[key], child[key], vm, key)
    }
    return options
}
```

## lifecycle生命周期钩子定义

```
    LIFECYCLE_HOOKS.forEach(hook => {
        strats[hook] = mergeHook
    })
```

> mergeOptions操作中会把Vue生命周期钩子在这里初始化,并且合并定义Vue实例的生命周期配置

> stracts实际上就是通过Object.create(null),创建了一个空对象

> mergeHook方法是判断当有parentVal,就通过concat拼接数组,否则判断传入的childVal是否是数组,是的话返回,不是就转成数组,最后通过dedupeHooks去掉重复定义
```
    const strats = config.optionMergeStrategies // optionMergeStrategies的定义在src\core\config.js中
```

```
// src\shared\constants.js
    export const LIFECYCLE_HOOKS = [
        'beforeCreate',
        'created',
        'beforeMount',
        'mounted',
        'beforeUpdate',
        'updated',
        'beforeDestroy',
        'destroyed',
        'activated',
        'deactivated',
        'errorCaptured',
        'serverPrefetch'
    ]
// src\core\util\options.js
    function mergeHook(
        parentVal: ?Array<Function>,
        childVal: ?Function | ?Array<Function>
    ): ?Array<Function> {
        const res = childVal
            ? parentVal
                ? parentVal.concat(childVal)
                : Array.isArray(childVal)
                    ? childVal
                    : [childVal]
            : parentVal
        return res
            ? dedupeHooks(res)
            : res
    }

    function dedupeHooks(hooks) {
        const res = []
        for (let i = 0; i < hooks.length; i++) {
            if (res.indexOf(hooks[i]) === -1) {
                res.push(hooks[i])
            }
        }
        return res
    }

    LIFECYCLE_HOOKS.forEach(hook => {
        strats[hook] = mergeHook
    })
```

## mergeData

> strats.data在这里定义,这里也提示了当我们声明的data不是函数时会抛出错误提示,当我们声明的data是对象时,我们在父组件同时注册多个子组件,我们通过调用子组件方法去修改data时,这几个子组件状态会一起变化

> 调用mergeDataOrFn,实际上合并parentVal和childVal去执行mergeData,然后去循环keys,递归调用mergeData

> 如果发现data已经是响应式了就跳过继续,否则调用set去建立响应式data,set实际上是去调用了defineReactive,这个我们在响应式原理中再细说

```
    if (key === '__ob__') continue
    ...
    // src\core\observer\index.js
    set(to, key, fromVal) 
    ...
```

```
function mergeData(to: Object, from: ?Object): Object {
    if (!from) return to
    let key, toVal, fromVal

    const keys = hasSymbol
        ? Reflect.ownKeys(from)
        : Object.keys(from)

    for (let i = 0; i < keys.length; i++) {
        key = keys[i]
        // in case the object is already observed...
        if (key === '__ob__') continue
        toVal = to[key]
        fromVal = from[key]
        if (!hasOwn(to, key)) {
            set(to, key, fromVal)
        } else if (
            toVal !== fromVal &&
            isPlainObject(toVal) &&
            isPlainObject(fromVal)
        ) {
            mergeData(toVal, fromVal)
        }
    }
    return to
}
export function mergeDataOrFn(
    parentVal: any,
    childVal: any,
    vm?: Component
): ?Function {
    if (!vm) {
        // in a Vue.extend merge, both should be functions
        if (!childVal) {
            return parentVal
        }
        if (!parentVal) {
            return childVal
        }
        return function mergedDataFn() {
            return mergeData(
                typeof childVal === 'function' ? childVal.call(this, this) : childVal,
                typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
            )
        }
    } else {
        return function mergedInstanceDataFn() {
            // instance merge
            const instanceData = typeof childVal === 'function'
                ? childVal.call(vm, vm)
                : childVal
            const defaultData = typeof parentVal === 'function'
                ? parentVal.call(vm, vm)
                : parentVal
            if (instanceData) {
                return mergeData(instanceData, defaultData)
            } else {
                return defaultData
            }
        }
    }
}

strats.data = function (
    parentVal: any,
    childVal: any,
    vm?: Component
): ?Function {
    if (!vm) {
        if (childVal && typeof childVal !== 'function') {
            process.env.NODE_ENV !== 'production' && warn(
                'The "data" option should be a function ' +
                'that returns a per-instance value in component ' +
                'definitions.',
                vm
            )

            return parentVal
        }
        return mergeDataOrFn(parentVal, childVal)
    }

    return mergeDataOrFn(parentVal, childVal, vm)
}
```

## resolveConstructorOptions

> 我们知道Vue实例可以通过new Vue构造函数实现,也可以通过Vue.extend方法构造一个子类,实现方法继承,super指向父类,Vue.extend我们在createElement中提到过

```
    Vue.extend = function (extendOptions: Object): Function {
        ...
        Sub['super'] = Super
        ...
    }
```

> 首先递归调用resolveConstructorOptions方法,返回"父类"上的options并赋值给superOptions变量.然后把"自身"的options赋值给cachedSuperOptions变量

[createComponent参考链接](https://github.com/leefinder/vue-analysis-sbs/tree/master/createComponent)

```
export function resolveConstructorOptions(Ctor: Class<Component>) {
    let options = Ctor.options
    // 如果能获取到super属性，说明Ctor是通过Vue.extend构建的子类，没有super则直接返回
    if (Ctor.super) {
        const superOptions = resolveConstructorOptions(Ctor.super) // 递归获取父类上的options
        const cachedSuperOptions = Ctor.superOptions
        if (superOptions !== cachedSuperOptions) {
            // super option changed,
            // need to resolve new options.
            Ctor.superOptions = superOptions
            // check if there are any late-modified/attached options (#4976)
            const modifiedOptions = resolveModifiedOptions(Ctor) // Ctor.sealedOptions 和 Ctor.options 作比较 把更改过得options取出来
            // update base extend options
            if (modifiedOptions) {
                extend(Ctor.extendOptions, modifiedOptions)
            }
            options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
            if (options.name) {
                options.components[options.name] = Ctor
            }
        }
    }
    return options
}
```
[resolveConstructorOptions上](https://segmentfault.com/a/1190000014587126)

[resolveConstructorOptions下](https://segmentfault.com/a/1190000014606817)
