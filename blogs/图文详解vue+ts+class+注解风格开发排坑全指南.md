ts的大势所趋，你还在吭哧吭中徘徊在vue+js大门口吗？来吧，是时候表演真正的技术了～～～

从vue开始火热起来到现在，已经基本上前端开发小伙伴入门的技能了。相信这么久时间过去之后，大家也早已习惯vue的开发模式了。那么，你和别人比比的时候，难道不想有些许亮点吗？虽然目前vue2+对ts的支持没有像react、ng等支持的更友好，但是随着社区相关工具链的完善，其生产项目使用vue+ts也是完全ok的。

相信很多小伙伴也早已磨刀霍霍、跃跃欲试了。那么在vue实际生产项目该如何规范、系统的使用vue+ts的开发模式呢？本文将系统的讲述如何在vue2+中开发`typescript+class+注解`风格的项目。图文详情、系统介绍、专注于排坑，一起在生产项目中大胆使用vue+ts吧。奥利给～～～

### ✨ 准备起飞 fly~~~

- 说到起步，首先说下基本的工具和环境吧。我这边使用的4+的cli脚手架，如图所示：

![环境查看.jpg](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172baca56e9925a0~tplv-t2oaga2asx-image.image)

- 使用vue-cli初始化项目

```bash
# 终端运行
vue create vue-ts-demo
```

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bacae51a84894~tplv-t2oaga2asx-image.image)

可以看到，命令运行后，让我们选择项目相关的配置参数，这里我们选择手动选择（Manually select fetures）

- 勾选配置

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bacb430775ddb~tplv-t2oaga2asx-image.image)

这里如上图，最重要的是勾选typescript，除了pwa之外，我们还是都选择配置吧。当然了你也可以根据你的需求而定，但是ts必须要勾选，笔记是ts的项目嘛～～～

- 选择class风格的语法

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bacca8742822d~tplv-t2oaga2asx-image.image)


这里选择class风格，回车就好。如果你不喜欢class风格，也可以选择否，但是我们建议ts开发时选择class风格，优雅（装逼…），如下图，最终class风格的代码会是这样：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172baccf68f397c3~tplv-t2oaga2asx-image.image)
仅演示部分截图，后续会详细讲解

- 继续选择配置，直到完成

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bacd76eb62cce~tplv-t2oaga2asx-image.image)

后面就是一些基本的配置选择了，像css预处理器、代码规范、配置文件位置、router mode等等，大家根据团队规范或者个人的风格进行选择吧，这个没什么要特别说明了。最后等待项目初始化完成即可。

- 运行

```bash
# 进入项目文件夹
cd vue-ts-demo

# 运行项目
npm run serve
```

cli4+项目在初始化完成之后，其实依赖是已经安装好了，是不需要再`npm i`的了

项目启动后，应该就可以看到很熟悉的页面了

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bacdcd90ff2dd~tplv-t2oaga2asx-image.image)


### ✨ 基础目录讲解

- 项目初始化的目录

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bace109483934~tplv-t2oaga2asx-image.image)

从图上可以看到，基础目录基本上和js版的vue项目没太大区别。额外需要注意的是，可以看到根目录新增了`tsconfig.json`作为ts的配置文件，还有其他的配置都独立成了配置文件的形式，这是因为我勾选的是配置文件独立存在。你初始化项目时也可以选择都放在browserList配置中。

另外，需要注意的是，以前所有的`.js`文件都变成了`.ts`文件。

- 基本的vue组件演示

```
<template>
  <div class="hello">
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';

@Component
export default class HelloWorld extends Vue {
  // 声明一个props
  @Prop() private msg!: string;
  
  // 声明一个变量
  private count: number = 0
  
  // 声明一个方法
  private addCount() {
    this.count++
  }
}
</script>

<style scoped lang="less">
</style>
```

可以看到，vue组件的三大件还是那三大件，template、script、style。但是需要注意的是script标签需要加`lang="ts"`的标识，来指明当前是ts的语法。


