# MVVM时代下仍需掌握的DOM - 基础篇

2019年07月18日 愣锤

在当前MVVM大行其道的环境下提到DOM一词，很多人可能会感到有些诧异。这种差诧异或许来自于类似“都什么年底了还操作DOM啊”的声音！说的没错，MVVM时代，虚拟dom东征西战，一枝独秀，着实不可否认其强大的威力。  

然而，DOM操作作为前端的基础，自诞生以来便左右着我们的页面效果。随着JQuery十年戎马生涯的落幕，DOM似乎暗淡了许多，但其在前端中的左右却从未动摇。即使是MVVM框架下的例如ElementUl/IView等等最流行的ui库，打开他们的源码，依旧会有类似的dom.js/event.js等工具集（这里暂且称其工具函数集合吧），这些ui库里面，避免不了基本的事件绑定啊/添加移除类啊等等。

不管任何时候，DOM依旧是前端必须掌握且需要投入一定时间研究的基础。不能只停留在jq事件的dom操作或者只是掌握那几个最常见的api。

👇下面开始有趣的DOM之旅吧！


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/5/6/16a88e53da01626d~tplv-t2oaga2asx-image.image)

### DOM
DOM全称Document Object Modal 文档对象模型

### Node
Node是js的构造函数，所有节点都从Node上继承最常见的属性和方法，例如：
- childNodes/firstChild/nodeName/nodeType等
- appendChild()/cloneNode()/removeChild()等
- 其他更多

### 节点类型
```
document.nodeType // 文档节点，9
document.doctype.nodeType // 文档类型声明节点，10
document.createElement('a').nodeType // 元素节点，1
document.createDocumentFragment().nodeType // 11
document.createTextNode('aaa').nodeType // 文本节点，3
```

### 节点的值
可以通过节点的`nodeValue`属性获取节点的值，但是除了`Text`和`Comment外`，其余节点基本都返回`null`

### 创建节点
```
// 创建元素节点，例如div
document.createElement('div')

// 创建文本节点
document.createTextNode('a text')

// 创建注释节点
document.createComment('a comment 节点')
```

### 插入元素或文本
```
// 替换#app内部的内容
document.getElementById('app').innerHTML = '<div>asdasdasd</div>'

// 替换#app及其内容，本身也会被替换掉
document.getElementById('app').outerHTML = '<div>asdasdasd</div>'

// 创建一个文本节点，并替换#app内的内容
document.getElementById('app').textContent = 'a text'

---
上面这些方法，如果不是赋值，而是直接作为属性取值，则会返回取到的节点字符串
---

var app = document.getElementById('app')
// 在#app开始标签之前插入，#app需要有父节点
app.insertAdjacentHTML('beforebegin', '<span>hello</span>')
// 在#app开始标签之后插入
app.insertAdjacentHTML('beforeend', '<span>beforeEnd</span>')
// #app结束标签之前插入
app.insertAdjacentHTML('afterbegin', '<span>afterbegin</span>')
// 在#app结束标签之后插入，#app需要有父节点
app.insertAdjacentHTML('afterend', '<span>afterEnd</span>')
```

### 插入节点
可以通过`appendChild()`和`insertBefore()`插入节点
```
// 插入节点
var div = document.createElement('div')
app.appendChild(div)
```

insertBefore控制插入的位置，第一个参数是待插入节点，第二个参数是插入位置（即一个插入这个节点的前面，类似于一个参考节点）
```
// 将div节点插入到#app的第二个p节点的前面
app.insertBefore(div, p[1])

// 如果忽略第二个参数，则和appendChild一样，默认插入到最后面
app.insertBefore(div)
```

### 移除节点/替换节点
移除一个节点，首先要找到该节点的父节点，然后在父节点上调用removeChild方法。
```
// 移除第二个p节点
p[1].parentNode.removeChild(p[1])
```
替换节点，先找到父节点，然后在父节点调用replaceChild方法，接收两个参数，第一个为新节点，第二个是待替换的节点
```
// 将第二个p节点替换成一个newdiv节点
p[1].parentNode.replaceChild(newdiv, p[1])
```
> 注意：这两个方法会返回被移除或替换的节点。该操作只是将节点从文档中移除，并不是真正的删除，其依旧存在于内存中，我们仍可以持有其引用。

