# dotenv源码解析

> version 14.3.2
> 愣锤 2022/01/30

### dotenv作用

[dotenv](https://github.com/motdotla/dotenv)主要用于nodejs环境中，将`.env`文件中定义的变量添加到`process.env`对象上，也就是通过配置的方式往`process.env`对象上添加变量。

该做法也是[软件设计12原则](https://12factor.net/zh_cn/)中的[在环境中存储配置](https://12factor.net/zh_cn/config)原则

### 用法说明

在项目根目录下创建.env文件，然后可以在该文件配置变量

```bash
DB_HOST=localhost
DB_USER=root
DB_PASS=s1mpl3
```

使用只需要导入该库暴露的`config`方法进行调用。下面是官网的例子：

```js
require('dotenv').config();

/**
 * {
 *   DB_HOST: localhost,
 *   DB_USER: root,
 *   DB_PASS: s1mpl3
 * }
 *
 */
console.log(process.env);
```

也可以直接将符合该配置规则的数据直接调用`parse`进行解析：

```js
const dotenv = require('dotenv')
const buf = Buffer.from('BASIC=basic')
const config = dotenv.parse(buf) // will return an object
console.log(typeof config, config) // object { BASIC : 'basic' }
```

### 源码总揽

> 解析的dotenv源码版本 14.3.0

`dotenv`的源码大约200+行左右，都在他的`src/main.js`中,下面先直接放整个源码，个人加了注释：

```js
const fs = require('fs')
const path = require('path')
const os = require('os')

function log (message) {
  console.log(`[dotenv][DEBUG] ${message}`)
}

const NEWLINE = '\n'
/**
 * 该正则描述的是：
 * 开头的正则 ^\s*([\w.-]+)\s*=描述的是：
 *  - 开头是0-n个空格
 *  - 后面紧跟中英文、下划线、点、连字符
 *  - 后面再紧跟0-n个空格
 *  - 再后面是一个等于号
 * 中间的正则 \s*("[^"]*"|'[^']*'|[^#]*)?描述的的是：
 *  - 在上述的基础上再紧跟0-n个空格
 *  - 后面紧跟（备注1）：
 *    - 收尾是双引号，中间是非双引号的其他0-n个字符
 *    - 或者收尾是单引号，中间是非单引号的其他0-n个字符
 *    - 或者除#号外的0-n个字符
 *  - 最后的问号表示 备注1 的整体都是可选的
 * 最后的正则(\s*|\s*#.*)?$描述的是：
 *  - 最后紧跟0-n个空格 或者 0-n个空格加上#号加上0-n个任意字符
 */
const RE_INI_KEY_VAL = /^\s*([\w.-]+)\s*=\s*("[^"]*"|'[^']*'|[^#]*)?(\s*|\s*#.*)?$/
const RE_NEWLINES = /\\n/g
const NEWLINES_MATCH = /\r\n|\n|\r/

// Parses src into an Object
function parse (src, options) {
  const debug = Boolean(options && options.debug)
  const multiline = Boolean(options && options.multiline)
  const obj = {}

  // convert Buffers before splitting into lines and processing
  /**
   * 通过换行符分割每一行的数据
   *  - windows系统换行符 \r\n
   *  - Mac系统下换行符 \r
   *  - Unix系统下换行符\n
   */
  const lines = src.toString().split(NEWLINES_MATCH)

  // 遍历每一行的数据，获取key和value作为变量和变量名
  for (let idx = 0; idx < lines.length; idx++) {
    let line = lines[idx]

    // matching "KEY' and 'VAL' in 'KEY=VAL'
    const keyValueArr = line.match(RE_INI_KEY_VAL)
    // matched?
    if (keyValueArr != null) {
      // 子表达式1匹配到的是=号前面的key
      const key = keyValueArr[1]
      // default undefined or missing values to empty string
      // 子表达式2匹配到的是=号后面的value，不包含行尾#的注释部分
      let val = (keyValueArr[2] || '')
      // value的最后一个字符的下标
      let end = val.length - 1

      // 判断value的首尾是否都是双引号
      const isDoubleQuoted = val[0] === '"' && val[end] === '"'
      // 判断value的首尾是否都是单引号
      const isSingleQuoted = val[0] === "'" && val[end] === "'"

      // 判断value是双引号开头，且结尾没有双引号
      const isMultilineDoubleQuoted = val[0] === '"' && val[end] !== '"'
      // 判断value是单引号开头，且结尾没有双引号
      const isMultilineSingleQuoted = val[0] === "'" && val[end] !== "'"

      // if parsing line breaks and the value starts with a quote
      /**
       * 如果符合条件：
       *  - 双引号开头且结尾没有双引号 / 单引号开头且结尾没有单引号
       *  - 用户设置了同意多行配置
       * 则继续递归查询下一行，直到找到某一行的结尾能匹配到开头的单双引号
       */
      if (multiline && (isMultilineDoubleQuoted || isMultilineSingleQuoted)) {
        const quoteChar = isMultilineDoubleQuoted ? '"' : "'"

        val = val.substring(1)

        /**
         * 继续递归查询下一行，直到找到某一行的结尾能匹配到开头的单双引号
         * 将每一行的值拼接起来
         */
        while (idx++ < lines.length - 1) {
          line = lines[idx]
          end = line.length - 1
          // 判断该行结尾是否是和开头匹配的单/双引号
          if (line[end] === quoteChar) {
            val += NEWLINE + line.substring(0, end)
            break
          }
          // 将值拼接起来
          val += NEWLINE + line
        }
      // if single or double quoted, remove quotes
      }
      /**
       * 如果当前行是合法的单引号开头结尾 或者 双引号开头结尾
       * 则正常取引号内的值
       */
      else if (isSingleQuoted || isDoubleQuoted) {
        val = val.substring(1, end)

        // if double quoted, expand newlines
        if (isDoubleQuoted) {
          val = val.replace(RE_NEWLINES, NEWLINE)
        }
      } else {
        // remove surrounding whitespa
        // 如果开头结尾没有单双引号
        val = val.trim()
      }

      // 最后将匹配到的key和value进行对obj赋值，最后将obj返回出去
      obj[key] = val
    }
    /**
     * 如果当前行不符合书写规则，且是debug模式则给出错误log
     */
    else if (debug) {
      // 去除首尾空格
      const trimmedLine = line.trim()

      // ignore empty and commented lines
      // 内容不为空且不是注释，则log提示
      if (trimmedLine.length && trimmedLine[0] !== '#') {
        log(`Failed to match key and value when parsing line ${idx + 1}: ${line}`)
      }
    }
  }

  return obj
}

/**
 * 处理.env文件路径
 * 如果是~开头的路径，则调用os.homedir()获取系统对应的home路径
 */
function resolveHome (envPath) {
  return envPath[0] === '~' ? path.join(os.homedir(), envPath.slice(1)) : envPath
}

// Populates process.env from .env file
function config (options) {
  // 默认使用utf8编码格式解析.env文件
  let dotenvPath = path.resolve(process.cwd(), '.env')
  let encoding = 'utf8'

  const debug = Boolean(options && options.debug)
  const override = Boolean(options && options.override)
  const multiline = Boolean(options && options.multiline)

  if (options) {
    // 如果用户设置了自定义环境变量的文件则优先使用
    if (options.path != null) {
      dotenvPath = resolveHome(options.path)
    }
    // 如果用户指定了编码格式，则优先使用
    if (options.encoding != null) {
      encoding = options.encoding
    }
  }

  try {
    // specifying an encoding returns a string instead of a buffer
    /**
     * - 先调用fs.readFileSync同步读取.env文件
     * - 再调用封装的parse函数解析.env内容
     */
    const parsed = DotenvModule.parse(fs.readFileSync(dotenvPath, { encoding }), { debug, multiline })

    /**
     * 将解析.env文件得到的key/value值，
     * 分别赋值到process.env对象上
     */
    Object.keys(parsed).forEach(function (key) {
      if (!Object.prototype.hasOwnProperty.call(process.env, key)) {
        process.env[key] = parsed[key]
      } else {
        // 对于process.env已存在的key，根据用户的override配置决定是否覆盖
        if (override === true) {
          process.env[key] = parsed[key]
        }

        // 如果指定了debug模式，则对于重复的key进行log提示
        if (debug) {
          if (override === true) {
            log(`"${key}" is already defined in \`process.env\` and WAS overwritten`)
          } else {
            log(`"${key}" is already defined in \`process.env\` and was NOT overwritten`)
          }
        }
      }
    })

    // 完成process.env对象的赋值后，将解析.env文件数据也返回出去
    return { parsed }
  } catch (e) {
    if (debug) {
      log(`Failed to load ${dotenvPath} ${e.message}`)
    }

    return { error: e }
  }
}

