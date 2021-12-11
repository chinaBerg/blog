underscore.js作为一个函数式库，也可以说是个工具库，在日常开发中可以显著的帮我们提示开发效率。当然了，同等的还有lodash等更为流行的库，但是这并不妨碍我们欣赏underscore.js的设计艺术，体验作者强大的代码抽象和函数复用功力。

代码更多的在于分享一些有用、但却容易被忽略的内容或设计逻辑。像一些已经烂大街的函数节流/去抖啊，就不再啰嗦了。  

还等什么呢？开始，宝贝儿～～

### 基础
- 立即执行函数
```
;(function() {
    // 代码内容
})();

// ES6一个文件本身就是一个模块
// 因此可以在ES6中不需要立即执行函数
```
- 实现一个可靠的undefined
```
// js中undefined是可以重写的
function test() {
    var undefined = 1;
    console.log(undefined) // 1
}
test()

// 实现一个可靠的undefined
void(0) // undefined
void 0 // undefined

// 或者jq的方式
;(function(window, undefined){
    // 在立即执行函数中不传递undefined来获取undefined
    console.log(undefined)
})(window)
```
- 环境判断，获取顶层的全局环境
保证在客户端和服务端都可用，要进行环境判断获取不同的顶层全局对象。
```
/**
 * underscore实现
 * 基本的一个判断思路是，如果是客户端则返回客户端的全局对象
 * 如果是服务端，返回服务端的全局对象
 * 否则返回this对象
 * 再否则，返回一个空对象
 */ 
var root = typeof self == 'object' && self.self === self && self ||
            typeof global == 'object' && global.global === global && global ||
            this ||
            {};

// 实现细节 
// 如果self是一个对象，检测self.self是否等于self，如果相等返回self。
typeof self == 'object' && self.self === self && self

// global在node环境中指代全局环境
```

客户端中,以下都表示全局对象window：
```
window
window.window
window.self
self
self.self
self.window
document.defaultView
```

作者采取self来判断，是因为考虑到一些没有窗口的上下文，例如Web Workers。


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/21/16b75d4632033b87~tplv-t2oaga2asx-image.image)

- 对象的创建
```
// 对象类型检测
var isObject = function(obj){
    var type = typeof obj;
    return type === 'function' || type === 'object' && !!type;
}

// 定义用来创建对象的构造函数
var Ctr = function(){};

// Es5原生支持的创建方法
var nativeCreate = Object.create;

// 创建对象
var baseCreate = function (prototype) {
    // 如果传入的参数不是对象，则返回空对象
    if (!isObject(prototype)) return {};
    // 如果是原生支持create方法，则使用原生方法
    if (nativeCreate) return nativeCreate(prototype);
    // 否则，通过Ctr构造函数继承prototype对象后实例化创建一个对象
    Ctr.prototype = prototype;
    var result = new Ctr;
    // 防止内存泄露，用完即销毁
    Ctr.prototype = null;
    return result;
};

// 换成我们可以使用如下方法创建
var baseCreate = Object.create || function(prototype) {
    if (!isObject(prototype)) return {};
    var F = function(){};
    F.prototype = prototype;
    var result = new F();
    F.prototype = null;
    return result;
};
```
- 判断一个对象是否含有一个属性
```
// 使用hasOwnProperty实现判断
var has = function(obj, path) {
    return obj != null && Object.hasOwnProperty.call(obj, path);
};
```
- 利用闭包返回获取特定属性的函数
```
var shallowProperty = function(key) {
    return function(obj) {
        return obj == null ? void 0 : obj[key];
    };
};
 
// 由此可以得到一个获取length属性的函数
var genLength = shallowProperty('length');
```
- 判断类数组
```
// 判断类数组的思想是：
// 该集合拥有length属性且类型为number
// 并且length值>= 0 && <= 数组长度的极限值
var MAX_ARRAY_INDEX = Math.pow(2, 53) - 1;
var isArrayLike = function(collection) {
    var length = getLength(collection);
    return typeof length == 'number' &&
            length >= 0 &&
            length <= MAX_ARRAY_INDEX;
};
```
- 乱序数组和洗牌算法
```
// 先定义一些工具

// 求min-max直接的随机数，含min，不含max
const random = (min, max) => {
    return min + Math.floor(Math.random() * (max - min + 1));
}
// 接收一个值并直接返回
const identity = arg => arg;

// 抽样算法
// 循环样本，不停的从剩余样本中随机抽出一个样本，
// 将抽出的样本与当前样本互换位置，直到满足抽样数量停止
const sample = (arr, n) => {
    var sampleArr = arr.map(identity);
    var index = 0,
        length = sampleArr.length,
        last = sampleArr.length - 1;
    // 最大抽样数为数组长度
    n = Math.max(Math.min(n, length), 0);
    for (; index < n; index++) {
        // 从剩余样本随机抽样
        var tempIndex = random(index, last);
        // 将抽样结果与当前样本互换
        var temp = sampleArr[index]
        sampleArr[index] = sampleArr[tempIndex]
        sampleArr[tempIndex] = temp;
    }
    // 返回抽样结果
    return sampleArr.slice(0, n)
}
sample([1,2,3,4,5,6], 4) // 随机抽出四个

// 洗牌算法
const shuffle = (arr) => sample(arr, Infinity);
```