### 克隆节点
- node.cloneNode()方法用来克隆节点，接收一个参数，如果为true则克隆该节点及其子节点，如果为false则只克隆该节点。  
- 该方法会克隆节点的所有属性和内联事件，但是不会克隆addEventListener或node.onclick等形式添加的事件。
```
// 克隆#app节点
app.cloneNode(false)

// 克隆#app及其子节点
app.cloneNode(true)
```
> 目前其默认值是false，但是DOM4规范其默认行为发生了变化为true。所以考虑到兼容，必须传参数使用。  

### childNodes
返回一个类数组包含所有直属子节点（包括文本节点/注释节点）
```
/// 返回#app的所有直属子节点
app.childNodes

// 验证该节点集合是实时的，而不是某一时刻的快照
var ns = app.childNodes
app.innerHTML = ""
console.log(ns) // #app被清空后，虽然之前定义了引用，这里依旧输出了空数组，因为是实时的。

// 可以借用数组方法将类数组转换成数组, es5:
Array.prototype.forEach.call(ns)
// es6中可以使用Array.from()
Array.from(ns).forEach(e => console.log(e))
```
> childNodes是实时的，而不是某一时刻的快照  
> html标签的换行会有文本节点产生，所以childNodes也会包含该文本节点。需要注意现代化开发的压缩代码，所以后续更多的只使用元素节点。

### 遍历DOM节点

普通节点
- app.parentNode 父节点
- app.firstChild 第一个子节点
- app.lastChild  最后一个子节点
- app.nextSibling 上一个兄弟节点
- app.previousSibling 下一个兄弟节点

元素节点
- app.parentElement 父元素节点
- app.children 所有子元素节点
- app.firstElementChild 第一个子元素节点
- app.lastElementChild 最后一个子元素节点
- app.nextElementSibling 上一个元素节点
- app.previousElementSibling 下一个元素节点

### 判断节点是否包含另一个节点
调用节点的contains方法，可以判断该节点是否包含参数节点，包含则返回true，否则false：
```
// #app是否包含p[1]这个节点
app.contains(p[1])
```

### 判断节点是否相等
具备以下条件，节点才相等：
- 节点类型相等
- 这些属性相等： nodeName/localName/namespaceURI/prefix/nodeValue
- attributes NameNodeMaps相等
- childNodes NodeLists相等
可以通过节点的isEqualNode方法判断

```
<input type="text">
<input type="text">
  
var ipts = document.querySelectorAll('input')
ipts[0].isEqualNode(ipts[1])

// 如果只是想判断是否是同一个节点引用，则可以使用全等运算符
ipts[0] === ipts[0]
```

### document下的节点
```
var doc = document

doc.doctype // 指向<!DOCTYPE>
doc.documentElement // 指向<html lang="en">
doc.head // 指向<head>
doc.body // 指向<body>
```

### 获取文档中聚焦/激活状态的元素引用

```
// 返回文档中聚焦或者激活状态的节点
document.activeElement

// 判断文档是否有激活或聚焦状态的节点，返回true/false
document.hasFocus()
```

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/5/6/16a88e5d654648b7~tplv-t2oaga2asx-image.image)

### 全局对象
可以通过`document.defaultView`获取顶部的对象（全局对象），在浏览器中全局对象是window，`document.defaultView`指向的是这个值，在非浏览器环境则访问到的是顶部对象的作用域。

### 元素节点
```
// 创建,接收一个参数，即元素类型tagName，元素节点的tagName和nodeName的一样。
// 传入的值在被创建元素前都会被转换成小写。
document.createElement('div')

// 获取元素标签名,返回的都是大写
var div = document.createElement('div')
div.nodeName // DIV
div.tagName // DIV
```

