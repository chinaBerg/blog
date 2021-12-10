在日常的项目中，有时候还是不可避免的会维护一些jq官网项目等。面对此类需求，很多还是以前的老套路，前端写页面交给后端去套数据。很烦有木有～～而改动之后还得交给后端再次修改，时间和沟通都是个麻烦。同时开发中，写静态页面也很麻烦，不能复用，不能使用现在的工具，心累有木有～～当然了，解决办法很多

- 自己写个webpack脚本维护起来，把工程化的那一套东西搬过来。
- 使用现成的nust\next等服务端渲染框架
- 借助于node+模板引擎等
- ...

而本文介绍一下node的egg.js框架配合模板引擎来快速开发项目的模式。上手简单快速，一个人搞定前后端。PS：又可以学习新知识来，我好（草）开（泥）心（马），奥利给～～～

![老罗扇脸图.gif](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/12/172a695a0da5ff74~tplv-t2oaga2asx-image.image)

### 初始化项目

```bash
# 初始化
cnpm init egg --type=simple

# 安装依赖
cnpm i

# 启动服务
npm run dev
```

简单看下生成的文件目录（ps：个别文件是没有的，后期自己添加的）

![目录.jpg](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/12/172a69680f6012e1~tplv-t2oaga2asx-image.image)

### 基本介绍

先介绍一下egg中app下的这些文件的基本作用，有个大概的概念：

- app/router.js中编写路由
- app/controller/下编写对应的控制器
- app/service/下编写service
- app/view/下编写对应的模板

![一脸懵逼图](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/12/172a6911b209c529~tplv-t2oaga2asx-image.image)

### 路由编写

- 路由，类似前端项目中的路由。是用来定义请求的URL。
- 当用户访问该路由地址时，将映射到对应的controller进行进一步的处理。
- 由此可见，路由router就是定义和controller之间的一种映射关系。


定义路由，首先要打开app/router.js文件，在里面进行定义，例如：

```javascript
'use strict';

module.exports = app => {
  const { router, controller } = app;
  // 定义首页的路由
  // 即当直接访问域名的时候，将请求映射到controller.home.home中进行进一步的处理
  router.get('/', controller.home.home);
  // 定义关于我们的路由
  router.get('/about', controller.home.about);
  // 定义新闻详情的路由
  router.get('/details/:id', controller.home.details);
};
```

- 此处的路由可以理解为就是我们访问的域名后面的具体的路径地址。例如`xxx.com/about中的/about`

- controller.xx.xx是指当我们访问了这个路由的时候，服务将当前路由映射到这个控制器中。具体的控制器作用，下面会详细介绍。


### 控制器Controller

- 控制器，主要的作用就是处理用户请求的参数，然后可以调用service服务得到我们需要的数据，最后返回数据或者直接渲染出一个模板。
- 而我们这里则是根据router（router参数非必须）渲染出一个页面，也就是渲染一个html页面返回，这样浏览器打开的就直接是页面了。

```javascript
'use strict';

const Controller = require('egg').Controller;

class HomeController extends Controller {
  async home() {
    const { ctx } = this;
    await ctx.render('index.njk')
  }
}

module.exports = HomeController

// 或者你也可以简化一下写法
module.exports = class extends require('egg').Controller {
    // ...
}
```

- 通过调用ctx.render('index.njk')方法，我们将渲染一个模板并返回（可以理解为返回给浏览器）
- 在渲染模板的时候我们也可以给模板传递一个数据对象，ctx.render('index.njk', {})，然后模板内就可以通过模板引擎的语法渲染我们的数据了。
- 一般这个数据是通过service读取的数据库或者调用其他接口得到的数据。这个会在下面的service中讲到。


下面演示一个调用service的例子：

```
// app/controller/home.js

'use strict';
module.exports = class extends require('egg').Controller {
    async details() {
    const { ctx } = this;
    try {
        // 调用service的home模块中的details服务
        // 得到数据后，塞给静态模板使用
        const data = await ctx.service.home.details(ctx.params.id)
        // 渲染一个模板
        await ctx.render('details.njk', data)
    } catch (error) {
      ctx.body = '新闻获取出错'
    }
  }
}

```

- 关于模板相关的内会在下面详细讲解
- 关于service的定义和使用接下来马上讲解

### Service服务

service服务主要是用来做什么的呢？上面在controller中也提到了，主要用来获取数据，拿到数据之后也可以格式化再返回给controller使用。

下面演示一个service调用某些接口服务得到数据并返回给controller使用：

