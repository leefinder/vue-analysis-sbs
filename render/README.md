# render

## renderMixin

> 给Vue原型添加$nextTick和_render两个方法

### $nextTick

> 代码位于/src/core/util/next-tick.js

> JS 执行是单线程的，它是基于事件循环的

1. 所有同步任务都在主线程上执行,组成一个执行栈

2. 主线程之外还有一个任务队列,当异步任务运行得到结果,就在任务队列中放置一个事件

3. 当执行栈所有同步任务执行完毕,js去读取任务队列,那些已经执行完的异步任务进入执行栈,进行执行

4. 主线程循环上面几个步骤

```
    const callbacks = []
    let pending = false
    export function nextTick (cb?: Function, ctx?: Object) {
        let _resolve
        callbacks.push(() => {
            if (cb) {
            try {
                cb.call(ctx)
            } catch (e) {
                handleError(e, ctx, 'nextTick')
            }
            } else if (_resolve) {
            _resolve(ctx)
            }
        })
        if (!pending) {
            pending = true
            timerFunc()
        }
        ...
    }
```

> nextTick源码可以看出,我们在nextTick中传入的回调函数会被存在callbacks数组中,最后一次性在timerFunc中执行

> 这里使用 callbacks而不是直接在nextTick中执行回调函数的原因是保证在同一个tick内多次执行nextTick,不会开启多个异步任务,而把这些异步任务都压成一个同步任务,在下一个tick执行完毕,这里通过pending锁定。

```
// $flow-disable-line
    if (!cb && typeof Promise !== 'undefined') {
        return new Promise(resolve => {
        _resolve = resolve
        })
    }
```
> 当nextTick不传入回调的时候,我们可以通过this.$nextTick().then去调用,即在函数中把Promise赋值给了_resolve

```
    let timerFunc

    if (typeof Promise !== 'undefined' && isNative(Promise)) {
        const p = Promise.resolve()
        timerFunc = () => {
            p.then(flushCallbacks)
            if (isIOS) setTimeout(noop)
        }
        isUsingMicroTask = true
    } else if (!isIE && typeof MutationObserver !== 'undefined' && (
        isNative(MutationObserver) ||
        // PhantomJS and iOS 7.x
        MutationObserver.toString() === '[object MutationObserverConstructor]'
        )) {
        let counter = 1
        const observer = new MutationObserver(flushCallbacks)
        const textNode = document.createTextNode(String(counter))
        observer.observe(textNode, {
            characterData: true
        })
        timerFunc = () => {
            counter = (counter + 1) % 2
            textNode.data = String(counter)
        }
        isUsingMicroTask = true
    } else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
        timerFunc = () => {
            setImmediate(flushCallbacks)
        }
    } else {
        // Fallback to setTimeout.
        timerFunc = () => {
            setTimeout(flushCallbacks, 0)
        }
    }
```

> Vue会先去判断是否原生支持Promise,如果支持就通过Promise.resolve().then(flushCallbacks)

> 如果不支持Promise,就降级为MutationObserver

> 如果Promise和MutationObserver都不支持,就执行setImmediate

> 以上三个都不支持就用setTimeout代替

```
    function flushCallbacks () {
        pending = false
        const copies = callbacks.slice(0)
        callbacks.length = 0
        for (let i = 0; i < copies.length; i++) {
            copies[i]()
        }
    }
```
> flushCallbacks很简单,就是把pending重置false,拷贝callbacks,对callbacks清0,统一执行传入的回调队列

### _render

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