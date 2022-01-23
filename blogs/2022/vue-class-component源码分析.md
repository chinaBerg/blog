# vue-class-component源码分析

愣锤，2022/01/22

### vue-class-component说明

vue官方团队开源的vue-class-component库，让vue2支持了class风格的语法，如下所示:

```js
import Vue from 'vue'
import Component from 'vue-class-component'

@Component({
    name: 'ComponentName',
    watch: {},
})
export default class Counter extends Vue {
  // data数据
  count = 0

  // 计算属性
  get myCount() {
    return this.count
  }
  set myCount(val) {
    this.count = val;
  }
  
  // 声明周期例子
  created() {}
  
  // render函数例子
  render() {}
}
```

而vue2的常规写法是下面这样的：

```js
export default {
  data() {
      return {
        count: 0,
      }
  },
  watch: {},
  computed: {},
  created() {},
  render() {},
}
```

### 文件结构

vue-class-component源码结构如下图所示：

![image](https://note.youdao.com/yws/res/18309/E4D99A662E134051A830EA299BF227B8)

- build 打包脚本
- dist 打包后的文件输出地址
- docs 文档源码（基于vuepress的文档）
- example 开发demo（其实就是一个vue项目）
- src 这是vue-component-class的源码

### 整体思路

vue-class-component的核心实现，是要将class风格的语法转换成options的语法。大家第一想法可能是要走代码编译实现，但是vue-class-component没有这么做。

大家知道，ts装饰器Decorator是为在类、类成员上通过元编程语法添加标注的功能。因此我们可以利用ts装饰器的功能，将class转换成vue实例。核心思路如下：

![image](https://note.youdao.com/yws/res/18340/925447EC5BA7425E9C6EC9376C05181D)

### Component实现

```js
function Component <V extends Vue>(options: ComponentOptions<V> & ThisType<V>): <VC extends VueClass<V>>(target: VC) => VC
function Component <VC extends VueClass<Vue>>(target: VC): VC
function Component (options: ComponentOptions<Vue> | VueClass<Vue>): any {
  /**
   * options为函数，说明Component是作为普通的装饰器函数使用
   * 注意，装饰器本身就是一个函数而已，此时的options参数就指代被装饰的类，
   * 把此处的options当成装饰器的target更好理解
   * @example
   *    @Component
   *    class Demo extends Vue {}
   */
  if (typeof options === 'function') {
    return componentFactory(options)
  }

  /**
   * options不是函数时，说明option被传入了值，
   * 则Component是作为装饰器工厂使用
   * @example
   *    @Component({ name: 'DemoComponent', filters: {} })
   *    class Demo extends Vue {}
   */
  return function (Component: VueClass<Vue>) {
    return componentFactory(Component, options)
  }
}
```

可以看到Component就是一个函数，这个函数就是一个装饰器。但是要注意当是，这个函数既是一个装饰器，也是一个装饰器工厂。原因如下，看个例子：

```js
// 定义一个装饰器
function Decorator(target) {
    // 这里的target其实就是类的构造函数
}

@Decorator
class Demo {}
```

装饰器直接修饰的class时，target就是类的构造函数。因为class本身是函数的语法糖，所以typeof options为函数时，我们把它作为装饰器使用，否则作为装饰器工厂使用。而装饰器工厂就是可以接收外部参数，并且返回一个装饰器。

所以Component函数就是判断如果是直接装饰的类，就作 为装饰器使用，否则作为装饰器工厂。最终的装饰器实现，是在componentFactory中。


### componentFactory实现

上述得知，`componentFactory`才是最终的装饰器实现。首先看下`componentFactory`轮廓：

```js
export function componentFactory (
  Component: VueClass<Vue>,
  options: ComponentOptions<Vue> = {}
): VueClass<Vue> {
    // ...
}
```

接收两个参数，第一个参数是装饰器装饰的类的构造函数本身。第二个参数是可选项，如果传递了该项说明`Component`是作为装饰器工厂使用的，而`options`也是部分vue参数，比如options就是下面例子中`@Component`参数的部分:

```ts
@Component({
    name: '',
    watch: {}
    // ... 
})
```

下面看内部实现：

```js
export function componentFactory (
  Component: VueClass<Vue>,
  options: ComponentOptions<Vue> = {}
): VueClass<Vue> {
  // ...
  options.name = options.name || (Component as any)._componentTag || (Component as any).name
  
  // 遍历被装饰类原型对象上所有的属性
  const proto = Component.prototype
  Object.getOwnPropertyNames(proto).forEach(function (key) {
    // 排除类的构造函数
    if (key === 'constructor') {
      return
    }

    // hooks
    // created等内置属性、钩子直接赋值
    if ($internalHooks.indexOf(key) > -1) {
      options[key] = proto[key]
      return
    }

    const descriptor = Object.getOwnPropertyDescriptor(proto, key)!
    if (descriptor.value !== void 0) {
      // methods
      // 将类上所有的实例方法拼接到options.methods
      if (typeof descriptor.value === 'function') {
        (options.methods || (options.methods = {}))[key] = descriptor.value
      } else {
        // typescript decorated data
        // 将类上所有的实例数据（注意不包含get\set）添加到options.mixins
        (options.mixins || (options.mixins = [])).push({
          data (this: Vue) {
            return { [key]: descriptor.value }
          }
        })
      }
    } else if (descriptor.get || descriptor.set) {
      // computed properties
      (options.computed || (options.computed = {}))[key] = {
        get: descriptor.get,
        set: descriptor.set
      }
    }
  })
}
```

从这里逻辑开始，就开始实现遍历class上的相关属性，拼接成vue的options参数对象的逻辑了，如上述逻辑架构图所示。

首先是确定组件实例的`name`值，依次从 `options.name、_componentTag、Component.name`顺序取值

然后遍历被装饰类的原型对象(注意是遍历的原型对象才能拿到实例属性/实例方法)，排除`constructor`；如果是`data、created等声明周期函数、render`等，则直接给`options`赋值；否则获取key对应的描述符，根据描述符中的`value`判断如果是函数，则放到`options.methods`属性中：

```js
if (typeof descriptor.value === 'function') {
    (options.methods || (options.methods = {}))[key] = descriptor.value
}
```

如果不是函数则说明是普通的原型对象上的实例属性，则作为vue实例的data中的响应式数据，因此放入options.mixins中：

```js
else {
    (options.mixins || (options.mixins = [])).push({
        data (this: Vue) {
            return { [key]: descriptor.value }
        }
    })
}
```

这种做法是巧妙的利用了vue的`mixins`属性，在数组中推入了多个`data`选项，最终vue实例化时会合并`mixins`。

如果上述判断描述的`value`不存在的时候，则判断描述符是否具有`set/get`属性，如果存在的话则说明是我们写的class的`setter`或getter`，那么我们则把它拼在`options.computed`属性中:

```js
if (descriptor.get || descriptor.set) {
    // computed properties
    (options.computed || (options.computed = {}))[key] = {
        get: descriptor.get,
        set: descriptor.set
    }
}
```

### 构造函数上的属性赋值处理

```js
;(options.mixins || (options.mixins = [])).push({
    data (this: Vue) {
      return collectDataFromConstructor(this, Component)
    }
})
```

再次添加了一个mixins值，具体的collectDataFromConstructor内容如下：

```js
export function collectDataFromConstructor (vm: Vue, Component: VueClass<Vue>) {
  // 重写_init方法
  const originalInit = Component.prototype._init
  Component.prototype._init = function (this: Vue) {
    // proxy to actual vm
    const keys = Object.getOwnPropertyNames(vm)
    keys.forEach(key => {
      Object.defineProperty(this, key, {
        get: () => vm[key],
        set: value => { vm[key] = value },
        configurable: true
      })
    })
  }

  // 获取类属性值
  const data = new Component()

  // 将_init重新指回原引用
  Component.prototype._init = originalInit

  // create plain data object
  const plainData = {}
  Object.keys(data).forEach(key => {
    if (data[key] !== undefined) {
      plainData[key] = data[key]
    }
  })

  return plainData
}
```

首先复写Component类的`_init`方法，然后在复写的`_init`方法中遍历所有属性设置`get/set`。然后通过`new Component`的方法拿到实例，注意的是，`Component`类是`extends Vue`的，所以这里实例化的是一个vue实例，而实例化vue的时候背后就是默认调用_init方法的，所以这里复写的`_init`
方法会被调用。最后遍历实例的所有属性得到我们需要的data数据。最终效果就是可以顺利拿到`Component constructor`中注册的响应式数据。

需要提醒一点的是虽然`vue-class-component`帮助我们把`constructor`中定义的数据也做了正常的转化，但是不建议我们在`constructor`中做这个事情，data数据的定义还是放在Component中的实例属性定义，同时也是不建议我们使用constructor，如下：

```js
@Component({
    name: 'ComponentName',
    watch: {},
})
export default class Counter extends Vue {
  // data数据
  count = 0
  
  str = 'this is a string'
}
```

原因是因为会被以为调用两次。[参考官网这里的说明](https://class-component.vuejs.org/guide/caveats.html#always-use-lifecycle-hooks-instead-of-constructor)

### 对外暴露可扩展的机制

```js
// decorate options
const decorators = (Component as DecoratedClass).__decorators__
if (decorators) {
    decorators.forEach(fn => fn(options))
    delete (Component as DecoratedClass).__decorators__
}
```

这里的处理逻辑就是如果Component类上挂有`__decorators__`属性（`__decorators__`属性是由其他基于`vue-class-component`进一步封装的库挂载到`Component`类型的装饰器数组），则依次执行装饰器。执行完毕移除`__decorators__`属性。

比如[vue-property-decorator](https://www.npmjs.com/package/vue-property-decorator)就基于vue-class-component扩展了很多其他装饰器。这部分内容会在后续的`vue-property-decorator`源码解析处再详细说明。

### 处理被装饰类上的静态属性/方法

大家知道，在vue2中如果我们直接在vue的对象上添加属性或方法，那么其是非响应式的，但是一般不建议我们这么做。例如：

```js
export default {
    data() {
        return {
            /// 响应式数据
        }
    },
    someNotReactiveData: 123,
    someNotReactiveFn() {},
}
```

那么在vue-class-component则是对应挂载类的静态属性和静态方法来达到同样的效果：

```js
@Component({
    name: 'ComponentName',
})
export default class Counter extends Vue {
  // 非响应式数据
  static myStaticData = 0
  
  // 非响应式的方法
  static myStaticFn() {}
}
```

`vue-class-component`是如何实现的呢？源码如下：

```js
// 查找到继承自Vue的父类，否则就取Vue
const superProto = Object.getPrototypeOf(Component.prototype)
const Super = superProto instanceof Vue
    ? superProto.constructor as VueClass<Vue>
    : Vue

// 利用Vue.extend扩展一个子类
const Extended = Super.extend(options)

// 处理类上的静态属性和静态方法
forwardStaticMembers(Extended, Component, Super)
```

下面看下`forwardStaticMembers`的具体实现：

```js
function forwardStaticMembers (
  Extended: typeof Vue,
  Original: typeof Vue,
  Super: typeof Vue
): void {
  // We have to use getOwnPropertyNames since Babel registers methods as non-enumerable
  Object.getOwnPropertyNames(Original).forEach(key => {
    // 排除prototype、arguments、callee、caller
    if (shouldIgnore[key]) {
      return
    }

    // 获取原始类上的属性的描述符
    const descriptor = Object.getOwnPropertyDescriptor(Original, key)!

    Object.defineProperty(Extended, key, descriptor)
  })
}
```

这里我移除了部分兼容和日志类的代码，核心代码如上所示，就是遍历被装饰类（注意，这里是直接遍历的Component，而不是Component.prototype），这样就可以得到所有的静态属性和方法，但是要排除`prototype、arguments、callee、caller`这几个属性。然后获取属性对应的描述符，然后通过`Object.defineProperty`添加到我们上面`Vue.extend`构造的子类上面。

### 处理元数据

基本上，到上面的步骤，所有的属性和方法等就已经处理完了，但是如果用户自定义了元数据如何处理呢？

```js
if (reflectionIsSupported()) {
    copyReflectionMetadata(Extended, Component)
}
```

首先要判断当前环境是否支持元数据，不知道元数据的翻阅[reflect-metadata文档](https://www.npmjs.com/package/reflect-metadata)。判断逻辑也就是看是否支持`Reflect.defineMetadata`和`Reflect.getOwnMetadataKeys`这俩方法，当然了reflect-metadata远不止这俩方法。

```js
export function reflectionIsSupported () {
  return typeof Reflect !== 'undefined' && Reflect.defineMetadata && Reflect.getOwnMetadataKeys
}
```

那么如果支持元数据，我们也需要处理元数据的转化，转化就是直接把Component中定义的所有元数据全部拷贝到我们Vue.extend构造的子类上面去。逻辑如下：

```js
// 拷贝元数据
export function copyReflectionMetadata (
  to: VueConstructor,
  from: VueClass<Vue>
) {
  // 这里我们先统一两个术语：
  //   - 原始类，即Component装饰的类
  //   - 扩展类，即Vue.extend(options)得到的类

  // 拷贝原始类上的元数据到扩展类上
  forwardMetadata(to, from)

  // 拷贝原始类上的实例属性和实例方法上的元数据到扩展类上
  Object.getOwnPropertyNames(from.prototype).forEach(key => {
    forwardMetadata(to.prototype, from.prototype, key)
  })

  // 拷贝原始类上的静态属性和静态方法上的元数据到扩展类上
  Object.getOwnPropertyNames(from).forEach(key => {
    forwardMetadata(to, from, key)
  })
}

/**
 * 元数据的拷贝
 * 核心实现就是：
 *  - 利用 Reflect.getOwnMetadata 获取元数据
 *  - 利用 Reflect.defineMetadata 设置元数据
 */
function forwardMetadata (to: object, from: object, propertyKey?: string): void {
  const metaKeys = propertyKey
    ? Reflect.getOwnMetadataKeys(from, propertyKey)
    : Reflect.getOwnMetadataKeys(from)

  metaKeys.forEach(metaKey => {
    const metadata = propertyKey
      ? Reflect.getOwnMetadata(metaKey, from, propertyKey)
      : Reflect.getOwnMetadata(metaKey, from)

    if (propertyKey) {
      Reflect.defineMetadata(metaKey, metadata, to, propertyKey)
    } else {
      Reflect.defineMetadata(metaKey, metadata, to)
    }
  })
}
```

拷贝逻辑有三个：

- 拷贝被装饰类上的元数据
- 拷贝被装饰类中所有的实例属性和实例方法上的元数据
- 拷贝被装饰类中所有的静态属性和静态方法上的元数据

具体的拷贝实现也就是利用`Reflect.getOwnMetadata`获取自身的元数据，再利用`Reflect.defineMetadata`在目标上定义元数据。