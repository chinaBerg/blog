# http-errors源码分析

> version: 2.0.0
> author: 愣锤
> 未经作者允许禁止转载

[http-errors](https://github.com/jshttp/http-errors)是一个轻松为`express、koa、connect`等库创建`HTTP`错误的库。

本文将基于`2.0.0`版本`http-errors`讲解其用法和源码实现，最后会总结从该库能学到什么内容。下面先了解下该库的使用，比如我们在`express-cli`初始化的项目中的入口文件会看到如下一段代码：

```js
const express = require('express');
const createError = require('http-errors');

const app = express();

/**
 * 捕获404错误，并推给errorHandler中间件处理
 * EG：根据express中间件原理
 * 此前没有匹配的路由时且没有错误产生，则并不会走到errorHandler中
 */
app.use(function captureNonMatch(req, res, next) {
  next(createError(404));
});

// 错误处理中间件
app.use(function errorHandler(err, req, res, next) {
  // ...

  // render the error page
  res.status(err.status || 500);
  res.render('error');
});

module.exports = app;
```

该段代码就是给`express`应用一个中间件，该中间件作用就是创建一个`404`的HTTP错误，然后通过`next`传递给下一个中间件。下一个中间件也是最后一个中间件，是一个用于四个参数的错误处理中间件，用于前面中间件出现错误时响应一个错误的`html`模板。


那这里为什么要有`captureNonMatch`处理呢？是因为在此之前如果没有任何的路由匹配时，我们希望给接收端响应一个`404`错误的模板，因此这里通过`next`一个`404`的`HTTP`错误，然后统一交由错误处理中间件处理。

学会了如何使用之后，我们接下来看其源码实现。`http-errors`库主要暴露`createError`和`isHttpError`两个方法：

- `createError`用于创建`HTTP`错误
- `isHttpError`用于判断是否为`HttpError`类型错误

`http-errors`的源码都在根目录下的`index.js`中，文件结构很简单，下面我们看起`index.js`中源码的核心结构：

```js
/**
 * 导出的模块
 * @public
 */
module.exports = createError
module.exports.HttpError = createHttpErrorConstructor()
module.exports.isHttpError = createIsHttpErrorFunction(module.exports.HttpError)

// 往导出的module.exports导出的函数上挂载所有错误类
populateConstructorExports(module.exports, statuses.codes, module.exports.HttpError)

function createError () {}

function createHttpErrorConstructor () {}

function createIsHttpErrorFunction (HttpError) {}

function populateConstructorExports (exports, codes, HttpError) {}
```

从主体源码结构可以看到就是导出了`createError`和`isHttpError`方法，`HttpError`虽然导出了但是并不是对外的`API`，只是用于导出单测的，是私有方法。最后调用`populateConstructorExports`方法对`module.exports`对象，也就是对`createError`做了一些处理，接下来我们就看他做了什么处理。

### populateConstructorExports源码分析

在使用`http-errors`库时，我们可以直接调用某个错误类创建错误，如下所示：

```js
var err = new createError.NotFound()

// 状态码和错误名称
const code = err.code;
const name = err.name
```

为什么`createError`函数上会有类似`NotFound`这些错误类呢？原因就在于源码中通过`populateConstructorExports`函数对`createError`做了如下处理：

```js
var statuses = require('statuses')

// 往导出的module.exports导出的函数上挂载所有错误类
populateConstructorExports(module.exports, statuses.codes, module.exports.HttpError)

/**
 * 将所有错误类的构造函数挂载到exports对象上
 * @private
 */
function populateConstructorExports (exports, codes, HttpError) {
  /**
   * 迭代 1xx - 5xx 的错误码
   * - 如果是4xx的则构造客户端错误类
   * - 如果是5xx的则构造服务端错误类
   */
  codes.forEach(function forEachCode (code) {
    var CodeError
    // 获取状态码的错误信息并去除空格转成大驼峰写法
    // EG：Not Found 转换成 NotFound
    var name = toIdentifier(statuses.message[code])

    switch (codeClass(code)) {
      // 4xx的客户端错误，生成用于创建错误对象的类
      case 400:
        CodeError = createClientErrorConstructor(HttpError, name, code)
        break
      // 5xx的服务端错误，生成用于创建错误对象的类
      case 500:
        CodeError = createServerErrorConstructor(HttpError, name, code)
        break
    }

    /**
     * 将错误类导出
     * 即挂载到exports对象上
     */
    if (CodeError) {
      exports[code] = CodeError
      exports[name] = CodeError
    }
  })
}
```

`populateConstructorExports`接收三个参数：

- 第一个参数`exports`其实就是传入的`createError`函数
- 第二个参数是传入`statues.code`，所有`HTTP`状态码的集合
- 第三个参数是`HttpError`抽象类

我们先`debug`看下`statuses.code`具体是什么内容，如下图所示，就是利用`statuses`库获取所有的`HTTP`状态码集合：

![image](https://note.youdao.com/yws/res/23833/28180BAECFB64689829CE8C5E37C18DB)

方法内部的处理逻辑就是迭代所有的状态码，只处理`4xx`和`5xx`范围的状态码，因为只认为`4xx`和`5xx`的才是错误。

- 如果是`4xx`的状态码则认为是客户端错误，调用`createClientErrorConstructor`抽象工厂用于创建状态码对应的错误生成类。
- 如果是`5xx`的状态码则认为是服务端错误，调用`createServerErrorConstructor`抽象工厂用于创建状态码对应的错误生成类
- 最后将错误生成类挂载到`createError`函数上进行对外暴露

挂载完成后，我们通过`Debug`看下`createError`函数被挂载完成后是什么样子。如下图所示，`createError`函数被挂载了状态码和错误名称对应的错误生成类：

![image](https://note.youdao.com/yws/res/23855/06C304389D534C2BA2CA651B6DBEE143)

### createClientErrorConstructor抽象工厂

我们知道了`createClientErrorConstructor`和`createServerErrorConstructor`都是一个抽象工厂，用于生成类。那就看下这两个方法里面做了什么事情：

```js
/**
 * 创建一个构造函数用于构造客户端错误
 * @private
 */
function createClientErrorConstructor (HttpError, name, code) {
  // 根据name转换成类名
  var className = toClassName(name)
  
  // 创建一个ClientError类
  function ClientError (message) {
    // ...
  }
  
  // ClientError继承自HttpError抽象类
  inherits(ClientError, HttpError)
  nameFunc(ClientError, className)

  // 定义status和expose等实例属性
  ClientError.prototype.status = code
  ClientError.prototype.statusCode = code
  ClientError.prototype.expose = true

  // 返回ClientError类
  return ClientError
}
```

`createClientErrorConstructor`抽象工厂内部就是创建了一个继承自`HttpError`抽象类的`ClientError`类，然后定义了`status`状态码等实例属性，`expose`实例属性用于表示这是可以该错误对象可以暴露给客户端，因为`4xx`一般是客户端错误。最后返回了`ClientError`类。下面我们看ClientError类的构造函数实现：

```js
function ClientError (message) {
  // 创建error对象，默认使用statuses.message中的错误消息
  var msg = message != null ? message : statuses.message[code]
  var err = new Error(msg)

  // capture a stack trace to the construction point
  Error.captureStackTrace(err, ClientError)

  // 修改error对象的原型对象指向ClientError.prototype
  setPrototypeOf(err, ClientError.prototype)

  // 重新定义error对象的message属性
  Object.defineProperty(err, 'message', {
    enumerable: true,
    configurable: true,
    value: msg,
    writable: true
  })

  // 重新定义error对象的name属性
  Object.defineProperty(err, 'name', {
    enumerable: false,
    configurable: true,
    value: className,
    writable: true
  })

  // 显示返回error对象
  return err
}
```

其实就是根据错误信息实例化一个`error`对象，然后将该`error`对象的原型指向`ClientError.prototype`，因此该`error`对象也就有了`status、expose`等实例属性，然后重新添加`message`和`name`属性信息，最后返回该`error`对象。

值得注意的一点是，`error`对象的原型本来是指向`Error`对象，但是修改后指向了`ClientError`类，那么`error`对象原来的`Error`相关的属性和方法不就丢失了吗？答案并不是，因为`ClientError`类的原型对象指向`HttpError`抽象类，`HttpError`抽象类本身是继承自`Error`的。所以我们看下`HttpError`抽象类的实现：

```js
module.exports.HttpError = createHttpErrorConstructor()

/**
 * 创建HTTP错误的抽象类
 * @private
 */
function createHttpErrorConstructor () {
  // 创建HttpError抽象类
  function HttpError () {
    throw new TypeError('cannot construct abstract class')
  }

  // HttpError继承自Error对象
  inherits(HttpError, Error)

  return HttpError
}
```

这里需要了解到的一点就是`module.exports.HttpError`虽然对外暴露了，但却是注释的方式定义为私有方法，因为它并不是真正对外暴露的API，只是用于单测的目的。

分析完了`createClientErrorConstructor`的实现，`createServerErrorConstructor`的实现其实一摸一样，区别在于`createServerErrorConstructor`的`expose`属性为`false`，因为`5xx`表示服务端错误，是不应该对客户端暴露的。

### createError源码分析

```js
/**
 * 创建一个HTTP错误
 *
 * @returns {Error}
 * @public
 */
function createError () {
  // 错误对象
  var err
  // 错误信息
  var msg
  // 错误状态码
  var status = 500
  // 自定义属性，会被挂载到error对象上
  var props = {}
  
  // 省略arguments参数处理部分
  // 根据不同的个数对上面对变量进行赋值....
  
  // 错误码不在4xx、5xx时，默认赋值为500
  if (typeof status !== 'number' ||
    (!statuses.message[status] && (status < 400 || status >= 600))) {
    status = 500
  }
  
  // ...
  
  return err
}
```

`createError`一开始就是根据`arguments`参数的个数和类型的区别，获取不同的错误状态码和错误信息，注意的是当传入一个`4xx、5xx`范围之外的状态码时，默认给`500`。

```js
/**
 * 尝试获取status对应的Error生成类
 * 例如400-451的状态码有对应的Error生成类，但是452就没有了，
 * 因此如果获取不到则尝试获取其所在范围的错误生成类，例如452-499就获取的是400的错误生成类
 */
var HttpError = createError[status] || createError[codeClass(status)]

if (!err) {
  // 实例化一个error对象
  err = HttpError
    ? new HttpError(msg)
    : new Error(msg || statuses.message[status])
  Error.captureStackTrace(err, createError)
}
```

根据状态码从createError上查找对应的错误生成类，如果用户在使用http-errors时没有主动传入error对象，则创建error对象。创建逻辑是尝试使用查找到的错误生成类进行创建，如果没有对应的类（比如452-499之间的状态码）则默认实例化一个普通的Error对象。

```js
/**
 * 如果不是HttpError构造的错误对象（比如调用时传递的），
 * 则expose属性由状态码决定，500以上说明是服务器错误则指明不对客户端暴露
 * err.status !== status用于判断用户同时传递了err和状态码，但是两者的状态码不一致则需要统一
 */
if (!HttpError || !(err instanceof HttpError) || err.status !== status) {
  // add properties to generic error
  err.expose = status < 500
  err.status = err.statusCode = status
}
```

紧接着判断如果传入的错误不是`HttpError`创建的或者同时传入了错误和状态码但是状态码不一致，则格式化或者添加`expose`和`status`等属性值。最后就是把传入的一些自定义属性添加到生成的`error`对象中，并返回`error`对象，代码如下所示：

```js
/**
 * 将其他自定义配置添加到error对象上
 */
for (var key in props) {
  if (key !== 'status' && key !== 'statusCode') {
    err[key] = props[key]
  }
}

return err
```

到这里`createError`的逻辑就分析完了，其实就是根据错误状态码寻找对应的错误生成类进行实例化错误对象，如果错误对象本身是传入的则不需要实例化，而是判断是否是符合`HttpError`类型的错误对象，不符合的话则格式化`status`和`expose`等属性。

### isHttpError源码分析

`createError.isHttpError()`方法用于判断一个对象是否是`HttpError`类型的错误对象，其源码实现如下：

```js
module.exports.isHttpError = createIsHttpErrorFunction(module.exports.HttpError)

/**
 * 创建一个用于判断是否是HttpError错误的函数
 * 注意：这里HttpError是通过外部传入而不是直接在isHttpError函数内部使用，原因在于方便单元测试。
 * - 直接使在isHttpError内部用HttpError的话，其单测过程中还要确保isHttpError测试通过。
 * - 通过外部传入的话，我们只需要在HttpError的单测中确保其正确就ok
 * @private
 */
function createIsHttpErrorFunction (HttpError) {
  return function isHttpError (val) {
    if (!val || typeof val !== 'object') {
      return false
    }

    // 如果是HttpError实例则直接返回true
    if (val instanceof HttpError) {
      return true
    }

    /**
     * 利用鸭子类型判断
     * 如果该error对象有布尔类型的expose属性、statusCode和status相同且都为number
     * 则认为是HttpError类型的错误
     */
    return val instanceof Error &&
      typeof val.expose === 'boolean' &&
      typeof val.statusCode === 'number' && val.status === val.statusCode
  }
}
```

这里的判断逻辑相对简单：

- 如果是`HttpError`类的实例，则直接返回`true`
- 否则利用**鸭式辨型**的思想，只要有`expose、statusCode、status`属性且类型都对，就认为是`HttpError`类的实例

这里有一个相对重要的点就是，`isHttpError`利用`createIsHttpErrorFunction`进行了一层的包裹，把`HttpError`以参数的形式传递进入使用，而不是直接在`isHttpError`函数内部直接使用`HttpError`是有原因的。

这样的做法并不是多余，而是为了单元测试的考量。如果是直接在内部使用`HttpError`，那么在进行`isHttpError`单测的时候还要考虑外部依赖的单测。而以参数的形式传入`HttpError`，则`isHttpError`是纯净的，单测时不需要考虑`HttpError`，只需要对`HttpError`做单独的单测即可。

### Error.captureStackTrace作用

在`http-errors`的源码实现中，有很多处创建错误时都添加了`Error.captureStackTrace`的使用，下面我们就来了解下该代码的作用是什么。

- Error的基本了解

我们可以通过`new Error(message)`实例化一个`error`对象，并且将`error.message`属性设置为提供的文本消息。如果`message`传入的是对象，则通过调用 `message.toString()`生成文本消息。 

`error.stack`属性将代表代码中调用 `new Error()` 的堆栈追踪信息。下面我们看一个在函数嵌套中使用`error`对象的堆栈追踪信息：

```js
function fn1() {
  fn2();
}
function fn2() {
  fn3();
}
function fn3() {
  const error = new Error('this is a error.');
  // Error.captureStackTrace(error, fn2);
  console.log(error.stack);
}

fn1();
```

我们有三个函数，`fn1`调用了`fn2`，`fn2`调用了`fn3`，`fn3`内部创建`error`对象并打印堆栈信息，执行后我们可以看到`fn3`函数内部的错误对象的堆栈信息如下图所示：

![image](https://note.youdao.com/yws/res/23957/86AD9D2229CF4939BBF9290F360E3571)

首先第一行显示的是`${error.name}: ${error.message}`格式的错误信息，然后显示的是错误的堆栈信息，从图中可以看到从上往下依次是调用栈`fn3 -> fn2 -> fn1`。如果想修改第一行的显示错误类型，可以指定`error`的`name`属性：

```js
const error = new Error('this is a error.');
error.name = 'NewError'

// 此时看到的第一行错误信息就是：
// NewError: this is a error.
console.log(error.stack);
```

- Error.captureStackTrace

`Error.captureStackTrace(targetObject[, constructorOpt])`用于捕获堆栈信息，会在 `targetObject` 上创建 `.stack` 属性，并且在访问`.stack`属性时返回 `Error.captureStackTrace()` 在代码中调用堆栈信息。

看下下面的例子更好理解一些：

```js
function fn() {
  const error = new Error('this is a error.');
  // Error.captureStackTrace(error);
  console.log(error.stack);
}

fn();
```

我们以不使用`Error.captureStackTrace(error);`追踪执行和使用`Error.captureStackTrace(error);`追踪执行的区别，可以看到如下图所示的区别，打印出来的堆栈行数不一样，说明`Error.captureStackTrace(error);`之后堆栈显示的是`Error.captureStackTrace(error);`的位置，否则的话显示的`new Error('this is a error.');`的位置。

![image](https://note.youdao.com/yws/res/23976/748B5F10339641649F504BD8B9C73A1D)

![image](https://note.youdao.com/yws/res/23978/9B942E0C6B55430D9932C49B1DEB78E9)

`Error.captureStackTrace`还一个重要的作用就是可以传入第二个参数来隐藏部分堆栈信息。第二个参数接收一个函数，如果传递则该函数以上的所有调用帧都将从`stack`调用栈中隐藏。看下面这个例子：

```js
function fn1() {
  fn2();
}
function fn2() {
  fn3();
}
function fn3() {
  const error = new Error('this is a error.');
  Error.captureStackTrace(error, fn2);
  console.log(error.stack);
}

fn1();
```

还是刚才的调用例子，我们使用了`Error.captureStackTrace`来追踪堆栈信息并且传入`fn2`来隐藏`fn2`及以上的所有堆栈信息，执行后如下图所示：

![image](https://note.youdao.com/yws/res/23993/F8B6EBFCCABE42458EA9FF80BE384AB7)


可以看到堆栈到`fn1`就停止了，`fn2`和`fn3`堆栈信息都被隐藏了。该功能对于希望隐藏部分调用堆栈信息是非常有用的。下面我们看下`http-error`源码中的实际使用场景：

```js
function createError () {
  // ...
  
  if (!err) {
    // 实例化一个error对象
    err = HttpError
      ? new HttpError(msg)
      : new Error(msg || statuses.message[status])
    // 隐藏err对象的createError帧以上的堆栈信息
    // 也就是隐藏了createError的错误调用细节
    Error.captureStackTrace(err, createError)
  }
  
  // ...
}
```

在`http-error`的`createError`源码实现中，在创建了`err`对象之后，调用了`Error.captureStackTrace`追踪堆栈信息并且隐藏`createError`函数及以上的堆栈信息。那么实际作用是用户在使用`http-errors`创建错误时，看到的错误堆栈信息只到用户的调用位置，而不会暴露`http-errors`库内部的调用堆栈。看下下面这都使用例子：

```js
const createHttpError = require('http-errors');

function fn() {
  const error = createHttpError(404);
  console.log(error.stack);
}

fn();
```

执行后查看错误堆栈信息，如下图所示，并没有看到`http-errors`库内部的错误堆栈追踪信息：

![image](https://note.youdao.com/yws/res/24007/6133460B79BF4673A057C7E36FC60488)

总结一下，`Error.captureStackTrace`以自身在代码中的位置作为传入的错误对象的堆栈追踪信息，并且可以通过第二个参数隐藏部分堆栈信息。

### 总结

`http-errors`源码本身没有多少内容，但是有下面几个点还是需要注意的：

- 使用抽象工厂来创建类
- 考虑到安全问题，对于`4xx`的客户端错误可以对接收端暴露，`5xx`的服务端错误不应该暴露
- 判断某个对象是否属于某一类时，除了利用`instanceof`判断是否是例子，还利用了鸭式辨型的思想
- 利用`Error.captureStackTrace`指定错误对象的堆栈追踪信息，以及对使用者隐藏库内部的堆栈信息
- 库必须要有完整的单测，并且考虑到单测的情况，部分代码组织格式会有些变化。例如上面提到的函数内部不使用外部依赖，而是把外部依赖传进来方便单测。即使不考虑单测，函数本身也应该尽量不依赖外部的内容。