另外，组件的基本使用方式，变成了注解+class风格的。从`vue-property-decorator`中引入了我们需要的内容，最后导出一个继承后的类，这个类也可以是匿名的（如果你想偷个小懒的话）：

```
@Component
export default class extends Vue {
    // ...
}
```

- class组件的详细说明

```javascript
import {
  Component, Vue, Prop, Watch, Emit,
} from 'vue-property-decorator';

@Component
export default class Header extends Vue {
  // data内的属性直接作为实例属性书写
  // 默认都是public公有属性
  nums = 0;
  // 也可以定义只读
  readonly msg2 = 'this is readonly data'
  // 定义私有属性
  private msg3 = 'this is a private data'
  // 定义一个私有、只读数据
  private readonly msg4 = 'this is a private and readonly data'
  
  
  // @Prop定义props
  // 花括号内定义prop的参数，例如default、type、required等
  @Prop({ default: 'this is title' }) title!: string;
  @Prop({ default: true }) fixed: boolean | undefined;

  // 通过getter书写计算属性
  get wrapClass() {
    return ['head', {
      'is-fixed': this.fixed,
    }];
  }

  // 观察属性
  // 可以通过第二个参数，设置immediate和deep
  @Watch('nums', { immediate: true, deep: true })
  handleNumsChange(n: number, o: number) {
    console.log(n, o);
  }

  // 定义emit事件，参数字符串表示分发的事件名
  // 如果没有，则使用方法名作为分发事件名，会自动转连字符写法
  // 注意，这样写法貌似无法派发多个参数，
  // 可以通过原有的this.$emit写法派发
  @Emit('add')
  emitAdd(n: number) {
    return n;
  }
  @Emit('reduce')
  emitReduce() {
    return (this.nums, 123, 321);
  }

  // 所有方法名，直接作为类的方法书写
  private handleClickAdd() {
    this.nums += 1;
    this.emitAdd(this.nums);
  }
  private handleClickReduce() {
    this.nums -= 1;
    this.emitReduce();
  }
}
```

上面组件，在注释里面详细说明了ts组件的基本常用语法。有了这些语法，基本页面书写和数据渲染应该没问题了。关于class组件的用法，项目其实是使用了[vue-class-component](https://github.com/vuejs/vue-class-component#readme)和[vue-property-decorator](https://github.com/kaorun343/vue-property-decorator#readme)两个库。更多的基本语法，小伙伴还是需要翻阅文档的。

这里要说明一下这两个库的关系。`vue-class-component`是官方维护的一个让vue支持class的风格的库，而`vue-property-decorator`也是基于`vue-class-component`的，但是在它基础之上支持了注解的语法，从而让我们的代码语法更加的简洁和易于复用。从我们的项目也可以看到，其实最终使用的是`vue-property-decorator`的语法的。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bacef65547fc8~tplv-t2oaga2asx-image.image)

- 补充一个npm查阅文档的命令

很多时候，如果我们像快速的打开一个库的文档地址或者源码地址时（又懒得去查），可以通过npm的命令快速做到，只需要终端运行：

```bash
# 查阅文档
npm docs 库名称 
# 例如查询vue-class-component的文档地址
npm docs vue-class-component

# 查看源码地址
npm repo 库名称
```

那么这是怎么做到的呢？

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bacf3b764eb7a~tplv-t2oaga2asx-image.image)

其实这些图在开发时是会在其package.json文件中指定的相关参数的。对npm感兴趣的小伙伴，可以继续翻阅[npm文档](https://docs.npmjs.com/)，学习更多的npm知识。

### TS下路由排坑指南

说完了vue单文件的基本用法之后，我们要来聊聊路由了。在ts模式下，路由还是有些东西需要说明的，不然你在项目里有些坑，会让你丈二和尚摸不着头脑的～～

- 基本的路由使用

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bad015db19397~tplv-t2oaga2asx-image.image)

