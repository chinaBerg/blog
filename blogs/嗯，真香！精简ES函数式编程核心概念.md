不过，不得不说，很多时候，函数式编程，是真香！本文也旨在以最简要的文笔，阐述函数式最核心的概念。  

JS本身支持多范式编程，比较熟知的面向对象以及愈发流行的函数式编程。本质并不是新的知识点，而是一种编程理念。这些范式各有各的特点，我们学习使用应该融会贯通，而不是互相排斥。合适的地方使用合适的技术做合适的事，才是更优解。  

下面，来看下函数式的一些基本原则概念：

### 原则、特点、概念
- 函数的第一条原则是小，第二条原则是更小。
- 函数式编程的世界中，没有外部环境的依赖，没有状态，没有突变。
- 函数相同的输入，必须得到相同的输出。这也被成为引用透明性。
- 函数式编程主张编写声明式和抽象的代码，而不是命令式。命令式是告诉编译器“如何去做”，声明式侧重于告诉编译器“做什么”。

### 纯函数
纯函数是相同的输入得到相同输出的函数。
- 纯函数不应该依赖任何外部变量，也不应该改变任何外部变量。
- 纯函数 第一个好处是容易测试。
- 纯函数必须有一个有意义的名称。
- 组合是函数式编程范式的核心

### 高阶函数
- 函数可以作为参数传递，称为高阶函数（简称HOC）
- 函数可以被其他函数返回。
```
// 函数可以作为参数传递
var fn = func => typeof func === 'function' && func();
var log = () => console.log('hello HOC');
fn(log)

// 函数可以被其他函数返回
var fn2 = () => log;
var func2 = fn2();
func2()
```
- 通过高阶函数实现抽象
```
// unless函数只在断言为false的时候执行函数
const unless = (predicate, fn) => {
    if (!predicate) fn();
}
// 所以我们很方便的求出一个数组中的偶数
const isEven = e => e % 2 === 0;
const isOdd = e => !isEven(e);
[1,2,3,4,5,6].forEach(e => unless(isOdd(e), () => console.log(e)))

// 定义times函数，指定函数执行n次
const times = (times, fn) => {
     for(var i = 0; i < times; i++) fn(i)
}
times(100, e => unless(isOdd(e), () => console.log(e)))
```
- 真正的高阶函数：some/every/map/filter/reduce/sort等
```
// 定义一个sortBy函数作为通用的数组排序的参数
// 根据某个属性返回一个从小到大排序的函数，作为sort的参数
const sortBy = (property) => (a, b) => a[property] > b[property] ? 1 : a[property] === b[property] ? 0 : -1;
const arr = [{name: '小猫', age: 5}, {name: '小狗', age: 1}];
arr.sort(sortBy('age'));
console.log(arr);
```

### 闭包
- 闭包的强大之处在于，可以访问其外部函数变量，使得函数的作用域得以延伸。
- 定义unary函数，将一个接收多个参数的函数转换成只接收一个函数的参数
```
// unary函数
const unary = fn => fn.length === 1 ? fn : (arg) => fn(arg);

const arrInt = [1,2,3].map(parseInt); // [1, NaN, NaN]
const arrInt2 = [1,2,3].map(unary(parseInt)); // [1, 2, 3]
```
- memoized缓存函数
```
// 纯函数的输出只依赖输入，所以可以对其做缓存操作
// 阶乘函数只依赖输入，所以是纯函数
const factorial = n => {
    if (n === 1) return 1;
    return n * factorial(n - 1);
}

// 定义memoized缓存函数
const memoized = fn => {
    const cache = {};
    return function (arg) {
        if (cache.hasOwnProperty(arg)) return cache[arg];
        return cache[arg] = fn(arg);
    }
}
// 定义阶乘的缓存函数
const memoFactorial = memoized(factorial);

// 调用
console.time('one');
memoFactorial(1000);
console.timeEnd('one'); // one: 0.115966796875ms

console.time('two');
memoFactorial(1000);
console.timeEnd('two') // two: 0.02490234375ms
```
- zip函数，用于将两个数组合并成一个数组
```
const zip = (arrLeft, arrRight, fn) => {
    let result = [];
    let index = 0;
    let maxLength = Math.max(arrLeft.length, arrRight.length);
    for (; index < maxLength; index++) {
        const res = fn(arrLeft[index], arrRight[index]);
        result.push(res);
    }
    return result;
}
zip([1,23,4], [2,4,5], (a, b) => a + b) // [3, 27, 9]
```

