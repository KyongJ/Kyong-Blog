---
title: 记录一次Webpack性能优化
description: 代码分析，利用 webpack-bundle-analyzer 检查当前 bundle
author: Kyong
date: 2023-04-27
tag: 
  - 性能优化
---
# 记录一次Webpack性能优化

## 开始项目

![image-20230301173729894](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20230301173729894.png)



### 1、代码分析

利用 `webpack-bundle-analyzer` 检查当前 bundle

```ts
// 安装
tarn add -D webpack-bundle-analyzer

// 配置
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin()
  ]
}

// 使用 构建时自动弹出
yarn build
```



> 使用webpack-bundle-analyzer后的图片

![image-20230301175505016](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20230301175505016.png)



> 路由懒加载后

![image-20230301201806008](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20230301201806008.png)

由图可以看到，webpack  不仅将路由分离成了单独的bundle，并且对依赖进行了拆解，只在需要的路由页进行加载，这大大减少了首屏资源加载体积（37%），将明显提前进入首屏渲染逻辑。



### 2、Prefetch 预获取

由于路由懒加载功能将非首屏依赖抽离出来，默认只在进入对应路由页面时刻进行加载，这会导致路由跳转时出现白屏或者`loading`，影响用户体验。所以我们使用 `Prefetch`预获取功能来帮助解决这个问题。

使用 `webpackPrefetch`：告诉浏览器将来可能需要该资源来进行某些导航跳转

```ts
const Archive = lazy(() => import(/* webpackPrefetch: true, webpackChunkName: "Archive" */'../pages/archive'));
const Article = lazy(() => import(/* webpackPrefetch: true, webpackChunkName: "Article" */'../pages/article'));
const ArticleHomeList = lazy(() => import('../pages/articleHomeList'));
const Category = lazy(() => import(/* webpackPrefetch: true, webpackChunkName: "Category" */'../pages/category'));
const Tag = lazy(() => import(/* webpackPrefetch: true, webpackChunkName: "Tag" */'../pages/tag'));
```



### 3、解决React+Ant Design 打包后vendors.js过大问题

按需引入

项目介绍：自研Web 端的 React 组件库，全部组件由 TypeScript 与 React Hooks 编写

项目功能：

- 已完成Input、Button、Select、InputDatePicker等组件，InputDatePicker支持用户手动输入日期
- 使用Eslint、prettier等工具提高代码，使用husky提高commit质量
- 使用 react-testing-library 完成单元测试，提高组件库功能完整性与正确性

**项目介绍**：适合于个人使用的技术博客，拥有后台管理系统进行协同管理。后台管理系统分为四个模块，分别为用户模块、文章模块、资源模块以及角色权限模块。 

**技术总结**：TypeScript +React Hooks+ Redux + Ant Design 

**项目职责**： 

- 使用gzip压缩资源文件；使用webpack中的external配合cdn分流一些稳定的第三方库；使用路由懒加载避免首屏一次性加载全部资源
- 通过以上优化工作，经测试在相同网速下，DOMContentload/loaded时间由原来的10秒提升到3.9秒,大大提升了用户的使用体验
- 采用图片懒加载技术，提高页面加载速度，减少卡顿，优化用户体验