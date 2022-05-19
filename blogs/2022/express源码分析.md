# express源码分析

> version 4.17.3
> 愣锤 2022/05/20

[express.js](https://www.expressjs.com.cn/)是一款基于 Node.js 平台，快速、开放、极简的 Web 开发框架。

```js
const Express = require('express');

const app = new Express();

app.get('/', (req, res) => {
  res.send('Hello Express!');
});

app.listen(3000, () => {
  console.log('express server running at port 3000.');
});
```

### 接口组成

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

### app.use()添加中间件原理

`express`很核心的一个功能模型就是中间件，`express`本身更多的是实现了一个中间件架构模型，很多功能都是通过添加中间件来扩展的。使用中间件的方式也很简单，就是通过`app.use()`进行添加：

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

`app.use`所依赖的中间件模型并没有在`express`初始化时创建，而是在使用`app.use`时才`lazy`式的初始化。在分析`app.use`的源码实现之前，我们先看一张相关的代码结构组织图：

![image](https://note.youdao.com/yws/res/22347/48A50F8BD760492C9B057E895DC9FF13)

上图梳理`app.use`所依赖的各个模块的依赖图，从图中可以看到`app.use`是在`application.js`文件中定义的。下面直接看`app.use`的源码实现：

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
     * 处理fn是中间件而不是express应用的情况
     * 注意的是express是支持主子应用的
     * app实例是包含handle方法和set方法的，这里利用了鸭式辨型的思想
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

- `app.use`中首先根据参数情况，获取中间件以及中间件生效的`path`路径
- 利用`this.lazyrouter();`进行初始化路由对象`router`
- 迭代传入的中间件，判断是中间件还是`express`应用
    - 是中间件，则交由`router.use`新增中间件
    - 是`express`应用，则指明父应用和挂载的路径等，同时触发一个子应用挂载的事件。判断是`express`子应用的逻辑是只有该`fn`参数包含`handle`和`set`属性则认为是`express`应用，利用的鸭式辨型的思想。

可以看到`app.use`本身是没有处理中间件逻辑的，只是处理了参数和区分中间件与子应用，最终中间件的处理还是代理到`Router`类了。这里逻辑如下图所示：

![image](https://note.youdao.com/yws/res/22539/B0BCF281FC7E4AB3B97213F923226134)


`router`对象也是通过手动调用`this.lazyrouter`进行`lazy`式的创建的。`lazyrouter`方法的实现也是在`application.js`中，我们看下源码实现：

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

    // layer.route用于标记该layer（中间件）是否是路由处理程序
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


### app\[method\]()添加中间件原理

除了`app.use()`放使用中间件外，我们还可以通过`app[method]()`的方式添加路由中间件，例如下面的使用方式:

```js
const app = express();

// 针对指定path的路由中间件
app.get('/api/v1/user/:id', function() {});
app.post('/api/v1/user', function() {});
```

`app[method]()`的原理究竟是怎样实现的，它又和`app.use()`的实现方式有什么区别呢？接下来我们首先找到`app[method]()`的源码位置，在`application.js`中有这样一段代码：

```js
var methods = require('methods');

/**
 * Delegate `.VERB(...)` calls to `router.VERB(...)`.
 * 将app[method]调用委托到router.route上调用
 */
methods.forEach(function(method){
  app[method] = function(path){
    // 如果是app.get调用且参数只有一个，则作为获取配置方法使用
    if (method === 'get' && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }

    this.lazyrouter();

    // 调用router.route方法获取route对象
    var route = this._router.route(path);
    // 将app[method]方法调用委托到route[method]
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```

这就意味着`application.js`文件加载时通过一个循环给`app`对象上挂载了`get、post、put、delete`等所有的请求方法，[methods](https://github.com/jshttp/methods)库的原理就是通过`http.METHODS`获取所有的请求方法名称的，同时兼容了`Node.js0.10`版本，有兴趣的可以翻着看看。

在`app[method]`的方法内部，主要做了如下几件事情：

- 如果是`get`方法且参数只有一个，则把`app.get`当作获取配置参数的方法使用
- 通过调用`this.lazyrouter();`确保`router`对象被加载
- 调用`router`对象的`route`方法创建一个`route`对象用于当前app[method]行为的委托
- 最后`app[method]`实际调用的是`route[method]`，完成了委托行为

![image](https://note.youdao.com/yws/res/22463/B7429DC9CC734B59A2A40D45177D0068)

接下来我们看`this._router.route(path);`是如何通过调用`router`的`route`方法来创建`route`对象的：

```js
var Route = require('./route');

/**
 * 根据指定的path创建一个新的Route对象
 * 每个route包含一个独立的中间件存储栈
 *
 * @param {String} path
 * @return {Route}
 * @public
 */
proto.route = function route(path) {
  // 初始化一个route对象
  var route = new Route(path);

  /**
   * 初始化一个Layer对象包裹中间件
   * 该Layer绑定的中间件是route.dispatch
   */
  var layer = new Layer(path, {
    sensitive: this.caseSensitive,
    strict: this.strict,
    end: true
  }, route.dispatch.bind(route));

  // 在layer上添加route属性指向route对象
  layer.route = route;

  // 将layer添加到router的栈中
  this.stack.push(layer);

  // 返回的是route
  return route;
};
```

这里可以看到也是首先是实例化`Route`对象，然后实例化`Layer`对象并把`route`对象的`dispatch`方法作为`layer`绑定的中间件，并在得到的`layer`对象对添加`route`属性指向`route`对象，接着后将`layer`对象添加到`router`的栈中，最后返回`route`对象。逻辑图如下：

![image](https://note.youdao.com/yws/res/22514/8FFDD18152A34D5CBB357E45CB3EB3FB)

但是这里有两点要特被注意：

- 第一点是`app[method]`的过程创建了`layer`并添加到了`router`的栈中，`app.use`也创建了`layer`并添加到了`router`的栈中。这样意味着不管是什么类型的中间件都会创建`layer`并添加到`router`栈中

- 第二点是`layer`对象上挂载的`rotue`属性是有值的，指向的就是`route`对象，而`app.use()`过程中创建的`layer`对象`route`属性是`undefined`，这就在`router`栈中区别开了是`app.use`添加的`layer`还是`app[method]`添加的`layer`

- 第三点是创建`layer`对象时把`route`对象的`dispatch`方法绑定为了`layer`的中间件函数，这里了先粗略提一下原因，`app[method]`借助`route[method]`将中间件添加到`route`独立的栈中，`route`对象通过`dispatch`执行所有的中间件。


接下来我们看Route类是什么，它做了那些事情？Route类的源码实现均在`lib/router/route.js`中：

```js
/**
 * 根据指定的path初始化Route
 *
 * @param {String} path
 * @public
 */
function Route(path) {
  this.path = path;
  this.stack = [];

  debug('new %o', path)

  // route handlers for various http methods
  this.methods = {};
}
```

可以看到`Route`类其实就是维护了一个独立的`stack`栈，用于存放属于该`route`的所有中间件。

但是通过上述我们知道，`Route`类上是定义了`get、post`等一系列方法的，这部分是如何实现的呢？在该文件的最后有这样一段源码：

```js
var methods = require('methods');

/**
 * 给Route类定义所有的请求方法
 * 例如：route.get、route.post等
 */
methods.forEach(function(method){
  Route.prototype[method] = function(){
    var handles = flatten(slice.call(arguments));

    // 迭代app[method](path, [...middleware])中所有的中间件添加到route的栈中
    for (var i = 0; i < handles.length; i++) {
      var handle = handles[i];

      // 省略部分参数校验代码

      /**
       * app[method]的调用本质是委托的route[method]
       * 注意：app[method](path, [...middlewares])中所有的中间件都是对该path生效的
       * 所以此处的Layer的path是'/'
       */
      var layer = Layer('/', {}, handle);
      layer.method = method;

      this.methods[method] = true;
      // 将中间件包裹成layer添加到route的栈中
      this.stack.push(layer);
    }

    // 返回this支持链式调用
    return this;
  };
});
```

这里也是一个循环往`Route`原型上挂载了所有的方法，在方法的实现中遍历了所有的传入参数（注意参数都是中间件），然后对每一个中间件调`Layer`进行包裹添加到`route`自己独立的栈中。

因此我们可以梳理出`app[method]`添加中间件的流程图如下所示：

![image](https://note.youdao.com/yws/res/22520/4ED5AF180F3341C39F48239BBF81BE40)

### express中间件总结

- 中间件栈模型总结

通过上述中间件的添加逻辑我们可以知道，不管是路由中间件（`app[method]`得到的`layer<route>`）还是非路由中间件（`app.use`得到的`layer(middleware)`），都是会使用`Layer`类包裹并添加到`router`对象（`app._router`指向的`Router`实例）的栈（`stack`属性）中，只不过区别在于路由中间件的处理程序是交由`route`对象管理的，因此中间件栈的模型如下图所示：

![image](https://note.youdao.com/yws/res/22555/51F33EEE78194BE9945E4DE4F3F418CF)

- router和route的区别

`router`对象是在实例化`express`应用时创建一个，在一个是唯一的，如果创建子应用则在子应用内也会创建一个`router`对象，在每个应用内添加的中间件都会存入隶属于当前应用的`router`实例的`stack`栈中。

`route`对象是在创建路由中间件时，根据当前路由`path`实例化的，它也有一个独立的`stack`栈存储当前路由`path`对应的多个中间件。但是创建路由中间件时会首先在`router`对象的栈中添加一个`layer`，只不过该`layer`的中间件运行后实际是调用对应`route`对象维护的所有中间件。

`route`对象在`express`实例内可以存在多个，个数由路由中间件创面的次数决定。

### 中间件调用原理

众所周知`express`中间件执行逻辑其实就是当接收到请求时依次执行所有匹配的中间件。但是具体的执行逻辑还是有些复杂的，要处理的情况很多，下面先放出来梳理的逻辑图，有了整体的概念之后再理解源码会更顺畅一些：

![image](https://note.youdao.com/yws/res/22583/E187A54F23B446CD8D0E055E7FF7C11F)
![image](https://note.youdao.com/yws/res/22585/564E1E2D7BA84242A1C3DF8AEE3E49A2)

在`app.listen`中的实现我们得知，`express`应用在接收到请求之后触发的是`app`函数调用，而app函数内部又是调用的`app.handle`，代码如下:

```js
var app = function(req, res, next) {
  app.handle(req, res, next);
};
  
app.listen = function listen() {
  // // 此处的this就是app函数
  var server = http.createServer(this);
  return server.listen.apply(server, arguments);
};
```

下面我们看`application.js`中`app.handle`的处理逻辑:

```js
/**
 * 将req和res分发给应用，进行管道式的处理
 *
 * 如果没有calllback提供，那么默认使用finalhandler作为错误处理程序，
 * 并根据中间件栈中出现的错误进行response响应
 * @private
 */
app.handle = function handle(req, res, callback) {
  var router = this._router;

  /**
   * final处理程序
   * 利用finalhandler处理final的情况
   */
  var done = callback || finalhandler(req, res, {
    env: this.get('env'),
    onerror: logerror.bind(this)
  });

  /**
   * 处理空路由情况，路由没有创建则直接给出报错
   * EG：没有使用任何中间件或者路由，则直接响应一个404的Cannot GET的HTML内容
   */
  if (!router) {
    debug('no routes defined on app');
    done();
    return;
  }

  // 将请求交由router系统处理
  router.handle(req, res, done);
};
```

这里只是做的执行中间件的前置操作：

- 自定义默认的错误处理程序，因为需要在中间件执行错误或者没有任何匹配的中间件时响应默认的`404`、`500`等
- 如果此时还没有路由对象的实例，说明没有注册中间件，则无法进行处理，直接调用`done`响应`404`
- 最后将中间件调用执行的逻辑委托给`router`对象执行

想了解[finalhandler](https://github.com/pillarjs/finalhandler#readme)的用法源码分析的可以查看我的这篇文章[《详解《finalhandler》源码中NodeJs服务错误响应原理》](https://juejin.cn/post/7077872017256480781)

有一点必须要提一下有利于后续内容的理解，`finalhandler`是根据响应码、错误状态码等值进行不同的响应内容，**但是在`finalhandler`之前如果已经有响应了，则`finalhandler`不会做任何处理。**

接下来，我们看`router.handle`做了什么事情，下面代码省略部分的参数边界情况的判断，只看核心的中间件部分：

```js
proto.handle = function handle(req, res, out) {
  var self = this;
  var idx = 0;

  // middleware and routes
  var stack = self.stack;

  /**
   * restore利用闭包缓存req上baseUrl、next、params的原始值
   * restore返回的done作用执行后作用就是就是恢复原始值，并调用out，即开始调用下个中间件
   */
  var done = restore(out, req, 'baseUrl', 'next', 'params');
  
  // 在req上挂载调用下一个中间件的引用
  req.next = next;

  next();
  
  function next() {
    // ... 先省略next实现
  }
}
```

首先可以看到主体逻辑就是获取存放所有中间件`layer`的栈，然后调用`next`开始执行中间件。下面我们看`next`内部做了什么事情，下面代码也省略了部分参数的处理情况，只关心核心的中间件部分:

```js
function next(err) {
    var layerError = err === 'route'
      ? null
      : err;

    // 如果错误是'router'则直接done响应404
    if (layerError === 'router') {
      setImmediate(done, null)
      return
    }

    /**
     * 迭代到最后一个中间件了，此时直接done
     * 注意：done内部执行的是finalhandler进行兜底的404返回，
     * 那如果在此前面的中间件已经调用过res.end怎么办？
     * 其实不用担心，因为finalhandler源码中已经判断如果已有响应请求发出则不再继续404响应
     */
    if (idx >= stack.length) {
      setImmediate(done, layerError);
      return;
    }

    // 获取请求的pathname
    var path = getPathname(req);

    // 无效的请求路径时直接done
    if (path == null) {
      return done(layerError);
    }

    var layer; // 当前包裹中间件的layer
    var match; // 当前中间件是否和请求路径匹配
    var route; // 是否是路由中间件，指向路由中间件的引用

    // 依次迭代栈中的layer<middleware>
    while (match !== true && idx < stack.length) {
      layer = stack[idx++];
      // 判断当前中间件是否和path匹配
      match = matchLayer(layer, path);
      route = layer.route;

      // match不是布尔类型时，说明matchLayer解析出错了
      if (typeof match !== 'boolean') {
        /**
         * 如果之前的中间件已经有错误产生了，则依旧使用之前的错误，否则使用matchLayer的错误
         * 此举保证将第一个出错的layer产生的错误传递到最后返回
         */
        layerError = layerError || match;
        // 其实这里直接continue就可以了
      }

      // 当前path和layer（中间件）不匹配，则跳过当前layer
      if (match !== true) {
        continue;
      }

      /**
       * 如果是非路由中间件则直接跳出while循环开始后续的调用逻辑，
       * 因为非路由中间件不需要检查方法类型
       * 下文都是针对路由中间件的方法类型是否匹配的判断
       */
      if (!route) {
        continue;
      }

      /**
       * 如果存在错误，则跳过当前中间件
       * 因为一旦产生错误，后续的中间件都不需要执行了，而是一直把错误往后传递，
       * 这样就要求错误处理中间件必须在最后一个中间件
       */
      if (layerError) {
        // routes do not match with a pending error
        match = false;
        continue;
      }

      /**
       * 下面一小段逻辑主要是处理当前路由中间件是否能匹配上实际的请求方法，
       * 如果无法匹配则跳到下一个中间件
       * 例如，req.method是get，但是只定义了app.post的中间件，那么是无法匹配的
       */
      var method = req.method;
      var has_method = route._handles_method(method);

      // build up automatic options response
      if (!has_method && method === 'OPTIONS') {
        appendMethods(options, route._options());
      }

      // don't even bother matching route
      if (!has_method && method !== 'HEAD') {
        match = false;
        continue;
      }
    }

    /**
     * 整个while下来之后match不是true，说明没有任何匹配的layer，
     * 没有任何layer与path能匹配，则直接结束本次请求
     */
    if (match !== true) {
      return done(layerError);
    }

    // this should be done for the layer
    self.process_params(layer, paramcalled, req, res, function (err) {
      /**
       * 如果产生错误了，则直接next往后跑，并将错误传递下去
       */
      if (err) {
        return next(layerError || err);
      }

      /**
       * 如果是路由中间件，则直接调用路由中间件
       * 这里直接调用就可以的原因是：
       *  - 路由中间件只是桥接，其真正的中间件在route对象的stack中
       *  - 这里layer.handle_request调用中间件也只不过是触发的route.dispatch()
       *  - 调用route.dispatch()后才是依次执行route.stack中的中间件
       */
      if (route) {
        return layer.handle_request(req, res, next);
      }

      // 校验参数后再调用中间件
      trim_prefix(layer, layerError, layerPath, path);
    });
}
```

`next()`的逻辑是中间件执行最核心的实现，整体逻辑如下：

- 首先判断中间件执行是否存在错误，如果存在错误且错误为`router`，则直接`done`响应`404`，停止后续的中间件执行。注意初次`next`调用的错误默认不存在

- 如果已经迭代完栈中所有的中间件，则直接`done`响应，响应的内容由中间件执行的错误决定，如果已经有中间件响应了，`done`本身不会做任何处理，这个在上面已经提到了

- 如果请求路径不存在，也直接调用`done`进行错误响应

- 紧接着利用`while`循环开始迭代中间件
    - 判断中间件是否和当前请求路径匹配，如果不匹配则继续迭代下一个中间件
    - 如果是非路由中间件则直接跳出`while`循环开始后续的调用逻辑，因为非路由中间件不需要检查方法类型,下文都是针对路由中间件的方法类型是否匹配的判断
    - 如果存在错误，则跳过当前中间件。因为一旦产生错误，后续的中间件都不需要执行了，而是一直把错误往后传递，这样就要求错误处理中间件必须在最后一个中间件

- 如果整个`while`下来之后没有匹配到中间件，则直接`done`响应错误处理

- 如果是路由中间件，则直接调用路由中间件。
- 如果是非路由中间件，则先调用`trim_prefix`进行参数校验
   - 路径校验不通过直接`done`响应错误
    - 校验通过，根据是否已有错误产生决定进行普通中间件调用还是错误中间件调用

下面看中间件调用的逻辑吧：

```js
function trim_prefix(layer, layerError, layerPath, path) {
  if (layerPath.length !== 0) {
    // ...省略路径参数校验不通过直接done的部分
  }

  /**
   * 判断有没有错误存在
   * - 有错误则调用handle_error方法处理
   * - 没有错误则调用handle_request方法处理
   */
  if (layerError) {
    layer.handle_error(layerError, req, res, next);
  } else {
    layer.handle_request(req, res, next);
  }
}
```

这部分可以看到针对非路由中间件的处理就是判断是否已存在错误，存在的化调用`handle_error`处理，不存在的话调用`handle_request`处理。下面我们看`handle_error`内部做的什么：

```js
/**
 * 处理layer包裹的错误处理中间件调用
 */
Layer.prototype.handle_error = function handle_error(error, req, res, next) {
  var fn = this.handle;

  /**
   * 如果传入的中间件处理程序的形参个数不是4，说明不是标准的错误处理中间件格式
   * 此时则把它作为普通中间件处理，直接调用next(error)将错误传递下去
   */
  if (fn.length !== 4) {
    return next(error);
  }

  try {
    // 调用错误处理中间件
    fn(error, req, res, next);
  } catch (err) {
    // 如果错误处理中间件调用出错了，则将错误继续往后传递
    // 直到触发最后的done的error响应
    next(err);
  }
};
```

这里的做法是如果有错误，但是该中间件却不是错误处理中间件，则直接`next(error)`将错误继续传递下去，是错误处理中间件则调用错粗处理中间件。然后`next`调用的控制权则移交给中间件内部控制，如果中间件执行出错，比如主动抛出错误的形式，则利用`catch`捕获后用`next`传递下去。

补充：`express`有普通中间件和错误处理中间件，两者参数格式不一样
- 普通中间件形参个数为`3`， `function middleware(req, res, next) {}`
- 错误处理中间件形参个数为`4`， `function errorMiddleware(err, req, res, next) {}`

`handle_request`的处理方式也类似，代码如下所示，就不过多讲解了：

```js
/**
 * 处理layer包裹的中间件调用
 */
Layer.prototype.handle_request = function handle(req, res, next) {
  var fn = this.handle;

  /**
   * 如果用户的中间件参数个数大于3，说明不是标准的中间件程序
   * 则直接next()忽略，进入到下一个中间件的处理中
   */
  if (fn.length > 3) {
    return next();
  }

  try {
    /**
     * 执行当前中间件，
     * 然后在中间件内部由中间件控制next调用跳到下一步
     */
    fn(req, res, next);
  } catch (err) {
    // 如果执行中间件的过程中出错了，调用next(err)将错误传递下去
    // 比如中间件会在错误的时候throw Error出来
    next(err);
  }
};
```

至此，一个中间件调用流程就结束了，但是有个很重要的细节大家可能注意到了，就是路由中间件（`route`对象）其实只是一个路由中间件集合的调用桥梁，路由中间件`layer.handle`绑定的中间件只是`route.dispatch`方法，真正的路由中间件执行其实是`route.dispatch`逻辑。

下面我们来看看`route.dispatch`到底做的什么事情：

```js
/**
 * 将req和res分发给route执行
 * @private
 */
Route.prototype.dispatch = function dispatch(req, res, done) {
  var idx = 0;
  var stack = this.stack;
  // 不存在路由中间件，则直接next到上层的router.stack中的下一个中间件
  if (stack.length === 0) {
    return done();
  }

  var method = req.method.toLowerCase();
  if (method === 'head' && !this.methods['head']) {
    method = 'get';
  }

  req.route = this;

  next();

  function next(err) {
    // signal to exit route
    if (err && err === 'route') {
      return done();
    }

    // signal to exit router
    if (err && err === 'router') {
      return done(err)
    }

    // 如果已经迭代完毕，则next到上层的router.stack中的下一个中间件
    var layer = stack[idx++];
    if (!layer) {
      return done(err);
    }

    // 如果请求类型不匹配，则执行下一个路由中间件
    if (layer.method && layer.method !== method) {
      return next(err);
    }

    // 根据有无错误进行不同的处理
    if (err) {
      layer.handle_error(err, req, res, next);
    } else {
      layer.handle_request(req, res, next);
    }
  }
};
```

`route.dispatch`的逻辑主要如下：

- 如果`route.stack`中不存在路由中间件，则直接next到上层的router.stack中的下一个中间件
- 迭代`route.stack`中的所有路由中间件
    - 存在错误为`route`，直接`next()`到上层`router.stack`中的下一个中间件
    - 存在错误为`router`，直接`next(err)`到上层`router.stack`中的下一个中间件
    - 如果已经迭代完毕，则`next(err)`到上层的`router.stack`中的下一个中间件
    - 如果请求类型与当前中间件不匹配，则执行下一个路由中间件
    - 根据有无错误进行不同的处理

至此一个完成的`express`中间件执行的完整闭环原理就讲完了。下面我们将抽离一个最最基本的`express`中间件执行逻辑，实现最基本的功能模型。

### 简易Express实现

实现一个简化版的Express，支持`app.use()`中间件模型，代码如下：

```js
const http = require('http');
const finalhandler = require('finalhandler');

class Express {
  use(path, handler) {
    if (!arguments.length) {
      throw Error('miss arguments');
    }
    if (arguments.length === 1) {
      handler = path;
      path = '/';
    }
    if (!this.router) {
      this.router = new Router();
    }
    this.router.use(path, handler);
  }

  listen() {
    const server = http.createServer(this._handle.bind(this));
    return server.listen.apply(server, arguments);
  }

  _handle(req, res) {
    const done = finalhandler(req, res);
    if (!this.router) {
      done();
      return;
    }
    this.router.handle(req, res, done);
  }
}

class Router {
  constructor() {
    this.stacks = [];
  }

  use(path, handler) {
    const layer = new Layer(path, handler)
    this.stacks.push(layer);
  }

  handle(req, res, done) {
    let index = 0;
    const stacks = this.stacks;
    const self = this;

    next();

    function next(error) {
      // 迭代完所有中间件后执行done逻辑
      if (index >= stacks.length) {
        done(error);
        return;
      }

      let layer;
      let isMatch;

      while(!isMatch && index < stacks.length) {
        layer = stacks[index++];
        isMatch = self.matchMiddleware(req.url, layer.path);
      }

      // 迭代完发现没有任何匹配的中间件则直接done
      if (!isMatch) {
        done(error);
        return;
      }

      // 调用中间件处理函数
      if (error) {
        layer.handleError(error, layer.handle, req, res, next);
      } else {
        layer.handleRequest(layer.handle, req, res, next);
      }
    };
  }

  // 最基本的中间件是否匹配的逻辑
  matchMiddleware(url, path) {
    return url.slice(0, path.length) === path;
  }
}

class Layer {
  constructor(path, fn, ops) {
    this.path = path;
    this.handle = fn;
    this.ops = ops || {};
  }

  // 调用错误处理中间件
  handleError(error, fn, req, res, next) {
    // 如果不是错误处理中间件则跳过
    if (fn.length !== 4) {
      next();
      return;
    }
    try {
      fn(error, req, res, next);
    } catch (error) {
      next(error);
    }
  }

  // 调用请求处理中间件
  handleRequest(fn, req, res, next) {
    // 如果不是普通中间件则跳过
    if (fn.length !== 3) {
      next();
      return;
    }
    try {
      fn(req, res, next);
    } catch (error) {
      next(error);
    }
  }
}
```

有了上面的实例，我们可以运行一个`demo`实例查看一下效果验证中间件的基本使用：

```js
const app = new Express();

app.use('/', (req, res, next) => {
  req.reqTime = Date.now().toString();
  next();
});

app.use('/a', (req, res, next) => {
  res.end(req.reqTime);
  next();
});

app.use('/', (req, res, next) => {
  req.reqTime = Date.now().toString();
  next();
});

app.use('/b', (req, res, next) => {
  throw Error('/b error');
});

app.use('/', (error, req, res, next) => {
  res.writeHead(error.status || 500);
  res.end('server error');
});

app.listen(9669, () => {
  console.log('express is running at port 9669');
});
```

# response相关API实现

在req和res对象的扩展上，express提供了很多实用的方法。代码将对部分实用api的实现进行详细的分析。

### res.status()实现

`res.status()`方法主要用于设置http响应对象的响应状态码。

```js
/**
 * 设置http的响应码
 *
 * @param {Number} code
 * @return {ServerResponse}
 * @public
 */
res.status = function status(code) {
  // this就是res对象
  this.statusCode = code;
  // 返回res支持链式调用
  return this;
};
```

`res.status()`方法定义在`response.js`中，实现比较简单，就是直接给`http`响应对象`response`设置`statusCode`。但是要注意的是为什么此处的`this`指代的是响应对象呢？

我们从`express`的`createApplication`实现中得知，只是在初始化的时候给`app`函数对象上挂载了`response.js`导出的对象上的内容，那这里`this`也不应该指向`res`对象，而是应该指向`app`对象呀。所以，为什么我们可以使用`res`对象上使用`status()`方法呢?

原因在于初始化中间件的时候给res对象做了扩展功能，我们回顾下`express`内置的`init`中间件的核心逻辑：

```js
function expressInit(req, res, next){
    // ...

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

    // ...
  };
```

从这里可以看到我们在默认中间件执行的时候给`req`和`res`对象进行了功能扩展，将`request.js`和`response.js`对象上的所有方法扩展到了对应的`req`和`res`对象。


### res.set()和res.get()实现

`res.set()`用于设置响应对象的响应头的字段，比如设置`Content-Type`等等，而`res.get()`则是从响应头获取相关的字段值。先看下`res.set()`的具体实现如下:

```js
/**
 * 设置header字段
 * Examples:
 *    res.set('Foo', ['bar', 'baz']);
 *    res.set('Accept', 'application/json');
 *    res.set({ Accept: 'text/plain', 'X-API-Key': 'tobi' });
 * Aliased as `res.header()`.
 * @param {String|Object} field
 * @param {String|Array} val
 * @return {ServerResponse} for chaining
 * @public
 */
res.set =
res.header = function header(field, val) {
  if (arguments.length === 2) {
    var value = Array.isArray(val)
      ? val.map(String)
      : String(val);

    // 如果是content-type字段，则特殊处理
    if (field.toLowerCase() === 'content-type') {
      // content-type不允许是数组
      if (Array.isArray(value)) {
        throw new TypeError('Content-Type cannot be set to an Array');
      }

      // 如果content-type中没有指定charset编码，则添加charset编码
      if (!charsetRegExp.test(value)) {
        var charset = mime.charsets.lookup(value.split(';')[0]);
        if (charset) value += '; charset=' + charset.toLowerCase();
      }
    }

    // 调用http server的response对象的setHeader方法更新header字段
    this.setHeader(field, value);
  } else {
    for (var key in field) {
      this.set(key, field[key]);
    }
  }
  return this;
};
```

`res.set()`的核心实现就是根据`key/value`，调用`node`原生语法`http`的`response`对象的`setHeader`方法设置请求头字段。唯一注意点是对`Content-Type`进行了特殊处理：如果`Content-Type`没有设置`charset`字符集，则利用`mime`库获取对应的字符集后添加上。

`res.get()`获取指定的响应头字段值的方法就比较简单了，直接利用`http`响应对象的`getHeader`方法获取:

```js
/**
 * 获取响应头指定字段的值
 *
 * @param {String} field
 * @return {String}
 * @public
 */
res.get = function(field){
  return this.getHeader(field);
};
```

### res.send()实现

`res.send()`是一个很重要的api，主要用于发送响应数据，例如`res.status(200).send('Hello world!')`先设置响应状态码为`200`然后响应数据为`Hello world`。下面我们看其内部是如何实现数据响应的：

```js
/**
 * 发送http响应
 *
 * Examples:
 *     res.send(Buffer.from('wahoo'));
 *     res.send({ some: 'json' });
 *     res.send('<p>some html</p>');
 * @param {string|number|boolean|object|Buffer} body
 * @public
 */
res.send = function send(body) {
  var chunk = body;
  var encoding;
  var req = this.req;
  var type;

  // settings
  var app = this.app;
 
  /**
   * 处理参数，兼容res.send()方法两个参数的旧格式写法
   * 使用旧格式写法时会给出使用新语法的提示
   */
  if (arguments.length === 2) {
    // res.send(body, status) backwards compat
    if (typeof arguments[0] !== 'number' && typeof arguments[1] === 'number') {
      deprecate('res.send(body, status): Use res.status(status).send(body) instead');
      this.statusCode = arguments[1];
    } else {
      deprecate('res.send(status, body): Use res.status(status).send(body) instead');
      this.statusCode = arguments[0];
      chunk = arguments[1];
    }
  }
  
  // disambiguate res.send(status) and res.send(status, num)
  if (typeof chunk === 'number' && arguments.length === 1) {
    // 如果没有设置Content-Type，res.send(status)则默认使用text/plain
    if (!this.get('Content-Type')) {
      this.type('txt');
    }

    // 对res.send(status)的调用给出新语法提示
    deprecate('res.send(status): Use res.sendStatus(status) instead');
    this.statusCode = chunk;
    // status对应的message
    chunk = statuses[chunk]
  }
  
  // ...省略
}
```

这里可以看到首先是根据send的参数格式和类型，做了旧语法的兼容，比如旧的语法`res.send(status)`、`res.send(status, body)`、`res.send(body, status)`。

处理完参数之后，紧接着就该对响应内容进一步处理了，代码如下所示：

```js
// 根据res.send()参数的类型做不同的处理
switch (typeof chunk) {
  case 'string':
    // 对应send内容为文本且没有设置Content-Type时默认使用text/html
    if (!this.get('Content-Type')) {
      this.type('html');
    }
    break;
  case 'boolean':
  case 'number':
  case 'object':
    if (chunk === null) {
      chunk = '';
    } else if (Buffer.isBuffer(chunk)) {
      // 如果是buffer数据且没有设置Content-Type，则默认使用application/octet-stream
      if (!this.get('Content-Type')) {
        this.type('bin');
      }
    } else {
      // 如果是布尔、数值、或者对象（非null非二进制流），则交由json方法处理
      // json方法处理完之后最后还是再次调用this.send以字符串的方式处理
      return this.json(chunk);
    }
    break;
}
```

这里根据send的参数数据进行不同的处理：

- 如果是字符串且没有设置`Content-Type`则默认设置为`text/html`
- 如果是null，则响应的数据为空字符串
- 如果是Buffer数据且没有主动设置`Content-Type`则默认设置为`application/octet-stream`
- 其他类型都交由`this.json()`序列化转成字符串之后再次调用`this.send()`进行处理。

接下来继续往后面看，如果是字符串，或者经由`this.json()`序列化成字符串之后是如何处理的：

```js
/**
 * 响应的数据是字符串时，指定字符编码为utf8
 * 且对已设置的Content-Type进行容错处理，确保有charset编码
 */
if (typeof chunk === 'string') {
  encoding = 'utf8';
  type = this.get('Content-Type');

  // reflect this in content-type
  if (typeof type === 'string') {
    this.set('Content-Type', setCharset(type, 'utf-8'));
  }
}
```

通过前面逻辑得知代码能走到这里，首先可以确定的是已经设置好了`Content-Type`，这时候就是拿到设置好的`Content-Type`数据再次进行容错处理，调用`setCharset`确保`Content-Type`设置了`charset`编码格式。

我们知道`res.send()`是自动帮我们计算了`Content-Length`的，因此接下来的实现逻辑就是计算`Content-Length`了：

```js
/**
 * etag的生成函数
 * - express的初始化工作时，调用了app.set('etag')
 * - 所以etagFn的生成函数时默认存在的，详细内容可以再翻看初始化时的app.set('etag')部分
 */
var etagFn = app.get('etag fn')
/**
 * 是否要生成ETag响应头
 * - 如果响应头中不包含ETag字段且上面的etagFn存在的话，则应该生成ETag
 * - 默认etagFn存在
 */
var generateETag = !this.get('ETag') && typeof etagFn === 'function'

// 生成Content-Length数据
var len
if (chunk !== undefined) {
  if (Buffer.isBuffer(chunk)) {
    // 如果是buffer数据，则直接获取buffer长度作为Content-Length
    len = chunk.length
  } else if (!generateETag && chunk.length < 1000) {
    // just calculate length when no ETag + small chunk
    len = Buffer.byteLength(chunk, encoding)
  } else {
    // 将字符串转换成buffer数据，再计算Content-Length
    chunk = Buffer.from(chunk, encoding)
    encoding = undefined;
    len = chunk.length
  }

  // 设置Content-Length值
  this.set('Content-Length', len);
}
```

这里先不看`ETag`的部分，先看len计算逻辑：

- 如果`chunk`是`Buffer`类型数据，则直接使用`chunk.length`获取`buffer`长度
- 如果是小字符串，且也不需要转换成`buffer`以及生成`ETag`则直接通过`Buffer.byteLength`获取字符串的字节长度
- 如果是大字符串则通过`Buffer.from`转换成`buffer`数据再获取长度

这里有个注意点是，对于小于`1000`字节的字符串数据，没有转换成`buffer`进行传输，因为不仅要考虑到传输的宽度和速率问题，还要考虑收到数据后的编解码的消耗，要做一个权衡考虑。

通过上面的逻辑，`generateETag`变量用于判断当前响应是否要生成`ETag`响应头，判断逻辑是如果响应头没有保护`ETag`字段且`app`设置中存在`ETag`的创建函数，则值为`true`表示需要生成`ETag`。接下来我们看是如何生成`ETag`的:

```js
// 如果需要生成ETag，则创建ETag并添加到响应头中
var etag;
if (generateETag && len !== undefined) {
  if ((etag = etagFn(chunk, encoding))) {
    this.set('ETag', etag);
  }
}
```

这里是调用`etagFn`函数来创建`ETag`，`etagFn`来自于`app`的设置，一开始应用初始化时默认添加了`etagFn`函数，该函数背后创建`ETag`的逻辑是调用[etag](https://github.com/jshttp/etag#readme)库实现的，这里不再继续展开。继续往后看：

```js
// 缓存尚未过期，则直接304不返回响应体
if (req.fresh) this.statusCode = 304;

/**
 * 响应码为204和304时移除内容相关的响应头``
 * - 204表示没有内容
 * - 304表示服务器资源没有变化，无需再次传输资源
 */
if (204 === this.statusCode || 304 === this.statusCode) {
  this.removeHeader('Content-Type');
  this.removeHeader('Content-Length');
  this.removeHeader('Transfer-Encoding');
  chunk = '';
}
```

这里通过`req.fresh`判断当前缓存数据是否未过期，如果未过期则设置响应状态码为`304`，然后针对`204`状态码（没有响应内容）和`304`状态码（资源未变化不需要重复传输）则直接移除响应的`Content-Type`等字段，并把响应内容设置为空。`req.fresh`的逻辑放在解析请求对象的时候讲解。

```js
if (req.method === 'HEAD') {
 /**
  * 处理完请求头相关设置后，
  * 如果是HEAD请求则直接end()，不响应任何实体，只包含了响应头
  */
  this.end();
} else {
  // 响应数据
  this.end(chunk, encoding);
}

return this;
```

代码到这里，`res.send()`就结束了，最后支持了`HEAD`类型的请求，对于`HEAD`请求只返回响应头数据，不返回响应实体数据。否则的话则直接调用`http response`对象的`end()`方法响应数据对象。

**划重点！划重点！！划重点！！！**

我们最后总结一下`res.send()`主要做了哪些事情，也就是一个`web`框架对响应数据的封装要做哪些事情：

- 根据响应的数据类型指定不同的`Content-Type`
- 计算好响应对象的`Content-Length`
- 支持`ETag`缓存是否过期，防止不必要的数据传输
- 对于`204`和`304`响应不做实体数据的响应
- 要支持`HEAD`请求

### res.sendFile()响应文件原理

`res.sendFile()`方法作用主要用于根据指定的文件路径响应给接收端资源文件。下面我们看是如何实现静态文件托管服务的：

```js
var send = require('send');

/**
 * 根据给定的path响应指定的资源文件
 * @public
 */
res.sendFile = function sendFile(path, options, callback) {
  var done = callback;
  var req = this.req;
  var res = this;
  var next = req.next;
  var opts = options || {};
  
  // 省略部分参数处理和类型校验...
  
  // 编码路径
  var pathname = encodeURI(path);
  // 创建文件的可读流
  var file = send(req, pathname, opts);
  
  // 利用sendfile函数响应资源文件
  sendfile(res, file, opts, function (err) {
    if (done) return done(err);
    if (err && err.code === 'EISDIR') return next();

    // next() all but write errors
    if (err && err.code !== 'ECONNABORTED' && err.syscall !== 'write') {
      next(err);
    }
  });
}
```

首先我们要了解一下`send`库，该库主要作用就是读取指定路径得到文件流，但是我们可以指定各种资源的处理逻辑和各个生命周期钩子的处理逻辑，非常灵活。想了解send库的使用和原理实现可以阅读我的这篇文章《[详解《send》源码中NodeJs静态文件托管服务实现原理](https://juejin.cn/post/7076093974938648612)》。

利用`send`库读取指定的文件得到文件流后，我们看下是如何处理流的，也就是这里的`sendfile`函数逻辑：

```js
/**
 * pipe的方式发送文件流
 */
function sendfile(res, file, options, callback) {
  var done = false;
  var streaming;
  
  // ...省略部分钩子函数，后面介绍
  
  file.on('directory', ondirectory);
  file.on('end', onend);
  file.on('error', onerror);
  file.on('file', onfile);
  file.on('stream', onstream);
  // res响应结束（完成、关闭、出错）之后，调用onfinish
  // 注意响应结束或者流读取出错之后，文件流会在send库内部被销毁，不需要手动销毁
  onFinished(res, onfinish);
  
  // 如果传递了响应头参数，则在流读取完成后更新响应头字段
  if (options.headers) {
    // set headers on successful transfer
    file.on('headers', function headers(res) {
      var obj = options.headers;
      var keys = Object.keys(obj);

      for (var i = 0; i < keys.length; i++) {
        var k = keys[i];
        res.setHeader(k, obj[k]);
      }
    });
  }

  // pipe方式将文件流响应给接收端
  file.pipe(res);
}
```

这里的主要逻辑是拿到调用`send`得到实例之后，监听一系列事件，最后将读取的流以`pipe`的形式响应给接收端。这些事件如下：

- `directory`事件表示读取的路径是一个文件夹
- `end`事件表示流读取结束了
- `error`事件表示流读取出错了
- `file`事件表示读取的是一个文件
- `stream`事件是在通过fs模块创建了可读流之后触发
- `headers`事件是在确定了路径能映射到资源之后触发，可以在此阶段添加响应头字段

接下来看各个事件到底做了什么事情？

```js
// 读取的资源是文件夹时的钩子
function ondirectory() {
  if (done) return;
  done = true;

  // 创建一个目标是文件夹的错误，e is dir
  var err = new Error('EISDIR, read');
  err.code = 'EISDIR';
  callback(err);
}
  
// 读取文件流出错的钩子
function onerror(err) {
  if (done) return;
  done = true;
  // 流出错时直接调用callback并传递错误
  callback(err);
}

// 读取文件流结束的钩子
function onend() {
  if (done) return;
  done = true;
  // 读取结束，调用callback
  callback();
}

// 读取的资源是文件的钩子
function onfile() {
  streaming = false;
}
  
// 开始读取流的钩子
function onstream() {
  streaming = true;
}
```

首先所有的钩子都判断了done的状态，也就是只要有兜底的操作处理过了就不再重复处理了。`ondirectory`说明读取的是个文件夹则直接创建一个`EISDIR`类型的错误，`onerror`说明出错了则直接把错粗传给回调函数，`onend`说明正常读取结束没有错误直接调用回调。`onfile`和`onstream`就是打标记当前流的读取状态，是未读还是开始读了。

```js
// res响应结束（完成、关闭、出错）的钩子
function onfinish(err) {
  // 客户端意外断开连接
  if (err && err.code === 'ECONNRESET') return onaborted();
  // 流读取或响应出错
  if (err) return onerror(err);
  // 如果已处理过结束状态，则不再做处理
  if (done) return;

  // 如果res响应结束了，但是还没有处理过callback，说明可能响应出现了意外
  setImmediate(function () {
    // 如果此时流还不处理结束的状态，则说明是连接意外关闭了，则直接onaborted
    if (streaming !== false && !done) {
      onaborted();
      return;
    }

    // 如果已经处理过了则不再继续处理
    if (done) return;
    done = true;
    // 否则调用callback
    callback();
  });
}

// 请求终止
function onaborted() {
  if (done) return;
  done = true;

  // 创建一个请求终止的错误，ECONNABORTED一般表示对方意外关闭了套接字
  var err = new Error('Request aborted');
  err.code = 'ECONNABORTED';
  // 调用callback并传入一个请求意外终止的错误
  callback(err);
}
```

我们重点看`onfinish`钩子的逻辑，也就是`res`响应结束（包括响应完成、关闭和出错）之后做了什么事情。

首先能走到`onfinish`逻辑说明`res`响应已经结束了，但是结束有可能是出错了、也有可能是正常结束。因此判断意外断开连接的情况创建一个`ECONNRESET`错误并调用`callback`，出错的情况则是的调用`callback`并传递错误。如果没出错则判断有没有已经调用过`callback`的操作了，有的话不做任何处理，没有的话则判断流的状态进行一些处理。

监听`res`响应结束逻辑是用的`on-finished`库，想了解该库的使用和实现原理的可以阅读我的这篇文章《[小而美的《on-finished》源码全解析](https://juejin.cn/post/7087741461990473735)》。


### res.render()模板渲染原理

利用`res.render()`可以渲染指定的视图，例如我们在访问`/`路由时返回`index.jade`的视图模板，使用代码如下：

```js
const express = require('express');
const router = express.Router();

const app = express();

/**
 * 设置视图引擎
 */
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'jade');

/* GET home page. */
router.get('/', function(req, res, next) {
  // 返回src/views/index.jade模板
  res.render('index', {
    title: 'Express',
  });
});
```

下面看`res.render()`内部是如何实现的：

```js
// application.js中
res.render = function render(view, options, callback) {
  var app = this.req.app;
  var done = callback;
  var opts = options || {};
  
  // 省略参数处理部分...

  // 响应后的callback
  done = done || function (err, str) {
    if (err) return req.next(err);
    self.send(str);
  };
  
  // 调用渲染方法
  app.render(view, opts, done);
}
```

可以看到`res.render()`渲染方法最终是调用的`app.render()`方法，并且传入指定的视图模板路径、选项参数和`done`的逻辑。下面看`app.render()`方法实现：

```js
/**
 * 根据模板名称渲染对应的视图
 * application.js文件
 */
app.render = function render(name, options, callback) {
  var cache = this.cache;
  var done = callback;
  var engines = this.engines;
  var opts = options;
  var renderOptions = {};
  var view;
  
  // 省略参数合并/处理代码...
  
  // 如果允许使用缓存则取缓存模板
  if (renderOptions.cache) {
    view = cache[name];
  }
  
  // 实例化视图
  if (!view) {
    // 暂时省略实例化view的代码
  }
  
  // 调用tryRender方法渲染
  tryRender(view, renderOptions, done);
}

function tryRender(view, options, callback) {
  try {
    view.render(options, callback);
  } catch (err) {
    callback(err);
  }
}
```

这里可以看到主要就是判断要不要使用缓存的视图，然后根据需要实例化视图，调用视图实例的`render`方法进行渲染。接下来看是如何实例化视图的：

```js
// 上述代码的if部分
if (!view) {
  // 获取View类
  var View = this.get('view');
  // 实例化View类
  view = new View(name, {
    defaultEngine: this.get('view engine'),
    root: this.get('views'),
    engines: engines
  });
  
  // 路径不存在则直接报错
  if (!view.path) {
    var err = new Error('...省略');
    err.view = view;
    return done(err);
  }
  
  // 创建缓存
  if (renderOptions.cache) {
    cache[name] = view;
  }
}
```

这里就是获取`View`类，然后实例化，然后将实例添加到缓存中。`View`类的来源是在我们`express`初始化配置的时候通过`this.set('view', View)`添加的类，其实现在`lib/view.js`中：

```js
/**
 * 根据name初始化一个视图
 * @param {string} name
 * @param {object} options
 * * Options:
 *   - `defaultEngine` 默认的模板引擎
 *   - `engines` 所有加载的模板引擎
 *   - `root` 视图模板的跟路径
 * @public
 */
function View(name, options) {
  var opts = options || {};

  // 默认的模板引擎
  this.defaultEngine = opts.defaultEngine;
  // 文件的后缀名
  this.ext = extname(name);
  // 模板文件名称
  this.name = name;
  // 模板的根路径
  this.root = opts.root;
  
  var fileName = name;

  if (!this.ext) {
    // 根据扩展视图引擎获取模板文件的后缀名
    // 例如根据jade引擎获取的是.jade
    this.ext = this.defaultEngine[0] !== '.'
      ? '.' + this.defaultEngine
      : this.defaultEngine;
    // 完整的文件名，name.[ext]
    fileName += this.ext;
  }
  
  // ... 省略代码
}
```

首先这里根据传入的视图文件名称，尝试获取其文件m名和后缀名，如果不存在后缀名则利用依赖的模板引擎尝试获取，比如指定的`jade`引擎，则对应获取`name.[ext]`文件名和后缀名。

```js
// 如果引擎还没有被加载，则调用require加载引擎
if (!opts.engines[this.ext]) {
  // 引擎名称
  var mod = this.ext.substr(1)

  // 加载对应的模板引擎
  var fn = require(mod).__express

  // 将模板引擎添加到engines缓存中
  opts.engines[this.ext] = fn
}

// 存储当前加载的引擎
this.engine = opts.engines[this.ext];

// 查找路径对应的文件，用于判断资源是否存在
this.path = this.lookup(fileName);
```

紧接着可以看到就是判断引擎有没有加载，没有加载则利用`require(引擎名)`加载模板引擎，加载后放在`opts.engines`缓存中，防止后续重复加载引擎。最后判断当前视图路径是否对应资源存在。

而我们前面指定，渲染视图的逻辑是拿到`View`类的实例后调用的其`render`方法，那么我们看下`View`类实例的`render`方法逻辑：

```js
/**
 * 调用渲染引擎
 * @param {object} options
 * @param {function} callback
 * @private
 */
View.prototype.render = function render(options, callback) {
  debug('render "%s"', this.path);
  this.engine(this.path, options, callback);
};
```

这里就比较简单了，其实就是调用的刚才加载的视图引擎。最后总结一下整体的渲染逻辑图如下所示：

![image](https://note.youdao.com/yws/res/23703/EB937578A7464BFA9AE49C5EAE0678A0)

### res.attachment()附件下载原理

如果在`http`响应中，需要接收端对响应资源进行附件下载并保存到本地的话，需要设置响应头的`Content-Disposition`字段，这块不清楚的话可以[查阅资料](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Disposition)。在`express`中则可以通过`res.attachment()`快速的实现该功能。接下来我们看其内部实现：

```js
var contentDisposition = require('content-disposition');

/**
 * 指示接收端将响应资源以附件形式下载并保存到本地
 * @param {String} filename
 * @return {ServerResponse}
 * @public
 */
res.attachment = function attachment(filename) {
  // 根据文件类型设置Content-Type
  if (filename) {
    this.type(extname(filename));
  }

  // 利用contentDisposition库生成响应头的Content-Disposition字段值
  // 从而支持附件下载
  this.set('Content-Disposition', contentDisposition(filename));

  return this;
};
```

可以看到内部就是通过`content-disposition`库实现的。那么为什么不直接自己拼接字符串呢？因为要考虑到大量的字符编码的问题。对`content-disposition`库源码实现有兴趣的话，可以查阅我的这篇博文[《详解Content-Disposition源码中Node附件下载服务原理》](https://juejin.cn/post/7077134813068525582)。

# request相关API实现

### req.get()实现

```js
/**
 * 获取指定的请求头字段值
 *
 * - `Referrer`和`Referer`都是一样的，两者是可互相替换的
 *
 * @param {String} name
 * @return {String}
 * @public
 */
req.get =
req.header = function header(name) {
  // 省略参数校验部分...

  var lc = name.toLowerCase();

  switch (lc) {
    case 'referer':
    case 'referrer':
      return this.headers.referrer
        || this.headers.referer;
    default:
      return this.headers[lc];
  }
};
```

这块没什么好说的，就是直接从请求对象上的headers上获取指定字段和值，利用的Node的原生语法。

### req.path()实现原理

`req.path`可以获取`req`上的`url`的`pathname`部分，比如`/users?name=jack`获取到的是`/users`，其实现原理如下:

```js
var parse = require('parseurl');

/**
 * 从req上解析path路径
 * @return {String}
 * @public
 */
defineGetter(req, 'path', function path() {
  return parse(this).pathname;
});

/**
 * 在一个对象上创建getter属性的辅助函数
 * @param {Object} obj
 * @param {String} name
 * @param {Function} getter
 * @private
 */
function defineGetter(obj, name, getter) {
  Object.defineProperty(obj, name, {
    configurable: true,
    enumerable: true,
    get: getter
  });
}
```

这里首先创建了一个`defineGetter`辅助函数，用于快速在`req`对象上创建属性，且属性只能读取不能修改。`req.path`的实现就是利用的[parseurl](https://github.com/pillarjs/parseurl#readme)库解析得到的`pathname`，有兴趣的可以了解下其实现，源码不多。

### req.fresh实现原理

`req.fresh`用于判断响应资源是否还新鲜（还未过期），比如在`res.send()`内部实现中，就判断当前`res`是否还新鲜，如果还新鲜的话则直接响应`304`，接下来我们看`req.fresh`的内部实现：

```js
/**
 * 检查请求是否还未过期，或者说叫做
 * Last-Modified或ETag是否匹配
 * @return {Boolean}
 * @public
 */
defineGetter(req, 'fresh', function(){
  var method = this.method;
  var res = this.res
  var status = res.statusCode

  /**
   * 只有GET和HEAD请求才存在未过期一说
   */
  if ('GET' !== method && 'HEAD' !== method) return false;

  // 2xx or 304 as per rfc2616 14.26
  if ((status >= 200 && status < 300) || 304 === status) {
    return fresh(this.headers, {
      'etag': res.get('ETag'),
      'last-modified': res.get('Last-Modified')
    })
  }

  return false;
});
```

响应数据是否还新鲜，针对的是GET和HEAD类型的请求，所以首先判断了非GET和非HEAD的请求就直接false了。然后对应`[200, 300)、200`范围的状态码，调用[fresh](https://github.com/jshttp/fresh#readme)库进行判断是否还新鲜，有兴趣的可以了解了解。

### req.query原理

`req.query`的实现都在middleware/query.js文件中实现的，这部分在讲解中间件架构中已经详细的提到了，可以回过头去翻看。

### req.body原理

`req.body`也是一个很重要的api，用于获取解析`post`的数据，该部分的实现在调用的第三方中间件中实现，类似的api还有一些，暂时不过多结束，在后续的中间件原理分析中会讲解。