```
// 首先需要在app/下新建service文件夹
// 然后在service下新建home.js，最终为app/service/home.js

'use strict';

module.exports = class extends require('egg').Service {
    // 根据文章id获取文章数据
    // 此处的id是controll调用service服务时传递的id
    async details(id) {
        try {
            const url = `https:xxxx.com/api/${id}`
            const { data } = await this.ctx.curl(url, { dataType: 'json' })
            if (data.data && data.code === 200) {
                // 此处也可以对数据进行处理后再返回
                
                // 返回数据
                return data.data
            }
            throw '数据获取失败'
        } catch (error) {
          throw error
        }
    }

}
```

- 之所以叫`home.js`，是因为我们`controller`中使用的是`ctx.service.home`
- `service.home`中的`home`就是`service`文件的文件名称
- 此处进行了最简单的数据获取，通过`this.ctx.curl`方法请求其他接口的数据。类似于使用`axios`获取数据的操作。
- 当然了，也可以进行数据库数据的增删改查操作，本次就不做演示了，后面会考虑新开一篇egg做项目开发的文章再详细介绍吧。
- 此处对于接口的域名配置可以放在配置文件中
- 对于curl方法也可以进一步的封装在helper中
- 对于数据返回的拦截也可以封装在中间件进行拦截
- 对于上面这些内容有兴趣的可以翻看egg文件中相应的说明，还是蛮详细的。


### 模板渲染

![nunjucks.jpg](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/12/172a693fc33213a5~tplv-t2oaga2asx-image.image)

定义好了基本的路由、控制器和service之后，就剩下模板了。首先模板，可能是前端小伙伴写的静态页面给到我们的（ps：这个前端充气小伙伴或许就是我们自己！哦呸，不是充气的）

