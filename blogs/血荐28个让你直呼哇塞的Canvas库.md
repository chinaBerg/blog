`Canvas`技术的诞生可谓是让绘图技术如虎添翼，本文将推荐一系列`Canvas图形绘制、流程图、组织图、甘特图、全景图、3D库、VR/AR、图像编辑`等方面的库，希望助你在Canvas绘图时寻得一把趁手的利器。

同时，愣锤也将`Canvas`的相关资源进行的收录整理分类，更多的资源请关注[awesome-canvas](https://github.com/chinaBerg/awesome-canvas)，项目地址 https://github.com/chinaBerg/awesome-canvas 。目前该库持续维护中，已收录大约`200+`的`Canvas库`，以及`资源网址、插件、书籍、博客`等资源。

## 图形处理库

图形绘制是Canvas中最基本的内容，但是Canvas本身提供的api比较基础，开发起来低效。而且也缺少完整的事件系统、区域监测、缓存等等。下面让我们来看几款高效的图形处理库吧。

### Konva
简介：`Konva`是一个 HTML5 Canvas JavaScript 框架, 通过扩展 Canvas 的 2D Context 让桌面端和移动端Canvas支持交互性，使其支持高性能动画、过渡、节点嵌套、分层、过滤、缓存、事件处理等等。[Konva传送门](https://konvajs.org/docs/sandbox/index.html)

除上述之外，文档相对友好，但也仅仅是相对于同类库的文档友好那么一滴滴，社区有维护一个中文文档。

![konva3.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07a494f9192d4a3292874626810345b8~tplv-k3u1fbpfcp-watermark.image?)

![konva2.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e1d7cd3293b4acd8f815d3df542c0f4~tplv-k3u1fbpfcp-watermark.image?)

### fabric.js

简介： **Fabric.js**是一个可以轻松处理 HTML5 Canvas元素的框架，使得Canvas元素支持**交互式对象模型**，同时也是一个**SVG-to-Canvas**和**Canvas-to-SVG**的解析器 [fabric.js传送门](http://fabricjs.com/demos/)

fabricjs是和konva同类型但是比konva更老牌的一个的Canvas库，目前github上Star数![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8fc770b5124f4ebb9eee4bf913c8c8cb~tplv-k3u1fbpfcp-zoom-1.image)

![fabricjs2.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0726bde6fb3849c48e82be6b87c7d0d8~tplv-k3u1fbpfcp-watermark.image?)

[更多示例](http://fabricjs.com/demos/)如下图所示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a2b81222dc7481c8e5c1dc856a45b9f~tplv-k3u1fbpfcp-watermark.image?)

更多关于Canvas图形处理的资源，请访问[awesome-canvas](https://github.com/chinaBerg/awesome-canvas)，项目地址 https://github.com/chinaBerg/awesome-canvas

## 图像编辑

市面上图像编辑的软件有很多，像大家所熟知的`PS、sketch、axure、激萌、剪映`等等。那么如果我们想开发类似的软件，有没有可以使用的库或者参考的开源软件呢？

### miniPaint

简介：[miniPaint](https://github.com/viliusle/miniPaint) [[在线演示](https://viliusle.github.io/miniPaint/)] - 在线版的PS。PS这款软件大家都不陌生，web版本的如何呢？请看下图：

![mini-paint.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b54308bb66524b878956d8d86bffca9f~tplv-k3u1fbpfcp-watermark.image?)

### DarkroomJS
简介：[DarkroomJS](https://github.com/MattKetmo/darkroomjs) [[在线演示](https://pqina.nl/pintura/?affiliate_id=854594675)] - 基于Fabricjs的浏览器端可扩展的图像编辑工具

![pintura-image.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/baf76bd23ea6455996b5281d4b05e991~tplv-k3u1fbpfcp-watermark.image?)

### fabric-brush
简介：[fabric-brush](https://github.com/tennisonchan/fabric-brush) [[在线演示](https://tennisonchan.github.io/fabric-brush/)] - 基于Fabric.js的Canvas笔刷工具

![brush.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e9b54091907466a926f53978cefa3bf~tplv-k3u1fbpfcp-watermark.image?)

### fabricjs-image-editor-origin
简介：[fabricjs-image-editor-origin](https://github.com/pegasus1982/fabricjs-image-editor-origin) [[在线演示](https://fabricjs-image-editor-f62330.netlify.app/)] - Fabricjs图像编辑器

![fabricjs-demo.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/095fb38ff1234f67910089c1d1718f67~tplv-k3u1fbpfcp-watermark.image?)

### react-sketch
简介：[react-sketch](https://github.com/tbolis/react-sketch) [[在线演示](http://tbolis.github.io/showcase/react-sketch/)] - 基于React、Fabricjs的素描应用

![sketch.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/221ebe8fddb645809a93f0cd6b2d22bf~tplv-k3u1fbpfcp-watermark.image?)

### glitch-canvas

简介：[glitch-canvas](https://github.com/snorpey/glitch-canvas) [[在线演示](https://snorpey.github.io/jpg-glitch/)] - 给画布元素添加故障效果

![jpg-glitch.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/997630d158ad4c1fad244f2b1c87495c~tplv-k3u1fbpfcp-watermark.image?)

### animockup

简介：[animockup](https://github.com/alyssaxuu/animockup) [[在线演示](https://animockup.com/)] - 在浏览器中创建动画模型，并导出为视频或动画GIF

![animo.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2281f526d9e84cd18547ab4875c60df1~tplv-k3u1fbpfcp-watermark.image?)

更多关于Canvas图像编辑/画板的资源，请访问[awesome-canvas](https://github.com/chinaBerg/awesome-canvas)，项目地址 https://github.com/chinaBerg/awesome-canvas

## 物理引擎

物理引擎使用质量、速度、摩擦力和空气阻力等变量，模拟了一个近似真实的物理系统，为刚性物体赋予真实的物理效果，比如重力、旋转和碰撞等效果，让物体的行为表现的更加趋向真实。例如，守望先锋的英雄在跳起时，系统所设置的重力参数就决定了他能跳多高，下落时的速度有多快，子弹的飞行轨迹等等。

### matter.js

简介： **matter.js**是一个用于 Web 的 JavaScript 2D 物理引擎库 [matter.js传送门](https://brm.io/matter-js/demo/#wreckingBall)

[matter.js](https://brm.io/matter-js/demo/#wreckingBall)相较于老牌的 Box2D 引擎库更为轻量级（压缩版仅有 87 KB），并且在性能和功能方面也不逊色。

![matter.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b63e15f8f2c4d0fb4fcf8c3816f12f4~tplv-k3u1fbpfcp-watermark.image?)

![matter3.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/968214be9a154418960a6806af5626cf~tplv-k3u1fbpfcp-watermark.image?)

更多关于Canvas物理引擎的资源，请访问[awesome-canvas](https://github.com/chinaBerg/awesome-canvas)，项目地址 https://github.com/chinaBerg/awesome-canvas

## 流程图/组织图/图编辑等

工业软件开发，一直是软件领域复杂而又重要的一环。其对技术人的要求要是更高的，下面看看有哪些可以辅助我们快速开发的库或者参考的场景吧。

### gojs

简介： **gojs**是一款可以非常方便的开发交互式流程图、组织结构图、设计工具、规划工具、可视化语言的JavaScript图表库。 [gojs.js传送门](https://gojs.net/latest/)

- GoJS用自定义模板和布局组件简化了节点、链接和分组。
- 给用户交互提供了许多先进的功能，如拖拽、复制、粘贴、文本编辑、工具提示、上下文菜单、自动布局、模板、数据绑定和模型、事务状态和撤销管理、调色板、概述、事件处理程序、命令和自定义操作的扩展工具系统。

![gojs.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9530224a8cc42f281b073124eea5e2f~tplv-k3u1fbpfcp-watermark.image?)

文档中提供了大量的[demo](https://gojs.net/latest/samples/)可供参考，基本对于常见的图编辑程序做到了全覆盖。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/959804132c414da3ad15a6e0b9bb79f4~tplv-k3u1fbpfcp-watermark.image?)

### butterfly

简介：[butterfly](https://github.com/alibaba/butterfly) [[在线演示](https://butterfly-dag.gitee.io/butterfly-dag/demo/analysis)] 一个基于JS的数据驱动的节点式编排组件库，同时适用于React/Vue2组件。

- 丰富的DEMO，开箱即用
- 全方位管理画布，开发者只需要更专注定制化的需求
- 利用DOM/REACT/VUE来定制元素；灵活性，可塑性，拓展性优秀
- 提供了中文文档，这点对英文不好的小伙伴很Nice

![butterfly2.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7c3bb3a005e464092b878c167f93fc1~tplv-k3u1fbpfcp-watermark.image?)

### wireflow
简介：[wireflow](https://github.com/vanila-io/wireflow) [[在线演示](https://app.wireflow.co/)] 用户流程图实时协作工具。

- Wireflow 有超过 100 种自定义构建图形/卡可供使用，涵盖了大多数 Web 元素、交互和使用案例。
- Wireflow 考虑到了协作。您可以邀请您的同事和他们一起实时设计下一个项目的用户流程。
- 它具有内置的实时聊天功能，让您能够与您的队友进行交流，并且在您实时协作时仍然在同一个应用程序中。

![wireflow.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b5a83716545442e08469052fd936ae57~tplv-k3u1fbpfcp-watermark.image?)

### flowy
简介：[Flowy](https://github.com/alyssaxuu/flowy) [[在线演示](https://alyssax.com/x/flowy/)] - 用于创建流程图的最小javascript库。使创建具有流程图功能的 WebApp 成为一项非常简单的任务。通可以在几分钟内构建自动化软件、思维导图工具或简单的编程平台。

- 响应式拖放、自动捕捉、自动滚动
- 块重排、删除块、自动块居中
- 条件捕捉、条件块移除、无依赖项

![flowy.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/048182452d8a49318e2094b7e38e8a7d~tplv-k3u1fbpfcp-watermark.image?)

### Workflow Designer

简介：[Workflow Designer](https://github.com/guozhaolong/wfd) [[在线示例](https://guozhaolong.github.io/wfd/)] - 基于G6和React的可视化流程编辑器。

![wfd.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a7565b536d84e50954b5eb757eb2f08~tplv-k3u1fbpfcp-watermark.image?)

### web-pdm

简介：[web-pdm](https://erd.zyking.xyz/) [[在线示例](https://erd.zyking.xyz/demo)] - 用G6做的ER图工具，最终目标是想做成在线版的powerdesigner.

![xyz.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9794b59c71674972b2785224ba45b4fa~tplv-k3u1fbpfcp-watermark.image?)

### X-Flowchart-Vue

简介：[X-Flowchart-Vue](https://github.com/OXOYO/X-Flowchart-Vue) [[在线演示](http://oxoyo.co/X-Flowchart-Vue/)] - 基于G6和Vue的可视化图形编辑器。

![x-flowchart.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/16ca7946093d4b498fde760ee322adb6~tplv-k3u1fbpfcp-watermark.image?)

### OrgChart

简介：[OrgChart](https://github.com/dabeng/OrgChart/blob/master/README.zh-cn.md) [[在线演示](https://dabeng.github.io/OrgChart/)] - 简单直接的组织图插件

![origin.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79c83e86a95f4af79706a46112c4a5db~tplv-k3u1fbpfcp-watermark.image?)

### welabx-g6

简介：[welabx-g6](https://github.com/claudewowo/welabx-g6) [[在线示例](https://claudewowo.github.io/welabx-g6/dist/?_blank)] - 基于G6和Vue的流程图编辑器。

![welabx.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b0aa8800bf443d6b03668f453b93360~tplv-k3u1fbpfcp-watermark.image?)

更多关于Canvas图编辑的资源，请访问[awesome-canvas](https://github.com/chinaBerg/awesome-canvas)，项目地址 https://github.com/chinaBerg/awesome-canvas

## 全景图/AR/VR

5g的普及、虚拟现实/增强现实落地、元宇宙的火热......似乎让Canvas再度推上了技术的顶峰。下面让我来看看开发这些场景常见的Canvas库吧。

### Pannellum

简介：[Pannellum](https://github.com/mpetroff/pannellum) [[在线演示](https://pannellum.org/)] - 轻量、免费、开源的web全景查看器。

![pannelum.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4c8b6b8717b84dc3a184be7ffa466ab5~tplv-k3u1fbpfcp-watermark.image?)

### Panolens.js

简介：[Panolens.js](https://github.com/pchen66/panolens.js) [[在线演示](https://pchen66.github.io/Panolens/)] - Panolens.js基于Three.JS，主要研究领域是全景、虚拟现实和潜在的增强现实。

![panolens.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5303c450c2342f8be7e27f7b0a4452d~tplv-k3u1fbpfcp-watermark.image?)

### JS-Cloudimage-360-View

简介：[JS-Cloudimage-360-View](https://github.com/scaleflex/js-cloudimage-360-view) [[在线演示](https://scaleflex.github.io/js-cloudimage-360-view/)] 一个简单的、交互式的资源，可以用来提供您的产品的虚拟游览。

![cloud-image.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b72026af2304a6db9636b9e7aeba74c~tplv-k3u1fbpfcp-watermark.image?)

### A-Frame

简介：[A-Frame](https://github.com/aframevr/aframe) [[在线演示](https://aframe.io/)] A-Frame 除了帮助您构建 360 度媒体播放器外，它还提供了许多附加功能。其他功能可帮助您增强网站的虚拟现实体验。

![aframe.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87fe83edb57a41378e4ad462d165a1a8~tplv-k3u1fbpfcp-watermark.image?)

**更多关于Canvas全景图/AR/VR的资源，请访问[Github](https://github.com/chinaBerg/awesome-canvas)项目地址[awesome-canvas](https://github.com/chinaBerg/awesome-canvas)。**

## 3D库

### three.js
简介：[three.js](https://github.com/mrdoob/three.js) [[在线演示](https://threejs.org/examples/)] - 创建易于使用、轻量级、跨浏览器的通用3d js库。three.js就不多介绍了，大家想必都很熟悉。

![three.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/203317199c2a4becb910577b9098ff6b~tplv-k3u1fbpfcp-watermark.image?)

![three2.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/934155f60c9849529724b67d175e00f8~tplv-k3u1fbpfcp-watermark.image?)

### zdog
简介：[zdog](https://github.com/metafizzy/zdog) [[在线演示](https://zzz.dog/)] - 基于canvas和SVG设计师友好的伪3D引擎

![zdog.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/580ce2bbcfc242a8936a61735e57d2b4~tplv-k3u1fbpfcp-watermark.image?)

### seen.js
简介：[seen](http://seenjs.io/) [[在线演示](http://seenjs.io/demo-simple-interactive.html)] - 使用SVG或Canvas渲染3D场景。

![seen.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72e68cb0d7bc4acc854f6b0cb1aa2c1d~tplv-k3u1fbpfcp-watermark.image?)

### Oimo.js
简介：[Oimo.js](https://github.com/lo-th/Oimo.js) [[在线演示](http://lo-th.github.io/Oimo.js/index.html#basic)] - 轻量级的JS 3D物理引擎。

![oimo.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/864ac8b523cf4c4cb9639fc9653df502~tplv-k3u1fbpfcp-watermark.image?)

### phoria.js
简介：[phoria.js](https://github.com/kevinroast/phoria.js) [[在线演示](http://www.kevs3d.co.uk/dev/phoria/index.html)] - 用于在 HTML5 画布 2D 渲染器上进行简单 3D 图形和可视化的 JavaScript 库。它不使用 WebGL。适用于所有 HTML5 浏览器，包括桌面、iOS 和 Android。

![phoria.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0a9e8240e9441019c1824b0981e0655~tplv-k3u1fbpfcp-watermark.image?)

更多关于Canvas 3D的资源，请访问[awesome-canvas](https://github.com/chinaBerg/awesome-canvas)，项目地址 https://github.com/chinaBerg/awesome-canvas

## 结尾
百尺竿头、日进一步，我是你们的老朋友愣锤😊。

[awesome-canvas](https://github.com/chinaBerg/awesome-canvas)精心收录了200+的Canvas库和对应的资源，基本覆盖了Canvas开发领域的方方面面，而且还在持续收录维护中。如果小伙伴们觉得收录的还不错，欢迎给点个Star支持一下。

最后，[壹沓科技](https://www.1data.info/)前端团队目前扩招中，大量HC岗位，地点在上海静安。公司主要做大数据和RPA数智化相关产品，涉及Antlr编译、图像引擎等内容。欢迎小伙伴们联系我或投递简历哦，期待你的加入～ 邮箱：berg.liu@1data.info

