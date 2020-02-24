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

> 主要完成的功能是将children类数组的第一层转换为一个一维数组,当children中包含组件是,函数式组件可能会返回一个数组而不是单独的根节点.因此,当child是数组时,我们
把整个数组利用Array.prototype.concat进行一次抹平.这里只进行1层抹平,函数式组件已经规范化了自己的子组件

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

> 当子级包含始嵌套数组,template,slot,v-for,手写的render函数,判断children中的元素是不是数组,如果是的话,就递归调用数组,并将每个元素保存在数组中返回

```
// 2. When the children contains constructs that always generated nested Arrays,
// e.g. <template>, <slot>, v-for, or when the children is provided by user
// with hand-written render functions / JSX. In such cases a full normalization
// is needed to cater to all possible types of children values.
    export function normalizeChildren (children: any): ?Array<VNode> {
        return isPrimitive(children)
            ? [createTextVNode(children)]
            : Array.isArray(children)
            ? normalizeArrayChildren(children)
            : undefined
    }
```

## createComponent

> 如果标签(tag)不是字符串,则创建组件VNode,代码在src/core/vdom/create-component.js中

```
    vnode = createComponent(Ctor, data, context, children, tag)
```

