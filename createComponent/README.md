# createComponent

> 我们在讲render函数的时候提到,_render会通过执行vm.$createElement去实现返回vnode,而vm.$createElement我们在讲createElement中也提到过,没有看过的同学可以点击链接看下

[render函数实现](https://github.com/leefinder/vue-analysis-sbs/tree/master/render))
[createElement](https://github.com/leefinder/vue-analysis-sbs/tree/master/createElement)

## _createElement回顾
> 下面我们来讲接createComponent,先回顾下_createElement的实现,如果是常规html节点则创建一个vnode,否则创建一个组件vnode
```
// src/core/vdom/create-element.js
    if (typeof tag === 'string') {
        let Ctor
        ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
        if (config.isReservedTag(tag)) {
            // platform built-in elements
            if (process.env.NODE_ENV !== 'production' && isDef(data) && isDef(data.nativeOn)) {
                warn(
                `The .native modifier for v-on is only valid on components but it was used on <${tag}>.`,
                context
                )
            }
            vnode = new VNode(
                config.parsePlatformTagName(tag), data, children,
                undefined, undefined, context
            )
            } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
            // component
            vnode = createComponent(Ctor, data, context, children, tag)
        } else {
            // unknown or unlisted namespaced elements
            // check at runtime because it may get assigned a namespace when its
            // parent normalizes children
            vnode = new VNode(
                tag, data, children,
                undefined, undefined, context
            )
        }
    } else {
        // direct component options / constructor
        vnode = createComponent(tag, data, context, children)
    }
```

## 正式进入createComponent

> 主要就是通过Vue.extend实例化子组件,分一下几个步骤,代码在src/core/vdom/create-component.js中

- Vue.extend实例化子组件
- resolveAsyncComponent处理异步组件
- resolveConstructorOptions合并配置
- transformModel处理v-model
- extractPropsFromVNodeData对props处理
- createFunctionalComponent处理函数式组件
- 对events做处理
- installComponentHooks安装全局钩子
- 返回子组件vnode

```
    export function createComponent(
        Ctor: Class<Component> | Function | Object | void,
        data: ?VNodeData,
        context: Component,
        children: ?Array<VNode>,
        tag?: string
    ): VNode | Array<VNode> | void {
        if (isUndef(Ctor)) {
            return
        }

        const baseCtor = context.$options._base

        // plain options object: turn it into a constructor
        if (isObject(Ctor)) {
            Ctor = baseCtor.extend(Ctor)
        }

        // if at this stage it's not a constructor or an async component factory,
        // reject.
        if (typeof Ctor !== 'function') {
            if (process.env.NODE_ENV !== 'production') {
                warn(`Invalid Component definition: ${String(Ctor)}`, context)
            }
            return
        }

        // async component
        let asyncFactory
        if (isUndef(Ctor.cid)) {
            asyncFactory = Ctor
            Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
            if (Ctor === undefined) {
                // return a placeholder node for async component, which is rendered
                // as a comment node but preserves all the raw information for the node.
                // the information will be used for async server-rendering and hydration.
                return createAsyncPlaceholder(
                    asyncFactory,
                    data,
                    context,
                    children,
                    tag
                )
            }
        }

        data = data || {}

        // resolve constructor options in case global mixins are applied after
        // component constructor creation
        resolveConstructorOptions(Ctor)

        // transform component v-model data into props & events
        if (isDef(data.model)) {
            transformModel(Ctor.options, data)
        }

        // extract props
        const propsData = extractPropsFromVNodeData(data, Ctor, tag)

        // functional component
        if (isTrue(Ctor.options.functional)) {
            return createFunctionalComponent(Ctor, propsData, data, context, children)
        }

        // extract listeners, since these needs to be treated as
        // child component listeners instead of DOM listeners
        const listeners = data.on
        // replace with listeners with .native modifier
        // so it gets processed during parent component patch.
        data.on = data.nativeOn

        if (isTrue(Ctor.options.abstract)) {
            // abstract components do not keep anything
            // other than props & listeners & slot

            // work around flow
            const slot = data.slot
            data = {}
            if (slot) {
                data.slot = slot
            }
        }

        // install component management hooks onto the placeholder node
        installComponentHooks(data)

        // return a placeholder vnode
        const name = Ctor.options.name || tag
        const vnode = new VNode(
            `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
            data, undefined, undefined, undefined, context,
            { Ctor, propsData, listeners, tag, children },
            asyncFactory
        )

        // Weex specific: invoke recycle-list optimized @render function for
        // extracting cell-slot template.
        // https://github.com/Hanks10100/weex-native-directive/tree/master/component
        /* istanbul ignore if */
        if (__WEEX__ && isRecyclableComponent(vnode)) {
            return renderRecyclableComponentTemplate(vnode)
        }

        return vnode
    }