const DotenvModule = {
  config,
  parse
}

module.exports = DotenvModule
```

仔细看一下，dotenv源码其实就只暴露了两个方法:

```js
const DotenvModule = {
  config,
  parse
}

module.exports = DotenvModule
```

### config方法的实现

config的作用上面已经说了，就是读取`.env`的变量配置然后天到`process.env`对象上，源码约60行，整体思路如下：

- 默认使用`utf8`编码格式通过`fs.readFileSync`读取根目录下的`.env`文件
- 如果用户使用了自定义的路径则读取自定义路径的配置文件
- 调用`parse`方法将`.env`中的数据解析成`key/value`形式
- 直接给`process.env`进行赋值
- 赋值过程中如存在重复的key，则根据用户的配置选择是覆盖还是输出日志

具体更详细的逻辑细节看上面的源码部分，这个方法没有复杂的逻辑。

### parse方法实现

parse方法是将符合规则的数据解析成`key/value`的格式。整体逻辑如下：

- 将`.env`的内容按行进行字符分割
- 遍历每一行数据，分割出`key`和`value`
    - 如果`value`部分没有首尾都是单引号或者都是双引号，则`value`取出引号内的部分
    - 如果`value`部分首尾没有单/双引号，直接取值
    - 如果开头是单引号或者双引号，但是结尾没有匹配的引号，则递归下一行继续匹配，直接到匹配到或者到最后，将匹配到的所有值进行字符拼接作为`value`


#### 按行分割字符串

要注意的是如何进行每一行的分割，这里需要考虑不同系统环境的换行符是不同的：

- windows系统换行符 `\r\n`
- Mac系统下换行符 `\r`
- Unix系统下换行符 `\n`

```js
const NEWLINES_MATCH = /\r\n|\n|\r/
```

#### 分割key和value

parse中从字符串分割出key和value的主要逻辑是通过正则匹配的，核心逻辑如下：

```js
const keyValueArr = line.match(RE_INI_KEY_VAL)
if (keyValueArr != null) {
  // 子表达式1匹配到的是=号前面的key
  const key = keyValueArr[1]
  // 子表达式2匹配到的是=号后面的value，不包含行尾#的注释部分
  let val = (keyValueArr[2] || '')
}
```

所以关键就是整个正则是如何实现的，详细的解读包含在了下面的注释里面：

```js
/**
 * 该正则描述的是：
 * 开头的正则 ^\s*([\w.-]+)\s*=描述的是：
 *  - 开头是0-n个空格
 *  - 后面紧跟中英文、下划线、点、连字符
 *  - 后面再紧跟0-n个空格
 *  - 再后面是一个等于号
 * 中间的正则 \s*("[^"]*"|'[^']*'|[^#]*)?描述的的是：
 *  - 在上述的基础上再紧跟0-n个空格
 *  - 后面紧跟（备注1）：
 *    - 收尾是双引号，中间是非双引号的其他0-n个字符
 *    - 或者收尾是单引号，中间是非单引号的其他0-n个字符
 *    - 或者除#号外的0-n个字符
 *  - 最后的问号表示 备注1 的整体都是可选的
 * 最后的正则(\s*|\s*#.*)?$描述的是：
 *  - 最后紧跟0-n个空格 或者 0-n个空格加上#号加上0-n个任意字符
 */
const RE_INI_KEY_VAL = /^\s*([\w.-]+)\s*=\s*("[^"]*"|'[^']*'|[^#]*)?(\s*|\s*#.*)?$/
```

### 日志输出的思想

dotenv中关于日志输出是通过用户的配置来决定是否需要输出，而不是通过判断生产环境/开发环境。这样灵活度更高

```js
// debug来源于用户的options参数配置
if (debug) {
  log(`Failed to load ${dotenvPath} ${e.message}`)
}
```

而根据环境判断是下面这样的：

```js
if (process.env.NODE_ENV !== 'production') {
  log('your log message.')
}
```

具体使用那种，则是根据实际场景仁者见仁智者见智了。