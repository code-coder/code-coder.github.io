---
layout: post
title: "Next 框架与主流工具的整合"
subtitle: " 一堆工具 redux + redux-saga + koa + sass + postcss + ant-design-mobile"
date: 2018-09-12 21:00:00
author: "张豆豆"
header-img: "img/post-bg-rwd.jpg"
catalog: true
tags:
  - 教程
  - SSR
  - react
  - next
  - redux
  - redux-saga
  - koa
  - ant-design-mobile
---

> “人生就是瞎折腾 ”

## 前言

老大说以后会用 next 来做一下 SSR 的项目，让我们有空先学学。又从 0 开始学习新的东西了，想着还是记录一下学习历程，有输入就要有输出吧，免得以后给忘记学了些什么~

---

# Next 框架与主流工具的整合

github 地址：[https://github.com/code-coder/next-mobile-complete-app](https://github.com/code-coder/next-mobile-complete-app)

##### 首先，clone [Next.js](https://github.com/zeit/next.js) 项目，学习里面的 templates。

##### 打开一看，我都惊呆了，差不多有 150 个搭配工具个 template，有点眼花缭乱。

##### 这时候就需要明确一下我们要用哪些主流的工具了:

- [x] 数据层：redux + saga
- [x] 视图层：sass + postcss
- [x] 服务端：koa

### 做一个项目就像造一所房子，最开始就是“打地基”：

#### 1. 新建了一个项目，用的是这里面的一个 with-redux-saga 的 template [戳这里](https://github.com/zeit/next.js/tree/canary/examples/with-redux-saga)。

#### 2. 添加 sass 和 postcss，参考的是 [这里](https://github.com/zeit/next-plugins/tree/master/packages/next-sass)

- 新建`next.config.js`，复制以下代码：

```js
const withSass = require("@zeit/next-sass");
module.exports = withSass({
  postcssLoaderOptions: {
    parser: true,
    config: {
      ctx: {
        theme: JSON.stringify(process.env.REACT_APP_THEME)
      }
    }
  }
});
```

- 新建`postcss.config.js`，复制以下代码：

```js
module.exports = {
  plugins: {
    autoprefixer: {}
  }
};
```

- 在`package.js`添加自定义 browserList，这个就根据需求来设置了，这里主要是移动端的。

```js
// package.json
"browserslist": [
    "IOS >= 8",
    "Android > 4.4"
  ],
```

- 顺便说一下 browserlist 某些配置会报错，比如直接填上默认配置

```
"browserslist": [
    "last 1 version",
    "> 1%",
    "maintained node versions",
    "not dead"
  ]
// 会报以下错误
Unknown error from PostCSS plugin. Your current PostCSS version is 6.0.23, but autoprefixer uses 5.2.18. Perhaps this is the source of the error below.
```

#### 3. 配置 koa，参照[custom-server-koa](https://github.com/zeit/next.js/tree/canary/examples/custom-server-koa)

- 新建`server.js`文件，复制以下代码：

```
const Koa = require('koa');
const next = require('next');
const Router = require('koa-router');

const port = parseInt(process.env.PORT, 10) || 3000;
const dev = process.env.NODE_ENV !== 'production';
const app = next({ dev });
const handle = app.getRequestHandler();

app.prepare().then(() => {
  const server = new Koa();
  const router = new Router();

  router.get('*', async ctx => {
    await handle(ctx.req, ctx.res);
    ctx.respond = false;
  });

  server.use(async (ctx, next) => {
    ctx.res.statusCode = 200;
    await next();
  });

  server.use(router.routes());
  server.listen(port, () => {
    console.log(`> Ready on http://localhost:${port}`);
  });
});

```

- 然后在配置一下`package.json`的 scripts

```
"scripts": {
    "dev": "node server.js",
    "build": "next build",
    "start": "NODE_ENV=production node server.js"
  }
