# vue-property-decorator源码解析

> version 9.1.2
> 愣锤 2022/02/06

### 介绍

[vue-property-decorator](https://github.com/kaorun343/vue-property-decorator)是对[vue-class-component](https://github.com/vuejs/vue-class-component)的进一步封装，让vue+ts开发时支持更多的装饰器写法，例如`@Prop、@Watch`等。

基本用法如下：

```js
@Component
export default class YourComponent extends Vue {
  @Prop({ default: 'default value' }) readonly propB!: string

  @Provide('bar') baz = 'bar'

  @Inject() readonly foo!: string

  @Watch('child')
  private onChildChanged(val: string, oldVal: string) {}

  @Emit()
  onInputChange(e: Event) {
    return e.target.value
  }
}
```

### 工程构建

从`package.json`文件中的`main`字段可知库的入口文件是`lib/index.umd.js`:

```json
{
  "main": "lib/index.umd.js",
}
```

但是`lib/index.umd.js`在源码中并不存在，因此这一定是打包后的产物，所以我们再看构建相关的脚本命令：

```json
{
  "scripts": {
    "build": "tsc -p ./src/tsconfig.json && rollup -c"
  }
}
```

从命令可知构建逻辑是先通过`tsc`命令编译ts文件，然后再通过`rollup`打包输出文件。再看`rollup.config.js`    文件的配置：

```js
export default {
  input: 'lib/index.js',
  output: {
    file: 'lib/index.umd.js',
    format: 'umd',
    name: 'VuePropertyDecorator',
    globals: {
      vue: 'Vue',
      'vue-class-component': 'VueClassComponent',
    },
    exports: 'named',
  },
  external: ['vue', 'vue-class-component', 'reflect-metadata'],
}
```

由此可知打包的入口`lib/index.js`也就是源码程序的入口， 输出地址是`lib/index.umd.js`，这也就和`package.json`文件中的`main`字段对应上了。

### 源码解析

入口文件内容如下：

```js
/** vue-property-decorator verson 9.1.2 MIT LICENSE copyright 2020 kaorun343 */
/// <reference types='reflect-metadata'/>
import Vue from 'vue'
import Component, { mixins } from 'vue-class-component'

export { Component, Vue, mixins as Mixins }

export { Emit } from './decorators/Emit'
export { Inject } from './decorators/Inject'
export { InjectReactive } from './decorators/InjectReactive'
export { Model } from './decorators/Model'
export { ModelSync } from './decorators/ModelSync'
export { Prop } from './decorators/Prop'
export { PropSync } from './decorators/PropSync'
export { Provide } from './decorators/Provide'
export { ProvideReactive } from './decorators/ProvideReactive'
export { Ref } from './decorators/Ref'
export { VModel } from './decorators/VModel'
export { Watch } from './decorators/Watch'
```

从上面的入口文件可以看到，该库直接从`vue-class-component`直接导出了`Component、Mixins`方法，然后导出了实现的`Prop、Emit、Inject、Provide、Watch`等装饰器。源码结构相对简单，就是直接导出了一系列装饰器方法。

### vue-class-component的插件原理

在具体分析`vue-property-decorator`装饰器原理之前，先看下`vue-class-component`的插件原理，因为`vue-property-decorator`是依赖`vue-class-component`暴露的创建装饰器的方法来封装新的装饰器的。

在我们分析`vue-class-component`源码的时候，我们知道`vue-class-component`本质原理就是在处理类上静态属性/方法、实例属性/方法等，转化成vue实例化所需要的`options`参数。但是在处理options的时候，其中有下面这点一段代码，我们是只提到它是用于处理例如`vue-property-decorator`等库的装饰器的：

```js
// decorate options
const decorators = (Component as DecoratedClass).__decorators__

if (decorators) {
    decorators.forEach(fn => fn(options))
    delete (Component as DecoratedClass).__decorators__
}
```

这段代码就是判断Component装饰的原始类上是否存在`__decorators__`属性（值是一个全是函数的数组，本质是一系列用于创建装饰器的工程函数），如果存在的话就依次调用数组的每一项，并把`options`参数的控制权交给当前项函数，这样的话外界就能力操作`options`参数了。

但是Component装饰的原始类本身是不会携带`__decorators__`属性的，只有在使用了例如`vue-property-decorator`库暴露的装饰器时，才会在断Component装饰的原始类上添加`__decorators__`属性。

之所以使用`vue-property-decorator`库的装饰器后会在Component装饰的原始类上添加`__decorators__`属性，是因为`vue-property-decorator`的装饰器中会使用`vue-class-component`暴露的创建装饰器的方法，该方法会在Component装饰的原始类上添加`__decorators__`属性。看下`vue-class-component`暴露的工具方法：

```js
/**
 * 一个抽象工厂函数，用于创建装饰器工厂
 * @param factory 用于创建装饰器的工厂
 */
export function createDecorator (factory: (options: ComponentOptions<Vue>, key: string, index: number) => void): VueDecorator {
  return (target: Vue | typeof Vue, key?: any, index?: any) => {
    // 获取Component装饰的类的构造函数
    const Ctor = typeof target === 'function'
      ? target as DecoratedClass
      : target.constructor as DecoratedClass
    // 如果该类上不存在__decorators__属性，则设置默认值
    if (!Ctor.__decorators__) {
      Ctor.__decorators__ = []
    }
    if (typeof index !== 'number') {
      index = undefined
    }
    // 在__decorators__加入处理装饰器的逻辑
    Ctor.__decorators__.push(options => factory(options, key, index))
  }
}
```

整体逻辑，咱们比如以使用`@Prop`为例：

- 在Component装饰的类上使用@Prop装饰器
- @Prop装饰器的内部实现中调用了`createDecorator`方法，并传入处理`@Prop`的逻辑（也就是如何处理`vue`中的`prop`属性）
- `createDecorator`方法会在`@Component`装饰的类上添加`__decorators__`属性，并在`__decorators__`数组中添加一项，该项是个函数，用于调用@Prop处理逻辑
- `@Component`装饰器在执行时生成`options`参数后，会判断`__decorators__`是否存在，如果存在则依次调用其中的函数。
- 因此也就通过钩子的调用实现了处理`@Prop`逻辑。

### @Prop装饰器原理

`@Prop`装饰器的实现都在`Prop.ts`文件中，代码如下：

```js
import Vue, { PropOptions } from 'vue'
import { createDecorator } from 'vue-class-component'
import { Constructor } from 'vue/types/options'
import { applyMetadata } from '../helpers/metadata'

/**
 * 封装的处理props属性的装饰器
 * @param  options @Prop(options)装饰器内的options参数选项
 * @return PropertyDecorator | void
 */
export function Prop(options: PropOptions | Constructor[] | Constructor = {}) {
  return (target: Vue, key: string) => {
    // 如果@Prop(options)的options不存在type属性，
    // 则通过ts元数据获取@Prop装饰器装饰的属性的类型赋值给options.type
    applyMetadata(options, target, key)
    // createDecorator是工具方法
    // 参数才是真正处理prop的逻辑
    createDecorator((componentOptions, k) => {
      /**
       * 给vue-class-component生成的options.props[key]赋值为
       * @Prop的参数options，注意这里两处的options概念不同
       *
       * 再重复下概念：
       *  - componentOptions 是vue-class-component生成的options参数
       *  - k 是@Prop装饰器装饰的属性
       *  - options 是@Prop(options)装饰器的options参数
       */
      ;(componentOptions.props || ((componentOptions.props = {}) as any))[
        k
      ] = options
    })(target, key)
  }
}

// 判断是否支持ts元数据的Reflect.getMetadata功能
const reflectMetadataIsSupported =
  typeof Reflect !== 'undefined' && typeof Reflect.getMetadata !== 'undefined'

// applyMetadata的实现
export function applyMetadata(
  options: PropOptions | Constructor[] | Constructor,
  target: Vue,
  key: string,
) {
  if (reflectMetadataIsSupported) {
    if (
      !Array.isArray(options) &&
      typeof options !== 'function' &&
      !options.hasOwnProperty('type') &&
      typeof options.type === 'undefined'
    ) {
      // 只有在装饰器参数为对象且不存在type属性时，
      // 才通过ts元数据获取数据类型给options.type赋值
      const type = Reflect.getMetadata('design:type', target, key)
      if (type !== Object) {
        options.type = type
      }
    }
  }
}
```

核心实现，就是通过`applyMetadata`处理`@Prop`参数没有`type`属性时，利用ts元数据获取`@Prop`装饰器装饰的属性的类型作为`vue prop`参数的`type`值。然后调用`createDecorator`创建`prop`处理的逻辑，处理逻辑更是直接把`@Prop`的`options`选项作为`vue prop key`的`value`值。

```js
@Component
export default class MyComponent extends Vue {
  @Prop() age!: number
}

// 上述代码也就被转化成了下面的代码
// 是因为上述代码利用元数据获取age的类型是Number拼接为options.type，
// 然后将options赋值给props.age
export default {
  props: {
    age: {
      type: Number,
    },
  },
}
```

### @Watch装饰器原理

先对比一下`vue-property-decorator`的watch写法和`vue2`的watch写法：

```js
/**
 * vue-property-decorator的watch写法
 */
import { Vue, Component, Watch } from 'vue-property-decorator'

@Component
export default class YourComponent extends Vue {
  @Watch('person', {immediate: true, deep: true })
  private onPersonChanged1(val: Person, oldVal: Person) {
    // your code
  }
}

/**
 * vue2的watch写法，
 * 需要注意的是watch是支持数组写法的
 */
export default {
  watch: {
    'path-to-expression': [
      exprHandler,
      {
        handler() {
            // code...
        },
        deep: true,
        immediate: true,
      }
    ]
  }
}
```

`@Watch`装饰器就是要处理Component装饰的原生类中的`watch`语法。

```js
import { WatchOptions } from 'vue'
import { createDecorator } from 'vue-class-component'

/**
 * decorator of a watch function
 * @param  path the path or the expression to observe
 * @param  watchOptions
 */
export function Watch(path: string, watchOptions: WatchOptions = {}) {
  return createDecorator((componentOptions, handler) => {
    /**
     * 获取Component装饰的类上定义的watch参数，没有就赋默认值为空对象
     * 注意a ||= b的写法等同于 a = a || b
     */
    componentOptions.watch ||= Object.create(null)
    const watch: any = componentOptions.watch

    /**
     * 把watch监听的key的回调统一格式化成数组
     * watch的key的回调是支持string | Function | Object | Array的
     */
    if (typeof watch[path] === 'object' && !Array.isArray(watch[path])) {
      watch[path] = [watch[path]]
    } else if (typeof watch[path] === 'undefined') {
      watch[path] = []
    }

    /**
     * 把 @Component 上 @Watch 装饰的函数和装饰器参数组合成vue的watch写法
     */
    watch[path].push({ handler, ...watchOptions })
  })
}
```

`@Watch`的实现还是利用`createDecorator`创建装饰器，其参数是具体的实现逻辑。具体逻辑就是获取`@Component`装饰的类上处理的`watch`参数，然后统一成数组格式，然后把`@Watch`装饰的函数逻辑以vue2的写法添加到其`watch`的数据的数组中。

### @Ref装饰器原理

`@Ref`装饰器的源码实现如下：

```js
import Vue from 'vue'
import { createDecorator } from 'vue-class-component'

/**
 * decorator of a ref prop
 * @param refKey the ref key defined in template
 */
export function Ref(refKey?: string) {
  return createDecorator((options, key) => {
    options.computed = options.computed || {}
    options.computed[key] = {
      // cache废弃语法
      cache: false,
      // 通过计算属性的get返回$refs属性
      // 例如 @Ref() readonly myDom: HTMLDivElement
      // 返回的是this.$refs.myDom
      get(this: Vue) {
        // 优先取@Ref装饰器的参数，否则取属性属性名，作为ref的key 
        return this.$refs[refKey || key]
      },
    }
  })
}

```

核心实现还是获取`@Component`装饰的类上的`computed`属性，然后增加一个计算属性，通过计算属性的`get`返回一个`this.$refs`的正常写法。这里提一点，`cache`是已经废弃的语法，这里仍然保留只是向前兼容。

### @Emit装饰器原理

```js
import Vue from 'vue'

// Code copied from Vue/src/shared/util.js
const hyphenateRE = /\B([A-Z])/g
// 字符串大写转连字符
const hyphenate = (str: string) => str.replace(hyphenateRE, '-$1').toLowerCase()

/**
 * decorator of an event-emitter function
 * @param  event The name of the event
 * @return MethodDecorator
 */
export function Emit(event?: string) {
  return function (_target: Vue, propertyKey: string, descriptor: any) {
    // 根据@Emit装饰器的函数名获取连字符的格式
    const key = hyphenate(propertyKey)
    const original = descriptor.value

    // 覆写装饰器函数的逻辑
    descriptor.value = function emitter(...args: any[]) {
      const emit = (returnValue: any) => {
        const emitName = event || key

        // 如果@Emit装饰的函数没有返回值，
        // 则直接emit装饰器函数的所有参数
        if (returnValue === undefined) {
          if (args.length === 0) {
            this.$emit(emitName)
          } else if (args.length === 1) {
            this.$emit(emitName, args[0])
          } else {
            this.$emit(emitName, ...args)
          }
        // 如果@Emit装饰的函数有返回值，
        // 则直接emit的值依次为：返回值、函数参数
        } else {
          args.unshift(returnValue)
          this.$emit(emitName, ...args)
        }
      }

      // 获取返回结果
      const returnValue: any = original.apply(this, args)

      // 如果是返回的promise则在then时emit
      // 否则直接emit
      if (isPromise(returnValue)) {
        returnValue.then(emit)
      } else {
        emit(returnValue)
      }

      return returnValue
    }
  }
}

// 判断是否是promise类型
// 判断手段为鸭式辨型
function isPromise(obj: any): obj is Promise<any> {
  return obj instanceof Promise || (obj && typeof obj.then === 'function')
}
```

### @VModel装饰器原理

```js
import Vue, { PropOptions } from 'vue'
import { createDecorator } from 'vue-class-component'

/**
 * decorator for capturings v-model binding to component
 * @param options the options for the prop
 */
export function VModel(options: PropOptions = {}) {
  const valueKey: string = 'value'
  return createDecorator((componentOptions, key) => {
    // 给props.value赋值为装饰器参数
    ;(componentOptions.props || ((componentOptions.props = {}) as any))[
      valueKey
    ] = options
    // 给computed[被装饰的属性key]赋值为get/set
    // get时直接返回props.value, set时触发this.$emit('input')事件
    ;(componentOptions.computed || (componentOptions.computed = {}))[key] = {
      get() {
        return (this as any)[valueKey]
      },
      set(this: Vue, value: any) {
        this.$emit('input', value)
      },
    }
  })
}
```

### 其他
s
其他装饰器的实现基本大同小异，就不再过多介绍。