### 柯里化与偏应用
- 只接收一个参数的成为一元函数，接收两个参数称为两元参数，接收多个参数的称为多元函数。接收不确定参数的称为变参函数。
- 柯里化是把一个多参数函数转换成一个嵌套的一元函数的过程。
```
// 定义柯里化函数
const curry = (fn) => {
    return function curryFunc(...arg) {
        if (arg.length < fn.length) {
            return function () {
                return curryFunc.apply(null, [...arg, ...arguments]);
            };
        }
        return fn.apply(null, arg);
    }
};
const func = (a, b) => console.log(a - b);
curry(func)(1)(2)
```
- 偏应用，允许开发者部分的应用函数
```
// 定义偏应用
// 只当partial时后续参数为udefined时才使用对应的实参替换
const partial = (fn, ...args) => {
    return function (...last) {
        let i = 0;
        const argsLen = args.length;
        const lastLen = last.length;
        for (; i < argsLen && i < lastLen; i++) {
            args[i] === undefined && (args[i] = last[i]);
        }
            return fn.apply(null, args);
        }
    }
const timer3s = partial(setTimeout, undefined, 3000)
timer3s(() => console.log('hello')) // 3s后输出hello
// bug原因在于undefined已经被替换掉了，后面再调用时发现没有undefined便不会再替换
timer3s(() => console.log('hello2')) // 依旧输出hello，而不是hello2
timer3s(() => console.log('hello3'))
```

### 组合和管道
- Unix理念，一个程序只做好一件事情，如果要完成一项新的任务，重新构建要好于在旧程序中添加新属性。（理解为，单一原则，如果要完成新的任务，重新组合多个小的功能比改造原有的一个程序要好)
- 组合的概念，一个函数的输出作为另一个函数的输入，从右到左依次传递下去，该过程就是组合。
```
// 定义组合函数
const compose = (...fns) => (val) => fns.reverse().reduce((acc, fn) => fn(acc), val);
 
// 定义一系列小的函数   
const splitStr = str => str.split(' ');
const getArrLen = arr => arr.length;
// 组合并输出
const getWords = compose(getArrLen, splitStr);
getWords('I am LengChui!') // 3
```
- 组合中，每一个函数只接收一个参数。如果现实中一个函数需要多个参数，可以利用curry和partil。
- 管道，和组合的功能一样，不过是从左到右的顺序。只是个人喜好问题。
```
// 定义管道函数
const pipe = (...fns) => val => fns.reduce((acc, fn) => fn(acc), val);
// 可以达到和compose同样的输出
const getWords2 = pipe(splitStr, getArrLen);
getWords2('I am LengChui!')
```
- 管道和组合函数发生错误时的定位
```
// 定义identity函数，将接收到的参数打印输出
const identity = arg => {
    console.log(arg);
    return arg;
}
// 在需要的地方直接插入即可
const getWords2 = pipe(splitStr, identity， getArrLen);
```

