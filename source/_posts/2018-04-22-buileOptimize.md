---
title: 记一次webpack构建优化
date: 2018-04-22 10:47:50
categories: js
tags:
  - webpack
  - 优化
---

一次偶然的机会，我知道了webpack有很多可视化工具，可以帮助你分析应用的依赖图表，找出不好的地方，让你去优化他们。

<!-- more --> 

## webpack Analyze

我用的是webpack的官方分析工具[Analyze](https://webpack.github.io/analyse/)，使用它之前，你需要先生成一份**包含关于模块的统计数据的json文件**，你可以参照[这篇文档](https://doc.webpack-china.org/api/stats/#src/components/Sidebar/Sidebar.jsx)，如果你使用的是Node APi，那么你可能需要参考[这篇文档](https://doc.webpack-china.org/api/node/#stats-%E5%AF%B9%E8%B1%A1-stats-object-)。

将生成好的json文件上传至Analyze，你会看到如下图所示界面。这是你应用的构建概览。你可以在Modules，Chunks，Assets目录里看见更详细的报告，也可以在Hints目录里查看优化提示。

{% asset_img 01.jpg Analyze Home %}


## 大量module被多个chunks依赖

让我们回到主题，点击Hints我发现，在我的应用中有很多个module被多个chunk依赖，这使得我chunk的尺寸比预期要大，会使页面访问速度变慢。我感到很奇怪？我的项目是用Vue-cli生成的，项目里配置了webpack的CommonsChunkPlugin插件，按理说这些被多个chunk依赖的module是会被打包进一个公共文件才对的。为什么这些module没有被打包进公共文件？问题出在哪？

{% asset_img 02.jpg 大量module被多个chunks依赖 %}

## 问题出在哪呢？

首先我猜测，可能是CommonsChunkPlugin插件配置不对，我依次尝试了[官方文档](https://doc.webpack-china.org/plugins/commons-chunk-plugin/)上的配置案例，问题任然存在。看来不是配置的问题。那问题出在哪呢？我整个人有点斯巴达，看样子又得去看源码了。

在看源码之前，我灵光一闪，有了新的发现。

我在配置代码里加了一行代码，打印资源路径。发现——我通过代码分割功能引入的子chunk的路径并没有出被打印在控制台中，也就是说，**CommonsChunkPlugin不会去提取被分割的子chunk中的公共文件**。

```js
new webpack.optimize.CommonsChunkPlugin({
  name: 'vendor',
  minChunks: function (module, count) {
    console.log(module.resource) // 打印资源路径
    return (
      module.resource &&
      /\.js$/.test(module.resource) &&
      module.resource.indexOf(
        path.join(__dirname, '../node_modules')
      ) === 0
    )
  }
}),
```

得出的这个令我很兴奋，我马上停用掉代码分割功能，来验证我的想法。

```js
// 通过代码分割功能异步引入的页面组件
const pageOne = () => import('@/page/pageOne' /* webpackChunkName: "pageOne" */)
const pageTwo = () => import('@/page/pageTwo' /* webpackChunkName: "pageTwo" */)

// 停用代码分割功能
import pageOne from '@/page/pageOne'
import pageTwo from '@/page/pageTwo'
```

验证的结果同我的猜想一样。

## 优化结果对比

找到原因问题就好解决了，既然**CommonsChunkPlugin不会去提取被分割的子chunk中的公共文件**，那我就手动将他们提取到一个公共文件里面去。

放一张优化前后的构建对比图。

{% asset_img 03.jpg 优化效果对比 %}

从图中可以看到，文件尺寸平均减少了50kb，画红线的三个文件更是减少了100kb。