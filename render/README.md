# render

```
    vnode = render.call(vm._renderProxy, vm.$createElement)
```

> 把Vue实例渲染成VNode形式,定义在src/core/instance/render.js中

```
    initProxy = function initProxy (vm) {
        if (hasProxy) {
            // determine which proxy handler to use
            const options = vm.$options
            const handlers = options.render && options.render._withStripped
                ? getHandler
                : hasHandler
            vm._renderProxy = new Proxy(vm, handlers)
        } else {
            vm._renderProxy = vm
        }
    }
```
> 在开发环境会去调用initProxy方法,否则就是vm本身,该方法会在new Vue()时调用_init方法执行,定义在src/core/instance/proxy.js中

```
export function initRender (vm: Component) {
    ...省略
    vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)

    vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
    ...省略
}
```
> vm.$createElement 方法定义是在执行 initRender 方法的时候, 调用vm.$createElement就是调用createElement方法,该方法定义在src/core/vdom/create-element.js中