```

### 现在只是把地基打好了，接着需要完成排水管道、钢筋架构等铺设：

- [x] 调整项目结构
- [x] layout 布局设计
- [x] 请求拦截、loading 状态及错误处理

#### 1. 调整后的项目结构

```
-- components
-- pages
++ server
|| -- server.js
-- static
++ store
|| ++ actions
||    -- index.js
|| ++ reducers
||    -- index.js
|| ++ sagas
||    -- index.js
-- styles
-- next.config.js
-- package.json
-- postcss.config.js
-- README.md
```

#### 2. layout 布局设计。

`ant design` 是我使用过而且比较有好感的 UI 框架。既然这是移动端的项目，[ant design mobile](https://mobile.ant.design/docs/react/introduce-cn) 成了首选的框架。我也看了其他的主流 UI 框架，现在流行的 UI 框架有[Amaze UI](http://amazeui.org/getting-started)、[Mint UI](https://mint-ui.github.io/#!/zh-cn)、[Frozen UI](http://frozenui.github.io/)等等，个人还是比较喜欢`ant`出品的。

恰好 templates 中有 ant design mobile 的 demo：[with-ant-design-mobile](https://github.com/zeit/next.js/tree/canary/examples/with-antd-mobile)。

- 基于上面的项目结构整合`with-ant-design-mobile`这个 demo。
- 新增 babel 的配置文件：.babelrc 添加以下代码：

```
{
  "presets": ["next/babel"],
  "plugins": [
    [
      "import",
      {
        "libraryName": "antd-mobile"
      }
    ]
  ]
}

```

- **修改 next.config.js 为：**

```
const withSass = require('@zeit/next-sass');
const path = require('path');
const fs = require('fs');
const requireHacker = require('require-hacker');

function setupRequireHacker() {
  const webjs = '.web.js';
  const webModules = ['antd-mobile', 'rmc-picker'].map(m => path.join('node_modules', m));

  requireHacker.hook('js', filename => {
    if (filename.endsWith(webjs) || webModules.every(p => !filename.includes(p))) return;
    const webFilename = filename.replace(/\.js$/, webjs);
    if (!fs.existsSync(webFilename)) return;
    return fs.readFileSync(webFilename, { encoding: 'utf8' });
  });

  requireHacker.hook('svg', filename => {
    return requireHacker.to_javascript_module_source(`#${path.parse(filename).name}`);
  });
}

setupRequireHacker();

function moduleDir(m) {
  return path.dirname(require.resolve(`${m}/package.json`));
}

module.exports = withSass({
  webpack: (config, { dev }) => {
    config.resolve.extensions = ['.web.js', '.js', '.json'];

    config.module.rules.push(
      {
        test: /\.(svg)$/i,
        loader: 'emit-file-loader',
        options: {
          name: 'dist/[path][name].[ext]'
        },
        include: [moduleDir('antd-mobile'), __dirname]
      },
      {
        test: /\.(svg)$/i,
        loader: 'svg-sprite-loader',
        include: [moduleDir('antd-mobile'), __dirname]
      }
    );
    return config;
  }
});

```

- **static 新增 rem.js**

```
(function(doc, win) {
  var docEl = doc.documentElement,
    // isIOS = navigator.userAgent.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/),
    // dpr = isIOS ? Math.min(win.devicePixelRatio, 3) : 1;
    // dpr = window.top === window.self ? dpr : 1; //被iframe引用时，禁止缩放
    dpr = 1;
  var scale = 1 / dpr,
    resizeEvt = 'orientationchange' in window ? 'orientationchange' : 'resize';
  docEl.dataset.dpr = dpr;
  var metaEl = doc.createElement('meta');
  metaEl.name = 'viewport';
  metaEl.content =
    'initial-scale=' + scale + ',maximum-scale=' + scale + ', minimum-scale=' + scale + ',user-scalable=no';
  docEl.firstElementChild.appendChild(metaEl);
  var recalc = function() {
    var width = docEl.clientWidth;
    // 大于1280按1280来算
    if (width / dpr > 1280) {
      width = 1280 * dpr;
    }
    // 乘以100，px : rem = 100 : 1
    docEl.style.fontSize = 100 * (width / 375) + 'px';
    doc.body &&
      doc.body.style.height !== docEl.clientHeight &&
      docEl.clientHeight > 360 &&
      (doc.body.style.height = docEl.clientHeight + 'px');
  };
  recalc();

  if (!doc.addEventListener) return;
  win.addEventListener(resizeEvt, recalc, false);
  win.onload = () => {
    doc.body.style.height = docEl.clientHeight + 'px';
  };
})(document, window);

