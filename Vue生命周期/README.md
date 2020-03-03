# 生命周期

> 先来回顾下Vue官网的生命周期图示

![Vue生命周期](./lifecycle.png)

> 我们在上一篇文章 [合并配置mergeOptions](https://github.com/leefinder/vue-analysis-sbs/tree/master/%E5%90%88%E5%B9%B6%E9%85%8D%E7%BD%AEmergeOptions) 也讲到了生命周期初始配置实在mergeOptions中完成的,每个生命周期钩子函数都会被存到vm.$options上,以数组形式存在,最后调用callHook,遍历执行hook

```
export function callHook(vm: Component, hook: string) {
    // #7573 disable dep collection when invoking lifecycle hooks
    pushTarget()
    const handlers = vm.$options[hook]
    const info = `${hook} hook`
    if (handlers) {
        for (let i = 0, j = handlers.length; i < j; i++) {
            invokeWithErrorHandling(handlers[i], vm, null, vm, info)
        }
    }
    if (vm._hasHookEvent) {
        vm.$emit('hook:' + hook)
    }
    popTarget()
}
```

## beforeCreate 和 created

> 我们在讲[new Vue 发生了什么](https://github.com/leefinder/vue-analysis-sbs/tree/master/newVue%E5%8F%91%E7%94%9F%E4%BA%86%E4%BB%80%E4%B9%88) 时提到过,beforeCreate会在initState之前调用,因此在这时我们获取不到实例注册的props,data,methods等,created会在initState之后执行,从而我们可以在created钩子中做一些后端接口获取的操作,但是这两个步骤调用时DOM还没有生成,因此我们不能去执行操作DOM的事件,之后我们会将vue-router和vuex,它们会在beforeCreate中混入

```
    Vue.prototype._init = function (options?: Object) {
        ...
        initLifecycle(vm)
        initEvents(vm)
        initRender(vm)
        callHook(vm, 'beforeCreate')
        initInjections(vm) // resolve injections before data/props
        initState(vm)
        initProvide(vm) // resolve provide after data/props
        callHook(vm, 'created')
        ...
    }
```

## beforeMount 和 mounted

> 通过new Vue构造函数实现的,判断vm.$vnode == null,这里会执行mounted钩子,代码定义在src\core\instance\lifecycle.js

```
export function mountComponent(
    vm: Component,
    el: ?Element,
    hydrating?: boolean
): Component {
    ...
    if (vm.$vnode == null) {
        vm._isMounted = true
        callHook(vm, 'mounted')
    }
    ...
}
```

> 通过createComponent创建的组件实例mounted是在__patch__中

> 我们在 [Vue update实现](https://github.com/leefinder/vue-analysis-sbs/tree/master/update) 中提到_update即调用了__patch__把vnode转成真是的DOM节点,会在__patch__中调用invokeInsertHook去执行mounted

```
export function mountComponent(
    vm: Component,
    el: ?Element,
    hydrating?: boolean
): Component {
    ...
    updateComponent = () => {
        vm._update(vm._render(), hydrating)
    }
    ...
}
```

> invokeInsertHook会去遍历执行定义的钩子函数

```
invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)

function invokeInsertHook (vnode, queue, initial) {
    // delay insert hooks for component root nodes, invoke them after the
    // element is really inserted
    if (isTrue(initial) && isDef(vnode.parent)) {
        vnode.parent.data.pendingInsert = queue
    } else {
        for (let i = 0; i < queue.length; ++i) {
            queue[i].data.hook.insert(queue[i])
        }
    }
}
```

> insert方法定义在src\core\vdom\create-component.js中,我们在 [createComponent](https://github.com/leefinder/vue-analysis-sbs/tree/master/createComponent) 中提到installComponentHooks会给组件安装全局钩子

```
    insert(vnode: MountedComponentVNode) {
        const { context, componentInstance } = vnode
        if (!componentInstance._isMounted) {
            componentInstance._isMounted = true
            callHook(componentInstance, 'mounted')
        }
        if (vnode.data.keepAlive) {
            if (context._isMounted) {
                // vue-router#1212
                // During updates, a kept-alive component's child components may
                // change, so directly walking the tree here may call activated hooks
                // on incorrect children. Instead we push them into a queue which will
                // be processed after the whole patch process ended.
                queueActivatedComponent(componentInstance)
            } else {
                activateChildComponent(componentInstance, true /* direct */)
            }
        }
    }
```

## beforeUpdate 和 updated

> beforeUpdate会在mountComponent中初始化渲染Watcher时添加before回调函数,在组件mounted后,destoryed之前,组件触发更新时调用

```
    new Watcher(vm, updateComponent, noop, {
        before() {
            if (vm._isMounted && !vm._isDestroyed) {
                callHook(vm, 'beforeUpdate')
            }
        }
    }, true /* isRenderWatcher */)
```

> update钩子的调用是在flushSchedulerQueue中,方法定义在src\core\observer\scheduler.js,这个我们在响应式原理中再讲

```
function flushSchedulerQueue() {
    currentFlushTimestamp = getNow()
    flushing = true
    let watcher, id

    queue.sort((a, b) => a.id - b.id)

    for (index = 0; index < queue.length; index++) {
        watcher = queue[index]
        if (watcher.before) {
            watcher.before()
        }
        id = watcher.id
        has[id] = null
        watcher.run()
        if (process.env.NODE_ENV !== 'production' && has[id] != null) {
            circular[id] = (circular[id] || 0) + 1
            if (circular[id] > MAX_UPDATE_COUNT) {
                warn(
                    'You may have an infinite update loop ' + (
                        watcher.user
                            ? `in watcher with expression "${watcher.expression}"`
                            : `in a component render function.`
                    ),
                    watcher.vm
                )
                break
            }
        }
    }
    const activatedQueue = activatedChildren.slice()
    const updatedQueue = queue.slice()

    resetSchedulerState()

    callActivatedHooks(activatedQueue)
    callUpdatedHooks(updatedQueue)

    if (devtools && config.devtools) {
        devtools.emit('flush')
    }
}

function callUpdatedHooks(queue) {
    let i = queue.length
    while (i--) {
        const watcher = queue[i]
        const vm = watcher.vm
        if (vm._watcher === watcher && vm._isMounted && !vm._isDestroyed) {
            callHook(vm, 'updated')
        }
    }
}

```

## beforeDestory 和 destoryed

> beforeDestory和destoryed会在组件销毁调用$destory时触发

```
    Vue.prototype.$destroy = function () {
        const vm: Component = this
        if (vm._isBeingDestroyed) {
            return
        }
        callHook(vm, 'beforeDestroy')
        vm._isBeingDestroyed = true
        // remove self from parent
        const parent = vm.$parent
        if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
            remove(parent.$children, vm)
        }
        // teardown watchers
        if (vm._watcher) {
            vm._watcher.teardown()
        }
        let i = vm._watchers.length
        while (i--) {
            vm._watchers[i].teardown()
        }
        // remove reference from data ob
        // frozen object may not have observer.
        if (vm._data.__ob__) {
            vm._data.__ob__.vmCount--
        }
        // call the last hook...
        vm._isDestroyed = true
        // invoke destroy hooks on current rendered tree
        vm.__patch__(vm._vnode, null)
        // fire destroyed hook
        callHook(vm, 'destroyed')
        // turn off all instance listeners.
        vm.$off()
        // remove __vue__ reference
        if (vm.$el) {
            vm.$el.__vue__ = null
        }
        // release circular reference (#6759)
        if (vm.$vnode) {
            vm.$vnode.parent = null
        }
    }
```

## activated & deactivated

> 这两个钩子是keep-alive组件配套,我会在 [keep-alive](https://github.com/leefinder/vue-analysis-sbs/tree/master/keep-alive) 章节中详细讲解