### 获取元素属性与值的集合
该属性是实时的类数组
```
doc.getElementById('txt').attributes
```

### 操作元素的属性节点
```
<a href="http://www.baidu.com" id="a" data-other="other prop">百度网</a>

var a = document.getElementById('a')

// 获取属性节点
a.getAttribute('href')
a.getAttribute('data-other')

// 设置属性节点
a.setAttribute('data-src', 'src string')

// 移除属性节点
a.removeAttribute('href')

// 监测元素是否含有某个属性节点
a.hasAttribute('href')
```
> getAttribute如果没取到则返回null  
> setAttribute必须传2个参数  
> hasAttribute不管这个属性有没有值，都返回true

### 元素类名
可以通过`a.className`或`a.classList`获取元素类名。  
className：
- 如果没有类名，返回""
- 类名会原样返回字符串，即使前后都有空格等
- 更改通过对其进行重新赋值

classList：
- ie9不支持
- 可以通过className模拟实现，有类似等profill库
- 有add/remove/contains/toggle等方法
```
// 添加
a.classList.add('f')
// 移除
a.classList.remove('a')
// 有则移除，无则添加
a.classList.toggle('e')
a.classList.toggle('b')
// 监测是否有某个类名
a.classList.contains('c')
```

### data-属性
```
// 获取data-属性：a.dataset.属性名，不存在则返回undefined
a.dataset.other

// 设置
a.dataset-other2 = 'data2'
```
> dataset在ie9中不支持，不过完全可以依旧使用getAttribute等属性使用


### 选择器
```
// id选择器
document.getElementById('app')

// 返回符合条件的首个元素节点
document.querySelector('#app')

// 返回符合条件的元素节点列表
document.querySelectorAll('li')

// 返回符合条件的标签列表
document.getElementsByTagName('div')

// 返回符合条件类名的节点
document.getElementsByClassName('flex1')
```
> querySelectorAll、getElementsByTagName、getElementsByClassName都是实时的，而不是快照。  
> 这些方法都可以作用在节点上，从而在上下文中进行局部查找。

```
// children: 查找所有直接子元素
document.querySelector('ul').children

// html文档中方便使用的类数组列表
document.forms // 获取文档中所有的表单
document.images // 获取文档中所有的图片
document.links // 获取文档中所有的a标签
document.scripts // 获取文档中所有的scripts
document.styleSheets // 获取文档中所有的link和style
```

### 元素偏移量  
首先普及offsetParent概念：一个元素的祖元素中第一个position值不为static的那个元素。
- `offsetTop`与`offsetLeft`是计算距其`offsetParent`元素的顶部距离和左边距离。（即距离祖元素中第一个position值不为static的祖元素的上边距离和左边距离）

### getBoundingClientRect
getBoundingClientRect获取元素相对于视口（可视区域）的各个距离，有如下值：
- left/right：元素左边/右边距视口左边的距离
- top/bottom：元素上边/下边距视口上边的距离
- width/height：元素的宽高（border+padding+content），该属性和`offsetWidth/offsetHeight`相等。

```
var rect = app.getBoundingClientRect()
rect.bottom
rect.height
rect.left
rect.right
rect.top
rect.width
```

### 元素尺寸
- `offsetWidth/offsetHeight`获取元素宽高（border + padding + content）
- clientWidth/clientHeight获取元素宽高（padding + content）

### 元素滚动距离
```
// 获取窗口的滚动距离
document.documentElement.scrollTop
document.body.scrollTop // ie
document.documentElement.scrollLeft
document.body.scrollLeft // ie

// 设置窗口的滚动位置
document.documentElement.scrollTop = 0
document.documentElement.scrollLeft = 0
document.body // ie

// 使某个元素滚动到可视区域
// 接收一个参数，true为滚动到可视区域顶部，false为滚动到可视区域底部。默认ture
document.querySelector('#app').scrollIntoView()
document.querySelector('#app').scrollIntoView(false)
```