- 筛选数组中的值有意义的项
```
// 利用filter和Boolean筛选
const arr = [1,2, '', false, undefined, null, NaN, '4'];
arr.filter(Boolean); // [1, 2, "4"]
```


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/21/16b75cdc829db6aa~tplv-t2oaga2asx-image.image)

### 函数相关
- 延迟执行
```
// 延迟执行，就是运用一个定时器，延迟执行函数
var delay = function(func, wait) {
    var args = Array.prototype.slice.call(arguments, 2);
    return setTimeout(function() {
        return func.apply(null, args);
    }, wait);
};

// 注意，在使用时最后先绑定好函数的this作用域，
// 避免之后的this被错误绑定
var log = console.log.bind(console);
delay(log, 1000, '123'); // 1s后输出123
```
- 函数组合
```
// 从右往左执行
var compose = function() {
    var args = arguments,
        start = args.length - 1;
    return function() {
        var i = start;
        var result = args[start].apply(this, arguments);
        while(i--) result = args[i].call(this, result);
        return result;
    }
}

// 执行
var funcA = (str) => str + 'aaaa';
var funcB = (str) => str + 'bbb';
var func = compose(funcB, funcA);
console.log(func('hello, ')); // hello, aaaabbb
```


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/21/16b75d5089953537~tplv-t2oaga2asx-image.image)

### 对象
- 检测对象是否含有某个属性
```
var has = function(obj, key) {
    return (obj !== null && obj !== undefined) &&
        Object.prototype.hasOwnProperty.call(obj, key);
}
var o = {a: 1};
console.log(has(o, 'a')); // true
console.log(has(o, 'b')); // false
```

- Object.keys的profill
```
// MDN上提供的profill
// 在不支持Object.keys的时候进行profill
// 增加了ie9以前for/in不支持Enmus的遍历的bug
if (Object.keys) {
    Object.keys = (function() {
        'use strict';
        var hasOwnProperty = Object.prototype.hasOwnProperty,
            hasDontEnumBug = !({toString: null}).propertyIsEnumerable('toString'),
            dontEnums = [
              'toString',
              'toLocaleString',
              'valueOf',
              'hasOwnProperty',
              'isPrototypeOf',
              'propertyIsEnumerable',
              'constructor'
            ],
            dontEnumsLength = dontEnums.length;
        return function(obj) {
            if (typeof obj !== 'object' && (typeof obj !== 'function' || obj === null)) {
                throw new TypeError('Object.keys called on non-object');
            }
            var result = [],
                prop, i;
            // 循环获取所有自身属性
            for (var prop in obj) {
                if (hasOwnProperty.call(obj, prop)) {
                    result.push(prop);
                }
            }
            // 如果是IE9以前，修复for/in无法遍历Enums类型的属性bug
            if (hasDontEnumBug) {
                for (i = 0; i < dontEnumsLength; i++) {
                    // 即检测，如果obj重写了Enums类型的属性，保证可以遍历到
                    if (hasOwnProperty.call(obj, dontEnums[i])) {
                        result.push(dontEnums[i]);
                    }
                }
            }
            return result;
        }
    })();
}
```