```

- **增加移动端设备及微信浏览器的判断**

```
(function() {
  // 判断移动PC端浏览器和微信端浏览器
  var ua = navigator.userAgent;
  // var ipad = ua.match(/(iPad).* OS\s([\d _] +)/);
  var isAndroid = ua.indexOf('Android') > -1 || ua.indexOf('Adr') > -1; // android
  var isIOS = !!ua.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/); // ios
  if (/(iPhone|iPad|iPod|iOS|Android)/i.test(navigator.userAgent)) {
    window.isAndroid = isAndroid;
    window.isIOS = isIOS;
    window.isMobile = true;
  } else {
    // 电脑PC端判断
    window.isDeskTop = true;
  }
  ua = window.navigator.userAgent.toLowerCase();
  if (ua.match(/MicroMessenger/i) == 'micromessenger') {
    window.isWeChatBrowser = true;
  }
})();
```

- **\_document.js 新增引用**：

```html
<head>
  <script src="/static/rem.js" />
  <script src="/static/user-agent.js" />
  <link
    rel="stylesheet"
    type="text/css"
    href="//unpkg.com/antd-mobile/dist/antd-mobile.min.css"
  />
</head>
```

- **构造布局**

1. 在 components 文件夹新增**layout**和**tabs**文件夹

```
++ components
|| ++ layout
|| || -- Layout.js
|| || -- NavBar.js
|| ++ tabs
|| || -- TabHome.js
|| || -- TabIcon.js
|| || -- TabTrick.js
|| || -- Tabs.js
```

2. 应用页面大致结构是（意思一下）

- 首页

| nav     |
| ------- |
| content |
| tabs    |

- 其他页

| nav     |
| ------- |
| content |

- **最后，使用 redux 管理 nav 的 title，使用 router 管理后退的箭头**

```
// other.js
static getInitialProps({ ctx }) {
    const { store, req } = ctx;
    // 通过这个action改变导航栏的标题
    store.dispatch(setNav({ navTitle: 'Other' }));
    const language = req ? req.headers['accept-language'] : navigator.language;

    return {
      language
    };
  }
```

```
// NavBar.js
componentDidMount() {
// 通过监听route事件，判断是否显示返回箭头
Router.router.events.on('routeChangeComplete', this.handleRouteChange);
}

handleRouteChange = url => {
if (window && window.history.length > 0) {
  !this.setState.canGoBack && this.setState({ canGoBack: true });
} else {
  this.setState.canGoBack && this.setState({ canGoBack: false });
}
};
```

```
// NavBar.js
let onLeftClick = () => {
  if (this.state.canGoBack) {
    // 返回上级页面
    window.history.back();
  }
};
```

#### 3、请求拦截、loading 及错误处理

- **封装 fetch 请求，使用单例模式对请求增加全局 loading 等处理。**
  > 要点：1、单例模式。2、延迟 loading。3、server 端渲染时不能加载 loading，因为 loading 是通过 document 对象操作的

```
import { Toast } from 'antd-mobile';
import 'isomorphic-unfetch';
import Router from 'next/router';

// 请求超时时间设置
const REQUEST_TIEM_OUT = 10 * 1000;
// loading延迟时间设置
const LOADING_TIME_OUT = 1000;

class ProxyFetch {
  constructor() {
    this.fetchInstance = null;
    this.headers = { 'Content-Type': 'application/json' };
    this.init = { credentials: 'include', mode: 'cors' };
    // 处理loading
    this.requestCount = 0;
    this.isLoading = false;
    this.loadingTimer = null;
  }

