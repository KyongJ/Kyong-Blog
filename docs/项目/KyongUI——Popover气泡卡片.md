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