# keep-alive

> keep-alive是Vue源码实现的一个抽象组件,在源码中我们可以看到keep-alive组件有一个abstract属性true,这个我们这new Vue发生了什么中有提到,在寻找父组件是抽象组件会被跳过

```
// src/core/instance/lifecycle.js
// locate first non-abstract parent
    let parent = options.parent
    if (parent && !options.abstract) {
        while (parent.$options.abstract && parent.$parent) {
        parent = parent.$parent
        }
        parent.$children.push(vm)
    }
```

## keep-alive组件支持3个props

- include 匹配的组件会被缓存
- exclude 不匹配的组件不会被缓存
- max 最大缓存的组件个数

> 在组件create的时候,创建了cache和keys用作缓存处理

```
    export default {
        name: 'keep-alive',
        abstract: true,

        props: {
            include: patternTypes,
            exclude: patternTypes,
            max: [String, Number]
        }
        ...
    }
```

## keep-alive组件内部实现

> keep-alive自定义实现了render,keep-alive默认支持插槽,keep-alive会去获取第一个子节点

```
    render () {
        // 获取默认的插槽
        const slot = this.$slots.default
        const vnode: VNode = getFirstComponentChild(slot)
        const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
        if (componentOptions) {
            // check pattern
            const name: ?string = getComponentName(componentOptions)
            const { include, exclude } = this
            if (
                // not included
                (include && (!name || !matches(include, name))) ||
                // excluded
                (exclude && name && matches(exclude, name))
            ) {
                return vnode
            }

            const { cache, keys } = this
            const key: ?string = vnode.key == null
                // same constructor may get registered as different local components
                // so cid alone is not enough (#3269)
                ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
                : vnode.key
            // 下次进入时命中缓存
            if (cache[key]) {
                // 直接从缓存中获取组件实例
                vnode.componentInstance = cache[key].componentInstance
                // make current key freshest
                // LRU缓存算法，移除当前缓存key
                remove(keys, key)
                // 把刚刚获取的key存到队列末端
                keys.push(key)
            } else {
                // 添加到缓存里
                cache[key] = vnode
                // 队列记录key
                keys.push(key)
                // prune oldest entry
                // 当超过最大缓存数时，取第一个keys移除，队列头部的key，默认以这个key作为过时的或者说访问最不频繁的key
                if (this.max && keys.length > parseInt(this.max)) {
                pruneCacheEntry(cache, keys[0], keys, this._vnode)
                }
            }
            vnode.data.keepAlive = true
        }
        return vnode || (slot && slot[0])
    } 
```
### matches

> matches就是去匹配include和exclude中定义的以数组、正则、字符串形式的组件名,组件名如果满足了配置include且不匹配或者是配置了exclude且匹配,那么就直接返回这个组件的vnode,否则的话走下一步缓存

```
    function matches (pattern: string | RegExp | Array<string>, name: string): boolean {
        if (Array.isArray(pattern)) {
            return pattern.indexOf(name) > -1
        } else if (typeof pattern === 'string') {
            return pattern.split(',').indexOf(name) > -1
        } else if (isRegExp(pattern)) {
            return pattern.test(name)
        }
        /* istanbul ignore next */
        return false
    }
```

### pruneCache

> 在keep-alive组件mounted时执行了include和exclude的监听,通过遍历cache去去除没有匹配规则的缓存

```
    mounted () {
        this.$watch('include', val => {
            pruneCache(this, name => matches(val, name))
        })
        this.$watch('exclude', val => {
            pruneCache(this, name => !matches(val, name))
        })
    },
    function pruneCache (keepAliveInstance: any, filter: Function) {
        const { cache, keys, _vnode } = keepAliveInstance
        for (const key in cache) {
            const cachedNode: ?VNode = cache[key]
            if (cachedNode) {
                const name: ?string = getComponentName(cachedNode.componentOptions)
                if (name && !filter(name)) {
                    pruneCacheEntry(cache, key, keys, _vnode)
                }
            }
        }
    }
```

### pruneCacheEntry

> pruneCacheEntry做的事就是去清除缓存,同时调用$destory()销毁实例

```
    function pruneCacheEntry (
        cache: VNodeCache,
        key: string,
        keys: Array<string>,
        current?: VNode
    ) {
        const cached = cache[key]
        if (cached && (!current || cached.tag !== current.tag)) {
            cached.componentInstance.$destroy()
        }
        cache[key] = null
        remove(keys, key)
    }
```

## 创建keep-alive组件

> 组件渲染会调用__patch__，patch中创建组件createComponent

```
    function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
        let i = vnode.data
        if (isDef(i)) {
            // 第一次创建的时候vnode.componentInstance不存在，所以isReactivated为false
            // 第二次进来vnode.componentInstance存在，keep-alive组件的keepAlive为true，所以isReactivated为true
            const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
            // 执行组件的init钩子
            if (isDef(i = i.hook) && isDef(i = i.init)) {
                i(vnode, false /* hydrating */)
            }
          
            if (isDef(vnode.componentInstance)) {
                initComponent(vnode, insertedVnodeQueue)
                insert(parentElm, vnode.elm, refElm)
                // isReactivated进入keep-alive组件的逻辑
                if (isTrue(isReactivated)) {
                    reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
                }
                return true
            }
        }
    }
```

> 组件init钩子，第一次进来createComponentInstanceForVnode去创建组件实例存到componentInstance上，keep-alive组件第二次进入会命中第一个逻辑
走prepatch，prepatch是直接从缓存里取componentInstance去更新组件

```
const componentVNodeHooks = {
    init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
        // keep-alive组件逻辑，第二次进来获取缓存里获取vnode
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
    prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
        const options = vnode.componentOptions
        const child = vnode.componentInstance = oldVnode.componentInstance
        updateChildComponent(
            child,
            options.propsData, // updated props
            options.listeners, // updated listeners
            vnode, // new parent vnode
            options.children // new children
        )
    }
}
```

> 最后执行invokeInsertHook，循环执行insert方法

```
const componentVNodeHooks = {
    insert (vnode: MountedComponentVNode) {
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
}

```