如图所示，基本的路由使用方式，项目初始化的时候已经配置好了，只不过在原来的基础之上增加了ts的类型。

这里需要注意一下，base的设置是`process.env.BASE_URL`。这个是哪里来的呢？其实读取的配置文件中的信息。cli4+初始化的项目是支持配置文件的。可以在根目录下根据环境不同新建几个配置文件：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bad0b1d987212~tplv-t2oaga2asx-image.image)

注意是`.env`开头，后面是环境类型，例如这里简单演示了本地开发环境，线上dev环境和线上生产环境的配置文件。在配置文件中，我们可以根据不同环境进行不同的配置，例如：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bad1030dc4f99~tplv-t2oaga2asx-image.image)

例如上面的路由base的配置，也大多时候会在这里配置不同的环境的api地址。可能小伙伴这里会有个小疑问，为什么配置了local环境还配置了一个dev环境，如果你本地开发出现跨域，而后端需要你来处理的时候，你跨域在这里单独配置一个本地环境处理api，然后做一些代理的事情，然后线上dev的时候不去处理跨域的问题。还是根据你的需求。

这里再补充一下，如果前端处理跨域该怎么处理吧。在根目录下新建一个`vue.config.js`文件，这是cli4中自定义webpack脚本配置的方法，可以翻阅[cli文档](https://cli.vuejs.org/zh/guide/webpack.html)，有详细的说明。这里看下具体怎么配置：

```javascript
// 以我上面为例
module.exports = {
    devServer: {
    proxy: {
      '/hermes/api/v1/': {
        // 需要代理到的域名地址
        // 即，当遇到'/hermes/api/v1/'时，将请求代理到'http:xxxx.com'
        target: 'http:xxxx.com',
        changeOrigin: true
      },
      '/bms/api/v1/': {
        target: ''http:xxxx.com',
        changeOrigin: true
      }
    }
  },
}
```

解释一下，上面通过设置proxy，来设置当遇到'/hermes/api/v1/'的请求地址时，则将请求代理到其'http:xxxx.com'域名，后面同理。

那这时候，需要给大家看一下api怎么写了：

```
// 简单演示一下，因为api是的单独管理处理的
// 都在src/api/文件夹下
// 例如这是一个account.ts模块

// 引入我们的环境变量中的域名前缀
// 本地开发中，肯定就是我们上面设置的域名前缀
// 这样的话，便会进行代理服务了，从而处理跨域问题
const { VUE_APP_API_KB } = process.env;

export interface ILogin {
  account: string;
  password: string;
}

// 登录
export const login = (data: ILogin): Promise<object> => {
  data.password = md5(data.password);
  return request({
    url: `${VUE_APP_API_KB}login`,
    method: 'post',
    data,
  });
};

```

- 路由钩子的注册

想使用路由钩子，很重要的一点，是要先注册，不然是无法使用的。那么如何注册呢？我们在router文件夹下新建一个`个class-component-hooks.ts`文件：

```javascript
import Component from 'vue-class-component';
// or
// import { Component } from 'vue-property-decorator';

// Register the router hooks with their names
Component.registerHooks([
  'beforeRouteEnter',
  'beforeRouteLeave',
  'beforeRouteUpdate',
]);
```

然后在`mian.ts`文件中，**最顶部进行引入**，保证在所有组件引入之前执行。这点很重要！！！

```
// main.ts顶部导入
import './router/class-component-hooks';
```

- 局部路由守卫参数类型

话题还是回到我们的路由上面。关于路由，有一点需要额外注意的就是路由守卫的使用，这点是要注意的，不然报错。

首先，我们看下页面内的路由守卫的参数问题：

```javascript
// 引入注解插件暴露出的相关内容
import { Vue, Component } from 'vue-property-decorator';
// 引入路由中的Route对象
import { Route } from 'vue-router';


@Component({})
export default class App extends Vue {
    // 参数类型的定义，使用Route，
    // next是函数，所以可以直接使用Function对象,
    // 或者是next: () => void
    beforeRouteEnter(to: Route, from: Route, next: Function) {
        next();
    }
}
```

可以看到，我们在使用类似`beforeRouteEnter`等路由钩子时，需要定义参数的类型，这里我们从`vue-router`中引入了`Route`类型作为`from`和`to`的参数类型。因为ts函数参数是需要定义类型的。而next参数，如果求简单的话，可以直接写个`Function`类型。


### ✨ TS下如何抉择Vuex及其坑点

说到vuex，可是vue项目的重点之一了。相信大家对于js版本的项目如何使用vuex已经是如鱼得水了。这里介绍一下ts版本中如何使用vuex。

在这里，我介绍的是[vuex-module-decorators](https://github.com/championswimmer/vuex-module-decorators#readme)，包括我的项目中也是使用的该库。除此之外也有一个[vuex-class](https://github.com/ktsn/vuex-class),这两个都是可以在ts中辅助我们更好的使用vuex的。但是我为什么选择和推荐`vuex-module-decorators`,后面我会进行说明。好了，下面先看下这个库该如何使用吧：

- 安装vuex-module-decorators

```bash
# 安装vuex-module-decorators
# 支持注解的方式开发store中的内容
cnpm i vuex-module-decorators -S
```

我先看下我们之前默认的store是什么内容：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bad185bba2629~tplv-t2oaga2asx-image.image)

可以看到基本上和正常我们的项目初始化的store结构并无二异，无非是index.js变成了index.ts。

下面我们先基本的组织一下文件结构，项目中基本上一定是会按模块划分的，除非项目真的很小。如下图，应该是我们比较熟悉的代码组织结构：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bad2091cd169b~tplv-t2oaga2asx-image.image)