### 滚动元素的尺寸
如果一个元素设置为超出滚动后，那么scrollHeight将获取其滚动元素的尺寸，例如一个div宽高50，overflow: scroll;里面有一个高度为1000px的p，那么该div的scrollHeight尺寸为1000。
```
div.scrollHeight
div.scrollWidth
```

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/5/6/16a88e6908a409a6~tplv-t2oaga2asx-image.image)

### style
元素的style属性返回一个CSSStyleDeclaration对象，该对象包含元素的内联样式，而不是计算后的样式，如果没有给元素写样式，则通过该属性获取的值是空置。

```
var domStyle = document.querySelector('#app').style
// 获取高度，宽度
style.height
style.width
// 连字符的属性需要使用驼峰命名法
domStyle.fontSize
// 对于暴露字属性在前面加上css
domStyle.cssFloat // domStyle.float 谷歌上测试也可以
```

> style获取的是内联的属性，如果是写在样式表中的属性，是获取不到的。  
> 获取的是实际的内联属性，而不是计算后的值。即使样式表中通过important等方式使得权重高于内联的，获取到的依旧是内联样式中写的值。
> 获取的颜色值是`rgb`的

style对象获取/设置/移除的其他方法
```
// 设置属性，不能写复合属性，例如background/margin，而是分开的写法：background-color/margin-left等
// 用-分割的写法，而不是驼峰
dom.setProperty('background-color', '#f00')
domStyle.setProperty('background-color', '#f00')

// 获取
domStyle.getPropertyValue(属性名)
domStyle.getPropertyValue('background-color')

// 移除
domStyle.removeProperty('background-color')
```

### style对象设置/获取/移除多个内联属性
```
// 批量设置多个内联属性
domStyle.cssText = 'background-color: #000;color: 20px;margin: 30px;'

// 移除全部内联属性
domStyle.cssText = ''

// 获取style属性的内联属性
domStyle.cssText

// 通过setAttribute/getAttibute/removeAttribute也是可以实现相同的效果
dom.setAttribute('style', 'background-color: #000;color: #f1f1f1; 20px;margin: 30px;')

dom.getAttribute('style')

dom.removeAttribute('style')
```

### 获取计算后的属性
```
var winStyle = window.getComputedStyle(dom)

winStyle.color
winStyle.border
winStyle.backgroundColor // 获取的是rgb颜色格式
winStyle.marginTop // 不能获取简写的格式，例如margin
```
> 返回的颜色格式是rgb的格式，背景色返回的是rgba  
> 不能获取简写的属性，例如margin/padding，而是marginTop

### 修改样式的最佳实践
更多的我们会通过给元素添加/移除某个class/id方式，来添加修改样式

### DocumentFragment文档片段
DocumentFragment文档片段可以看作是一个空的文档模板，行为与实时DOM树类似，但是仅在内存中存在，可以附加到实时DOM中。

```
// 创建
document.createDocumentFragment()
// 例如：
var lis = ['hello! ', 'Every', 'bady'];
var fragment = document.createDocumentFragment();
lis.forEach(e => {
  var liElem = document.createElement('li');
  liElem.textContent = e;
  fragment.appendChild(liElem)
})
dom.appendChild(fragment)

// 文档片段插入到dom后，自身的节点内容就没了。例如上面的例子：
dom.appendChild(fragment) // 第一次将文档片段的内容插入到dom后
dom.appendChild(fragment) // 执行相同的操作，并不会插入了，因为此时的文档片段内容没了。

// 为了文档片段的内容可以多次利用，可以利用克隆的方式
dom.appendChild(fragment.cloneNode(true))
```

### 绑定事件

```
// 内联事件，基本不用
<div onclick="alert('a')"></div>

// 属性事件(DOM 0 级事件)
window.onload = function () {}

// 绑定事件（DOM 2 级事件），ps：没有1级事件
window.addEventListener('scroll', (e) => {
    console.log(e)
}, false)
```

### 易混事件区分

常见的click/onload/scroll/resize等事件就不介绍了。

