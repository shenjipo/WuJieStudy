## 源码

[wujie框架源码](https://github.com/Tencent/wujie)

[wujie官方文档](https://wujie-micro.github.io/doc/)

[本文源码](https://github.com/shenjipo/WuJieStudy)

## 工程化部分

### lerna介绍

wujie使用`lerna`工具作为单一源码仓库的管理工具，其内部主要包含4个模块

* wujie-core 包含了微前端业务代码，像核心的`iframe`、`shadowroot`、`html自定义组件`、`代理dom`都在里
* wujie-vue2 封装了一个vue2组件，可以按照vue2的语法来使用wujie
* wujie-vue3 封装了一个vue3组件，可以按照vue3的语法来使用wujie
* wujie-react 封装了一个react组件，可以按照react的语法来使用wujie

使用lerna的作用在于

1. 在文件结构上隔离核心业务代码和外部组件代码，方便维护
2. 方便本地调试，当改变了核心业务代码后无需发布，即可在组件代码中生效
3. 方便各部分代码独立打包发布

### lerna使用

1. 全局或者本地安装`lerna`

> pnpm install --save-dev lerna

2. 进入到项目的目录下执行

> lerna init

生成的文件结构如图所示

![image_45ad5eb091b79160cfcd8f5eae4a2fa2.png](http://101.133.143.249/api/getImage/image_45ad5eb091b79160cfcd8f5eae4a2fa2.png)

3. 执行如下命令，创建两个子目录

> lerna create package-a
>
> lerna create package-b

4. 根据本文源码，修改根目录下的package.json文件，在script字段下添加一行命令，当配置了这个命令之后，如果在命令行执行`pnpm run start`命令，lerna会自动进入到各个子目录执行子目录下预先配置好的start命令，方便一键编译、启动

> "start": "lerna run start --parallel"

5. 修改package-a和package-b目录下的package.json文件，添加start命令
6. 修改package-a和package-b目录下的package.json文件，引入babel包用于把ts文件编译成esm模式下的js文件
7. 在命令行执行`pnpm run start`命令，如果在package-a和package-b文件夹的esm文件夹下出现了编译后的js文件，那么lerna就运行成功，wujie项目采用的就是上述工程化配置方案。

![image_9a07efc9b05a1bc0883f84dc8e3293ad.png](http://101.133.143.249/api/getImage/image_9a07efc9b05a1bc0883f84dc8e3293ad.png)

## 功能部分

首先介绍下基于官方的源码调试，首先使用`pnpm i`安装依赖，然后执行`pnpm run start`启动，启动成功之后会自动弹出两个网页

> http://localhost:7700/#/home
>
> http://localhost:8000/home

根据上面工程化部分介绍，由于使用了`lerna`，修改src目录下的代码会自动触发重新编译，修改的结果可以在网页中立即看到

所有的微前端框架都要解决`js`隔离和`css`隔离这两个问题，`wujie`使用iframe来解决`js`隔离问题，使用`shadowroot`解决css隔离问题，使用代理模式来把iframe中的`dom`和`winodw`的一些方法代理到`shadowroot`中去执行，在`wujie`中把`iframe`称为`js`沙箱，把`shadowroot`称为`css`沙箱，`wujie`的使用方法在官网上介绍的比较清晰，本文研究源码主要是为了弄清楚`wujie`如何实现把iframe中的操作代理到`shadowroot`中

下面以 wujie-vue 组件为例说明wujie的启动过程

1. 当加载wujie组件库的时候，就会创建自定义html元素 wujie-app `window.customElements.define("wujie-app", WujieApp);`
2. wujie-vue组件的模板只有一个 div 标签，当div标签按照vue2的生命周期渲染完成后，进入mounted生命周期时，调用了wujie提供的startApp方法
3. 初始化一个wujie实例 new Wujie()，内部包含两个重要对象

this.shadowRoot  shdaowRoot实例，也就是所谓的css沙箱

this.iframe iframe实例，也就是所谓的js沙箱，此时iframe已经被插入到dom中，接着在iframeReady的回调函数中，去劫持iframe.contentWindow.Document原型链的的一些方法，例如dom操作，widnow操作，把这些操作全部代理到shaodwRoot的`proxyDocument`等属性上，而proxyDocument是被proxy的属性， 相当于在iframe内部进行的一些dom操作，实际执行在了shadowRoot中，在这个被劫持的函数中，我们还可以执行一些“副作用函数”

4. wujie.active()，在dom中加入生成的shadowroot实例，并且完成shadowroot的proxyDocument的属性上方法的改写
5. wujie.start()，加载远程js脚本，在js沙箱中执行

下面提供两个手写的例子来说明简化实现上述的过程

[iframe代理到shadowroot的原理](https://github.com/shenjipo/WuJieStudy/blob/master/wujie/iframe%E7%9A%84document%E6%93%8D%E4%BD%9C%E4%BB%A3%E7%90%86%E5%88%B0shadowRoot%E4%BE%8B%E5%AD%90.html)

[wujie的启动简化流程](https://github.com/shenjipo/WuJieStudy/blob/master/wujie/wujie%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E6%A8%A1%E6%8B%9F.html)

打开 `wujie启动过程模拟.html` 文件调试时需要先启动`wujie`（把官方代码git clone下来后依次执行pnpm i 和 pnpm run start即可）

在控制台可以看到iframe中的**部分**dom操作确实被代理到了shadowroot中，如下图所示。

![image_802dc6cded1892deeeb20520f3c48b74.png](http://101.133.143.249/api/getImage/image_802dc6cded1892deeeb20520f3c48b74.png)

根据源码部分分析，如果要实现wujie的效果，代理部分特殊处理的属性和方法特别多，因此本文只做了实现原理验证，并没有完全实现把dom元素渲染到shadowroot中