正常来说，我们会在`modules`文件夹中划分功能模块，`index.ts`作为出口，`constant.ts`作为常量文件；`mutation-types.ts`作为`mutations`的类型管理文件。这里暂时先不说store.ts，因为这是我们等下改造后的文件结构；

接下来，我们就挨个仔细说说每个文件该如何书写和组织。

- store/index.ts，sotre文件的统一暴露出口

```
// store/index.ts

export { default as AccountModule } from './modules/account';
export { default as DownloadModule } from './modules/download';
export { default as DownlableModule } from './modules/kbAnalyse';

// ....
```

如上所示，是我们的index.ts文件，我们在这个文件中做了什么事情呢？其实就是把我们的moudles文件夹下的模块导入进来，然后再统一暴露出去。如果不清楚语法为何这样写的，可以[点击这里了解一下ES6导入导出的复合写法](https://es6.ruanyifeng.com/#docs/module#export-%E4%B8%8E-import-%E7%9A%84%E5%A4%8D%E5%90%88%E5%86%99%E6%B3%95)。

那我们为什么要这么做呢？为什么要统一导入导出呢？实则是为了使用方便，下面我们看下是如何在页面中使用store中的数据和方法的。

- 组件中如何使用store数据

我们知道，在正常模式下的vue组件使用store通常是如下这样的：

```
// 组件中
import { mapState, mapGetters, mapActions } from 'vuex'

export default {
    computed: {
        ...mapGetters('analyse', [
            'someGetterName'
        ])
    },
    methods: {
        ...mapActions('analyse', [
            'customResetState'
        ]),
    }
}
```

下面我们看下在引入了`vuex-module-decorators`之后是如何使用vuex中数据和方法的：

```
// class风格的组件中
import { Component, Vue, Ref } from 'vue-property-decorator';
// 首先引入我们的store中的模块
import { AccountModule } from '@/store/index';

@Component({
  name: 'Login',
})
export default class extends Vue {
    // 省略其他代码

    /**
     * 发送登录的request请求
     */
    private async submitLogin() {
        if (this.isLoading) return;
        try {
          this.isLoading = true;
          const params = clone(this.formData);
          
          // 这里直接通过该模块调用store中的login这个action
          await AccountModule.login(params);
          const { redirect } = this.$route.query;
          this.$router.replace(redirect ? decodeURIComponent(redirect) : '/');
        } catch (error) {
          this.$Notice.error({ title: error });
        } finally {
          this.isLoading = false;
        }
    }
    
    // 省略其他代码
}
```

上述演示了一个action的基本调用，其他state、getter等都一样，直接调用模块里面对应的内容即可。如此，还算是蛮方便的。更有ts的类型推导的加持，非常方便。

- 如何定义一个store模块

上面介绍了store/index.ts以及组件中如何使用，现在我们看下如何定义模块。说到开发模块，其实我们看到，我们只是在index.ts中对模块进行了导入导出，其实我们还是没有实例化我们的store实例的。因为本来是在index中实例化的，我们把index.ts用来导入导出了，所以实例化的部分没了。

那么先说说为什么要在index.ts中进行导入导出，其实是为了在组件人引入时store模块时方便，根据node.js的文件查找规则，我们只需要写到'@/store'即可。说完了这个，该来看我们store/store.ts了：

```
// store/store.ts

import Vue from 'vue';
import Vuex from 'vuex';

Vue.use(Vuex);

export default new Vuex.Store({});
```

可以看到我们的store.ts，其实是被我们用来实例化store了。注意，这里我们没有简化演示，确确实实我们实例化store时什么都没有，因为我们想做的是动态注册store模块，store是支持这个功能的。下面我们看下如何动态注册store模块。

- 动态注册store模块

我们来看下我们书写store模块的基本的一个格式：

```
import store from '@/store/store';
import {
  VuexModule, Module, Mutation, Action, getModule,
} from 'vuex-module-decorators';

@Module({
  dynamic: true, namespaced: true, name: 'account', store,
})
class Account extends VuexModule {
  // state演示，用户token
  public token = getToken() || ''
  
  // getter演示，平台code数组
  public get platformGetter(): string[] {
    return this.userConfig.platform;
  }

  // mutation演示
  @Mutation
  private [types.SET_TOKEN](token: string) {
    this.token = token;
    setToken(token);
  }
  
  // action演示
  // 用户登录
  @Action({ commit: types.SET_TOKEN, rawError: true })
  public async login(params: accountApi.ILogin) {
    const data = await accountApi.login(params);
    return data;
  }
}
```

通过如上演示，我们要注意几个小点：

一、我们需要定义一个继承VuexModule的类来作为导出的module

二、我们通过实例属性来属性state对象，通过类的get属性来属性vex的getter属性，通过@Mutation、@Action来开发对于的mutation和action

三、我们在`@Module({})`中动态注册模块，需要我们手动传入`store`和通过`dynamic: true`指定是动态的；通过`namespaced`开启命名空间，`name`来指定模块名称；

四、注意一下mutation的名称写法，和vue中其实是一样的，不过可能还有一些小伙伴不知道`[types.SET_TOKEN]() {}`到底是什么意思，这其实是es6的属性名表达式，就是把表达式作为属性的key，不清楚的小伙伴可以点击[属性名表达式](https://es6.ruanyifeng.com/#docs/object#%E5%B1%9E%E6%80%A7%E7%9A%84%E7%AE%80%E6%B4%81%E8%A1%A8%E7%A4%BA%E6%B3%95)补补课。

五、action这里，有2点需要注意一下。首先是，如果在action参数中指定来commit是谁，那么可以在action方法内直接return数据，这样其实也是会调用上面的commit方法的

```
@Action({ commit: types.SET_TOKEN})
public async login() {
    const data = await accountApi.login();
    return data;
}

// 等同于
@Action()
public async login() {
    const data = await accountApi.login();
    // 注意，一定是context对象上调用commit方法
    this.context.commit(types.SET_TOKEN, data)
}
```

正常调用commit都是通过this.context来传递的，只不过如果只要一个就可以简写而已；如果是需要多个commit，那只能是老老实实commit了。

关于action，还有一点需要注意的是错误的捕获，如下所示：

```
@Action({ commit: types.SET_TOKEN, rawError: true })
```

[rawError的issue](https://github.com/championswimmer/vuex-module-decorators/issues/26)

[rawError全局开启配置，在文档最后](https://github.com/championswimmer/vuex-module-decorators#readme)

这里如果不配置rawError: true会使得你的主动抛错会有问题。这个一定要注意。

六、最后一点，至于则么取得其他模块的state、getter、action等，直接打印一些相关的参数和载体对象就知道了，暂不多说了。


- mutation-types集中管理所有的muation

这一块没什么好说的，和平常项目一样，把所有的mutation名称集中管理起来。但是如果名称越来越多，其实也是不好管理的，命名容易冲突，如果真的很多，建议还是分模块吧。

```
export const SET_TOKEN = 'SET_TOKEN';
export const RESET_TOKEN = 'RESET_TOKEN';
export const SET_USER_INFO = 'SET_USER_INFO';
export const SET_USER_CONFIG = 'SET_USER_CONFIG';
export const LOGOUT = 'LOGOUT';
```

- store模块中ts的一些说明

我们会在store中定义state，在引入了ts之后，我们其实更好的可以要求我们的module的类实现我们的定义的好的类型：

```
// store/module/account.ts
// 省略其他...

interface IState {
  token: string | undefined;
  userInfo: IUserInfo;
  userConfig: IConfig;
}

@Module({
  dynamic: true, namespaced: true, name: 'account', store,
})
class Account extends VuexModule implements IState {
  // 用户token
  public token = getToken() || ''

  // 用户信息
  public userInfo: IUserInfo = getDefaultUserInfo()

  // 客户配置
  public userConfig: IConfig = getUserConfig()
}

// ...省略其他
```

可以看到，我们要求我们的Account在继承VuexModule后，必须实现我们定义的IState这个类型，即我们的类必须要有
IState中所含有的类型，否则ts就会给我报错。同时，这些实例的类型也必须得是我们之前定义好的类型。这样就会使得代码更严谨和更利于多人合作，这也正体现了ts优势所在。


### ✨ vuex-module-decorators与vuex-class对比PK

前面也提到了说要对比一下两个库，为什么选择vuex-module-decorators。

- 首先用法上来说，两者是各种千秋的。

vuex-module-decorators是在定义模块时通过注解（装饰器）的语法来做的，在使用的时候通过直接引入对应的module让，然后直接使用module上的属性或方法。

vuex-class是在定义模块时，和普通项目基本一样，只不过是需要对变量增加类型而已。但是使用的时候是通过注解（装饰器的语法），如下图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bad3139b98c3d~tplv-t2oaga2asx-image.image)

可以看到，在定义模块时的语法，vuex-module-decorators是更优雅一些的，而在使用时，vuex-class是更胜一筹的。

- 两者结合？

两者结合也是完全可以的，笔者也曾试验过，确实可以。那这样岂不是完美来？那不禁要问了，为何还要放弃vuex-class呢？究其原因，是因为vuex-class在使用时，部分地方会失去ts的类型支持，而只能指定为any。这使得我们在某种程度上失去引入ts的意义，故而笔者更多的选后的另外一个。具体的例子，现在也不想演示了，也或许是有更好的处理方式，而笔者未曾发现吧，有清楚的小伙伴也欢迎指出。

### ✨ Eslint不识别“别名”的处理


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bad6a91e2ebc8~tplv-t2oaga2asx-image.image)

- 安装eslint插件

```bash
# 如果eslint-plugin-import已安装，则不需要重复安装
cnpm install eslint-plugin-import  eslint-import-resolver-alias --save-dev
```

- .eslintrc.js文件增加配置
```javascript
// ...省略其他

settings: {
    'import/resolver': {
      alias: {
        map: [
          ['@', './src'],
        ],
        extensions: ['.ts', '.js', '.jsx', '.json']
      }
    }
}

// ...省略其他
```

[eslint-import-resolver-alias文档](https://github.com/johvin/eslint-import-resolver-alias)


### ✨ 增强类型，处理全局挂载问题

```
// 如果需要在Vue.prototype上挂载一个方法，例如:
// 仅演示用法
const noticeDefaultConfig = {
  duration: 2,
};
Vue.prototype.$success = (msg = '', config = {}) => {
  Vue.prototype.$Notice.success({
    title: msg,
    ...noticeDefaultConfig,
    ...config,
  });
};

// 这时候如果直接使用，
// 则会在编译阶段报错$success不存在，
// 所以需要处理，给vue增强类型

// src/shims-vue.d.ts
import Vue from 'vue'
import VueRouter, { Route } from 'vue-router'

// 原有的
declare module '*.vue' {
  import Vue from 'vue';

  export default Vue;
}
// 增加的扩展
// 增强扩展vue的类型
declare module 'vue/types/vue' {
  interface Vue {
    $router: VueRouter
    $route: Route
    $success: Function
  }
}
```


### ✨ 不支持ts的第三方库如何处理

- 去@types中去寻找

```
// 社区维护了@types/xxx的库
// 目前主流的js库，@types/上基本上都有维护其ts版本
// 所以直接上npm去搜索，例如lodash
@types/lodash

// 如果有的话则直接安装使用,
// 注意安装在本地开发依赖就好
cnpm i @types/lodash -D
```

对于@types没有支持到的库该怎么办？

- 自己给库增加ts支持，限于篇幅，更多的内容可以查看ts文档。后续如果可以，会考虑写篇ts的内容，再详细说吧。

### ✨ interface如何组织

最后说下interrace，随着interface好type声明的越来越多，把interface按模块集中管理起来或许会更好。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172b8efd7e745541~tplv-t2oaga2asx-image.image)

### ✨ Components如何组织

说到components文件夹，也就是我们的公共组件，这块我更倾向于使用index.ts文件对所有的组件统一进行导入导出。这样在引入组件的时候，就不必关注组件实际的位置了，只需要通过从components/index进行导入。来看下components文件夹目录：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bb2d882013fb6~tplv-t2oaga2asx-image.image)

对于组件文件的命名，大家可以参考官网文档的建议，最终还是以团队规范为主。比如我们这里基础组件全部Base开头，对于一个组件在一个模块中只会使用一次的组件，以The开头。全部采用大驼峰命名，包括组件调用也是。这个具体的看规范吧。再看下index.ts文件如何组织：

```
export { default as BaseButton } from './BaseButton/index.vue';
export { default as BaseSideContainer } from './BaseSideContainer/index.vue';
export { default as BaseNofify } from './baseNotify/index.vue';

// ...省略其他
```

再来看下导入时：

```
import { BasePopoverConfirm } from '@/components';
```

最大的好处还是在外部不需要care组件的组件文件夹结构和组件位置关系。components内组件的组织结构发生变化，是完全不会影响到外部的引入路径的。

## 更新（2020-08-13）

### ✨ 处理第三方库挂载在vue原型上的方法的类型定义

- 推荐

最好的做法直接在src/shims-vue.d.ts中，把其定义的文件类型全部导入进来，便会自动合并了。

```
import Vue from 'vue'
import 'vue-router/types/vue'
import 'element-ui/types'

declare module '*.vue' {
  import Vue from 'vue';

  export default Vue;
}
```

如图，想element-ui已经把需要合并的类型，已经合并过了，我们只需要引入即可：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b1cc9e0a145b4aaea5a1d640703d2be7~tplv-k3u1fbpfcp-zoom-1.image)


