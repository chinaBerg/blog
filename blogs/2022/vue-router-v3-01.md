## vue-router源码解析 - 项目架构

### 版本
> vue-router 3.5.3版本

### 文件夹目录

![image](https://note.youdao.com/yws/res/18122/DDF0597C74B64172ACDD56BCD2785A7A)

需要关注的如下几个文件夹：

- build - 生产构建脚本
- examples - 开发环境脚本和demo资源
- src - vue-router源码文件
- scripts - git相关脚本文件
- flow - flow静态检查相关文件

## 开发环境脚本搭建

- `package.json`文件中运行命令

由此可知，vue-router开发环境运行的就是`examples/server.js`文件

```json
{
    "scripts": {
        "dev": "node examples/server.js"
    }
}
```

- 打开`server.js`文件

```js
const express = require('express')
const rewrite = require('express-urlrewrite')
const webpack = require('webpack')
const webpackDevMiddleware = require('webpack-dev-middleware')
const WebpackConfig = require('./webpack.config')

/**
 * 初始化一个express应用
 */
const app = express()

/**
 * 在express应用中使用webpack的热更新服务
 */
app.use(
  // webpack(WebpackConfig)是通过api的方式使用webpack构建
  webpackDevMiddleware(webpack(WebpackConfig), {
    publicPath: '/__build__/',
    stats: {
      colors: true,
      chunks: false
    }
  })
)

const fs = require('fs')
const path = require('path')

/**
 * 读取当前路径下的所有同级文件夹目录
 * 对当前路径下的url访问，全部重定向到对应目录文件夹下的index.html路径
 * @example /active-links/* 实际访问url的是/actives-links/index.html
 */
fs.readdirSync(__dirname).forEach(file => {
  if (fs.statSync(path.join(__dirname, file)).isDirectory()) {
    /**
     * express-urlrewrite库作用就是url重定向
     * @see https://www.npmjs.com/package/express-urlrewrite
     */
    app.use(rewrite('/' + file + '/*', '/' + file + '/index.html'))
  }
})

/**
 * 利用express托管静态文件
 * @example /index.html 会实际访问 __dirname 路径下的 index.html 文件
 */
app.use(express.static(__dirname))

const host = process.env.HOST || 'localhost'
const port = process.env.PORT || 8080

/**
 * 监听端口，默认8080
 * 此时通过http://localhost:8080即可访问dev服务
 * '/' 默认访问的是 __dirname下的index.html
 */
module.exports = app.listen(port, host, () => {
  console.log(`Server listening on http://${host}:${port}, Ctrl+C to stop`)
})
```

server.js文件的整体逻辑：

![image](https://note.youdao.com/yws/res/18207/94A7226DC70546DE9DB2928915A14369)

- 初始化一个express应用

```js
const app = express()
```

- 在express应用中使用webpack的热更新服务

```js
app.use(
  // webpack(WebpackConfig)是通过api的方式使用webpack构建
  webpackDevMiddleware(webpack(WebpackConfig), {})
)
```

- 对同级文件夹目录的文件url访问进行url重定向

```js
/**
 * 读取当前路径下的所有同级文件夹目录
 * 对当前路径下的url访问，全部重定向到对应目录文件夹下的index.html路径
 * @example /active-links/* 实际访问url的是/actives-links/index.html
 */
fs.readdirSync(__dirname).forEach(file => {
  if (fs.statSync(path.join(__dirname, file)).isDirectory()) {
    /**
     * express-urlrewrite库作用就是url重定向
     * @see https://www.npmjs.com/package/express-urlrewrite
     */
    app.use(rewrite('/' + file + '/*', '/' + file + '/index.html'))
  }
})
```

- 利用express托管静态文件

```js
app.use(express.static(__dirname))
```
 
- 监听端口，启动服务

```js
app.listen(port, host, () => {
  console.log(`Server listening on http://${host}:${port}, Ctrl+C to stop`)
})
```


### URL重定向到指定资源

```js
/**
 * 读取当前路径下的所有同级文件夹目录
 * 对当前路径下的url访问，全部重定向到对应目录文件夹下的index.html路径
 * @example /active-links/* 实际访问url的是/actives-links/index.html
 */