### 函子
- 函子是一个普通的对象，它实现了map函数，在遍历每一个对象值的时候生成一个新的对象。简言之，函子是一个持有值的容器。
```
// 函子其实就是一个持有值的容器
const Containter = function (value) {
    this.value = value;
}
// of静态方法用来生成Container实例，省略new而已
Containter.of = function (value) {
    return new Containter(value)
}
Containter.prototype.map = function (fn) {
    return Containter.of(fn(this.value));
}

// 可以简化一下(省略of)
const Containter = function (value) {
    if (!(this instanceof Containter)) return new Containter(value);
    this.value = value;
}
Containter.prototype.map = function (fn) {
    return Containter.of(fn(this.value));
}

// es6写法
class Containter {
    constructor (value) {
        this.value = value;
    }
    // 静态方法of返回类实例
    static of(value) {
        return new Containter(value);
    }
    // map函数允许Container持有的值调用任何函数
    map(fn) {
        return Containter.of(fn(this.value));
    }
}
console.log(Containter.of(123).map(e => 2 * e)
    .map(e => e + 1).value
) // 247
```
- Maybe函子，是一种对错误处理的强大抽象。
```
// 定义Maybe函子，和普通函子的区别在于map函数
// 会对传入的值进行null和undefined检测
class Maybe {
    constructor(value) {
        this.value = value;
    }
    static of(value) {
        return new Maybe(value);
    }
    isNothing() {
        return this.value === undefined || this.value === null;
    }
    // 检测容器持有值是否为null或undefined，如果是则直接返回null
    map(fn) {
        if (this.isNothing()) return Maybe.of(null);
        return Maybe.of(fn(this.value));
    }
}
// 可以保证程序在处理值为null或undefinede的时候不至于崩溃
// eg1
const res = Maybe.of(null).map(e => null).map(e => e - 10);
console.log(res);
// eg2
const body = {data: [{type: 1}]};
const typeAdd = e => {
    e.type && e.type ++;
    return e;
}
const res = Maybe.of(body).map(body => body.data)
    .map(data => data.map(typeAdd))
console.log(res)
```
> MayBe函子能轻松处理所有null和undefined错误  
> 但是MayBe函子不能知道错误来自于哪里。  

- Either函子，能解决分支扩展问题
```
// ES6方式实现
class EitherParent {
    constructor(value) {
        this.value = value;
    }
    // 子类会继承该方法
    static of(value) {
        // new this.prototype.constructor使得返回的实例是子类
        // 这样子类调用of方法后才可以继续链式调用
        return new this.prototype.constructor(value);
    }
}
class Nothing extends EitherParent {
    constructor(...arg) {
        super(...arg)
    }
    map() {
        return this;
    }
}
class Some extends EitherParent {
    constructor(...arg) {
        super(...arg)
    }
    map(fn) {
        return new Some(fn(this.value))
    }
}

// 实例使用
function getData() {
    try {
        throw Error('error'); // 模拟出错
        return Some.of({code: 200, data: {a: 13}})
    } catch (error) {
        return Nothing.of({code: 404, message: 'net error'})
    }
}
console.log(getData().map(res => res.data).map(data => data.a))
```
> MayBe和Either都是Pointed函子  

### Monad函子
- Monad就是一个拥有chain方法的函子
- 类似于MayBe函子，
```
class Monad {
    constructor(value) {
        this.value = value;
    }
    static of(value) {
        return new Monad(value);
    }
    isNothing() {
        return this.value === undefined || this.value === null;
    }
    // 用于扁平化MayBe函子，但是只扁平一层
    join() {
        if (this.isNothing()) return Monad.of(null);
        return this.value;
    }
    // 直接将map后的join扁平操作封装在chain方法中
    // 使得更简便的调用
    chain(fn) {
        return this.map(fn).join();
    }
    map(fn) {
        if (this.isNothing()) return Monad.of(null);
    return Monad.of(fn(this.value));
    }
}
console.log(Monad.of(123).chain(e => {
    e += 1;
    return e;
}))
```

### 内容参考
- ES6函数式编程入门经典

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/9/16b3c2f4ec791acf~tplv-t2oaga2asx-image.image)

> 百尺竿头、日进一步  
> 我是愣锤，一名前端爱好者。  
> 欢迎批评与交流。