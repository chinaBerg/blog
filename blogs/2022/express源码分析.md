# express源码分析

> version 4.17.3
> 愣锤

[express.js](https://www.expressjs.com.cn/)是一款基于 Node.js 平台，快速、开放、极简的 Web 开发框架。

```js
const Express = require('express');

const app = new Express();

app.get('/', (req, res) => {
  res.send('Hello Express!')
});

app.listen(3000, () => {
  console.log('express server running at port 3000.');
});
```

# 源码分析

### 基本构成

通过源码根目录下的`index.js`暴露的内容得知，`express`的入口文件在`./lib/express`中，`lib/express`的基本结构如下：

```js
exports = module.exports = createApplication;

/**
 * 创建express应用
 *
 * @return {Function}
 * @api public
 */
function createApplication() {
  // ...
}

/**
 * 导出一些原型对象
 */
exports.application = proto;
exports.request = req;
exports.response = res;

/**
 * 导出一些构造函数
 */
exports.Route = Route;
exports.Router = Router;

/**
 * 导出一些中间件
 */
exports.json = bodyParser.json
exports.query = require('./middleware/query');
exports.raw = bodyParser.raw
exports.static = require('serve-static');
exports.text = bodyParser.text
exports.urlencoded = bodyParser.urlencoded
```

从上述可以看到`express`主要是对外暴露一个`createApplication`函数用于创建`express`应用，同时暴露出内置的路由对象、一些中间件等。`express`对外暴露的内容下图所示：

![image](https://note.youdao.com/yws/res/22179/6887FB6E3F6E40F8BFCA5B8E8A541230)

`4x`的`express`一些内置的中间件已经改变或移除，因此对于旧的中间件使用则会给出提示报错，让使用者自行添加或修改中间件。如下所示：

```js
/**
 * 已经被移除的对外暴露的api列表，
 * 当尝试方法已经移除的api时给适当的错误提示
 */
var removedMiddlewares = [
  'bodyParser',
  'compress',
  'cookieSession',
  'session',
  'logger',
  'cookieParser',
  'favicon',
  'responseTime',
  'errorHandler',
  'timeout',
  'methodOverride',
  'vhost',
  'csrf',
  'directory',
  'limit',
  'multipart',
  'staticCache'
]
removedMiddlewares.forEach(function (name) {
  Object.defineProperty(exports, name, {
    get: function () {
      throw new Error('Most middleware (like ' + name + ') is no longer bundled with Express and must be installed separately. Please see https://github.com/senchalabs/connect#middleware.');
    },
    configurable: true
  });
});
```

### createApplication实现

对于`express`有两种不同的用法，其一是把它当作一个`http`服务使用，如下所示：

```js
const app = express();

app.listen(3000, () => {});
```

其二是把它当作一个已有的`http/https`服务的中间件模型使用，也就是此时的`express`仅仅承担一个已有服务的扩展功能，使已有服务支持中间件模型。使用例子如下所示:

```js
const http = require('http');

const app = express();
app.use(middlewareA);
app.use(middlewareB);

const server = http.createServer(app);

server.listen(3000, () => {});
```

![image](https://note.youdao.com/yws/res/22209/9B6F0FD6DA7440F0B39755052D4254C7)

通过导出的内容可以看到，`express()`的背后就是调用的`createApplication()`函数，那么`createApplication`背后做了什么呢？是如何支持两种不同的使用模式呢？接下来我们带着问题看下`createApplication`的源码实现：

```js
/**
 * 创建express应用
 *
 * @return {Function}
 * @api public
 */
function createApplication() {
  /**
   * app被设计成一个兼容http/https服务callback格式的函数
   *
   * - 把express作为http服务使用
   *   const app = express();
   *   app.listen(3000, () => {});
   *
   * - 把express作为http服务的callback使用
   *   const http = require('http');
   *   const app = express();
   *   const server = http.createServer(app);
   *   server.listen(3000, () => {});
   */
  var app = function(req, res, next) {
    app.handle(req, res, next);
  };

  // 通过mixin的方式 “继承” EventEmitter
  mixin(app, EventEmitter.prototype, false);
  /**
   * 通过mixin的方式 “继承” application.js
   * 从而将express的真正实现抽离到application.js中
   */
  mixin(app, proto, false);

  /**
   * 给app上挂载request和response对象
   * request和response对象上分别添加一个指向app的引用
   */
  app.request = Object.create(req, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })
  app.response = Object.create(res, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  // 调用init方法完成默认配置的初始化
  app.init();

  // 返回app
  return app;
}
```

我们看到`createApplication`函数内容主要逻辑是:

- 创建了一个`app`函数并并返回`app`函数。
- 通过`mixin`来 “继承” `events`触发器的功能，就是支持发布订阅功能。
- 通过`mixin`的方式将所有功能的时候抽离到`application.js`中
- 给`app`函数上挂载`request`和`response`对象
- 最后调用`app.init();`进行初始化参数

首先需要注意的是`app`函数的入参是按照中间件函数的格式定义的。这时候要先了解下中间件的编写要求了。引用一个官方的例子：

![image](https://note.youdao.com/yws/res/22218/27879D95DEB24D7887AF6343894C9750)

中间件是要求接收三个参数，第一个参数是网络请求对象，第二个参数是网络响应对象，第三个参数`next`是个函数，被调用后将执行下一个中间件。

这里的中间件妙在和`http/https`服务的`callback`是兼容的，例如看下`http`服务的`callback`是怎样的：

```js
const http = require('http');

const server = http.createServer((req, res) => {
  res.end('Hello Http Server!');
});

server.listen(3000, () => {});
```

此时我们再回到`app`函数的实现，可以看到`app`函数内就是`app.handle(req, res, next);`，`app.handle`的作用就是调用中间件，稍后会详细讲解。此时便不难理解为什么`express`可以给已有`http/https`服务扩展中间件能力了。因为把`express`实例作为已`有http/https`服务的`callback`使用时，`callback`被调用时实际上调用就是上述的`app`呀，也就是调用的`app.handle`来执行已有的中间件。

![image](https://note.youdao.com/yws/res/22245/0CD651B1B0DF462EB591A8063F6B3B23)

现在我们思考下`express`是如何独立作为`http`服务的呢？这时候我先回忆下，我们把`express`独立使用时，是需要调用`express`实例的`listen`方法来监听端口然后启动服务的：

```js
const app = express();

app.listen(3000, () => {
  console.log('[express] server running at port 3000.');
});
```

所以我们要看下`app上listen`方法的实现。`listen`方法是在上述`mixin`的时候从`application.js`中导出的对象上拷贝到`app`上的，因此其实现是在`application.js`中。如下所示的`listen`的代码实现：

```js
/**
 * Application prototype.
 *
 * 可以简单理解为是express的原型对象的实现
 */
var app = exports = module.exports = {};

/**
 * 监听连接
 *
 * @return {http.Server} 返回node的http.Server服务
 * @public
 */
app.listen = function listen() {
  var server = http.createServer(this);
  return server.listen.apply(server, arguments);
};
```

listen方法自动帮我们创建了一个`http`服务，并把当前`express`实例作为服务的`callback`，因此在`callback`调用时也是通过之前讲到的`app.handle`执行的中间件。

要注意的是为什么这里的`this`指代的是当前`express`实例，因为虽然`listen`方法是在`application.js`中的导出的`app`对象上实现的，但是`createApplication`内通过`mixin`将`listen`方法拷贝到了`createApplication`内的`app`上，可以理解为`listen`方法就是在`createApplication`内的`app`上实现的。因此此处的`this`指向的当前`express`实例。

最后，提一下`mixin`做了什么？`mixin`其实是引用的[merge-descriptors](https://github.com/component/merge-descriptors)库，该库的作用是拷贝目标对象所有属性的descriptors来实现`merge`的。其源码核心实现如下所示：

```js
// 
function merge (dest, src, redefine) {
  Object.getOwnPropertyNames(src).forEach(function forEachOwnPropertyName (name) {
    // 如果属性已存在且不允许覆盖，则不拷贝
    if (!redefine && hasOwnProperty.call(dest, name)) {
      return;
    }

    // 拷贝 descriptor
    var descriptor = Object.getOwnPropertyDescriptor(src, name);
    Object.defineProperty(dest, name, descriptor);
  });

  return dest;
}
```

通过`descriptor`方式的拷贝和普通的拷贝有什么区别呢？看下面的例子就知道了：

```js
const mergeDescriptor = require('merge-descriptors');

const obj = { a: 0 }

Object.defineProperty(obj, 'a', {
  value: 1,
  writable: false,
});

function merge(dest, src) {
  for (let key in src) {
    if (Object.prototype.hasOwnProperty.call(src, key)) {
      dest[key] = src[key];
    }
  }
  return dest;
}

function logDescriptor(obj, key) {
  console.log(Object.getOwnPropertyDescriptor(obj, key));
}

// {a: 1}
console.log(merge({}, obj));
// {a: 1}
console.log(mergeDescriptor({}, obj));
// { value: 1, writable: true, enumerable: true, configurable: true }
logDescriptor(merge({}, obj), 'a');
// { value: 1, writable: false, enumerable: true, configurable: true }
logDescriptor(mergeDescriptor({}, obj), 'a');
```

可以看到最主要的一个区别在于普通的`merge`无法拷贝已修改的`descriptor`相关属性，只拷贝了值。这里就不过多赘述了。

总结一下：

- `express`在初始化的时候并没有做太多的事情，仅仅支持了两种的使用模式和挂载请求对象。
- 中间件等逻辑并没有在一开始进行初始化，相关的功能都在真正使用的时候才会`lazy`加载的，这部分源码实现会在下面讲解。
- `express`整体的代码结构有着很好的分层和模块化思想，但是代码的组织没有真正的`OOP`范式，而是利用函数本质是对象进行扩展函数功能的伪`OOP`的感觉。这里如果利用传统`JS`的`OOP`方式舒服一些：

```js
var http = require('http');
var req = require('./request');
var res = require('./response');

function Express() {
  if (!(this instanceof Express)) {
    return new Express();
  }
  // ... 省略部分实现
  this.request = Object.create(req, {});
  this.response = Object.create(res, {});
}

Express.prototype.listen = function() {
  return http.createServer(this).listen.apply(server, arguments);
}

Express.prototype.callback = function() {
  return function handler(req, res, next) {
    this.handle(req, res, next);
  }
}

// 把Express作为已有服务的callback使用
var app = new Express();
var server = http.createServer(app.callback());
server.listen(3000);

// 把Express单独作为http服务
const app2 = new Express();
app2.listen(3200);
```

### 添加中间件原理

express很核心的一个功能模型就是中间件，express本身更多的是实现了一个中间件架构模型，很多功能都是通过添加中间件来扩展的。使用中间件的方式也很简单，就是通过`app.use()`进行添加：

```js
const express = require('express');

const app = express();

// 添加全局的中间件
app.use((req, res) => {
  console.log('use a global middleware.');
  next();
});

// 添加局部的中间件
app.use('/admin', (req, res, next) => {
  console.log('middleware with target path');
  next();
});

app.listen(3000, () => {
});
```

`app.use`所依赖的中间件模型并没有在express初始化时创建，而是在使用`app.use`时才`lazy`式的初始化。下面直接看`app.use`在`application.js`中的源码实现：

```js
/**
 * app.use()添加中间件
 * 最终是代理到router.use()添加中间件
 * 这部分的使用文档可以查阅官网的`Router#use()`部分
 *
 * 如果fn参数是一个express应用（而非中间件），则会被挂载在指定的_route_上
 * @public
 */
app.use = function use(fn) {
  var offset = 0;
  var path = '/';

  // default path to '/'
  // disambiguate app.use([fn])
  if (typeof fn !== 'function') {
    var arg = fn;

    while (Array.isArray(arg) && arg.length !== 0) {
      arg = arg[0];
    }

    // first arg is the path
    if (typeof arg !== 'function') {
      offset = 1;
      path = fn;
    }
  }

  // 获取所有的中间件函数
  var fns = flatten(slice.call(arguments, offset));

  // app.use()没有传入中间件时给出报错
  if (fns.length === 0) {
    throw new TypeError('app.use() requires a middleware function')
  }

  // lazy路由，仅在没初始化过路由时才初始化路由
  this.lazyrouter();
  // 路由用于添加管理中间件等
  var router = this._router;

  fns.forEach(function (fn) {
    /**
     * fn是中间件，而不是express应用
     * 注意的是express是支持主子应用的
     */
    if (!fn || !fn.handle || !fn.set) {
      // 将中间件添加到路由router的中管理
      return router.use(path, fn);
    }

    debug('.use app under %s', path);

    // 处理挂载的是express应用的情况
    fn.mountpath = path;
    fn.parent = this;

    // restore .app property on req and res
    router.use(path, function mounted_app(req, res, next) {
      var orig = req.app;
      fn.handle(req, res, function (err) {
        setPrototypeOf(req, orig.request)
        setPrototypeOf(res, orig.response)
        next(err);
      });
    });

    // 触发一个子应用挂载的事件
    fn.emit('mount', this);
  }, this);

  // 让app.use支持链式调用
  return this;
};
```

- `app.use`中首先根据参数情况，获取中间件以及中间件生效的path路径
- 利用`this.lazyrouter();`进行初始化路由对象`router`
- 迭代传入的中间件，判断是中间件还是`express`应用
    - 是中间件，则交由`router.use`新增中间件
    - 是`express`应用，则指明父应用和挂载的路径等，同时触发一个子应用挂载的事件

可以看到`app.use`本身是没有处理中间件逻辑的，只是处理了参数和区分中间件与子应用，最终中间件的处理还是代理到`Router`类了。`Router`类的也是通过手动调用`this.lazyrouter`进行`lazy`式的创建的。`lazyrouter`方法的实现也是在`application.js`中，我们看下源码实现：

```js
/**
 * 如果尚未添加路由，则初始化路由
 *
 * 注意：路由没在defaultConfiguration时初始化，
 * 原因是会读取这些默认，但是默认配置可能会在程序启动后改变
 *
 * @private
 */
app.lazyrouter = function lazyrouter() {
  // router尚未被初始化则创建router
  if (!this._router) {
    /**
     * 实例化路由类，用于管理路由中间件（中间件架构就在该类中实现）
     *
     * Router类的参数最终传递给path-to-regexp库
     * - caseSensitive用于指定解析url参数时是否忽略大小写
     * - strict用于指定解析url参数时是否允许匹配结尾的分界符
     */
    this._router = new Router({
      caseSensitive: this.enabled('case sensitive routing'),
      strict: this.enabled('strict routing')
    });

    /**
     * 默认添加query中间件和expressInit中间件
     * - query中间件作用是解析url中的query参数为键值对集合
     * - middleware.init中间件作用是对res和req对象做一些初始化配置和挂载一些引用
     *
     * 注意：this.get的实现是在methods.forEach()逻辑中，该点比较隐晦，
     * this.get实际触发的是this.set('query parser fn')得到的get效果
     */
    this._router.use(query(this.get('query parser fn')));
    this._router.use(middleware.init(this));
  }
};
```

`Router`实例化之后会在`app`上挂载一个_router属性，因此也是判断当`_router`不存在的时候才创建。`Router`本身的作用是创建管理中间件，这里在实例化之后，默认添加了两个中间件：

- `query` 中间件解析`url`中的`query`参数为键值对集合

```js
// 文件在lib/middleware/query.js
/**
 * query解析中间件
 * 将url中的query参数解析成key/value键值对集合
 *
 * @param {Object} options
 * @return {Function}
 * @api public
 */
module.exports = function query(options) {
  var opts = merge({}, options)
  var queryparse = qs.parse;

  // 如果用户自定义了query解析函数，则优先使用用户传入的解析函数
  if (typeof options === 'function') {
    queryparse = options;
    opts = undefined;
  }

  if (opts !== undefined && opts.allowPrototypes === undefined) {
    // back-compat for qs module
    opts.allowPrototypes = true;
  }

  // 返回query中间件函数
  return function query(req, res, next){
    /**
     * 如果req请求对象中已经包含了query字段，则该中间件不做任何处理，
     * 说明此时req对象已经被处理过了，此时便把req的query控制权交与使用者
     */
    if (!req.query) {
      // 否则，使用parseurl和qs库解析url中的查询参数解析成键值对
      var val = parseUrl(req).query;
      req.query = queryparse(val, opts);
    }

    next();
  };
};
```

- `middleware.init`中间件对`res`和`req`对象做一些初始化配置和挂载一些引用

```js
// 文件在lib/middleware/init.js
/**
 * 初始化中间件，将req和res互相暴露给对象，
 * 并添加一个非标准的X-Powered-By响应头
 *
 * @param {Function} app
 * @return {Function}
 * @api private
 */
exports.init = function(app){
  return function expressInit(req, res, next){
    // 设置非标准的X-powered-by响应头为Express
    if (app.enabled('x-powered-by')) res.setHeader('X-Powered-By', 'Express');
    // req增加res引用
    req.res = res;
    // res增加req引用
    res.req = req;
    req.next = next;

    /**
     * 扩展req和res对象，支持定义在request.js和response.js中的所有功能，
     * 做法是：
     *  - 设置req的原型对象为app.requset
     *  - 设置res的原型对象为app.response
     * 需要注意的是：虽然修改了req和res的原型对象，但是req和res并未丢失原来的原型对象，
     * 原因是在app.request和app.response的实现中是基于正确的原型对象创建的：
     * - var req = Object.create(http.IncomingMessage.prototype)
     * - var res = Object.create(http.ServerResponse.prototype)
     */
    setPrototypeOf(req, app.request)
    setPrototypeOf(res, app.response)

    res.locals = res.locals || Object.create(null);

    next();
  };
};
```

下面我们看下`Router`类的源码实现，`Router`的构造函数和`express`有些类似，都是定义一个兼容中间件函数格式的函数并返回，因为`router`在业务代码中是可以单独导入使用的。这里最主要的是定一个了`stack`属性，用于存放所有的中间件。下面代码中省略了大部分的参数处理代码：

```js
/**
 * 根据options参数初始化路由对象
 * Router类主要用于管理路由，比如路由对应的中间件管理和执行
 */
var proto = module.exports = function(options) {
  var opts = options || {};

  function router(req, res, next) {
    router.handle(req, res, next);
  }

  // 省略部分参数...

  // 存放中间件的集合
  router.stack = [];

  return router;
};


/**
 * use方法添加中间件
 */
proto.use = function use(fn) {
  var offset = 0;
  var path = '/';

  // 省略部分参数的处理，和app.use部分有些类似...

  // 获取path对应的所有中间件
  var callbacks = flatten(slice.call(arguments, offset));

  // 遍历app.use时添加的url对应的中间件集合
  for (var i = 0; i < callbacks.length; i++) {
    var fn = callbacks[i];

    // 对中间件使用Layer类进行包裹一层
    var layer = new Layer(path, {
      sensitive: this.caseSensitive,
      strict: false,
      end: false
    }, fn);

    layer.route = undefined;

    // 将包裹后的中间件添加到中间件栈中（数据结构本质是队列）
    this.stack.push(layer);
  }

  // 返回this，支持链式调用
  return this;
};
```

在`router.use`的实现中，对中间件使用了`Layer`类进行了一层包裹之后才添加到`stack`集合中的。下面我们看`Layer`做了哪些事情：

```js
/**
 * 对中间件进行一次包裹，
 * 添加path、handle等属性，并对path的动态参数进行解析
 */
function Layer(path, options, fn) {
  if (!(this instanceof Layer)) {
    return new Layer(path, options, fn);
  }

  debug('new %o', path)
  var opts = options || {};

  this.handle = fn;
  this.name = fn.name || '<anonymous>';
  this.params = undefined;
  this.path = undefined;
  /**
   * 通过path-to-regexp库解析path中的路由参数
   * 将解析后的参数存放在this.keys中
   */
  this.regexp = pathRegexp(path, this.keys = [], opts);

  // set fast path flags
  this.regexp.fast_star = path === '*'
  this.regexp.fast_slash = path === '/' && opts.end === false
}
```

`Layer`类的主要目的就是对中间件进行一次包裹，包裹成统一的数据格式，并且对`path`的动态参数部分利用`path-to-regexp`解析成键值对集合，方便后续使用。有关`path-to-regexp`的使用和源码解析可以翻阅我的这篇博文 “[面试官：Vue-Router是如何解析URL路由参数的？小明：卒......](https://juejin.cn/post/7077156554721460255)”。

到这里基本上就完成了添加中间件的一整条链路，再梳理一下整条链路图：

![image](https://note.youdao.com/yws/res/22347/48A50F8BD760492C9B057E895DC9FF13)

### 消费中间件原理