  /**
   * 请求1s内没有响应显示loading
   */
  showLoading() {
    if (this.requestCount === 0) {
      this.loadingTimer = setTimeout(() => {
        Toast.loading('加载中...', 0);
        this.isLoading = true;
        this.loadingTimer = null;
      }, LOADING_TIME_OUT);
    }
    this.requestCount++;
  }

  hideLoading() {
    this.requestCount--;
    if (this.requestCount === 0) {
      if (this.loadingTimer) {
        clearTimeout(this.loadingTimer);
        this.loadingTimer = null;
      }
      if (this.isLoading) {
        this.isLoading = false;
        Toast.hide();
      }
    }
  }

  /**
   * 获取proxyFetch单例对象
   */
  static getInstance() {
    if (!this.fetchInstance) {
      this.fetchInstance = new ProxyFetch();
    }
    return this.fetchInstance;
  }

  /**
   * get请求
   * @param {String} url
   * @param {Object} params
   * @param {Object} settings: { isServer, noLoading, cookies }
   */
  async get(url, params = {}, settings = {}) {
    const options = { method: 'GET' };
    if (params) {
      let paramsArray = [];
      // encodeURIComponent
      Object.keys(params).forEach(key => {
        if (params[key] instanceof Array) {
          const value = params[key].map(item => '"' + item + '"');
          paramsArray.push(key + '=[' + value.join(',') + ']');
        } else {
          paramsArray.push(key + '=' + params[key]);
        }
      });
      if (url.search(/\?/) === -1) {
        url += '?' + paramsArray.join('&');
      } else {
        url += '&' + paramsArray.join('&');
      }
    }
    return await this.dofetch(url, options, settings);
  }

  /**
   * post请求
   * @param {String} url
   * @param {Object} params
   * @param {Object} settings: { isServer, noLoading, cookies }
   */
  async post(url, params = {}, settings = {}) {
    const options = { method: 'POST' };
    options.body = JSON.stringify(params);
    return await this.dofetch(url, options, settings);
  }

  /**
   * fetch主函数
   * @param {*} url
   * @param {*} options
   * @param {Object} settings: { isServer, noLoading, cookies }
   */
  dofetch(url, options, settings = {}) {
    const { isServer, noLoading, cookies = {} } = settings;
    let loginCondition = false;
    if (isServer) {
      this.headers.cookies = 'cookie_name=' + cookies['cookie_name'];
    }
    if (!isServer && !noLoading) {
      loginCondition = Router.route.indexOf('/login') === -1;
      this.showLoading();
    }
    const prefix = isServer ? process.env.BACKEND_URL_SERVER_SIDE : process.env.BACKEND_URL;
    return Promise.race([
      fetch(prefix + url, { headers: this.headers, ...this.init, ...options }),
      new Promise((resolve, reject) => {
        setTimeout(() => reject(new Error('request timeout')), REQUEST_TIEM_OUT);
      })
    ])
      .then(response => {
        !isServer && !noLoading && this.hideLoading();
        if (response.status === 500) {
          throw new Error('服务器内部错误');
        } else if (response.status === 404) {
          throw new Error('请求地址未找到');
        } else if (response.status === 401) {
          if (loginCondition) {
            Router.push('/login?directBack=true');
          }
          throw new Error('请先登录');
        } else if (response.status === 400) {
          throw new Error('请求参数错误');
        } else if (response.status === 204) {
          return { success: true };
        } else {
          return response && response.json();
        }
      })
      .catch(e => {
        if (!isServer && !noLoading) {
          this.hideLoading();
          Toast.info(e.message);
        }
        return { success: false, statusText: e.message };
      });
  }
}

export default ProxyFetch.getInstance();

```

## 写在最后

一个完整项目的雏形大致出来了，但是还是需要在实践中不断打磨和优化。

如有错误和问题欢迎各位大佬不吝赐教 :)
