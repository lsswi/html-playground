# Bottom Sheet 两阶段滚动商品展示页 — 设计文档

## 概述

实现移动端商品展示页面的两阶段滚动交互：白色容器（bottom sheet 风格）先整体上滑到导航栏下方，到顶后保持圆角可见，容器内部商品内容继续滚动。

## 需求

- 导航栏 + 背景是一个连续渐变的整体（橙色→深棕），不是两块割裂的区域
- 白色圆角容器浮在背景之上，从下方滑上来覆盖背景
- 默认状态：白色容器位于屏幕中下部，露出上方促销 banner
- 上滑：白色容器整体上滑，直到顶部紧贴导航栏（覆盖 banner）
- 容器到顶后：**圆角始终保持可见**，继续上滑滚动容器内部商品内容
- 置顶商品卡片跟随内容滚走，不吸顶
- 单向上滑，不支持下拉回初始位置
- 纯 HTML + CSS + JS 单文件 demo

## 技术方案：三层架构 + 幽灵占位映射滚动

### 三层结构

```
z-index: 100  ── nav-overlay（导航栏，fixed，始终在最上层）
z-index:   1  ── scroll-layer（滚动层，页面唯一滚动源）
                   ├── scroll-spacer（220px 透明占位，让背景透出来）
                   ├── product-container（白色容器，sticky top:88px）
                   │   └── product-scroll（内部内容区，overflow:hidden，JS 控制 scrollTop）
                   └── scroll-phantom（幽灵占位，JS 动态设高，撑起页面滚动空间）
z-index:   0  ── bg-layer（固定背景，fixed，导航栏+banner 为同一渐变）
```

### 背景一体化

导航栏和背景使用**同一渐变色系**，在 88px（导航栏底边）处色值完全一致，视觉上融为一体：
- `bg-layer`: `linear-gradient(180deg, #FF6A00 0%, #FF5500 88px, #CC3300 45%, #3D1500 100%)`
- `nav-overlay`: `linear-gradient(180deg, #FF6A00 0%, #FF5500 100%)`
- 两层在 88px 交界处无色差

### 滚动机制：幽灵占位 + scroll 映射

核心思路：**始终只有外层页面在原生滚动**，不拦截任何事件。通过幽灵元素撑高页面，把超出阈值的滚动量映射到内部 scrollTop。

1. **阶段1（容器上滑）**：`scrollY <= spacerHeight(220px)`
   - 页面正常滚动，spacer 被推出视口
   - 白色容器跟随滚动上移，逐渐覆盖背景
   - 内部 `scrollTop = 0`，内容不动

2. **切换点**：`scrollY = spacerHeight`
   - 白色容器到达 `sticky top:88px`，粘住
   - 圆角始终可见（容器 `overflow:hidden` + 固定高度 `calc(100vh - 88px)`）

3. **阶段2（内部内容滚动）**：`scrollY > spacerHeight`
   - 页面继续滚动（由 phantom 撑起的额外高度）
   - `scrollEl.scrollTop = scrollY - spacerHeight`
   - 内容在白色容器圆角窗口内流动

```
页面总滚动高度 = spacer(220) + container(100vh-88px) + phantom(innerOverflow)
                 ↑阶段1↑                               ↑阶段2↑
```

### 页面结构

```
html/body（整体滚动）
├── div.bg-layer（position: fixed, z-index: 0）
│   │  background: 连续渐变 #FF6A00 → #3D1500
│   ├── div.bg-top（导航栏占位，88px）
│   └── div.banner-content
│       ├── "超值特惠专区"（金色 #FFCC80）
│       └── "逛一逛 得奖励✨"（金色渐变 #FFD700→#FFEE88）
│
├── nav.nav-overlay（position: fixed, z-index: 100, 88px）
│   │  background: 渐变 #FF6A00 → #FF5500（与 bg-layer 顶部一致）
│   ├── 返回按钮
│   └── "浏览商品 15 秒，得奖励" 提示条
│
└── div.scroll-layer（position: relative, z-index: 1）
    ├── div.scroll-spacer（220px, 透明，让 bg-layer 透出）
    ├── section.product-container（sticky top:88px, 固定高度, overflow:hidden）
    │   └── div.product-scroll（overflow:hidden, JS 控制 scrollTop）
    │       ├── div.featured-product（横排：左图140×140 + 右详情）
    │       ├── div.shop-row（店铺信息 + 购买按钮）
    │       └── div.product-grid（双列 CSS Grid, gap 10px）
    │           └── div.product-card × 20
    └── div.scroll-phantom（高度 = innerOverflow, JS 动态计算）
```

### 视觉规格

| 元素 | 样式 |
|------|------|
| 导航栏 | 渐变 #FF6A00→#FF5500, 高度 88px（含安全区）, 文字白色 |
| 背景层 | 渐变 #FF6A00→#FF5500→#CC3300→#3D1500, 全屏 fixed |
| 促销文字 | 副标题 #FFCC80, 主标题金色渐变 #FFD700→#FFEE88 |
| 白色容器 | 白色, border-radius: 16px 16px 0 0, 阴影, 固定高度 calc(100vh-88px) |
| 置顶商品卡 | 横排布局, 左图(140×140) + 右详情, 圆角 12px, 阴影 |
| 双列网格 | CSS Grid 2列, gap 10px |
| 商品卡片 | 白色, 圆角 10px, 阴影 |
| 价格 | 红色 #FF3B30, 粗体 |
| 原价 | 灰色 #bbb, 删除线 |

### 数据

所有数据为硬编码 mock 数据：
- 1 个置顶商品（大卡片）
- 20 个双列商品（确保内容足够长可滚动）
- 图片用纯色渐变 div 占位
- 文字为模拟的商品名称/价格

## 文件结构

```
double-scroll/
├── index.html    ← v1: 早期原型（背景随页面滚动）
└── index2.html   ← v2: 最终版（三层架构 + 幽灵占位映射）
```

## 约束

- 单个 HTML 文件，无外部依赖
- 移动端优先（375px 宽度视口）
- 图片用纯色渐变矩形代替
- 文字使用模拟数据
