# Vue 实例挂载的实现

> 在执行new Vue时,vue会去判断是否定义的了el参数,去执行$mount去挂载vm

## $mount

```
// 在src/core/instance/init.js中
    if (vm.$options.el) {
        vm.$mount(vm.$options.el)
    }
```

## 重写原型上的$mount方法

> 纯前端浏览器环境compiler版本的$mount

- 如果el指向body或者html根节点则报错
- 判断当前实例有没有定义render函数,如果没有就去取template,通过compileToFunctions方法转换出render方法
- 最后去调用原型上mount方法

```
// src/platform/web/entry-runtime-with-compiler.js
    // 缓存原型上的$mount方法
    const mount = Vue.prototype.$mount
    // 重写$mount 第一个参数表示挂载的dom元素，第二个是服务端渲染的配置
    Vue.prototype.$mount = function (
        el?: string | Element,
        hydrating?: boolean
    ): Component {
        // 获取el，通过document.querySelector获取根节点
        el = el && query(el)

        // 根节点不能是body或者html
        if (el === document.body || el === document.documentElement) {
            process.env.NODE_ENV !== 'production' && warn(
            `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
            )
            return this
        }
        // 把合并配置后的$options赋值给options
        const options = this.$options
        // resolve template/el and convert to render function
        // 判断组件是否定义了render函数，如果没有定义则去获取定义的模版template
        if (!options.render) {
            // 没有render方法时尝试获取template
            let template = options.template
            if (template) {
                if (typeof template === 'string') {
                    if (template.charAt(0) === '#') {
                    template = idToTemplate(template)
                    /* istanbul ignore if */
                    if (process.env.NODE_ENV !== 'production' && !template) {
                        warn(
                        `Template element not found or is empty: ${options.template}`,
                        this
                        )
                    }
                }
                } else if (template.nodeType) {
                    template = template.innerHTML
                } else {
                    if (process.env.NODE_ENV !== 'production') {
                    warn('invalid template option:' + template, this)
                    }
                    return this
                }
            // 没有template定义，尝试获取根节点的html内容
            } else if (el) {
                template = getOuterHTML(el)
            }
            // 获取到template后 通过compileToFunctions生成render函数
            if (template) {
                /* istanbul ignore if */
                if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
                    mark('compile')
                }
                // 获取到的模版字符串传入compile中执行parse optimize generate 返回render函数
                // compileToFunctions接收2个参数 模版字符串 用户定一个编译配置
                const { render, staticRenderFns } = compileToFunctions(template, {
                    outputSourceRange: process.env.NODE_ENV !== 'production',
                    shouldDecodeNewlines,
                    shouldDecodeNewlinesForHref,
                    delimiters: options.delimiters,
                    comments: options.comments
                }, this)
                options.render = render
                options.staticRenderFns = staticRenderFns

                /* istanbul ignore if */
                // vue开发环境的性能检测
                if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
                    mark('compile end')
                    measure(`vue ${this._name} compile`, 'compile', 'compile end')
                }
            }
        }
        // 执行原型$mount
        return mount.call(this, el, hydrating)
    }
```
## 原型$mount的实现

> 原型上的$mount方法定义在 src/platforms/web/runtime/index.js中,实际上是调用了mountComponent方法,
这个方法定义在src/core/instance/lifecycle.js中

> 传入2个参数,第一个是el,它表示挂载的元素,可以是字符串,也可以是DOM对象,如果是字符串在浏览器环境下会调用query方法转换成DOM对象的.第二个参数是和服务端渲染相关,在浏览器环境下我们不需要传第二个参数

```
// public mount method
    Vue.prototype.$mount = function (
        el?: string | Element,
        hydrating?: boolean
    ): Component {
        el = el && inBrowser ? query(el) : undefined
        return mountComponent(this, el, hydrating)
    }
```

## mountComponent方法

- 如果options上没有render方法,就创建空的VNode
- 执行beforeMount钩子
- 定义updateComponent方法,实际上是调用vm._update去触发更新
- 实例化一个渲染Watcher,在回调中执行updateComponent，在这个方法中调用vm._render生成虚拟dom，该方法在这里可以用来监听更新,在before钩子中添加了beforeUpdate钩子函数,在组件生成后,销毁前会去执行beforeUpdate钩子
- vm._update()把VNode patch到真实DOM后,执行mounted钩子.注意,这里对mounted钩子函数执行有一个判断逻辑,vm.$vnode如果为null,则表明这不是一次组件的初始化过程,而是我们通过外部 new Vue 初始化过程

```
export function mountComponent(
    vm: Component,
    el: ?Element,
    hydrating?: boolean
): Component {
    vm.$el = el
    if (!vm.$options.render) {
        vm.$options.render = createEmptyVNode
        if (process.env.NODE_ENV !== 'production') {
            /* istanbul ignore if */
            if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
                vm.$options.el || el) {
                warn(
                    'You are using the runtime-only build of Vue where the template ' +
                    'compiler is not available. Either pre-compile the templates into ' +
                    'render functions, or use the compiler-included build.',
                    vm
                )
            } else {
                warn(
                    'Failed to mount component: template or render function not defined.',
                    vm
                )
            }
        }
    }
    callHook(vm, 'beforeMount')

    let updateComponent
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        updateComponent = () => {
            const name = vm._name
            const id = vm._uid
            const startTag = `vue-perf-start:${id}`
            const endTag = `vue-perf-end:${id}`

            mark(startTag)
            const vnode = vm._render()
            mark(endTag)
            measure(`vue ${name} render`, startTag, endTag)

            mark(startTag)
            vm._update(vnode, hydrating)
            mark(endTag)
            measure(`vue ${name} patch`, startTag, endTag)
        }
    } else {
        updateComponent = () => {
            vm._update(vm._render(), hydrating)
        }
    }

    // 创建渲染watcher
    new Watcher(vm, updateComponent, noop, {
        before() {
            if (vm._isMounted && !vm._isDestroyed) {
                callHook(vm, 'beforeUpdate')
            }
        }
    }, true /* isRenderWatcher */)
    hydrating = false

    // manually mounted instance, call mounted on self
    // mounted is called for render-created child components in its inserted hook
    if (vm.$vnode == null) {
        vm._isMounted = true
        callHook(vm, 'mounted')
    }
    return vm
}

```