同理，像vue-router也是一样：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c1a7334c92a14878a0ea87c33e2585af~tplv-k3u1fbpfcp-zoom-1.image)

- 老的做法

需要自己手动导入然后合并到vue这个接口上
```
import Vue from 'vue'
import VueRouter, { Route } from 'vue-router'
import { Message } from 'view-design/types/message.d'

declare module '*.vue' {
  import Vue from 'vue';

  export default Vue;
}

declare module 'vue/types/vue' {
  interface Vue {
    $router: VueRouter
    $route: Route
    $success: Function
    $error: Function
    $Message: Message
  }
}
```

### ✨ 动态添加路由

以vue为例，经常遇见的场景是通过router.addRoute动态注册路由。

只需要注意一点，通配符匹配路由。**一定要放在最后!一定要放在最后!一定要放在最后!**

```
@Mutation
  private [types.GENERATE_USER_ROUTES]() {
    const routes: any = [];
    (this.userAuths || []).forEach((auth) => {
      if (auth.children?.length) {
        auth.children.forEach((item) => {
          const curRoute = dynamicDicts.find((e) => e.name === item.name);
          if (curRoute) routes.push(curRoute);
        });
      }
    });
    // 注意，添加到最后
    routes.push(route404);
    this.userRoutes = routes;
}
```

