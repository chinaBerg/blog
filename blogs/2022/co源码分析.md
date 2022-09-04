# co源码分析

> author 愣锤

### 背景介绍

`ES2017` 标准引入了 `async` 函数，使得我们操作异步变得更加简单了，它让我们真的可以使用同步的语法编写异步的逻辑，算是彻底解决了 `javascript` 嵌套地狱苦恼。

```js
// 定义一个异步函数
const asyncFn = (timeout) => {
  return new Promise((resolve, reject) => {
    setTimeout(resolve, timeout, 'data');
  });
}

// 依次执行异步函数
async function asyncService() {
  // 等待异步执行的结果
  const result = await asyncFn(3000);
  // 等待异步执行的结果
  const result2 = await asyncFn(1000);
  // 返回结果
  return result + result2;
}

asyncService().then(data => {
  console.log('res', data);
}).catch(err => {
  console.log(err);
});
```

如上述代码所示，`async` 函数允许内部 `await` 异步函数，并且 `await` 会等待异步逻辑的执行结果，最终 `async` 函数返回一个 `promise` 实例。关于 `async/await` 想必大家都是非常熟悉的了，业务中应该都是在大量使用的。

那么在 `async/await` 标准被实现之前，是否可以像上述一样使用同步方式编写异步逻辑呢？答案是可以的，下面我们看下 `async/await` 标准之前的`hack`方案吧！

### CO模块介绍

