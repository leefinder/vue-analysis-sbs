# 响应式原理

## Object.defineProperty

> Object.defineProperty() 方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回此对象

- obj
> 要定义属性的对象。
- prop
> 要定义或修改的属性的名称或 Symbol 。
- descriptor
> 要定义或修改的属性描述符。


1. configurable
> 当且仅当该属性的 configurable 键值为 true 时，该属性的描述符才能够被改变，同时该属性也能从对应的对象上被删除。默认为 false。

2. enumerable
> 当且仅当该属性的 enumerable 键值为 true 时，该属性才会出现在对象的枚举属性中。默认为 false。enumerable 定义了对象的属性是否可以在 for...in 循环和 Object.keys() 中被枚举

3. value
> 该属性对应的值。可以是任何有效的 JavaScript 值（数值，对象，函数等）。默认为 undefined。

4. writable
> 当且仅当该属性的 writable 键值为 true 时，属性的值，也就是上面的 value，才能被赋值运算符改变。默认为 false。当 writable 属性设置为 false 时，该属性被称为“不可写的”。它不能被重新赋值。

5. get
> 属性的 getter 函数，如果没有 getter，则为 undefined。当访问该属性时，会调用此函数。执行时不传入任何参数，但是会传入 this 对象（由于继承关系，这里的this并不一定是定义该属性的对象）。该函数的返回值会被用作属性的值。默认为 undefined。

6. set
> 属性的 setter 函数，如果没有 setter，则为 undefined。当属性值被修改时，会调用此函数。该方法接受一个参数（也就是被赋予的新值），会传入赋值时的 this 对象。默认为 undefined。

```
Object.defineProperty(obj, prop, descriptor)
```

## initState

> 在Vue初始化阶段，调用this._init，会执行initState的逻辑，代码定义在src/core/instance/state.js

- initProps
- initMethods
- initData
- initComputed
- initWatch

```
export function initState (vm: Component) {
    vm._watchers = []
    // 把合并后的options赋值给opts
    const opts = vm.$options
    // 分别对props methods data computed watch 进行初始化操作 变成响应式
    if (opts.props) initProps(vm, opts.props)
    if (opts.methods) initMethods(vm, opts.methods)
    if (opts.data) {
        initData(vm)
    } else {
        observe(vm._data = {}, true /* asRootData */)
    }
    if (opts.computed) initComputed(vm, opts.computed)
    if (opts.watch && opts.watch !== nativeWatch) {
        initWatch(vm, opts.watch)
    }
}
```