### ✨ 以插件形式开发全局filters

插件的基本格式就不说了，具体的目录结构也不多说。这里只强调一点，就是vm的类型，使用`typeof Vue`。同理其他像插件等等需要使用vue实例的地方。

```
import Vue from 'vue';
import dayjs from 'dayjs';
import { isFalsy, Falsey } from 'utility-types';
import { DateFormat } from '@/store/constant';

export default {
  install(vm: typeof Vue) {
    // 格式化日期
    vm.filter('formatDate', (t: string | number | Date | Falsey, format = DateFormat.default) => {
      if (isFalsy(t)) return '-';
      return dayjs(t).format(format);
    });
  },
};

### ✨ ts中共享less/scss等的变量

比如项目中很常见的场景，我们在一些组件中需要使用我们的主题色等，而这些颜色很多时候是通过less、sass等变量定义的。那么这就需要我们处理在ts中共享less、sass变量的需求了（使用css-in-js的除外，但是css-in-js也会有一些不舒服的地方）。当然了，我们可以不在js中使用less变量，但是这样我们就需要再常亮里面再定义一次，这是没必要的也是冗余的。下面看看如何共享变量：

- 定义less等变量

var.less文件中定义less变量

```css
/* 主色 */
@primary-color: #007EE6;

/* 主背景色 */
@primary-bg: #f0f2f5;