```

### Vue.extend

> Vue.extend实际上就是构造一个Vue的子类,通过原型继承的方式把一个纯对象转换一个继承于Vue的构造器Sub并返回,然后对Sub这个对象本身扩展了一些属性,如扩展options,添加全局API等;并且对配置中的props和computed做了初始化工作;最后对于这个Sub构造函数做了缓存,避免多次执行 Vue.extend的时候对同一个子组件重复构造

> 代码定义在src/core/global-api/extend.js

```
    Vue.extend = function (extendOptions: Object): Function {
        extendOptions = extendOptions || {}
        const Super = this
        const SuperId = Super.cid
        const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
        if (cachedCtors[SuperId]) {
            return cachedCtors[SuperId]
        }

        const name = extendOptions.name || Super.options.name
        if (process.env.NODE_ENV !== 'production' && name) {
            validateComponentName(name)
        }

        const Sub = function VueComponent(options) {
            this._init(options)
        }
        Sub.prototype = Object.create(Super.prototype)
        Sub.prototype.constructor = Sub
        Sub.cid = cid++
        Sub.options = mergeOptions(
            Super.options,
            extendOptions
        )
        Sub['super'] = Super

        // For props and computed properties, we define the proxy getters on
        // the Vue instances at extension time, on the extended prototype. This
        // avoids Object.defineProperty calls for each instance created.
        if (Sub.options.props) {
            initProps(Sub)
        }
        if (Sub.options.computed) {
            initComputed(Sub)
        }

        // allow further extension/mixin/plugin usage
        Sub.extend = Super.extend
        Sub.mixin = Super.mixin
        Sub.use = Super.use

        // create asset registers, so extended classes
        // can have their private assets too.
        ASSET_TYPES.forEach(function (type) {
            Sub[type] = Super[type]
        })
        // enable recursive self-lookup
        if (name) {
            Sub.options.components[name] = Sub
        }

        // keep a reference to the super options at extension time.
        // later at instantiation we can check if Super's options have
        // been updated.
        Sub.superOptions = Super.options
        Sub.extendOptions = extendOptions
        Sub.sealedOptions = extend({}, Sub.options)

        // cache constructor
        cachedCtors[SuperId] = Sub
        return Sub
    }
```

### installComponentHooks

> installComponentHooks安装全局钩子,就是遍历componentVNodeHooks对象把钩子函数合并到data.hook上,在合并过程中,如果某个时机的钩子已经存在 data.hook 中,那么通过执行mergeHook函数做合并,依次执行这两个钩子函数

```
const hooksToMerge = Object.keys(componentVNodeHooks)

function installComponentHooks(data: VNodeData) {
    const hooks = data.hook || (data.hook = {})
    for (let i = 0; i < hooksToMerge.length; i++) {
        const key = hooksToMerge[i]
        const existing = hooks[key]
        const toMerge = componentVNodeHooks[key]
        if (existing !== toMerge && !(existing && existing._merged)) {
            hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge
        }
    }
}

const componentVNodeHooks = {
    init(vnode: VNodeWithData, hydrating: boolean): ?boolean {
        if (
            vnode.componentInstance &&
            !vnode.componentInstance._isDestroyed &&
            vnode.data.keepAlive
        ) {
            // kept-alive components, treat as a patch
            const mountedNode: any = vnode // work around flow
            componentVNodeHooks.prepatch(mountedNode, mountedNode)
        } else {
            const child = vnode.componentInstance = createComponentInstanceForVnode(
                vnode,
                activeInstance
            )
            child.$mount(hydrating ? vnode.elm : undefined, hydrating)
        }
    },

    prepatch(oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
        const options = vnode.componentOptions
        const child = vnode.componentInstance = oldVnode.componentInstance
        updateChildComponent(
            child,
            options.propsData, // updated props
            options.listeners, // updated listeners
            vnode, // new parent vnode
            options.children // new children
        )
    },

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
    },

    destroy(vnode: MountedComponentVNode) {
        const { componentInstance } = vnode
        if (!componentInstance._isDestroyed) {
            if (!vnode.data.keepAlive) {
                componentInstance.$destroy()
            } else {
                deactivateChildComponent(componentInstance, true /* direct */)
            }
        }
    }
}
```