```
// 鼠标按下，都是在输入法接收到键值之前
keydown // 任何按键按下都会触发，不管他是否产生字符码
keypress // 只有实际产生字符码才会触发，例如command键/option键/shift键等并不会触发

// 鼠标滑入
mouseenter // 鼠标滑入元素及其子元素时触发，不冒泡
mouseover // 鼠标滑入某个元素时触发，会冒泡

// 页面展示
window.onpageshow = function () {} // 展示页面时，触发
window.onload = function () {} // 页面加载完成后触发
// 两者的区别在于，从浏览器缓存读取的页面，并不会触发load事件，例如操作浏览记录的前进后退时

// 其他
offline // 离线时触发
online // 在线时触发
message // 跨文档传递时触发
hashchange // url中hash值的变化时触发
DOMContentLoaded // 页面解析完成后触发，资源不一定下载完成
```

### 事件中的this/target/currentTarget

```
document.body.addEventListener('click', function (e) {
    console.log(this)
    console.log(e.currentTarget)
    console.log(e.target)
    console.log(this === e.target, this === e.currentTarget)
}, false)

// this
this指的是该事件绑定的元素或对象，这里指向body

// currentTarget
指的是该事件绑定的元素或对象，这里指向body，同this

// target
指的是事件的目标，可以理解为开始触发冒泡时的那个元素，或者说是鼠标点击的嵌套在最里面的那个元素。
这里指向div
```

### preventDefault
阻止事件的默认行为，例如a标签的跳转、输入框的输入等 。但是并不能阻止冒泡。
```
// 假设a是某个a元素
a.addEventListener('click', function (e) {
    e.preventDefault()
}, false)
```

### stopPropagation
阻止事件冒泡,但不会阻止默认事件。
```
a.addEventListener('click', function (e) {
    e.stopPropagation()
}, false)
```

### stopImmediatePropagation
stopImmediatePropagation方法不仅会阻止事件冒泡，还会阻止该元素在调用该方法后面的绑定事件的触发
```
app.addEventListener('click', function () {
    console.log('app first')
}, false)

app.addEventListener('click', function (e) {
    console.log('app second, 阻止app后面绑定的click事件的冒泡')
    e.stopImmediatePropagation()
}, false)

// 此次的app事件绑定不会触发，因为已经被上面的stopImmediatePropagation方法阻止掉了
app.addEventListener('click', function (e) {
    console.log('app third')
}, false)

document.body.addEventListener('click', function () {
    console.log('body click')
}, false)

// 最终输出如下：
// app first
// style.html:98 app second, 阻止app后面绑定的click事件的冒泡
```

### 自定义事件
```
// 自定义事件
var cusEvent = document.createEvent('CustomEvent');

// 配置自定义事件的详情
cusEvent.initCustomEvent('myNewEvent', true, true, {
    myNewEvent: 'hello this is my new custom event!'
})

// 给#app绑定我们自定义的事件
var app = document.querySelector('#app');
app.addEventListener('myNewEvent', function (e) {
    console.log(e.detail.myNewEvent)
}, false)

// 在app上触发自定义事件
app.dispatchEvent(cusEvent)
```

initCustomEvent接收四个参数：事件名称，是否冒泡，是否可以取消事件，传递给event.detail的值

### 事件委托
事件委托利用事件流来完成，给父级绑定事件，然后判断触发事件的target，执行对应的事件。

例如：给表个中的td添加事件。
```
var tableBox = document.querySelector('#table-box');
tableBox.addEventListener('click', function (e) {
    var target = e.target
    if (target.tagName.toLowerCase() === 'td') {
        console.log(target.textContent)
    }
}, false)
```

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/5/6/16a88d6aac9ad086~tplv-t2oaga2asx-image.image)

参考内容：
- 文章参考《dom启蒙》一书
- [MDN资料文档](https://developer.mozilla.org/zh-CN/)

> 百尺竿头、日进一步。  
> 我是愣锤，一名前端爱好者。