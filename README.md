# HTML Playground

一些前端交互效果的 Demo 合集。

## 项目列表

### [double-scroll](./double-scroll/)

移动端两阶段滚动交互 Demo。白色容器（Bottom Sheet 风格）先整体上滑覆盖背景，到顶后容器保持圆角，内部商品内容继续滚动。

- `index.html` — v1 原型：背景随页面滚动
- `index2.html` — v2 最终版：三层架构 + 幽灵占位映射滚动

核心技术：`position: sticky` + phantom scroll 映射，零事件拦截，纯原生滚动体验。
