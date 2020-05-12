# createElement

> createElement是在执行initRender时注册到当前实例的, 代码在src/core/instance/render.js中

```
    export function initRender (vm: Component) {
        ...省略
        
        vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
        
        vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

        ...省略
    }
```

# createElement方法实际上是对_createElement方法的封装

```
    export function createElement (
        context: Component,
        tag: any,
        data: any,
        children: any,
        normalizationType: any,
        alwaysNormalize: boolean
        ): VNode | Array<VNode> {
            // 参数不一致时， 往前移一位 
        if (Array.isArray(data) || isPrimitive(data)) {
            normalizationType = children
            children = data
            data = undefined
        }
        if (isTrue(alwaysNormalize)) {
            normalizationType = ALWAYS_NORMALIZE
        }
        return _createElement(context, tag, data, children, normalizationType)
    }
```

> 在createElement中,首先检测data的类型,通过判断data是不是数组,以及是不是基本类型,来判断data是否传入.如果没有传入,则将所有的参数向前赋值,且data = undifined. 然后,判断判断传入的alwaysNormalize参数是否为真,为真的话,赋值给ALWAYS_NORMALIZE常量.最后,再调用_createElement函数,可以看到,createElement是对参数做了一些处理以后,将其传给_createElement函数

> _createElement,第一步判断data不能是响应式的,vnode中的data如果是响应式的就抛出警告,返回一个空的vnode

```
    export function _createElement (
        context: Component,
        tag?: string | Class<Component> | Function | Object,
        data?: VNodeData,
        children?: any,
        normalizationType?: number
    ): VNode | Array<VNode> {
        if (isDef(data) && isDef((data: any).__ob__)) {
            process.env.NODE_ENV !== 'production' && warn(
            `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
            'Always create fresh vnode data objects in each render!',
            context
            )
            return createEmptyVNode()
        }
        ...
    }
```
## normalize-children

> 对传入的children进行拍平, 代码在src/core/vdom/helpers/normalize-children.js中

```
// support single function children as default scoped slot
    if (Array.isArray(children) &&
        typeof children[0] === 'function'
    ) {
        data = data || {}
        data.scopedSlots = { default: children[0] }
        children.length = 0
    }
    if (normalizationType === ALWAYS_NORMALIZE) {
        children = normalizeChildren(children)
    } else if (normalizationType === SIMPLE_NORMALIZE) {
        children = simpleNormalizeChildren(children)
    }
```

### simpleNormalizeChildren

> 主要完成的功能是将children类数组的第一层转换为一个一维数组,当children中包含组件时,函数式组件可能会返回一个数组而不是单独的根节点.因此,当child是数组时,我们把整个数组利用Array.prototype.concat进行一次抹平.这里只进行1层抹平,函数式组件已经规范化了自己的子组件

```
// 1. When the children contains components - because a functional component
// may return an Array instead of a single root. In this case, just a simple
// normalization is needed - if any child is an Array, we flatten the whole
// thing with Array.prototype.concat. It is guaranteed to be only 1-level deep
// because functional components already normalize their own children.
    export function simpleNormalizeChildren (children: any) {
        for (let i = 0; i < children.length; i++) {
            if (Array.isArray(children[i])) {
            return Array.prototype.concat.apply([], children)
            }
        }
        return children
    }
```

### normalizeChildren

> 判断children是否基础类型，如果是返回文本节点；如果是数组类型，执行normalizeArrayChildren，遍历传入的children，继续判断是否数组，如果是递归执行normalizeArrayChildren

```
// 2. When the children contains constructs that always generated nested Arrays,
// e.g. <template>, <slot>, v-for, or when the children is provided by user
// with hand-written render functions / JSX. In such cases a full normalization
// is needed to cater to all possible types of children values.

    export function normalizeChildren (children: any): ?Array<VNode> {
        // 判断基础类型 string number boolean symbol
        return isPrimitive(children) 
        // 创建文本节点
            ? [createTextVNode(children)] 
            : Array.isArray(children)
        // 遍历children，判断children[i]，如果是数组类型递归执行normalizeArrayChildren，判断基础类型创建文本节点，返回res数组
            ? normalizeArrayChildren(children) 
            : undefined
    }
```

```
    // 判断传入的tag是string类型 创建一个普通的html节点
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
            // 创建普通的html节点
            vnode = new VNode(
                config.parsePlatformTagName(tag), data, children,
                undefined, undefined, context
            )
        } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
            // component 如果是组件节点，调用createComponent
            vnode = createComponent(Ctor, data, context, children, tag)
        } else {
            // unknown or unlisted namespaced elements
            // check at runtime because it may get assigned a namespace when its
            // parent normalizes children
            // 其他的自定义节点
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


