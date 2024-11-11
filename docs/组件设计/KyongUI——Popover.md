---
title: KyongUI——Popover气泡卡片
description: 在YY直播实习的时候，负责的是YY520交友活动页面的搭建，其中有一个需求就是页面会存在一个倒计时组件，在活动结束的前一天在完成某些任务后，可以点击抽奖。由于 `UI` 库使用的是 `antd`，所以我第一反应是使用 `antd` 的 `CountDown` 组件。
author: Kyong
date: 2023-05-23
tag: 
  - 组件设计
hidden: true
---
# KyongUI——Popover气泡卡片

## 一、设计Props

| 属性            | 介绍                           | 类型              | 默认值   |
| --------------- | ------------------------------ | ----------------- | -------- |
| className       | 自定义类名                     | `string`          | `--`     |
| style           | 自定义样式                     | `CSSProperties`   | `--`     |
| type            | 触发形式 hover/click           | `string`          | `hover`  |
| align           | 对齐方式 left/right/top/bottom | `string`          | `bottom` |
| content         | 卡片内容                       | `ReactNode`       | `<></>`  |
| width           | 卡片宽度                       | `number / string` | `200px`  |
| noBorder        | 无边框                         | `boolean`         | `false`  |
| defaultShow     | 默认显示气泡卡片               | `boolean`         | `false`  |
| closeDeps       | 气泡卡片关闭依赖项             | `any[]`           | `[]`     |
| onVisibleChange | 卡片显示隐藏回调               | Function          | `--`     |

二、