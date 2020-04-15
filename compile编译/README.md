# compile

> vue的编译，我们从vue的runtime版本中过一下，我么在平时开发过程中，编译过程都通过vue-loader给我们提前处理了，runtime版本的vue的编译代码定义在src/platforms/web/entry-runtime-with-compiler.js中，runtime的$mount实现是通过获取$options上定义的render函数，如果没有定义，则获取template，然后执行compileToFunctions方法，传入template和配置参数，返回一个render函数，compileToFunctions定义在src/platforms/web/compiler/index.js中，最后执行mount方法，mount方法是我们在Vue原型上定义的$mount，而原型上的$mount方法定义在src/platforms/web/runtime/index.js中，实际上执行$mount是调用了mountComponent方法，方法定义在src/core/instance/lifecycle.js中

```
    // 浏览器端$mount 定义在src/platforms/web/runtime/index.js
    const mount = Vue.prototype.$mount 
    Vue.prototype.$mount = function (
        el?: string | Element,
        hydrating?: boolean
    ): Component {
        // 获取传入的根节点
        el = el && query(el)

        /* istanbul ignore if */
        // 如果根节点定义在body上或者html根节点上则在开发环境报错提示
        if (el === document.body || el === document.documentElement) { 
            process.env.NODE_ENV !== 'production' && warn(
            `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
            )
            return this
        }
        // 获取合并配置后的$options
        const options = this.$options 
        // resolve template/el and convert to render function
        // 如果没有定义render方法，就去获取定义的template字段
        if (!options.render) { 
            let template = options.template
            // 如果template存在，对其做类型处理
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
            } else if (el) {
                template = getOuterHTML(el)
            }
            if (template) {
                /* istanbul ignore if */
                if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
                    mark('compile')
                }
                // 调用compileToFunctions对template编译，定义在src/platforms/web/compiler/index.js
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
                if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
                    mark('compile end')
                    measure(`vue ${this._name} compile`, 'compile', 'compile end')
                }
            }
        }
        // 执行vue原型上的$mount
        return mount.call(this, el, hydrating) 
    }
```

## compile函数实现

> compileToFunctions传入template模板字符串和配置参数，执行返回render和staticRenderFns，compileToFunctions是通过执行createCompiler返回，createCompiler方法定义在src/compiler/index.js中，createCompilerCreator这个函数我们可以看到实际上编译的过程是在这个方法传入的baseCompile实现的，我们后面再看，createCompilerCreator方法定义在src/compiler/create-compiler.js中，createCompilerCreator接收baseCompile，返回createCompiler函数，createCompiler内部定义了compile函数，它对我们传入的options做了一层扩展，返回compile和compileToFunctions，compileToFunctions实际上是通过createCompileToFunctionFn对compile做了一层包装，createCompileToFunctionFn方法定义在src/compiler/to-function.js中，实质是通过createFunction把我们编译后的code传入，返回一个new Function

```
    export function createCompilerCreator (baseCompile: Function): Function {
        return function createCompiler (baseOptions: CompilerOptions) {
            function compile (
                template: string,
                options?: CompilerOptions
            ): CompiledResult {
                ...
                ...
            }

            return {
                compile,
                compileToFunctions: createCompileToFunctionFn(compile)
            }
        }
    }

