# keep-alive组件实现

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

## keep-alive render

> keep-alive自定义实现了render,keep-alive默认支持插槽,keep-alive会去获取第一个子节点

```
    render () {
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
            if (cache[key]) {
                vnode.componentInstance = cache[key].componentInstance
                // make current key freshest
                remove(keys, key)
                keys.push(key)
            } else {
                cache[key] = vnode
                keys.push(key)
                // prune oldest entry
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