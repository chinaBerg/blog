# mocha使用教程

[mocha](https://mochajs.org/#configuring-mocha-nodejs)是一个javascript测试框架，可以将测试的异常映射到对应的测试用例上。

### 基本使用

- 安装

```bash
# 安装mocha
npm install mocha -D
```

- 添加测试用例

`mocha`默认会对项目根目录下的`test`文件夹下面的所有`.js`文件进行测试

```bash
# 创建测试文件夹
mkdir test
```

在`test`文件夹下创建`demo.js`文件，写入一段基本的测试用例代码:

```js
// demo.js
const assert = require('assert');

// 定义一组测试套件
describe('start a mocha test suit.', function() {
  // 定义一个测试用例
  it('start a first test.', function() {
    assert.equal(1, 1);
  });
});
```

- 配置npm命令

```json
{
    "scripts": {
        "test": "mocha"
    }
}
```

配置好`npm`命令后运行`npm test`即可以运行测试用例了，结果如下图所示：

![image](https://note.youdao.com/yws/res/24825/AE18D87DEC2149C8984B035A6B8CC041)

- `describe()` 表示一组测试套件（一组测试用例）
    - 第一个参数是测试套件描述
    - 第二个参数是一个函数，可以在该函数内添加具体的测试用例
- `it()` 表示一个测试用例
    - 第一个参数是该测试用例的描述
    - 第二个参数是一个函数，在函数内部具体实现测试用例内容


### 参数配置

- 通过命令行参数修改配置

命令行的使用方式是`--key=value`的格式，如下所示，设置超时时间为`3000`ms:

```js
"scripts": {
    "test": "mocha --timeout=3000"
},
```

- 通过配置文件修改参数

`mocha`支持多种配置文件格式，在`mocha`的`6.0.0`版本及以后，支持正在项目根目录创建`.mocharc.js | .mocharc.cjs`、`.mocharc.json | .mocharc.jsonc`、`.mocharc.yaml | .mocharc.yml`。下面演示`json`的配置方式，其文件内容是`mocha`参数的默认配置：

```json
{
  "diff": true,
  "extension": ["js", "cjs", "mjs"],
  "package": "./package.json",
  "reporter": "spec",
  "slow": "75",
  "timeout": "2000",
  "ui": "bdd",
  "watch-files": ["lib/**/*.js", "test/**/*.js"],
  "watch-ignore": ["lib/vendor"]
}
```

更多配置文件写法可以[查阅这里](https://github.com/mochajs/mocha/tree/master/example/config)。

在`6.0.0`以前的版本，配置文件是存放在`test`目录下的`mocha.opts`中，具体的翻阅对应版本的文档。

- 修改配置文件位置

可以在使用`mocha`命令时添加命令行参数`--config [your path]`指定配置文件路径:

```js
{
    "scripts": {
        "test": "mocha --config ./a.json"
    },
}
```

### 接口风格

`mocha`运行开发者选择自己喜欢的风格的接口模式，支持`BDD`、`TDD`、`Exports`、`QUnit`和`Require`风格的接口。默认使用的是`BDD`风格。

可以通过修改配置参数来修改`mocha`的接口风格：

```json
{
    // 修改mocha的接口风格，默认是bdd
    "ui": "bdd"
}
```

- `BDD`提供了`describe()`、`it()`、`before()`, `after()`, `beforeEach()`, `afterEach()`方法
    - `describe()`定义一组测试套件
    - `it()`定义一个测试用例
    - `before()`钩子函数，在当前区块内第一个测试用例运行之前运行一次
    - `after()`钩子函数，在区块内最后一个测试用例运行之后运行一次
    - `beforeEach()`钩子函数，在当前区块内的每一个测试用例运行前运行一次
    - `afterEach()`钩子函数，在当前区块内的每一个测试用例运行后运行一次

```js
const assert = require('assert');

describe('start a mocha test suit.', function() {
  before(function beforeHook() {
    console.log('should run before this suit.');
  });
  beforeEach(function beforeEachHook() {
    console.log('should run before each test.');
  });

  it('shuold return true.', function() {
    assert.equal(1, 1);
  });
  it('should return false.', function() {
    assert.equal(1, 2);
  });
});
```

通过上面的例子可以看到钩子的执行情况，如下图所示：

![image](https://note.youdao.com/yws/res/24900/A089C4B964384A22A3588CA040BA9E36)


### 异步支持

- 基于回调的异步支持

通过向 `it()` 回调函数添加一个参数（通常命名为`done`）的方式可以支持异步测试。`done`函数可以传入一个`Error`对象或`Error`子类对象或假值表示不通过测试用例，不传表示通过测试用例：

```js
// 待测试的函数
function asyncFn(msg) {
  return new Promise((resolve, reject) => {
    setTimeout(reject, 1000, new Error(msg));
  })
}

// 测试用例
escribe('start a mocha test suit.', function() {
  it('async test with done', function(done) {
    asyncFn('a error')
      .then(() => {
        // 通过测试
        done();
      }).catch(err => {
        // 不通过测试
        done(err);
      });
});
```

- 基于`promise`的异步支持

除了使用`done`支持异步用例，也可以直接返回一个`promise`对象，`mocha`会判断`promise`的状态来决定测试用例通过或不通过：

```js
// 待测试的函数
function asyncFn(msg) {
  return new Promise((resolve, reject) => {
    setTimeout(reject, 1000, new Error(msg));
  })
}

// 返回promise对象支持异步测试用例
it('async test with promise.', function() {
    return asyncFn('a error');
});
```

注意，不能既传递`done`参数又返回`promise`对象，会出现错误。

- 基于`async/await`的异步支持

```js
it('async test with promise.', function() {
    return asyncFn('a error');
});
```

上述三种支持异步的方式的测试用例，运行后如下图所示：

![image](https://note.youdao.com/yws/res/24938/272FC97E498043D0941668D195166140)

- 异步钩子

`before`、`after`等钩子也是既支持同步也支持异步，异步的写法和上述测试用例支持异步一样，如下图所示：

```js
// mocha钩子既可以是同步的也可以是异步的
// 异步钩子则使用done函数
after(function(done) {
    setTimeout(() => {
      console.log('async after hook');
      done();
    }, 1000);
});
```

- 补充，思考`done`异步的实现

如果测试的是同步代码，需要省略回调函数，而且`mocha`再执行完一个用例后会自动执行下一个测试用例。

那么思考一下，如何实现一段`js`执行一组包含同步或异步的任务，如果异步任务传递了参数`done`则等待异步完成，如果是同步则直接调用下一个异步任务。

```js
function run(queue) {
  function next(index) {
    if (index < queue.length) {
      // 核心在于判断任务定义时是否定义了形参
      const fn = queue[index];
      // 如果没有形参定义（例如done），则运行完同步任务直接调用下一个任务
      if (fn.length === 0) {
        fn();
        next(++index);
      // 如果定义了形参，则等待用户手动调用done后开始下一个任务，依次支持异步
      } else {
        fn(function done() {
          console.log('done length', queue[index].length);
          next(++index);
        });
      }
    }
  }
  next(0);
}
```

测试下上述代码的结果，先测试一组同步任务：

```js
const tasks = [
  function() {
    console.log(1);
  },
  function() {
    console.log(2);
  },
  function() {
    console.log(3);
  },
];

// 依次输出1，2，3
run(tasks);

const tasks2 = [
  function() {
    console.log(1);
  },
  function(done) {
    setTimeout(() => {
      console.log(2);
      done();
    }, 1000);
  },
  function() {
    console.log(3);
  },
];

// 先输出1，1s后输出2，紧接着输出3
run(tasks);
```

如果传入了`done`但是没有调用，结果是否符合预期呢？看下面的例子：

```js
const tasks = [
  function() {
    console.log(1);
  },
  function(done) {
    setTimeout(() => {
      console.log(2);
    }, 1000);
  },
  function() {
    console.log(3);
  },
]；

// 先输出1，1s后输出2，没有输出3
run(tasks);
```

可以看到结果是符合预期的，第二个异步任务没有调用`done`，也就没有再运行到下一个任务了。

### 测试执行时间

`mocha`的很多`reporter`会认为测试用例的执行周期时间（默认`75ms`）超过一定值的时候就是慢的，看下面的这个例子：

```js
describe('slow test suit.', function() {
  it('should run 200ms.', function() {
    return asyncFn(200);
  });

  it('should run 500ms.', function() {
    return asyncFn(500);
  });
});
```

![image](https://note.youdao.com/yws/res/24972/A68AC1866D894AFCBE1BF7972853D2F4)

如上图所示，两个测试用例都超过了75ms，因此执行测试用例的时候红色的部分提示了是慢的。根据执行的时间，会将测试用例执行的效率氛围下面三种：

- fast：低于设置时间的一半
- normal：大于设置时间的一半到设置时间之间
- slow：超过设置时间

如果有些测试用例就是比较耗时的话，我们可以自定义这个目标时间：

```js
describe('slow test suit.', function() {
  // 设置slow时间是700ms，对当前套件内所有测试用例生效
  this.slow(700);

  it('should run 200ms.', function() {
    return asyncFn(200);
  });

  it('should run 500ms.', function() {
    return asyncFn(500);
  });
});
```

如下图所示，`200ms`的变成了`fast`，`500ms`的变成了`normal`。

![image](https://note.youdao.com/yws/res/24986/B77638FFECBD4BFCAFEFFC19FC1E403B)

当然了，也可以仅对单个测试用例生效：

```js
it('should run 200ms.', function() {
    // 仅对当前测试用例生效
    this.slow(700);
    return asyncFn(200);
});
```


### 程序式使用