好了，言归正传！Egg中有关于[模板渲染](https://eggjs.org/zh-cn/core/view.html)的文档，可以看一下。Egg本身内置了egg-view作为模板的解决方案，其中View支持插件[egg-view-nunjucks](https://github.com/eggjs/egg-view-nunjucks)，本文也是通过该插件进行的模板开发。

```bash
# 首先安装
cnpm i egg-view-nunjucks --save

# 然后在根目录下的config/plugin.js中使用插件
'use strict';

module.exports = {
  nunjucks: {
    enable: true,
    package: 'egg-view-nunjucks',
  }
};

# 完成了插件的安装和引入，我们还需要配置插件的参数
# 根目录下的config/config.default.js中
module.exports = appInfo => {
    // 其他代码
    // ...
    
    // 配置我们的插件参数
    config.view = {
        // 定义映射的文件后缀名
        // 此处我们定义为.njk，那么我们的模板都需要以.njk结束，
        // 这样该类型的文件才会被我们的模板插件进行处理
        mapping: {
          '.njk': 'nunjucks',
        }
    }

    // 其他代码
    // ...
}
```

- 注意，如果不叫`.njk`也是可以的，这个是自定义的，只要满足你的模板文件的后缀名和你定义的一样就行，这样才会被插件处理。
- `egg-view-nunjucks`封装的是`nunjucks`,`nunjucks`推荐叫`.njk`

[egg-view-nunjucks文档](https://github.com/eggjs/egg-view-nunjucks) &nbsp;&nbsp;&nbsp;[nunjucks文档](http://mozilla.github.io/nunjucks/)


有了模板引擎，我们嵌套数据等就会方便很多了。下面来简单看下一个模板的文件夹：

![模板文件夹.jpg](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/12/172a6931fe14a15d~tplv-t2oaga2asx-image.image)
- 如上图所示，需要新建app/view文件夹，所有的模板放在该文件夹下。这个也是可以通过配置修改的（如果你有需求的话）；
- 上面的模板文件，根据你的项目来，这里仅作为演示。通常的，我们可能会把页面一些通用的头部啊、底部、菜单等单独抽离处理复用，具体的还是看你的项目。

下面我们简单看下`base.njk`这个模板，做了什么事情：

```html
<!DOCTYPE html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0,minimum-scale=1.0,maximum-scale=1.0,user-scalable=no">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>{% block title %}title默认内容{% endblock %}</title>
    <meta name="keywords" content="{% block keywords %}keywords默认内容{% endblock %}">
    <meta name="description" content="{% block description %}description默认内容{% endblock %}">
    

    <link rel="stylesheet" type="text/css" href="../../public/css/base.css">
    {% block head %}{% endblock %}
  </head>

  <body>
    <!-- 主体内容部分 -->
    {% block content %}{% endblock %}

    <script src="../../public/js/jquery.js"></script>
    <!-- script部分 -->
    {% block script %}{% endblock %}
  </body>
</html>
```

- 很多时候，静态页面基本的结构都是一样的，或者说类似的。所以我们可以定义一个基本的模板，然后所有的页面继承我们这个基础的模板就好。如何继承，后面会讲。
- 比如，这里我们定义的基本模板中有头部，但是头部中的title等内容可能各个页面不一样，所以我们自定义一个`block`块，这样就可以在页面中替换掉这块的内容。有些类似于`vue的slot`插槽。

```
// 定义block块
{% block title %}默认内容{% endblock %}
```
- 简单说一下这里个人定义的几个基本的block的作用吧：

名称 | 说明
---|---
head块 | 用于在head内最底部插入一些代码。例如，公共模板引入了公共css，但是每个页面还有可能单独引入css，全局只有一个css文件的除外
content块 | 用于放置每个页面的主体部分
script块 | 用于放置每个页面需要单独引入的js文件

定义好了基本的模板和块，下面我们看下页面中如何使用：

```
{% extends "./base/base.njk" %}

{% block title %}这是一个新的title{% endblock %}
{% block description %}这是一个新的description{% endblock %}
{% block keywords %}壹沓科技,加入壹沓,联系我们{% endblock %}
{% set navActive = "about" %}

{% block head %}
  <link rel="stylesheet" href="../public/css/swiper.min.css">
{% endblock %}

{% block content %}
  <div class="page-wrapper">
    <!-- 导入公共的nav模板 -->
    {% include './base/nav.njk' %}

    <!-- 背景图 -->
    <section class="banner-wrapper">
      <img src="../public/img/icon/about-banner.jpg" alt="背景LOGO">
      <span>{{ userName }}</span>
    </section>
    
    <!-- 渲染html模板演示 -->
    <div>{{content | safe}}</div>

    <!-- 导入公共的底部模板 -->
    {% include './base/foot.njk' %}
  </div>
{% endblock %}

{% block script %}
  <script src="../public/js/swiper.min.js"></script>
  <script src="../public/js/about.js"></script>
{% endblock %}
```

- 通过上面一个简单的模板，却包含了最常用和最关键的很多特性。首先第一行，我们要让当前页面继承我们定义的基础模板：

```
// 相对路径即可
{% extends "./base/base.njk" %}
```

- 覆盖我们在基础模板中自定义的块。也可以理解为给vue的slot传入新的内容：

```
// 例如，设置该页面的title
// 如果不设置，就会使用基础模板中默认的
{% block title %}这是一个新的title{% endblock %}
```

- 设置变量，在当前模板中都可以使用。包括引入的其他模板，只要是出现在了当前模板中，都可以取到该值进行使用。

```
// 定义了一个navActive变量
// 后面会讲解一个常见的场景
{% set navActive = "about" %}
```

- 使用变量、渲染内容/html

```
// 变量的使用，和vue的{{}}插值一样
// 比如这个userName是我们从controller中调用service获取的数据对象中的一个属性，
// 然后把对象传递给了模板，
// 那么在模板中可以直接取对象中的属性进行渲染
// 注意，传递给模板的是一个对象，例如{userName: 'jack'},但是我们使用的时候直接取userName
{{userName}}


// 同样的，我们在模板内定义的变量也可以直接使用的

// 渲染html，比如很常见的富文本
// 类似vue的过滤器，这个safe是内置的，
// 过滤器也可以自定义，具体的查看文档吧
{{content | safe}}
```

- 引入其他模板。比如上面可以看到，我们继承了基础的模板，但是我们还可以会提取一些公共的像nav模板进行复用。

```
// 导入我们的nav模板
{% include './base/nav.njk' %}
```

- nav模板一般很常见的会有这么一个场景，比如就是当前页面选项需要高亮。来，先看一下nav模板：

```
<!-- 导航头部 -->
<nav class="nav-wrapper">
  <ul class="menu-content">
    <li>
      <a href="/news" class="{{ 'active' if navActive == 'news' else '' }}">动态资讯</a>
    </li>
    <li>
      <a href="/about" class="{{ 'active' if navActive == 'about' else '' }}">关于我们</a>
    </li>
  </ul>
</nav>
```

可以看到，我们对`class名`进行了一个if判断，判断当变量`navActive`是某些值的时候给他增加一个`active`类名，否则就是各空的`class名`。这里仔细注意一下写法即可。
那么我们怎么定义变量呢？此时再回到上面定义变量的部分介绍，我们已经在页面中定义了一个变量`navActive`，所以我们只需要每个页面定义一个`navActive`且值为我们需要的即可了。

- 最后再补充说明一下，模板内的调整路径，就直接写成我们在`router.js`中定义的路由即可，而不是写的模板地址。

```
// a标签跳转
<a href="/news"></a>

// js调整也是一样，使用router.js定义的路由
window.location.href="/news"
```

![王境特真香警告](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/12/172a69268a07a976~tplv-t2oaga2asx-image.image)

> 百尺竿头、日进一步  
> 我是你们的老朋友愣锤～  
> 喜欢的小伙伴欢迎关注点赞，一起分享交流更多的前端好玩技术！