/**
 * :export指令是webpack提供的用于在js和less中共享变量的功能
 * https://mattferderer.com/use-sass-variables-in-typescript-and-javascript
 */
:export {
  primaryColor: @primary-color;
}
```

上面只有注意`:export`是webpack提供的一个用于在js和less/sass等css处理器之间共享变量的方式就好。但是，还需要注意的时候，在ts里面，如果直接使用，还是会报错未类型的错误，所以这就需要我们给变量文件写声明文件了。

- 定义var.less文件的声明文件

因为是less文件，必须要声明，不然报类型声明不存在的问题，同级目录下创建var.less.d.ts文件：

```
export interface LessVariables {
  primaryColor: string
}

export const variables: LessVariables

export default variables
```

- 使用

```
// 导入less变量文件
import variables from '@/styles/var.less';

// 直接使用即可
/** 鼠标划过时图标的颜色 */
  @Prop({ type: String, default: variables.primaryColor }) hoverColor!: string
// 也可以用在计算属性等
```

### ✨ interface管理

建议在src目录下创建一个interface文件夹，按模块划分，用于专门管理所有的接口和type等。如图：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/592be40998a748c7957e0a6819a1f340~tplv-k3u1fbpfcp-zoom-1.image)

### 👍👍👍最后

希望小伙伴们再看完本篇文章，还没有使用vue+ts开发的小伙伴们能早日在实际项目中开发使用。我相信掌握了这些，进行vue+ts开发实际项目应该是基本OK的。希望本文对想使用ts的小伙伴有那么一丢丢帮助，记得收藏点赞哦！！！



> 百尺竿头、日进一步
> 我是你们的老朋友愣锤，如果觉得喜欢，欢迎点赞收藏！！！