```

> createCompileToFunctionFn，创建cache缓存编译节点，因为编译是特别耗时的，这里通过缓存优化编译过程；通过new Function('return 1')，判断当前环境支不支持new Function来构建方法，尝试获取缓存，最后执行compile方法得到compiled，compile是在上一个函数createCompiler中当做参数传入的

```
    export function createCompileToFunctionFn (compile: Function): Function {
        const cache = Object.create(null)

        return function compileToFunctions (
            template: string,
            options?: CompilerOptions,
            vm?: Component
        ): CompiledFunctionResult {
            options = extend({}, options)
            const warn = options.warn || baseWarn
            delete options.warn

            /* istanbul ignore if */
            if (process.env.NODE_ENV !== 'production') {
                // detect possible CSP restriction
                try {
                    new Function('return 1')
                } catch (e) {
                    if (e.toString().match(/unsafe-eval|CSP/)) {
                        warn(
                            'It seems you are using the standalone build of Vue.js in an ' +
                            'environment with Content Security Policy that prohibits unsafe-eval. ' +
                            'The template compiler cannot work in this environment. Consider ' +
                            'relaxing the policy to allow unsafe-eval or pre-compiling your ' +
                            'templates into render functions.'
                        )
                    }
                }
            }

            // check cache
            const key = options.delimiters
            ? String(options.delimiters) + template
            : template
            if (cache[key]) {
                return cache[key]
            }

            // compile
            const compiled = compile(template, options)

            // check compilation errors/tips
            if (process.env.NODE_ENV !== 'production') {
                if (compiled.errors && compiled.errors.length) {
                    if (options.outputSourceRange) {
                        compiled.errors.forEach(e => {
                            warn(
                            `Error compiling template:\n\n${e.msg}\n\n` +
                            generateCodeFrame(template, e.start, e.end),
                            vm
                            )
                        })
                    } else {
                        warn(
                            `Error compiling template:\n\n${template}\n\n` +
                            compiled.errors.map(e => `- ${e}`).join('\n') + '\n',
                            vm
                        )
                    }
                }
                if (compiled.tips && compiled.tips.length) {
                    if (options.outputSourceRange) {
                        compiled.tips.forEach(e => tip(e.msg, vm))
                    } else {
                        compiled.tips.forEach(msg => tip(msg, vm))
                    }
                }
            }

            // turn code into functions
            const res = {}
            const fnGenErrors = []
            res.render = createFunction(compiled.render, fnGenErrors)
            res.staticRenderFns = compiled.staticRenderFns.map(code => {
                return createFunction(code, fnGenErrors)
            })

            // check function generation errors.
            // this should only happen if there is a bug in the compiler itself.
            // mostly for codegen development use
            /* istanbul ignore if */
            if (process.env.NODE_ENV !== 'production') {
                if ((!compiled.errors || !compiled.errors.length) && fnGenErrors.length) {
                    warn(
                    `Failed to generate render function:\n\n` +
                    fnGenErrors.map(({ err, code }) => `${err.toString()} in\n\n${code}\n`).join('\n'),
                    vm
                    )
                }
            }

            return (cache[key] = res)
        }
    }
```

> 回过头来看createCompiler中声明的当做参数传入的compile，compile会处理传入的配置参数options（对平台相关的编译配置和自定义的配置做合并），做最后执行baseCompile

```
function compile (
    template: string,
    options?: CompilerOptions
): CompiledResult {
    // 传入的web平台编译的相关配置 vue\src\platforms\web\compiler\options.js
    const finalOptions = Object.create(baseOptions)
    const errors = []
    const tips = []

    let warn = (msg, range, tip) => {
        (tip ? tips : errors).push(msg)
    }

if (options) {
    if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
        // dev环境的提示忽略先
        ...
    }
    // merge custom modules
    // 合并自定义配置
    if (options.modules) {
        finalOptions.modules =
            (baseOptions.modules || []).concat(options.modules)
        }
        // merge custom directives
        if (options.directives) {
            finalOptions.directives = extend(
                Object.create(baseOptions.directives || null),
                options.directives
            )
        }
        // copy other options
        for (const key in options) {
            if (key !== 'modules' && key !== 'directives') {
                finalOptions[key] = options[key]
            }
        }
    }

    finalOptions.warn = warn
    // 传入模板字符串 最后的编译配置 执行编译
    const compiled = baseCompile(template.trim(), finalOptions)
    if (process.env.NODE_ENV !== 'production') {
        detectErrors(compiled.ast, warn)
    }
    compiled.errors = errors
    compiled.tips = tips
    return compiled
}
```

> baseCompile 在执行 createCompilerCreator 方法时作为参数传入，真正的编译过程parse->optimize->generate，parse解析模板字符串生成ast，通过optimize优化处理ast树，把ast和编译配置传入generate生成code，返回render函数

```
// \vue\src\compiler\index.js
const createCompiler = createCompilerCreator(function baseCompile (
    template: string,
    options: CompilerOptions
    ): CompiledResult {
        const ast = parse(template.trim(), options)
        if (options.optimize !== false) {
            optimize(ast, options)
        }
        const code = generate(ast, options)
        return {
            ast,
            render: code.render,
            staticRenderFns: code.staticRenderFns
        }
    })
```