[CO](https://github.com/tj/co#readme)是大名鼎鼎的`TJ`巨佬编写的一个基于`Generator`语法实现的用同步方式编写异步逻辑的库，在`Node`端和浏览器端都可以使用。下面我们看下如何使用`CO`达到和上述`async/await`一样的效果：

```js
const co = require('co');

// 定义一个异步函数
const asyncFn = (timeout) => {
  return new Promise((resolve, reject) => {
    setTimeout(resolve, timeout, 'data');
  });
}

const promise = co(function* () {
  // 等待异步执行的结果
  const result = yield asyncFn(3000);
  // 等待异步执行的结果
  const result2 = yield asyncFn(1000);
  // 返回结果
  return result + result2;
});

promise.then(data => {
  console.log(data);
}).catch(err => {
  // ...
});
```

可以看到，`co`利用`Generator`语法同样实现了`async/await`的效果，并且通过`yield`可以等待异步执行结果，最终也返回一个`promise`实例。

除此之外，`co`内部的`yield`除了支持异步函数，还可以是`Generator`构造函数、`Generator`实例、`Thunk`函数等。我们再看个复杂的例子：

```js
function* GenFn() {
  return yield Promise.resolve(123);
}

function resolveThunk(done) {
  setTimeout(() => {
    done(null, 'thunk response')
  }, 1000);
}

function rejectThunk(done) {
  setTimeout(() => {
    done(new Error('thunk error'))
  }, 1000);
}

co(function* (){
  try {
    // 等待一个Generator的异步结果
    const res1 = yield GenFn;
    // 等待Generator实例的异步结果
    const res2 = yield GenFn();
    // 输出 123 123
    console.log(res1, res2);

    // 等待一个异步Thunk函数的结果，1s后输出thunk response
    const res3 = yield resolveThunk;
    console.log(res3);

    // 1s后抛出一个错误
    yield rejectThunk;
  } catch (err) {
    // 输出 try/catch error: thunk error
    console.log('try/catch error:', err.message);
  }
  return 'co data';
}).then(data => {
  // 输出 co resolved: co data
  console.log('co resolved:', data);
}).catch((error) => {
  console.log('co rejected:', error);
});
```

通过上面这个复杂的小例子可以看到，`co`中`yield`支持的表达式的多样性。通过这里的错误抛出情况可知，`yield`后面表达式抛出的异步错误会被 `function*(){}` 内部的 `try/catch` 捕获，如果没有 `try/catch` 捕获，则会被上抛到 `co` 外部，也就是`co()`调用后返回的 `promise` 实例的 `catch` 捕获到。

要知道`Generator`语法本身是没有这些功能的，`co`基于`Generator`实现这一些的功能，真的是非常强悍，不由得让人竖起大拇指。理解`Generator`也是更好的理解`async/await`的逻辑。讲解`co`实现之前，先把`Generator`基础回顾一下。


### Generator

`Generator` 生成器函数是 `ES6` 提供的一种异步编程解决方案，并且把js异步编程猛的带到一个新高度。有两个明显的语法特征：

- `function`关键字与函数名之间有个 `*` 号，类似`async`
- 函数内部可以使用`yield`关键词，类似`await`

```js
// 定义一个Generator函数
function* gen() {
  yield 123;
  yield true;
  return false;
}
```

调用`Generator`函数会创建一个`Generator`对象，但是要注意的是此时`Generator`函数内部的代码逻辑并不会立即执行，而是需要通过`Generator`对象调用`next`方法才会执行。

```js
function* Gen() {
  console.log('Gen run.')
  yield 123;
  yield 456;
}

// 没有任何输出
const gen = Gen();
```

从这里可以看到，仅调用`Gen()`函数，其内部代码是没有执行的。接着上面的代码我们继续调用：

```js
// 打印 Gen run.
const res1 = gen.next();
// 输出 { value: 123, done: false }
console.log(res1);

const res2 = gen.next();
// 输出 { value: 456, done: false }
console.log(res2);

const res3 = gen.next();
// 输出 { value: undefined, done: true }
console.log(res3);
```

可以看到，第一次调用`next`的时候才开始执行内部代码，并且`next`调用返回一个对象，包含`value`和`done`两个属性：

- `value`是`yield`后面表达式的执行结果
- `done`的值为`true`或`false`，表达当前`gen`迭代器有没有执行完毕

这里有个重点得提醒一下，`gen.next()` 返回值中的 `value` 是 `yield` 后面表达式值的执行结果，就是说 `yield` 后面的表达式的执行结果赋值给的是  `gen.next()` 的 `value` 值，而不是 `yield expression` 的值。

`yield expression`返回的默认是`undefined`的值，那么`yield expression`的值是由谁决定的呢？看下面的例子:

```js
function* Gen() {
  const res = yield 123;
  // 输出456
  console.log(res);
  return res;
}

const gen = Gen();
const res1 = gen.next();
// 输出 { value: 123, done: false }
console.log(res1);

const res2 = gen.next(456);
// 输出 { value: 456, done: true }
console.log(res2);
```

在调用`gen.next()`时可以传入一个参数，该参数会作为上一次`yield expression`的返回值，但是要注意的是，第一次调用`gen.next()`是不可以传递的，即使传递也没有生效的。为什么呢？因为第一次调用`gen.next()`是让代码执行到第一个`yield`位置，还不存在上一个`yield`。

为了让大家理清楚`Generator`的执行逻辑，总结了下面这张图：

![image](https://note.youdao.com/yws/res/24490/4D852442C0CC4C84AA7843CA4CD713A5)

`Generator` 函数返回的遍历器对象，除了拥有`next`方法外，还有`throw`方法。`throw`方法的主要作用是可以在Generator函数外部抛错，然后在函数内部捕获错误。如果函数内部没有捕获错误，则错误会上抛到外部。看下面这个例子：

```js
function* Gen() {
  try {
    yield 123;
  } catch (error) {
    // 输出 inside:  Error: gen throw error
    console.log('inside: ', error);
  }
}

const gen = Gen();

try {
  // 先让gen函数运行到yield
  gen.next();
  // 在外边调用抛错逻辑
  gen.throw(new Error('gen throw error'))
} catch (error) {
  console.log('outside: ', error);
}
```

到这里，基本上 `Generator` 的主要用法就涵盖了。总结一下，`Generator`函数会创建一个生成器对象，巧妙之处在于可以控制内部代码的暂停，并把执行权交给其他协作者（或者通俗讲，交给外部）。

什么意思呢？就是内部的代码每次执行到 `yield` 命令时都会暂停，只有在外部再次调用 `next` 方法时才会继续执行到下一个 `yield` 命令，因此便可以方便 的控制代码的启停。基于此可以实现非常强大的异步用法。接下来我们就看`co`模块如何基于 `Generator` 函数实现强大的异步编程吧。

### Co源码分析

知其然，知其所以然。

上面知道了co实现的功能是非常强大的，那么我们自然要了解一下其原理实现了，到底是如何玩转generator的？

`co`的源码仅一个`index.js`文件，结构相对简单，主要暴露出一个co函数，下面看下主体结构如下：

```js
/**
 * 导出 `co` 模块
 */
module.exports = co['default'] = co.co = co;

/**
 * 执行一个generator函数或generator对象并返回一个promise
 * @param {Function} fn
 * @return {Promise}
 * @api public
 */
function co(gen) {
  var ctx = this;
  var args = slice.call(arguments, 1);

  return new Promise(function(resolve, reject) {
    // ......
  }
}
```

从上述代码可知道`co`库导出了一个cmd格式的函数，既包含默认导出也包含了按需导出。`co`函数内部则返回了一个`promise`实例，这样便支持了`co().then().catch();`调用。

`co`内部返回的是一个`Promise`实例，因此`co`调用时其`new Promise`内部代码是立即执行的，下面我们看`Promise`内部做了什么事情？

```js
return new Promise(function(resolve, reject) {
  // 调用generator函数，得到迭代器对象
  if (typeof gen === 'function') gen = gen.apply(ctx, args);
  /**
  * 如果gen不存在或者不存在.next方法，
  * 说明不是generator函数，而是普通函数，则直接返回函数执行结果
  */
  if (!gen || typeof gen.next !== 'function') return resolve(gen);
    
  onFulfilled();
    
  function onFulfilled(res) {
    // ...
  }
    
  function onRejected(err) 
    // ...
  }
    
  function next(ret) {
    // ...
  }
}
```

`co`()调用时参数可以是`Generator`函数、`Generator`实例等，所以上述代码首先判断传入的参数是否是函数，如果是函数则调用该函数得到结果，结果由如下几个情况：

- 参数是`Generator`函数则调用后得到`Generator`实例
- 参数是普通函数则就是普通函数的执行结果

紧接着判断函数执行结果，如果执行结果不是函数，说明传入的参数不是`Generator`函数，比如传递的是普通函数，非函数等，则直接`resolve`函数执行结果或传入的参数。

通过上述的处理，主要保证了拿到的结果一定是个`generator`实例或者类似`generator`实例（鸭式辨型思想）。处理完了参数，接下来就是调用`onFulfilled`函数开始处理co()参数函数的内部逻辑了：

```js
/**
 * @param {Mixed} res
 * @return {Promise}
 * @api private
 */
function onFulfilled(res) {
  var ret;
  try {
    // 调用迭代器的next方法获取yield的结果
    ret = gen.next(res);
  } catch (e) {
    // 调用失败直接reject错误，co().catch()可以捕获错误
    return reject(e);
  }
  // 调用成功继续next执行下去
  next(ret);
  return null;
}
```

`onFulfilled`的逻辑是拿到`generator`对象后，直接调用`next`方法开始执行`generator`函数内部逻辑到下一个`yield`位置处，`gen.next(res);`执行后得到`yield expression`的执行结果和当前`generator`函数是否执行结束的结果，然后将执行结果传递`给next`函数继续处理。如果`gen.next(res);`这行逻辑执行过程中出错则捕获错误直接`reject`。

这里有个细节点要注意下，调用`gen.next(res)`传入了参数，从代码逻辑可以看到，第一次调用`onFulfilled`时传递的是`undefined`，后续则是调用`onFulfilled`时如果传入了参数，该参数是会被作为`yield express`的返回结果的，这点非常重要，要画**重点！重点！重点！** 比如下面这个例子，`onFulfilled(res)`的参数就是`asyncFnResoledData`。而`onFulfilled(res)`的参数其实就是`yield`后面异步函数`resolved`的值，后续分析会详细解释为什么：

```js
co(function* {
  const asyncFnResoledData = yield asyncFn(); 
});
```

接下来我们看`next`函数的逻辑处理：

```js
/**
 * 在generator对象中获取next value，返回promise
 * @param {Object} ret
 * @return {Promise}
 * @api private
 */
function next(ret) {
  // 如果迭代器已经执行到最后，resolve结果，此时co().then()可以拿到结果
  if (ret.done) return resolve(ret.value);
  // 将当前值尝试转换为promise
  var value = toPromise.call(ctx, ret.value);
  /**
   * promise.then时调用onFulfilled，promise.catch时调用onRejected
   * then时把结果给到onFulfilled，onFulfilled内部继续调用gen.next(data)，
   * 因此达到了yield的结果就是then时的data结果
   * 注意：调用gen.next()时传入的结果会作为yield的返回数据
   */
  if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
  // 如果yield后面跟的内容最终不能转换成promise则抛出错误
  return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
    + 'but the following object was passed: "' + String(ret.value) + '"'));
}
```

`next`的逻辑很关键，也是`co`的核心实现。这里首先根据`gen.next()`后的值进行判断：

- 如果`generator`已经执行结束，则`resolve`结果出去，可以在`co().then()`中获取`resolve`的值
- 如果未执行结束，则判断值是否是`promise`，则通过`value.then(onFulfilled, onRejected)`处理`promise`实例的`resoled`和`rejected`逻辑。

上述这步`value.then(onFulfilled, onRejected)`逻辑很关键，这也是为什么`co`内部的`yield`等待一个异步时可以等待异步的代码执行，就像`await`一样。

`promise`实例`rejected`时则调用`onRejected`处理错误逻辑，或者根本就不是`promise`实例时（`yield`后面的表达式能得到`promise`的表达式）则调用`onRejected`抛出一个参数不对的错误。接下来我们看`onRejected`的逻辑：

```js
/**
 * @param {Error} err
 * @return {Promise}
 * @api private
 */
function onRejected(err) {
  var ret;
  try {
    // 利用gen.throw抛出错误，
    // 如果调用处yield有try/catch则在function*(){}内部的try/catch内捕获到错误
    ret = gen.throw(err);
  } catch (e) {
    // 如果yield处没有trycatch捕获错误，则会被外部捕获，也就是此处
    // 此处捕获到错误后直接reject出去就可以在外部co().catch()处捕获到了
    return reject(e);
  }
  // gen.throw返回{done: boolean, value: any}后继续next
  next(ret);
}
```

`onRejected`函数的逻辑就是对`reject`错误调用`gen.throw(err)`抛错，注意这里调用的是`generator`实例的抛错，而不是`js`语法的`throw`抛错。这里是因为我们期望如果`co()`内的`generator`函数内部有`try/catch`逻辑时则由内部的`try/catch`捕获错误，而不是上抛到`co.catch()`，只有当内部没有`try/catch`时才错误上抛到外部。

弄清楚这块还是需要上述对`generator`语法的`throw`逻辑学习，`gen.throw(err)`主要作用是在外部抛错在内部捕获，如果内部没有`try/catch`捕获错误则错误才会上抛到`gen.throw(err)`调用处或再外部。因此这里如果`co(function* { //... })`内部没有捕获错误，则错误发生时会被`onRejected`函数的`catch`部分捕获，捕获后直接`reject`出去，就可以被`co.catch()`逻辑捕获了。内部由catch处理的话则继续调用`next`往后执行。

至此，`co`的核心实现已经结束，总结一下核心实现的流程图如下：

![image](https://note.youdao.com/yws/res/24783/WEBRESOURCE61c34de756a15dafc7c26f11ab3feb3f)

最后我们再补充一个重要的知识点，**`co`模块如何`thunk`函数的？**

我们从一开始`co`的学习使用得知，`co`是至此如下`thunk`函数的，也就是`yield`后面的表达式可以是一个`thunk`函数：

```js
function resolveThunk(done) {
  setTimeout(() => {
    done(null, 'thunk response')
  }, 1000);
}

co(function* () {
  const res = yield resolveThunk;
  
  // ...
});
```

关键就在在于刚才的`next`函数内部有下面这一行代码：

```js
// 将当前值尝试转换为promise
var value = toPromise.call(ctx, ret.value);
```

这里对于yield后面的表达式先进行了一次promise尝试转换，转换的逻辑主要是如果已经是`promise`了就不再重复转换，否则的根据`value`的数据类型进行不同的处理，其中有如下逻辑：

```js
function toPromise(obj) {
  // 如果是null | undefined | ''则直接返回，不转换成promise
  if (!obj) return obj;
  // 如果已经是promise，不再重复转换
  if (isPromise(obj)) return obj;
  /**
   * 如果是Generator函数，或者Generator调用后的迭代器，
   * 则直接调用co()执行其内部逻辑，co后最终返回一个promise，
   * 通过此方式使得yield后面支持了Generator函数或者Generator迭代器
   */
  if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);

  // 如果是函数，则支持thunk风格
  if ('function' == typeof obj) return thunkToPromise.call(this, obj);

  // 如果yield的是数组，则对数组每一项转换成promise，并用Promise.all包裹
  // 即所有pormise都resolved才resolved
  if (Array.isArray(obj)) return arrayToPromise.call(this, obj);
  if (isObject(obj)) return objectToPromise.call(this, obj);
  return obj;
}
```

比如这里就是判断如果是函数，比如`thunk`函数，就直接调用`thunkToPromise`转换，其他还有需要主要的就是如果`yield`后面是`Generator`或者`Generator`实例则先调用`co`进行结果获取，就像套娃一样。接下来我们重点看`thunkToPromise`的实现:

```js
/**
 * 将一个thunk函数转换成一个promise
 *
 * @param {Function}
 * @return {Promise}
 * @api private
 */
function thunkToPromise(fn) {
  var ctx = this;
  // 返回一个promise
  return new Promise(function (resolve, reject) {
    // 核心做法在于把resolve和reject的机会交由用户触发
    // 触发逻辑是用户的 function thunk(done) {} 函数内部调用done时传入的参数
    // 如果第一个参数传入了有效值则reject，否则第二位及以后的参数都作为resolve值
    // 参数格式是nodejs风格的
    fn.call(ctx, function (err, res) {
      if (err) return reject(err);
      if (arguments.length > 2) res = slice.call(arguments, 1);
      resolve(res);
    });
  });
}
```

`thunkToPromise`其实就是一次对`yield`后面函数的一次包装调用并返回一个`promise`实例。这里封装的思路是：

- `new Promise`是立即执行的，因此`fn.call(ctx, function() {})`直接调用用户的`thunk`函数
    - `fn`是`thunk`函数
    - `fn`的第二个参数是传递给`thunk`函数的`done`参数
- 调用`thunk`时传递了一个`done`函数让使用者根据业务逻辑调用`done`函数
    - 通过这种方式支持的异步，比如用户可以在一个异步`resolved`或`rejected`时进行`done`
- `done`参数调用时会根据传给`done`的参数格式对`thunk`进行`resolve`或`reject`
    - `done`的参数格式是符合`nodejs`标准的，第一个参数表示错误，后续参数都是`resolved`的值。


### 总结

`co`的核心实现就是利用`generator`控制代码执行的启动停止，并处理`yield`异步表达式的`resolved`和`rejected`状态。



