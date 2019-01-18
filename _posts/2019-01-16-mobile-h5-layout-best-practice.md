---
layout: post
title: "移动端优雅布局实践"
subtitle: "在项目初期，没有定一个好的布局简直是一个噩梦的开始。在经历了两周多的优化过后，总结出了一条关于布局的经验：使主容器脱离文档流+fixed布局，既可以兼容ios滚动效果也可以方便实现单屏自适应。"
date: 2019-01-12 21:00:00
author: "张豆豆"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
  - 总结
  - SSR
  - react
  - next
  - flex
  - 移动端web app
---

> “学无止境！坑越大，变得足够强了才能越过去~”

# 移动端优雅布局实践

> 前言：移动端有非常多的坑，布局首当其冲。

## 背景

移动端应用有各种复杂的页面需求，不仅要解决单屏、多屏、固定头部或底部等多个场景，还要兼容 ios 和 Android 内核，在经历了[项目实战(手机模式打开)](https://www.unicareer.com/academy/)过后，总结出了一些经验，在这里和大家分享一下。

这篇文章是基于 →
[Next 轻量级框架与主流工具的整合](https://segmentfault.com/a/1190000016383263)  
最新的代码在这里 →
[next-mobile-complete-demo](https://github.com/code-coder/next-mobile-complete-app)

## 心路历程

首先，需要实现的首页类似于 app 应用。

![截图示例](http://doudou-static.oss-cn-shanghai.aliyuncs.com/%E5%B8%83%E5%B1%80.png)

- #### 第一站：统一 flex 布局的坑

  > → [flex 教程](http://www.runoob.com/w3cnote/flex-grammar.html)

  首页如上面的图片所示，然后偷了个懒，直接用了[antd-mobile 的 tab 标签栏](https://mobile.ant.design/components/tab-bar-cn/)，它需要指定容器高度。于是，在项目最开始的时候直接通过 js 将容器设置为浏览器 html 窗口的高度：

```
// html高度 = body高度 = 主容器高度
doc.body.style.height = docEl.clientHeight + 'px';
```

这样做也有许多好处：纵向布局十分方便，不管是 tab 标签栏、tab 标签页、头部固定元素或者底部固定元素实现起来都很简单；也能很简单就能实现元素居中对齐、两端对齐、自动分配空间等；同时也避免了 Android 输入框引起传统布局的问题。

不过这里有个唯一的不好的地方就是兼容不了 ios 的前进返回的操作栏和上下滚动回弹：

- 不能在滚动时隐藏前进返回的操作栏、
- 在上下滚动时容易触发页面的回弹效果，造成滚动卡顿。

> 这里补充一下：在 ios 上如果以正常流布局，内容超过一定高度时，向下滚动会隐藏底部的前进返回栏，向上滚动的时候再显示出来。当滚动到底部或者顶部时，再继续拉动页面，会有一个回弹的效果。

这两个问题极大的降低了 ios 的用户体验，又恰逢项目间隔期，有大把的时间，于是被推着到了布局优化上面。

- #### 第二站：寻找良好用户体验的布局案例

这里我们在寻找的过程中发现两个具有代表性的移动端应用：

1. [bilibili 的 m 站](https://m.bilibili.com)、[腾讯新闻](xw.qq.com)

   它用的是传统式流布局，头部和底部 fixed 固定，所有的内容全部向下平铺。

2. [花瓣网 H5](http://huaban.com/)

   它采用的是设置主容器的 position 属性为 absolute 来脱离文档流的形式。

这两种布局方式在 ios 上都用户体验极好。但同时也发现他们都存在的一些问题，比如：登录页面明明不足一屏，却依然存在滚动；没有兼容 iPhone X 的安全域；弹窗滚动穿透等等。

如果能将这两种布局和 flex 进行整合一下，聚合各自的优点，基本上能打造一个用户体验和兼容性都令人满意的移动端应用了。

那么，如何基于现有的 flex 布局去整合这两种布局又成了一个问题。

- #### 第三站：基于 flex 调整布局结构

在调整之前，我的预期是这样的：**去掉外层高度限制，去掉纵向 flex 布局（除极少数单屏页面），将顶部、底部固定和弹窗设置为 fixed**。

考虑到少数单屏页面需要继承 html 窗口的高度，于是采用了主容器脱离文档流的方式。

调整过后 dom 结构是这样的：

    html > body > div#root > div.main-content(position: absolute)

只需要给**_main-content_**加上`height: 100%`,就可以满足单屏页面的需求。

> 好事多磨！

按照上面的想法进行改造过后又发现了新的问题：**脱离文档流过后，切换页面，html 会保持最后滚动的位置**。

- #### 第四站：滚动条位置、滚动穿透、兼容 iPhone X

1. **有可能出现滚动条位置被记录的问题**  
   找了一下发现`history`有个`scrollRestoration`的属性，它有两个值，一个是默认值`auto`还有一个是`manual`。如果设置为`manual`就可以手动设置滚动的位置。

   ```
   if ('scrollRestoration' in history) {
     history.scrollRestoration = 'manual';
   }
   ```

   然后在切换路由过后设置滚动的位置为起始位置：

   ```
   Router.events.on('routeChangeComplete', () => {
   document.scrollingElement.scrollTop = 0;
   });
   ```

   这样每次切换页面都是从初始位置开始。

   > 开发遇见的问题，这个 demo 中没有出现。这里有点鸡肋，前进后退也不会记录滚动位置。如果需要前进后退记录滚动的位置，就不能用这种脱离文档流的形式，需要用 body 滚动的形式，也就是。

2. **解决滚动穿透**  
   这个问题，需要针对有滚动的弹窗和无滚动的弹窗单独处理。

   - 针对无滚动的弹窗，在弹窗弹出的时候禁用 touchmove 事件，隐藏的时候移除事件监听。
   - 有滚动的弹窗，需要找到页面的滚动的容器设置`overflow:hidden`

3. **兼容 iPhone X**

   在 meta 标签后增加`viewport-fit=cover`

   ```
   <meta name="viewport" content="initial-scale=1,maximum-scale=1,minimum-scale=1,user-scalable=no,viewport-fit=cover"/>
   ```

   然后留出安全域距离

   ```
   padding-bottom: 0.49rem; // 底部button的高度
   padding-bottom: calc(0.49rem + constant(safe-area-inset-bottom));
   padding-bottom: calc(0.49rem + env(safe-area-inset-bottom));
   ```

   这里需要注意些`padding-bottom`的时候需要写三个，第一个是为了兼容 Android。

## 总结

总的来说 flex 布局还是带了十分大的便利，尤其是能根治居中、适配等一些老大难的问题。

每种布局的方式都有一定的缺陷，到目前为止还没有一种万能的方案来解决移动端各种复杂的场景。还是需要根据自己的需求还选择使用哪种实现方式。

**\*前面这三种方案各自的缺陷：**
纵向 flex 布局（不建议使用） | body 流式平铺（传统） | absolute 脱离文档流（正在使用）
---|---|---
ios 前进后退的操作栏无法隐藏、回弹效果引起卡顿 | 内部容器无法继承 html 窗口高度 | 路由跳转可能出现滚动条被记录
