---
layout: post
title: "Next 框架与主流工具的整合（二）"
subtitle: " 一堆工具 redux + redux-saga + koa + sass + postcss + ant-design-mobile"
date: 2019-01-19 00:40:00
author: "张豆豆"
header-img: "img/post-bg-rwd.jpg"
catalog: true
tags:
  - 教程
  - 移动端web app
  - SSR
  - react
  - next
  - ant-design-mobile
---

# Next 框架与主流工具的整合（二）—— 完善与优化

> 前言：18 年 12 月 24 日项目成功上线了，在经历了两周的线上 bug、UI 以及代码优化后，解决了不少问题，于是再完善与优化一下这个[项目](https://github.com/code-coder/next-mobile-complete-app)。

- 布局优化
- 高清配置
- antd-mobile 自定义配置
- antd-mobile Toast 组件封装

## 布局优化

布局优化在这篇文章完成了 → [移动端优雅布局实践]()，最后我使用的方案是——**absolute 脱离文档流**（好处是设置容器的`height: 100%`，可以直接继承 html 窗口高度）。

## 高清配置

### 问题发现

开始项目之前没有对 dpr（device pixel radio）与缩放做过多的了解，在项目开发的时候就将它们都直接写死为 1 了。到后来 UI 验收的时候发现并没有实现 UI 设计师预期的细线效果。我在解决这个问题的时候才去认真看了一下 dpr 的介绍。[这篇详解 dpr 的文章](https://blog.csdn.net/a419419/article/details/79295799)写得还不错。

从概念来说，dpr 就是设备的物理像素与设备独立像素（也就是 css 逻辑像素，以下就称为 css 逻辑像素）的比率。

比如：iPhone 6 的分辨率是 750\*1334，`window.screen.width`（css 逻辑像素）为 375，因此

```math
dpr = 750 /375 = 2
```

再比如：iPhone X 的分辨率是 1125\*2436，`window.screen.width`（css 逻辑像素）也是 375，因此

```math
dpr = 1125 /375 = 3
```

那么 dpr 有什么用呢？

在这之前先提一下我们移动端必备的一个`meta`标签：

```
<meta name="viewport" content="width=device-width,maximum-scale=1,minimum-scale=1,user-scalable=no" />
```

> device-width 在 html 中也同样被解读为理想（基准）视口的宽度，即 320px，375px，414px，这里的 px 就是指 css 像素，通常也被称为逻辑像素；那我们可以认为 html 中的 css 像素的显示尺寸应该和 NA 中的 pt、dp 的显示尺寸相等。

通过这个 meta 标签，我们可以实现`initial-scale=1`初始缩放 100%，**就可以达到`1px的css逻辑像素 = 眼睛在设备上看起来的1px`，换句话说`body { width: 375px; }`可以在 iPhone 6 上充满竖屏的整个宽度**。

那么问题就来了，如果我们要给一个盒子加上一个 1px 的细线：
`border-bottom: 1px solid red;`。那么在 iPhone 6 上真的是 1px 吗？

iPhone 6 真机截图（宽度为 702px）：
![image](http://doudou-static.oss-cn-shanghai.aliyuncs.com/blog/next/1px-no-scale.jpeg)

可以看出高度明显不止 1 像素。这就是由于 dpr 造成的，因为 iPhone 6 的 dpr 为 2，且缩放比例为 100%，1px 的 css 渲染出来就是 2px 物理像素。

> #### 这就是我们 UI 粑粑和产品们不满意的地方。

那接下来如何去解决这个问题呢？

### 解决方案

根据设备的 dpr 来动态计算缩放比例，以及根节点的`font-size`。

```js
// rem.js
(function(doc, win) {
  var docEl = doc.documentElement,
    dpr = Math.min(win.devicePixelRatio, 3);
  dpr = window.top === window.self ? dpr : 1; //被iframe引用时，禁止缩放
  var scale = 1 / dpr,
    resizeEvt = "orientationchange" in window ? "orientationchange" : "resize";
  docEl.dataset.dpr = dpr;
  var metaEl = doc.createElement("meta");
  metaEl.name = "viewport";
  metaEl.content =
    "initial-scale=" +
    scale +
    ",maximum-scale=" +
    scale +
    ", minimum-scale=" +
    scale +
    ",user-scalable=no,viewport-fit=cover";
  docEl.firstElementChild.appendChild(metaEl);
  var recalc = function() {
    var width = docEl.clientWidth;
    // 大于1280按1280来算
    if (width / dpr > 1280) {
      width = 1280 * dpr;
    }
    // px : rem = 100 : 1
    docEl.style.fontSize = 100 * (width / 375) + "px";
  };
  recalc();
  if (!doc.addEventListener) return;
  win.addEventListener(resizeEvt, recalc, false);
})(document, window);
```

> 如果有显示富文本元素，则需要处理富文本元素的样式

移动端为了适配不同的机型，使用`rem`为单位是一个不错的选择，而且我们也一直在用它。

这里除了根据 dpr 来计算`initial-scale`，还调整了根节点的`font-size`，以至于在缩放的时候能够还原到视窗大小（因为要缩放，所以要相应的增加`rem`的基数）。

这样我们上面写的`border-bottom: 1px solid red;`在这个方案显示出来就是这样的：
![image](http://doudou-static.oss-cn-shanghai.aliyuncs.com/blog/next/1px-scale.jpeg)

哇咔咔，可以看出明显变细了，这才是我们 UI 粑粑们想要的 O(∩_∩)O~~

但是这样的设置在结合`ant-design-mobile`的时候，发现`ant-design-mobile`的组件都被缩小了。原来是，它的元素都是以 px 为单位，而我们缩放前没有对它的最小单位乘以相应的基数。**那么，我们需要给它配置一个基数**！

## antd-mobile 自定义配置

查文档发现`ant-design-mobile`提供了[主题配置](https://mobile.ant.design/docs/react/customize-theme-cn)，而且它提供了一个`@hd`的变量做为长度基本单位，它的默认值是`1px`。

##### 我们只需要把`@hd`设置为`0.01rem`就可以解决问题。

这个[主题配置](https://mobile.ant.design/docs/react/customize-theme-cn)的文档是以 webpack 项目来做的例子。那如何在`next.js`项目中完成自定义配置呢？

我在[next.js 的 examples](https://github.com/zeit/next.js/tree/canary/examples)中没有找到我所需要的 example，不过找到了两个相关的例子：一个是[with-antd-mobile](https://github.com/zeit/next.js/tree/canary/examples/with-antd-mobile)，一个是[with-ant-design-less](https://github.com/zeit/next.js/tree/canary/examples/with-ant-design-less)。第二个是`ant-design`的自定义主题配置，那应该就可以仿照这个 example 去增加[with-antd-mobile](https://github.com/zeit/next.js/tree/canary/examples/with-antd-mobile)的自定义主题配置。

> 这里提一下，这个[with-antd-mobile](https://github.com/zeit/next.js/tree/canary/examples/with-antd-mobile)在我写[Next 框架与主流工具的整合](https://segmentfault.com/a/1190000016383263)之后更新了`next.config.js`的配置，这里也改成了最新的配置。

1. 安装解析 **Less** 与 **normalize.css** 的包

```shell
npm i @zeit/next-less @zeit/next-css less less-vars-to-js -S
```

2. 修改`.babelrc`配置

```
{
  "presets": ["next/babel"],
  "plugins": [
    [
      "import",
      {
        "libraryName": "antd-mobile",
        "style": true
      }
    ]
  ]
}

```

3. 修改`next.config.js`配置

```
/* eslint-disable */
const withCSS = require('@zeit/next-css');
const withSass = require('@zeit/next-sass');
const withLess = require('@zeit/next-less');
const lessToJS = require('less-vars-to-js');
const fs = require('fs');
const path = require('path');

// Where your antd-custom.less file lives
const themeVariables = lessToJS(fs.readFileSync(path.resolve(__dirname, './antd-custom.less'), 'utf8'));

// fix: prevents error when .less files are required by node
if (typeof require !== 'undefined') {
  require.extensions['.less'] = file => {};
  require.extensions['.css'] = file => {};
}

module.exports = withCSS(
  withLess(
    withSass({
      lessLoaderOptions: {
        javascriptEnabled: true,
        modifyVars: themeVariables
      }
    })
  )
);

```

4. 在项目下新建 Less 变量文件 `antd-custom.less`

```
@hd: 0.01rem;
```

重启项目，就大功告成了。

## antd-mobile Toast 组件封装

`antd-mobile`的`Toast.info()`组件在显示的时候不能点击背景就消失，与原生的 Toast 有些差异，为了体验，这里再做了一层封装，在点击背景的时候隐藏 Toast。

```
// utils/toast.js
static info = (content, duration, onClose, mask) => {
    Toast.info(content, duration, onClose, mask);
    const toastElement = document.getElementsByClassName('am-toast-mask')[0];
    toastElement &&
      toastElement.addEventListener('click', () => {
        Toast.hide();
        onClose && onClose();
      });
  };
```

`antd-mobile`的 loading 图在 Android 上有些怪异，这里也自定义了 Loading:

```html
static loading = (content, duration, onClose, mask) => {
    Toast.info(
      <div>
        <svg className="rotate360-800" width="0.26rem" height="0.26rem" viewBox="0 0 26 26">
          <title>加载</title>
          <desc>Created with Sketch.</desc>
          <g id="首页" stroke="none" strokeWidth="1" fill="none" fillRule="evenodd">
            <g transform="translate(-185.000000, -519.000000)" fill="#FFFFFF" id="加载">
              <g transform="translate(185.000000, 519.000000)">
                <g id="分组" transform="translate(0.625011, 0.625011)">
                  <path
                    d="M12.3750144,0.0259531552 C5.53983848,0.0259531552 0,5.56600648 0,12.4009676 C0,19.2359286 5.53983848,24.775982 12.3750144,24.775982 C19.2099755,24.775982 24.7500288,19.2359286 24.7500288,12.4009676 C24.7500288,5.56579163 19.2099755,0.0259531552 12.3750144,0.0259531552 Z M12.3750144,22.028385 C7.05736759,22.028385 2.74781179,17.7185714 2.74781179,12.4009676 C2.74781179,7.08332074 7.05741056,2.7735501 12.3750144,2.7735501 C17.6926612,2.7735501 22.0026467,7.08336371 22.0026467,12.4009676 C22.0026467,17.7186144 17.6926182,22.028385 12.3750144,22.028385 Z"
                    id="形状"
                    fillOpacity="0.2"
                    fillRule="nonzero"
                  />
                  <path
                    d="M12.3749972,0.0259402646 L12.3750144,2.77353721 C17.6926612,2.77353721 22.0026467,7.08335082 22.0026467,12.4009547 L24.7500116,12.4009547 C24.7500116,5.56577874 19.2099583,0.0259402646 12.3749972,0.0259402646 Z"
                    id="路径"
                  />
                </g>
              </g>
            </g>
          </g>
        </svg>
        <div style={{ fontSize: '0.12rem' }}>{content}</div>
      </div>,
      duration,
      onClose,
      mask
    );
  };
```

## 写在最后

一个项目需要在不断的优化与完善中才能变得更好。搭建项目是对一个项目负责人很大的考验，如果在项目设计的初期有许多的问题没有考虑到，就很有可能导致优化的时候需要耗费很大的精力。

总之，不要逃避困难与问题，这些都是成长路上不可或缺的。
