# mime源码分析

> version 3.0.0
> 愣锤 2022/03/21

[mime](https://github.com/broofa/mime)是Js的一个非常丰富的`MIME type`的模块。例如你可以根据文件扩展名获取`MIME type`, 也可以根据`MIME type`获取扩展名。下面演示基本的使用例子：

```js
const mime = require('./index');

// 通过extension获取MIME
const cssMimeType = mime.getType('css');
// 通过path获取MIME
const cssMimeType2 = mime.getType('dir/demo.css');

// text/css
console.log(cssMimeType);
// text/css
console.log(cssMimeType2);

// 通过MIME获取extension
const cssExt = mime.getExtension(cssMimeType);
// css
console.log(cssExt);
```

### 基础知识

**媒体类型**（通常称为 `Multipurpose Internet Mail Extensions` 或 `MIME` 类型 ）是一种标准，用来表示文档、文件或字节流的性质和格式。它在`IETF RFC 6838`中进行了定义和标准化。

MIME的语法结构为：

```bash
type/subtype
```

- 由类型与子类型两个字符串中间用`/`分隔而组成
- `type` 表示可以被分多个子类的独立类别。`subtype` 表示细分后的每个类型
- 对大小写不敏感，但是传统写法都是小写
- 不允许空格存在

常见的重要的MIME类型有：

- `application/octet-stream` 应用程序文件的默认值，浏览器一般不会自动执行或询问执行。浏览器会像对待 设置了HTTP头 `Content-Disposition` 值为 `attachment` 的文件一样来对待这类文件。
- `text/plain` 文本文件默认值，即使它意味着未知的文本文件，但浏览器认为是可以直接展示的。
- `text/css` 在网页中要被解析为CSS的任何`CSS`文件必须指定`MIME`为`text/css`，特别要注意为`CSS`文件提供正确的MIME类型。
- `text/html` 所有的HTML内容都应该使用这种类型
- `text/javascript` 据 HTML 标准，应该总是使用 MIME 类型 text/javascript 服务 JavaScript 文件。其他值不被认为有效，使用那些值可能会导致脚本不被载入或运行

所有的MIME类型请[查询这里](https://www.iana.org/assignments/media-types/media-types.xhtml)

### 源码分析

`mime`库的源文件如下图所示：

![image](https://note.youdao.com/yws/res/20396/B6B84C7FC454485ABBA1EEDA5AC6EF3F)

- 红色框中的部分是完整实现，其中`index.js` 入口文件，`types`文件夹下是原始数据字典
- 绿色部分是简易版实现，`lite.js`是入口文件
- 蓝色部分是命令行实现，就一个`cli.js`文件

下面我们先看完整的mime实现，打开`index.js`文件：

```js
'use strict';

// 导入Mime类
let Mime = require('./Mime');

// 对外暴露一个Mime的实例
// 类的参数分别是两个mime相关的原数据字典
module.exports = new Mime(require('./types/standard'), require('./types/other'));
```

接下来我们看`Mime`的实现：

```js
'use strict';

/**
 * @param typeMap [Object] Map of MIME type -> Array[extensions]
 * @param ...
 */
function Mime() {
  /**
   * _types字段存储 ext -> mimeType的映射
   * {
   *   css: 'text/css',
   *   html: 'text/html',
   *   htm: 'text/html',
   *   shtml: 'text/html',
   * }
   */
  this._types = Object.create(null);
  /**
   * _extensions存储 mimeType -> ext的映射
   * TODO：在数据字典中ext的值为多个值，只取第一个值与mimeType绑定
   * 且第一个值如果是*开头的要去的*
   * {
   *   'text/css': 'css',
   *   'text/html': 'html', // 数据字典中 "text/html": ["html", "htm", "shtml"]
   *   'text/rtf': 'rtf', // 数据字典中 "text/rtf": ["*rtf"]
   * }
   */
  this._extensions = Object.create(null);

  // 如果初始化时传递了参数则多为define方法的参数进行新增mime类型
  for (let i = 0; i < arguments.length; i++) {
    this.define(arguments[i]);
  }

  // 绑定this作用域
  this.define = this.define.bind(this);
  this.getType = this.getType.bind(this);
  this.getExtension = this.getExtension.bind(this);
}

/**
 * 定义 mimetype -> extension的映射
 */
Mime.prototype.define = function(typeMap, force) {
};

/**
 * 通过path或者extension获取mimetype
 */
Mime.prototype.getType = function(path) {
};

/**
 * 通过mimetype获取默认的extension
 */
Mime.prototype.getExtension = function(type) {
};

module.exports = Mime;
```

`Mime`类包含的主要属性和方法如下：

- `_types`属性，存储 `ext -> mimeType` 的映射
- `_extensions`属性，存储 `mimeType -> ext` 的映射
- `define`方法，用于定义 `mimetype -> extension` 的映射
- `getType`方法，通过`path`或者`extension`获取`mimetype`
- `getExtension`方法，通过`mimetype`获取默认的`extension`

`Mime`构造函数在实例化的主要逻辑就是遍历所有的参数，每个参数都是一份`MIME`类型相关的字典数据，对每一项字典调用`define`方法生成`ext -> MIME`以及`MIME -> ext`的映射关系。传递给`Mime`类的参数数据格式，我们可以看下`standard.js`文件：

```js
module.exports = {
  "application/andrew-inset": ["ez"],
  "application/applixware": ["aw"],
  "application/atom+xml": ["atom"],
  // ....其他更多数据
  "text/html": ["html", "htm", "shtml"],
  "text/jade": ["jade"],
  "text/jsx": ["jsx"],
  "text/less": ["less"],
  "text/markdown": ["markdown", "md"],
  // ....其他更多数据
}
```

这里要注意的是多种文件类型都可能映射到同一个`MIME`，因此`MIME`的值是一个`extension`数组。


我们继续看`define`方法的是吧，探究一下定义时做了什么事情：

```js
/**
 * 定义 mimetype -> extension的映射。
 * 每一个key都是mimetype，值是该mimetype相关的extension数组。
 * extension数组的第一个值作为该mimetype的默认extension值。
 *
 * e.g. mime.define({'audio/ogg', ['oga', 'ogg', 'spx']});
 *
 * 定义时，如果一个extension已经被定义过了，则会抛出一个错误。
 * 可以通过设置第二个参数`force`的值为true来强行覆盖。
 *
 * e.g. mime.define({'audio/wav', ['wav']}, {'audio/x-wav', ['*wav']});
 *
 * @param map (Object) 类型定义数据
 * @param force (Boolean) 如果值为true，则强行覆盖原有定义
 */
Mime.prototype.define = function(typeMap, force) {
  // 遍历类型定义数据对象
  for (let type in typeMap) {
    // 获取当前mimetype相关的extension集合
    let extensions = typeMap[type].map(function(t) {
      return t.toLowerCase();
    });
    type = type.toLowerCase();

    // 遍历extensions，将每一项作为key，对应的mimetype作为value绑定映射关系
    for (let i = 0; i < extensions.length; i++) {
      const ext = extensions[i];

      // '*' prefix = not the preferred type for this extension.  So fixup the
      // extension, and skip it.
      if (ext[0] === '*') {
        continue;
      }

      // 如果该扩展已被定义过则直接抛出
      if (!force && (ext in this._types)) {
        throw new Error(
          'Attempt to change mapping for "' + ext +
          '" extension from "' + this._types[ext] + '" to "' + type +
          '". Pass `force=true` to allow this, otherwise remove "' + ext +
          '" from the list of extensions for "' + type + '".'
        );
      }

      this._types[ext] = type;
    }

    // 把extension集合的第一项作为mimetype的默认extension
    if (force || !this._extensions[type]) {
      const ext = extensions[0];
      this._extensions[type] = (ext[0] !== '*') ? ext : ext.substr(1);
    }
  }
};
```

`define`的逻辑基本都写在注释里了，核心就是遍历字典数据，然后打平字典数据成`MIME`和`Ext`的一对一映射关系。以`text/html`为例子就是:

```js
// 原始的html MIME数据格式
{
  "text/html": ["html", "htm", "shtml"],
}

// 打平后的_types
{
  "html": "text/html",
  "htm": "text/html",
  "shtml": "text/html"
}
// 打平后的_extensions
{
  "text/html": "html",
}
```

打平数据后，我们再看下是如何获取MIME类型的吧。`getType`方法实现如下所示：

```js
/**
 * 通过path或者extension获取mimetype
 */
Mime.prototype.getType = function(path) {
  path = String(path);
  /**
   * last 获取basename
   * TODO：这里没有通过path.basename获取的原因在于
   *  要支持'dir\\text.txt'的参数类型
   * EG：css => css, text/css => css，dir\\text.txt => text.txt
   */
  let last = path.replace(/^.*[/\\]/, '').toLowerCase();
  /**
   * ext 根据last获取到对应的ext扩展名
   * EG: css => css, a.css => css, a.css.css => css
   */
  let ext = last.replace(/^.*\./, '').toLowerCase();

  // last长度小于path长度，说明是传递的路径参数，而不是扩展名参数
  let hasPath = last.length < path.length;
  // ext长度小于last，说明是明确了带.的扩展名
  let hasDot = ext.length < last.length - 1;

  // 如果是直接传递的扩展名参数，例如 css
  // 或者是传递的带扩展名的路径，例如dir/demo.css
  // 则返回对应的mimeType，否则返回null
  return (hasDot || !hasPath) && this._types[ext] || null;
};
```

`getType`方法主要通过传入的`path`或者`extension`参数，解析出其中的`extension`值，然后根据之前的`_types`字典数据取出对应的`MIME`值。需要注意的一点，这里获取`basename`并没有使用`path.basename`方法，是因为要支持`dir\\deno.txt`这种格式的路径。

最后我们看下如何根据MIME获取extension吧。`getExtension`实现如下：

```js
/**
 * 通过mimetype获取默认的extension
 */
Mime.prototype.getExtension = function(type) {
  type = /^\s*([^;\s]*)/.test(type) && RegExp.$1;
  return type && this._extensions[type.toLowerCase()] || null;
};
```

这里的逻辑是根据MIME的值返回对应的extension，但是核心在于这个正则是如何获取MIME的type的。

```js
// 正则表示，前后可以有0-n个空格，中间匹配除去分号和空格之外的任意0-n个字符
// 最终返回的是RegExp.$1，即第一个子表达式的内容
// 也就是返回小括号内匹配到的内容
type = /^\s*([^;\s]*)/.test(type) && RegExp.$1;
```

按理说传入的就是例如`text/css`这样的值，为什么还要做这些处理呢？大家可以看下，比如`html`的`MIME`很多时候是这样的`text/html; charset=utf8`，会带有编码相关的内容，因此这个正则也就是为了处理这些情况，值获取`text/html`的部分。

### 简易版的实现

`mime`库的作者对外暴露的`lite.js`用于提供简易版的`mime`库，其实现都在`lite.js`中：

```js
'use strict';

let Mime = require('./Mime');
module.exports = new Mime(require('./types/standard'));
```

可以看到和完整版本的区别就在于丢掉了`other`的`MIME`数据。

### 命令行的实现

`mime`库的命令行实现都在`cli.js`中，完整实现如下：

```js
#!/usr/bin/env node

'use strict';

process.title = 'mime';
let mime = require('.');
let pkg = require('./package.json');
// 获取命令行脚本参数
let args = process.argv.splice(2);

// 获取版本
if (args.includes('--version') || args.includes('-v') || args.includes('--v')) {
  console.log(pkg.version);
  process.exit(0);
// 获取库的名称
} else if (args.includes('--name') || args.includes('-n') || args.includes('--n')) {
  console.log(pkg.name);
  process.exit(0);
// 获取库的帮助信息
} else if (args.includes('--help') || args.includes('-h') || args.includes('--h')) {
  console.log(pkg.name + ' - ' + pkg.description + '\n');
  console.log(`Usage:

  mime [flags] [path_or_extension]

  Flags:
    --help, -h                     Show this message
    --version, -v                  Display the version
    --name, -n                     Print the name of the program

  Note: the command will exit after it executes if a command is specified
  The path_or_extension is the path to the file or the extension of the file.

  Examples:
    mime --help
    mime --version
    mime --name
    mime -v
    mime src/log.js
    mime new.py
    mime foo.sh
  `);
  process.exit(0);
}

// 获取文件参数
let file = args[0];
// 调用mime获取MIME
let type = mime.getType(file);

// 输出到终端
process.stdout.write(type + '\n');
```

主要逻辑比较简单，就是获取命令行参数，输出相关信息，通过命令行的文件参数，调用`mime`库获取`MIME`，然后通过`process.stdout.write`写入到控制台。

