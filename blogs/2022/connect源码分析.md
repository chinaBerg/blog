# connect源码分析

> version 3.7.0
> 愣锤 2022/02/23

[connect](https://github.com/senchalabs/connect)是NodeJs的中间层服务，既可以作为http/https服务的中间层使用，也可以单独作为服务使用。

- 单独作为支持中间件模型的服务使用

```js
const connect = require('connect');

const app = connect();

function middlewareA(req, res, next) {
  setTimeout(() => {
    console.log('run middlewareA');
    next();
  }, 2000);
}

function middlewareB(req, res, next) {
  setTimeout(() => {
    console.log('run middlewareB');
    next();
  }, 1000);
}


app.use(middlewareA).use(middlewareB);

app.listen(3200, () => {
  console.log('[connect server] running at port 3200.')
});
```

- 作为http/https服务的中间件模型使用

```js
const http = require('http');
const connect = require('connect');

const app = connect();

function middlewareA(req, res, next) {
  setTimeout(() => {
    console.log('run middlewareA');
    next();
  }, 2000);
}

function middlewareB(req, res, next) {
  setTimeout(() => {
    console.log('run middlewareB');
    next();
  }, 1000);
}

app.use(middlewareA).use(middlewareB);

const server = http.createServer(app);

server.listen(3200, () => {
  console.log('[connect server] running at port 3200.')
});
```

### 源码实现

connect库整体就是对外导出了一个createServer函数用于创建server，server既可以作为http/https服务的回调使用，也可以单独作为服务使用。

```js
module.exports = createServer;

var proto = {};

function createServer() {
  function app(req, res, next){ app.handle(req, res, next); }
  merge(app, proto);
  merge(app, EventEmitter.prototype);
  app.route = '/';
  // 存储中间件集合的栈
  app.stack = [];
  return app;
}

// 注册中间件
proto.use = function use(route, fn) {}

// 依次调用中间件
proto.handle = function handle(req, res, out) {}

// 启动监听服务
proto.listen = function listen() {}
```

如上所示：

- `createServer`函数内创建了一个`app`函数并返回`app`，`app`函数基本和`http/https.createServer`方法的回调函数一样，因此`createServer`返回的实例可以作为`http/https.createServer`服务的回调函数使用。作为回调使用时，实际执行的是`app.handle`方法，该方法主要用于依次调用添加的中间件服务。
- `merge(app, proto);`将`proto`上的方法浅拷贝给`app`，这样`app`函数对象上就添加了listen等方法。因此便可以通过`app.listen(3000, () => {})`把app单独作为服务使用。

下面看中间件如何注册的:

```js
proto.use = function use(route, fn) {
  var handle = fn;
  var path = route;

  // default route to '/'
  // 注册中间件时没有明确指定中间件范围，
  // 则默认指定为'/'，即对全部生效
  if (typeof route !== 'string') {
    handle = route;
    path = '/';
  }

  // wrap sub-apps
  // 对参数是对象时的参数格式化
  if (typeof handle.handle === 'function') {
    var server = handle;
    server.route = path;
    handle = function (req, res, next) {
      server.handle(req, res, next);
    };
  }

  // wrap vanilla http.Servers
  if (handle instanceof http.Server) {
    handle = handle.listeners('request')[0];
  }

  // strip trailing slash
  // 去除末尾的/符号
  if (path[path.length - 1] === '/') {
    path = path.slice(0, -1);
  }

  // add the middleware
  debug('use %s %s', path || '/', handle.name || 'anonymous');
  // 把中间件放入栈中，实际是队列的意思
  this.stack.push({ route: path, handle: handle });

  return this;
};
```

注册的逻辑就是把中间件和中间件应当匹配的范围存入数组中。

下面看中间件是如何依次调用的，这也是本库的核心关键:

```js
proto.handle = function handle(req, res, out) {
  var index = 0;
  var protohost = getProtohost(req.url) || '';
  var removed = '';
  var slashAdded = false;
  var stack = this.stack;

  // final function handler
  // 对请求的最终处理
  var done = out || finalhandler(req, res, {
    env: env,
    onerror: logerror
  });

  // store the original URL
  req.originalUrl = req.originalUrl || req.url;

  function next(err) {
    if (slashAdded) {
      req.url = req.url.substr(1);
      slashAdded = false;
    }

    if (removed.length !== 0) {
      req.url = protohost + removed + req.url.substr(protohost.length);
      removed = '';
    }

    // next callback
    // 即将调用的中间件对象, layer.route中间件路由 layer.handle是中间件函数
    var layer = stack[index++];

    // all done
    // 中间件执行结束，如果有错误则会进行错误输出
    if (!layer) {
      defer(done, err);
      return;
    }

    // route data
    // 请求的pathname路径
    var path = parseUrl(req).pathname || '/';
    // 当前中间件匹配的路径范围
    var route = layer.route;

    // skip this layer if the route doesn't match
    // 如果请求的pathname不符合当前中间件的匹配范围，则跳过当前中间件处理
    if (path.toLowerCase().substr(0, route.length) !== route.toLowerCase()) {
      return next(err);
    }

    // skip if route match does not border "/", ".", or end
    /**
     * 跳过不匹配的请求路径
     * 例如，中间件期望处理的是/a的请求，但是请求的url可能是/a、/abc、/a/bc
     * 因此这里处理的就是排除/abc的情况
     */
    var c = path.length > route.length && path[route.length];
    if (c && c !== '/' && c !== '.') {
      return next(err);
    }

    // trim off the part of the url that matches the route
    if (route.length !== 0 && route !== '/') {
      removed = route;
      req.url = protohost + req.url.substr(protohost.length + removed.length);

      // ensure leading slash
      if (!protohost && req.url[0] !== '/') {
        req.url = '/' + req.url;
        slashAdded = true;
      }
    }

    // call the layer handle
    call(layer.handle, route, err, req, res, next);
  }

  next();
};
```

- 通过`index`变量标记当前执行到的中间件位置
- 通过调用`next`函数执行当前中间件，并且`index`变量加1，用于指向下一个中间件的位置
- `next`内部通过`call`函数实际进行中间件的调用，并把当前next的引用传递给`call`，这样便`call`便取得了调用下一个中间件的手段。

```js
function call(handle, route, err, req, res, next) {
  // handle中间件函数的形参个数
  var arity = handle.length;
  var error = err;
  var hasError = Boolean(err);

  debug('%s %s : %s', handle.name || '<anonymous>', route, req.originalUrl);

  try {
    if (hasError && arity === 4) {
      // error-handling middleware
      handle(err, req, res, next);
      return;
    // 一旦调用出错，便不会调用后续中间件执行，而是直接一直next到最后结束
    } else if (!hasError && arity < 4) {
      // request-handling middleware
      // 请求处理的中间件调用
      // 在第三方中间件内部，手动调用next时
      handle(req, res, next);
      return;
    }
  } catch (e) {
    // replace the error
    error = e;
  }

  // continue
  next(error);
}
```

call函数内部就是实际调用中间件函数，并且把`req、res、next`这几个参数传递给调用的中间件:

```js
handle(req, res, next);
```

因此中间件执行的时候，便可以处理`req、res`，已经在执行结束后可以手动调用`next`进入下一个中间件:

```js
// 模拟一个异步处理的中间件
function middlewareA(req, res, next) {
  setTimeout(() => {
    console.log('run middlewareA');
    next();
  }, 2000);
}
```

抽象一个类似的中间件模型则如下所示:

```js
const http = require('http');

class Server {
  constructor() {
    this.stack = [];
    this.server = http.createServer((req, res) => {
      this.iterate(req, res)
    });
  }
  listen(port, cb) {
    this.server.listen(port, cb);
  }
  use(middle) {
    this.stack.push({
      middle,
    });
    return this;
  }
  iterate(req, res) {
    let index = 0;
    const stack = this.stack;

    function next(err) {
      const layer = stack[index++];
      if (index > stack.length) return;
      if (err) next(err);

      if (layer.middle) {
        layer.middle(req, res, next);
      } else {
        next(new Error('middle miss'));
      }
    }
    next(req, res);
  }
}

function middlewareA(req, res, next) {
  setTimeout(() => {
    console.log('run middlewareA');
    next();
  }, 1000);
}

function middlewareB(req, res, next) {
  setTimeout(() => {
    console.log('run middlewareB');
    next();
  }, 2000);
}

const app = new Server();
app.use(middlewareA).use(middlewareB);
app.listen(3000, () => {
  console.log('server running at port 3000');
});
```

### 总结

该库主要看其中间件的实现方式，以及该库是如何同时支持以下功能的:

- 单独作为服务使用
- 仅让已知服务支持中间件模型