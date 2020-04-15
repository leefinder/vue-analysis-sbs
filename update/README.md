# update

> _update方法会在首次渲染和数据更新时触发,目的是把VNode渲染成真实的DOM节点,
该方法是通过lifecycleMixin方法添加到Vue原型上的,
代码定义在src/core/instance/lifecycle.js中

> lifecycleMixin给Vue原型添加了3个方法

- _update
- $forceUpdate
- $destroy

```
    export function lifecycleMixin (Vue: Class<Component>) {
        Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
            const vm: Component = this
            const prevEl = vm.$el
            const prevVnode = vm._vnode
            const restoreActiveInstance = setActiveInstance(vm)
            vm._vnode = vnode
            // Vue.prototype.__patch__ is injected in entry points
            // based on the rendering backend used.
            if (!prevVnode) {
                // initial render
                vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
            } else {
                // updates
                vm.$el = vm.__patch__(prevVnode, vnode)
            }
            restoreActiveInstance()
            // update __vue__ reference
            if (prevEl) {
                prevEl.__vue__ = null
            }
            if (vm.$el) {
                vm.$el.__vue__ = vm
            }
            // if parent is an HOC, update its $el as well
            if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
                vm.$parent.$el = vm.$el
            }
            // updated hook is called by the scheduler to ensure that children are
            // updated in a parent's updated hook.
        }

        Vue.prototype.$forceUpdate = function () {
            const vm: Component = this
            if (vm._watcher) {
                vm._watcher.update()
            }
        }

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
    }

```

## _update方法实现

> _update 的核心就是调用 vm.__patch__ 方法,定义在src/platforms/web/runtime/index.js中

```
    // install platform patch function
    Vue.prototype.__patch__ = inBrowser ? patch : noop
```

> 在浏览器环境中__patch__实现是通过patch方法,定义在src/platforms/web/runtime/patch.js中

### createPatchFunction

> patch方法实现是通过调用createPatchFunction,该方法支持传入一个对象,包含nodeOps和modules两个参数

- nodeOps封装了一系列DOM操作的方法,定义在src/platforms/web/runtime/node-ops.js中
- modules定义了一些模块的钩子函数的实现
    1. baseModules,vdom相关,定义在src/core/vdom/modules/index.js
    2. platformModules,web平台相关,定义在src/platforms/web/runtime/modules/index.js

> Vue首次渲染时,传入的是我们定义的vm.$el根节点,看一下执行流程

```
// src/core/instance/lifecycle.js
    if (!prevVnode) {
      // initial render
        vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
        vm.$el = vm.__patch__(prevVnode, vnode)
    }
```

- 第一次创建会去判断是否是真实的节点,显然isRealElement = true
- 接下来又通过emptyNodeAt方法把oldVnode转换成VNode对象,然后再调用createElm方法
- 先尝试创建子组件,然后通过调用createChildren创建子元素,createChildren其实就是
- 通过invokeCreateHooks去执行create钩子,并用insertedVnodeQueue缓存vnode
- 最后调用insert方法,把 DOM 插入到父节点中,因为是递归调用,子元素会优先调用insert

### createElm

```
    function createElm (
        vnode,
        insertedVnodeQueue,
        parentElm,
        refElm,
        nested,
        ownerArray,
        index
    ) {
        if (isDef(vnode.elm) && isDef(ownerArray)) {
        // This vnode was used in a previous render!
        // now it's used as a new node, overwriting its elm would cause
        // potential patch errors down the road when it's used as an insertion
        // reference node. Instead, we clone the node on-demand before creating
        // associated DOM element for it.
        vnode = ownerArray[index] = cloneVNode(vnode)
        }

        vnode.isRootInsert = !nested // for transition enter check
        if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
            return
        }

        const data = vnode.data
        const children = vnode.children
        const tag = vnode.tag
        if (isDef(tag)) {
            if (process.env.NODE_ENV !== 'production') {
                if (data && data.pre) {
                    creatingElmInVPre++
                }
                if (isUnknownElement(vnode, creatingElmInVPre)) {
                    warn(
                        'Unknown custom element: <' + tag + '> - did you ' +
                        'register the component correctly? For recursive components, ' +
                        'make sure to provide the "name" option.',
                        vnode.context
                    )
                }
            }

            vnode.elm = vnode.ns
                ? nodeOps.createElementNS(vnode.ns, tag)
                : nodeOps.createElement(tag, vnode)
            setScope(vnode)

            /* istanbul ignore if */
            if (__WEEX__) {
                // in Weex, the default insertion order is parent-first.
                // List items can be optimized to use children-first insertion
                // with append="tree".
                const appendAsTree = isDef(data) && isTrue(data.appendAsTree)
                if (!appendAsTree) {
                    if (isDef(data)) {
                        invokeCreateHooks(vnode, insertedVnodeQueue)
                    }
                    insert(parentElm, vnode.elm, refElm)
                }
                createChildren(vnode, children, insertedVnodeQueue)
                if (appendAsTree) {
                    if (isDef(data)) {
                        invokeCreateHooks(vnode, insertedVnodeQueue)
                    }
                    insert(parentElm, vnode.elm, refElm)
                }
            } else {
                createChildren(vnode, children, insertedVnodeQueue)
                if (isDef(data)) {
                    invokeCreateHooks(vnode, insertedVnodeQueue)
                }
                insert(parentElm, vnode.elm, refElm)
            }

            if (process.env.NODE_ENV !== 'production' && data && data.pre) {
                creatingElmInVPre--
            }
        } else if (isTrue(vnode.isComment)) {
            vnode.elm = nodeOps.createComment(vnode.text)
            insert(parentElm, vnode.elm, refElm)
        } else {
            vnode.elm = nodeOps.createTextNode(vnode.text)
            insert(parentElm, vnode.elm, refElm)
        }
    }
```

### createChildren

> createChildren就是循环遍历虚拟节点的子节点,递归调用createElm方法

```
    function createChildren (vnode, children, insertedVnodeQueue) {
        if (Array.isArray(children)) {
            if (process.env.NODE_ENV !== 'production') {
                checkDuplicateKeys(children)
            }
            for (let i = 0; i < children.length; ++i) {
                createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
            }
        } else if (isPrimitive(vnode.text)) {
            nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
        }
    }
```

> invokeCreateHooks循环执行所有的create钩子,并把vnode push到insertedVnodeQueue数组中

```
    function invokeCreateHooks (vnode, insertedVnodeQueue) {
        for (let i = 0; i < cbs.create.length; ++i) {
            cbs.create[i](emptyNode, vnode)
        }
        i = vnode.data.hook // Reuse variable
        if (isDef(i)) {
            if (isDef(i.create)) i.create(emptyNode, vnode)
            if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
        }
    }
```

> 把生成的子节点插入到父容器中

```
    function insert (parent, elm, ref) {
        if (isDef(parent)) {
        if (isDef(ref)) {
            if (nodeOps.parentNode(ref) === parent) {
                nodeOps.insertBefore(parent, elm, ref)
            }
        } else {
            nodeOps.appendChild(parent, elm)
        }
        }
    }
```
