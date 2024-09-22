# Taro引入Ant Design-Mobile

> 引言
>
> 最近在使用Taro开发微信小程序项目，由于Taro自带的UI内容有限，遂想引入Antd-mobile，方便开发



本文主要是基于凹凸实验室的文章做一点小修改（完全按照凹凸实验室发的，UI会消失）

[Taro 3.3 alpha 发布：用 ant-design 开发小程序？](https://juejin.cn/post/6955290839220224031)

[Ant Design Mobile](https://github.com/NervJS/taro-antd-mobile)



### Get Starting

> 如果已经创建好了项目，请直接跳到第四步

##### 1、进入Taro官网 [Taro](https://docs.taro.zone/docs/GETTING-STARTED)

##### 2、安装 CLI （安装alpha版本）

```bash
# 安装 CLI
$ npm i @tarojs/cli@alpha -g
```

接下来跟着官网操作下去就行了

##### 3、安装antd-mobile

```bash
$ npm install --save antd-mobile
# or
$ yarn add antd-mobile
# or
$ pnpm add antd-mobile
```

##### 4、如果已经创建好了项目，需要更换taro版本

```bash
# 卸载之前的版本
$ npm uninstall @tarojs/cli -g

# 安装alpha版本
$ npm i @tarojs/cli@alpha -g

# 进入项目目录
$ taro update project [taro版本号]

# 删除node_modules,重新安装
```



### 兼容工作

相对于官方示例的 H5 代码，本项目主要做了以下兼容工作：

##### 1、首先需要安装插件 `@tarojs/plugin-html`

```bash
$ npm i @tarojs/plugin-html
```

然后配置使用即可：

```js
// config/index.js
const config = {
  // ...
  plugins: [
    '@tarojs/plugin-html'
  ]
}
```

##### 2、引入css样式

>  凹凸实验室的方法   本人亲测根据凹凸实验室发的文章安装会导致css样式丢失

Taro 提供两种内置样式我们可以直接引入生效：

- `@tarojs/taro/html.css`: W3C HTML4 的内置样式，只有 HTML4 标签样式，体积较小，兼容性强，能适应大多数情况。
- `@tarojs/taro/html5.css`: Chrome(Blink) HTML5 的内置样式，内置样式丰富，包括了大多数 HTML5 标签，体积较大，不一定支持所有小程序容器。

```
// app.css
// html4
import '@tarojs/taro/html.css';
// html5
import '@tarojs/taro/html5.css';
```



> 我的方法

将`@tarojs/taro/html.css` 和`@tarojs/taro/html5.css` 复制到`src`文件下，然后引入`app.ts`文件中

```js
// app.ts
import './html.css'
import './html5.css'
```



### 组件支持列表

详见上文链接