fs.readdirSync(__dirname).forEach(file => {
  if (fs.statSync(path.join(__dirname, file)).isDirectory()) {
    /**
     * express-urlrewrite库作用就是url重定向
     * @see https://www.npmjs.com/package/express-urlrewrite
     */
    app.use(rewrite('/' + file + '/*', '/' + file + '/index.html'))
  }
})
```

- 利用的fs模块读取所有同级的文件夹，然后利用[express-urlrewrite](https://www.npmjs.com/package/express-urlrewrite)库对url访问进行重定向到指定的`index.html`，具体表现为，比如访问的是`/transitions`会访问`/transitions/index.html`, `/transitions/sub-path` 也实际访问的是`/transitions/index.html`文件。

- 可以看到每个同级文件夹下都有`index.html`文件

![image](https://note.youdao.com/yws/res/18158/1FB6B2D31126416F8A464066D4663E34)

### 开发环境的热更新

```js
/**
 * 在express应用中使用webpack的热更新服务
 */
app.use(
  // webpack(WebpackConfig)是通过api的方式使用webpack构建
  webpackDevMiddleware(webpack(WebpackConfig), {
    publicPath: '/__build__/',
    stats: {
      colors: true,
      chunks: false
    }
  })
)
```

- 在server.js中实例化express应用后，注册了[webpack-dev-middleware](https://github.com/webpack/webpack-dev-middleware)中间件服务。

- 该中间价通过webpack api的方式使用webpack开发对应服务。

- webpack的配置基本就是常规配置入口、出口、loader和plugin

- 需要注意的是entry配置

```js
/**
 * 多入口打包配置
 * 读取__dirname路径下所有子文件夹路径
 * 将子文件路径下的app.js文件作为入口文件
 */
entry: fs.readdirSync(__dirname).reduce((entries, dir) => {
    const fullDir = path.join(__dirname, dir)
    const entry = path.join(fullDir, 'app.js')
    if (fs.statSync(fullDir).isDirectory() && fs.existsSync(entry)) {
      entries[dir] = ['es6-promise/auto', entry]
    }

    return entries
}, {}),
```

利用fs模块读取所有同级目录，并且把每个目录下的`app.js`作为入口文件，最终返回的是一个webpack多入口配置，也就是多页打包配置。如下图所示，每个demo目录下都是有一个`app.js`文件的:

![image](https://note.youdao.com/yws/res/18182/FCE69B11D5A749B98220794A01EEEFE4)

- 看下resolve别名配置

```js
resolve: {
    alias: {
      // 指定vue的实际访问路径是node_modules中的vue/dist/vue.esm.js
      vue: 'vue/dist/vue.esm.js',
      // 指定vue-router的访问路径是/src/index.js
      'vue-router': path.join(__dirname, '..', 'src')
    }
},
```

注意这里通过别名将`vue`指向了`node_modules`中安装的`vue`模块，而将`vue-router`指向了我们的`src`下的源码文件。

### demo文件夹中app.js和index.html

可以看到开发环境下，examples文件夹中的每个demo文件夹下基本都有一个`app.js`和`index.html`

![image](https://note.youdao.com/yws/res/18188/70EB6B7DB40141C6A9603F3C6623F771)

```html
<!DOCTYPE html>
<div id="app"></div>
<script src="/__build__/shared.chunk.js"></script>
<script src="/__build__/route-props.js"></script>
```

可以看到html文件，就是引入了2个js文件，这回两个文件都是webpack编译生成的，在webpack配置中我们指定了输出参数:

```js
output: {
    path: path.join(__dirname, '__build__'),
    filename: '[name].js',
    chunkFilename: '[id].chunk.js',
    publicPath: '/__build__/'
},
```

所以开发时的输出目录都是`/__build__/`文件夹下，所以此时生成的两个文件，一个是demo文件夹名称的js，一个是代码分割的公共chunk文件，而webpack配置中也指定了chunk的名称是`share`:

```js
optimization: {
    // 公共代码提取
    splitChunks: {
      cacheGroups: {
        shared: {
          name: 'shared',
          chunks: 'initial',
          minChunks: 2
        }
      }
    }
},
```

app.js基本就是一个vue的实例化：

```js
import Vue from 'vue'
import VueRouter from 'vue-router'
import Hello from './Hello.vue'

Vue.use(VueRouter)