- 对象的扩展
```
// underscore实现了一个很巧妙的扩展函数的工厂函数
// 通过该工厂函数来创建各种不同的扩展函数
var createAssigner = function(keysFunc, defaults) {
    // 参数keysFunc是用来获取keys的函数
    // default表示是否覆盖原属性,，默认覆盖
    return function(obj) {
        var length = arguments.length;
        if (defaults) obj = Object(obj);
        if (length < 2 || obj === undefined || obj === null) return obj;
        // 遍历多有待扩展对象
        for (var index = 1; index < length; index++) {
            var source = arguments[index],
                keys = keysFunc(source),
                l = keys.length;
            // 遍历每个对象的所有属性
            for (var i = 0; i < l; i++) {
                var key = keys[i];
                // 只有default为true或者原属性为default的时候才覆盖
                if (!defaults || obj[key] === void 0) obj[key] = source[key];
            }
        }
        return obj;
    }
}

// 创建一个只扩展自身属性的函数，不包含继承的
// 这里Object.keys只是为了简化，underscore中有_.keys实现来实现同样的效果
var extendOwn = assign = createAssigner(Object.keys);
console.log(extendOwn({a: 1, b: 2}, {c: 1}, {d: 2}));

// 扩展所有属性，包含继承的
var entends = createAssigner(_.allKeys)
```

- Object.prototype.hasOwnProperty
```
// 这里想说的是，可以看到很多库都会使用下面这种方式，而不是obj.hasOwnProperty
// 原因是防止用户的对象重载了该属性，从而导致报错
// 因为js并没有把hasOwnProperty作为关键词被平板2
Object.prototype.hasOwnProperty
```

- 数据类型检测
```
// 以下数据类型都可以通过
// Object.prototype.toString的方法监测数据类型
// 和underscore的检测原理是一样的，实现方法不一样
'Arguments',
'Function',
'String',
'Number',
'Date',
'RegExp',
'Error',
'Symbol',
'Map',
'WeakMap',
'Set',
'WeakSet'

// 写一个类型检查函数的工厂函数
var createTypeCheck = function(name) {
    return function(obj) {
        return Object.prototype.toString.call(obj) === '[object ' + name + ']';
    }
}

// 通过工厂函数创建类型检查函数
var isNumber = createTypeCheck('Number');
console.log(isNumber(123), isNumber(''));

var isString = createTypeCheck('String');
console.log(isString(123), isString(''));

var isObject = createTypeCheck('Object');
console.log(isObject(123), isObject(''), isObject({}));

// 数组检查，可以优先使用原生的方法
// 也正体现了该写法比underscore的灵活性
var isArray = Array.isArray || createTypeCheck('Array');
console.log(isArray([]), isArray({})); // true false

// 其他同理
……

// 检查是否是元素
var isElement = function (obj) {
  return !!(obj && obj.nodeType === 1);
};

// undefined检查
var isUndefined = function(obj) {
    return obj === void 0;
}
```


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/21/16b75ce39bd9495b~tplv-t2oaga2asx-image.image)

- 解决冲突
```
// 解决库的命名冲突
// 像underscore库的_，很容易与其他lodash等库冲突，还有jq/zepto的$等
// 因此提供了noConflict方法来解决冲突
// 核心思想就是交还原库的控制权，重新命名
_.noConflict = function() {
    // 交还原库的控制权
    root._ = previousUnderscore;
    // 返回this可以给该库重新命名
    return this;
};

// 例如
var underscore = _.noConflict();
```

- 随机数
```
// 生成 [min, max] 直接是随机数，包含min，不含max
// 如果只传递一个参数，则生成[0, max]直接的随机数
var random = function(min, max) {
    if (max == null) {
        max = min;
        min = 0;
    }
    return min + Math.floor(Math.random() * (max - min))
}
```

- 缓存常量
```
// 天了撸，利用闭包，绕了一大圈，
// 结果只是把值原封不动的返回了。
// 看似无用，实则可以帮我们缓存当时的值
var memoConstant = function(value) {
    return function() {
        return value;
    };
};

// 实例
var a = 1;
var constantA = memoConstant(a);
a = 2;
console.log(constantA())
```

### 参考资料
- [undersercore 源码分析](https://legacy.gitbook.com/book/yoyoyohamapi/undersercore-analysis/details)
- [underscore源码](https://github.com/jashkenas/underscore)
- [underscore中文文档](https://www.html.cn/doc/underscore1.8.2/#)


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/21/16b75ce8b203fe29~tplv-t2oaga2asx-image.image)

### By End

> 百尺竿头、日进一步  
> 我是愣锤，一名前端爱好者  
> 欢迎批评与交流  

本期配图主题：百变女神陈钰琪小姐姐。
喜欢的点个赞吧～～～