const router = new VueRouter({
  mode: 'history',
  base: __dirname,
  routes: [
    { path: '/', component: Hello }, // No props, no nothing
    { path: '/hello/:name', component: Hello, props: true }, // Pass route.params to props
})

new Vue({
  router,
  template: `
    <div id="app">
      <h1>Route props</h1>
      <ul>
        <li><router-link to="/">/</router-link></li>
        <li><router-link to="/hello/you">/hello/you</router-link></li>
      </ul>
      <router-view class="view" foo="123"></router-view>
    </div>
  `
}).$mount('#app')
```


## 生产环境构建脚本

通过package.json中的构建命令可以得知，构建脚本是`build/build.js`

```json
{
    "scripts": {
        "build": "node build/build.js",
    }
}
```

下面看下build.js文件：

```js
const fs = require('fs')
const path = require('path')
const zlib = require('zlib')
const terser = require('terser')
const rollup = require('rollup')
const configs = require('./configs')

/**
 * 先在根目录下创建dist文件夹目录
 */
if (!fs.existsSync('dist')) {
  fs.mkdirSync('dist')
}

// 执行构建函数的调用
build(configs)

/**
 * 遍历打包配置数组依次进行打包
 * @param builds 打包配置数组
 */
function build (builds) {
  let built = 0
  const total = builds.length
  const next = () => {
    buildEntry(builds[built])
      .then(() => {
        built++
        if (built < total) {
          next()
        }
      })
      .catch(logError)
  }

  next()
}

function buildEntry ({ input, output }) {
  const { file, banner } = output
  const isProd = /min\.js$/.test(file)
  // 利用rollup进行构建输出
  return rollup
    .rollup(input)
    .then(bundle => bundle.generate(output))
    .then(bundle => {
      // console.log(bundle)
      const code = bundle.output[0].code
      if (isProd) {
        /**
         * 生产环境下利用terser对文件进行压缩优化js文件
         * 文件顶部的banner注释信息不再terser且加个换行符
         */
        const minified =
          (banner ? banner + '\n' : '') +
          terser.minify(code, {
            toplevel: true,
            output: {
              // 转义字符串和正则表达式中的 Unicode 字符
              ascii_only: true
            },
            compress: {
              // 压缩时抛弃所有makeMap函数
              // 显示指明了makeMap不会有任何副作用
              pure_funcs: ['makeMap']
            }
          }).code
        return write(file, minified, true)
      } else {
        return write(file, code)
      }
    })
}

/**
 * 构建文件写入磁盘，
 * 同时在终端输出构建文件信息
 */
function write (dest, code, zip) {
  return new Promise((resolve, reject) => {
    function report (extra) {
      console.log(
        blue(path.relative(process.cwd(), dest)) +
          ' ' +
          getSize(code) +
          (extra || '')
      )
      resolve()
    }

    // 将构建的文件输出到磁盘
    fs.writeFile(dest, code, err => {
      if (err) return reject(err)
      /**
       * 在控制台输出构建信息
       * 如果打的生产包，则在控制台同时输出gzipped后的大小
       */
      if (zip) {
        zlib.gzip(code, (err, zipped) => {
          if (err) return reject(err)
          report(' (gzipped: ' + getSize(zipped) + ')')
        })
      } else {
        report()
      }
    })
  })
}

/**
 * 计算文件大小，单位kb
 */
function getSize (code) {
  return (code.length / 1024).toFixed(2) + 'kb'
}

function logError (e) {
  console.log(e)
}

/**
 * 利用ASNI编码，在console.log时可以在控制台输出蓝色字体
 */
function blue (str) {
  return '\x1b[1m\x1b[34m' + str + '\x1b[39m\x1b[22m'
}
```

- 主体逻辑

```js
const configs = require('./configs')

/**
 * 先在根目录下创建dist文件夹目录
 */
if (!fs.existsSync('dist')) {
  fs.mkdirSync('dist')
}

// 执行构建函数的调用
build(configs)
```

主体逻辑就是导入打包的配置参数，然后调用build函数进行打包。

- build函数

```js
/**
 * 遍历打包配置数组依次进行打包
 * @param builds 打包配置数组
 */
function build (builds) {
  let built = 0
  const total = builds.length
  const next = () => {
    buildEntry(builds[built])
      .then(() => {
        built++
        if (built < total) {
          next()
        }
      })
      .catch(logError)
  }

  next()
}
```

build函数就是读取打包参数数组（数组的每一项都是一种环境的打包配置，比如dev环境、生产环境，es环境等），然后依次进行打包。

- config配置

```js
// 来自于config.js文件
module.exports = [
  // browser dev
  {
    file: resolve('dist/vue-router.js'),
    format: 'umd',
    env: 'development'
  },
  {
    file: resolve('dist/vue-router.min.js'),
    format: 'umd',
    env: 'production'
  },
  {
    file: resolve('dist/vue-router.common.js'),
    format: 'cjs'
  },
  {
    file: resolve('dist/vue-router.esm.js'),
    format: 'es'
  },
  {
    file: resolve('dist/vue-router.esm.browser.js'),
    format: 'es',
    env: 'development',
    transpile: false
  },
  {
    file: resolve('dist/vue-router.esm.browser.min.js'),
    format: 'es',
    env: 'production',
    transpile: false
  }
].map(genConfig)

function genConfig (opts) {
  const config = {
    input: {
      input: resolve('src/index.js'),
      plugins: [
        // 移除flow类型静态检查的代码
        flow(),
        node(),
        cjs(),
        replace({
          __VERSION__: version
        })
      ]
    },
    output: {
      file: opts.file,
      format: opts.format,
      banner,
      name: 'VueRouter'
    }
  }

  if (opts.env) {
    config.input.plugins.unshift(replace({
      'process.env.NODE_ENV': JSON.stringify(opts.env)
    }))
  }

  if (opts.transpile !== false) {
    config.input.plugins.push(buble())
  }

  return config
}
```

配置就是最终输出的各种环境下的rollup打包配置

- buildEntry

```js
function buildEntry ({ input, output }) {
  const { file, banner } = output
  const isProd = /min\.js$/.test(file)
  // 利用rollup进行构建输出
  return rollup
    .rollup(input)
    .then(bundle => bundle.generate(output))
    .then(bundle => {
      // console.log(bundle)
      const code = bundle.output[0].code
      if (isProd) {
        /**
         * 生产环境下利用terser对文件进行压缩优化js文件
         * 文件顶部的banner注释信息不再terser且加个换行符
         */
        const minified =
          (banner ? banner + '\n' : '') +
          terser.minify(code, {
            toplevel: true,
            output: {
              // 转义字符串和正则表达式中的 Unicode 字符
              ascii_only: true
            },
            compress: {
              // 压缩时抛弃所有makeMap函数
              // 显示指明了makeMap不会有任何副作用
              pure_funcs: ['makeMap']
            }
          }).code
        return write(file, minified, true)
      } else {
        return write(file, code)
      }
    })
}
```

该函数才是真正调用rollup进行打包的函数：

```js
rollup
    .rollup(input)
    .then(bundle => bundle.generate(output))
```

构建数据生成后，还会判断如果是prod环境的构建，则对构建内容进一步调用`terser`库进行压缩优化代码。最后调用`write`函数写入到磁盘。

- write的文件生成和控制台输出

```js
/**
 * 构建文件写入磁盘，
 * 同时在终端输出构建文件信息
 */
function write (dest, code, zip) {
  return new Promise((resolve, reject) => {
    function report (extra) {
      console.log(
        blue(path.relative(process.cwd(), dest)) +
          ' ' +
          getSize(code) +
          (extra || '')
      )
      resolve()
    }

    // 将构建的文件输出到磁盘
    fs.writeFile(dest, code, err => {
      if (err) return reject(err)
      /**
       * 在控制台输出构建信息
       * 如果打的生产包，则在控制台同时输出gzipped后的大小
       */
      if (zip) {
        zlib.gzip(code, (err, zipped) => {
          if (err) return reject(err)
          report(' (gzipped: ' + getSize(zipped) + ')')
        })
      } else {
        report()
      }
    })
  })
}
```

- 计算文件大小

```
/**
 * 计算文件大小，单位kb
 */
function getSize (code) {
  return (code.length / 1024).toFixed(2) + 'kb'
}
```

- ASNI码在控制台输出彩色文字

```js
/**
 * 利用ASNI编码，在console.log时可以在控制台输出蓝色字体
 */
function blue (str) {
  return '\x1b[1m\x1b[34m' + str + '\x1b[39m\x1b[22m'
}
```

- 生成环境构建的逻辑架构图

![image](https://note.youdao.com/yws/res/18246/9FCF6DFB4A674042833B6CD4CB283D3F)