# PPTX 兼容规范 / pptx_compat

本文件融合中车风格 PPT 的 PPTX 兼容规范与 dom-to-pptx 引擎的工程白名单。**冲突时一律以本文件为准**（即最严口径）。

dom-to-pptx 引擎原理：读取 DOM 的 `getBoundingClientRect()` 与 computed styles → 转为原生 PPTX shape。所以**只有"能被静态计算出来的样式"才能可靠转换**；任何动态、依赖渲染过程或视觉合成的特性都会被丢弃或失真。

---

## 1. HTML 与布局

### 必须

- 输出**完整 HTML**，不依赖本地文件、命令行脚本、外部样例或后处理工具。
- 每页必须是 `<div class="slide">`，且固定：
  ```css
  width: 1920px;
  height: 1080px;
  position: relative;
  overflow: hidden;
  background: #FFFFFF;
  ```
- 幻灯片内容**全部位于 `.slide` 内**。
- `.slide` **不得位于带 `transform` 的父容器中**（包括 rotate）—— 父级 transform 会破坏 bounding-rect 计算。
- 幻灯片内容样式以**内联 `style`** 为主。`<style>` 仅用于：
  - 页面背景
  - Export 按钮样式
  - 全局 reset
  - 非幻灯片 chrome（如 `.slide-stage` 容器）
- 位置优先使用**绝对定位**：`position: absolute; left/top/width/height: Npx;`。
- 固定尺寸容器内部可使用 `display: flex` / `display: grid`（最终 rect 会被测量，布局方式对引擎透明）。

### 尺寸单位

| 单位 | 状态 | 说明 |
|---|---|---|
| `px` | ✅ 默认 | 全部幻灯片内容必须用 px |
| `%` | ⚠️ 限制 | 仅可用于已有固定 px 宽高的父容器内部 |
| `rem` / `em` | ❌ 禁用 | 行为不可预测 |
| `vh` / `vw` / `vmin` / `vmax` | ❌ 禁用 | 视口单位绝对禁止 |
| `pt` / `cm` / `in` | ❌ 禁用 | 别混用度量体系 |
| `fr`（grid） | ✅ | grid 轨道在测量前已解析 |
| `clamp()` / `min()` / `max()` | ❌ 禁用 | 即使引擎"勉强可用"也不要用 |

### 防溢出

- 所有固定尺寸容器、卡片和文本区域**必须有明确 `width` 与 `height`**，并加：
  ```css
  box-sizing: border-box;
  min-height: 0;
  overflow: hidden;
  ```
- `validateSlides()` 会对 **flex/grid 的直接子项**强制校验 `min-height: 0`（缺失会触发纵向 blow-out 警告）。
- **长文本必须拆分、压缩、分行或降级为要点**。不得依赖 `padding`、flex/grid 自撑高或导出器自动扩容。
- 表格可用原生 `<table>` `<tr>` `<td>` `<th>`，但**必须放在固定 px 容器中**，且单元格样式需内联。dom-to-pptx v1.1.6+ 才将原生 table 转为 PPTX table（含边框、单元格边距、富文本）。
- 预览页面不得让 1920px 宽幻灯片在常见桌面视口中被水平裁切。如保留页面 padding 或居中外层容器，必须设置足够 `min-width` 或调整预览结构。
- Export 按钮等非幻灯片 chrome 元素不受 1920px 约束，但应确保**不遮挡 `.slide` 主要内容区域**。

---

## 2. 颜色与背景

| 特性 | 状态 | 说明 |
|---|---|---|
| `background-color: #hex` / `rgb()` / `rgba()` / 命名色 | ✅ | 全部支持 |
| `background: linear-gradient(...)` | ✅ | 任意角度，多 stop，含透明度 |
| `background: radial-gradient(...)` | ❌ **禁用** | 导出 PPT 后效果丢失（引擎静默丢弃装饰性背景层），不要使用。 |
| `background: conic-gradient(...)` | ❌ | 不支持 |
| 多重背景（gradient + url） | ⚠️ | 只有第一层可靠，**复杂叠层拆为多个绝对定位 div** |
| `background-blend-mode` / `mix-blend-mode` | ❌ | 忽略 |
| `opacity` | ✅ | |
| CSS 自定义属性 `var(--brand)` | ⚠️ | 工作但不利于调试，色值优先直写 |

---

## 3. 边框、圆角、阴影

| 特性 | 状态 | 说明 |
|---|---|---|
| `border: Npx solid <color>` | ✅ | |
| 单边 `border-top` / `border-left-width` 等 | ✅ | |
| `border-style: dashed` / `dotted` | ⚠️ | 常退化为 solid，强分隔慎用 |
| `border-radius`（px / % / 单角） | ✅ | 单角 `border-top-right-radius: 24px;` 支持 |
| `outline` | ❌ | 不映射，用 border |
| `box-shadow: x y blur <color>`（外阴影） | ✅ | 转为 PPTX outer shadow |
| `box-shadow: inset ...` | ❌ | 内阴影忽略 |
| 多重 `box-shadow` | ⚠️ | **只渲染第一个** —— 只写一层 |
| `text-shadow` | ❌ **禁用** | 经常掉，用色重 + 字重对比代替 |

---

## 4. 字体与文本

| 特性 | 状态 | 说明 |
|---|---|---|
| `font-family` 含 web-safe 后备 | ✅ | 固定用 `'Microsoft YaHei', 'Inter', Arial, sans-serif` |
| `font-size: Npx` | ✅ | 小数会保留 1/10 pt 精度，**优先整数 px** |
| `font-size: rem/em` | ❌ 禁用 | |
| `font-weight: 100-900` | ✅ | |
| `font-style: italic` | ✅ | |
| `line-height`（无单位或 px） | ✅ | |
| `letter-spacing` | ✅ | v1.1.6+ 映射为 PPTX charSpacing |
| `text-transform: uppercase/lowercase/capitalize` | ✅ | v1.1.6+ |
| `text-align: left/right/center/justify` | ✅ | |
| `white-space: nowrap/pre/pre-wrap` | ✅ | |
| `text-decoration: underline/line-through` | ✅ | |
| `writing-mode: vertical-rl/vertical-lr` | ✅ | v1.1.7 路由到 PPTX 竖排 |
| `word-break` / `overflow-wrap` | ⚠️ | CJK 自动撑高文本框（v1.1.7 autoFit），但仍按本文 §1 防溢出处理 |
| `<strong>` / `<em>` / `<span style="...">` 内联富文本 | ✅ | 富文本 run 保留，含表格单元格内 |
| Google Fonts `<link>` | ❌ **禁用** | 禁止任何外链 CSS，包括 Google Fonts |
| 图标字体（FontAwesome / Material 等） | ❌ **禁用** | 禁止外链图标字体，icon 仅手写内联 SVG |

---

## 5. 图片

| 特性 | 状态 | 说明 |
|---|---|---|
| `<img src="https://...">` 含 CORS | ✅ | 必须 HTTPS 且 CORS 启用 |
| `<img src="data:image/...;base64,...">` | ✅ | 注意文件体积 |
| `<img src="./local.jpg">` / `file://...` | ❌ | 相对路径与本地路径都失败 |
| 无 CORS 头的图片 URL | ❌ | 圆角/遮罩引擎会失败，图片可能掉 |
| `object-fit: cover / contain / fill` | ✅ | **本规范要求统一用 `cover`** |
| `border-radius` 应用于 `<img>` | ✅ | v1.1.0+ 离屏 canvas masking，无白边 |
| `loading="lazy"` | ❌ **禁用** | 可能在导出时还未加载 |
| `srcset` / `<picture>` / `<source>` | ❌ **禁用** | 行为不可预测 |

---

## 6. SVG 与矢量

| 特性 | 状态 | 说明 |
|---|---|---|
| 内联 `<svg>` | ✅ | 默认栅格化 |
| `svgAsVector: true` 导出选项 | ✅ **默认开启** | 保留矢量，PowerPoint 可"转换为形状"编辑 |
| 内嵌外链 `<image href="https://...">` | ❌ **禁用** | 即使有 CORS 也禁用 |
| SVG `<filter>` / `<mask>` 在 vector 模式 | ⚠️ | 可能 round-trip 失真，避免使用 |
| 外部 `<img src="x.svg">` | ✅ | 当作图片处理（按 §5 规则） |

### SVG 元素白名单

`<path>` / `<line>` / `<polyline>` / `<polygon>` / `<circle>` / `<rect>` / `<ellipse>` / `<g>` / `<text>`

### SVG 禁止项

| 元素 / 属性 | 原因 |
|---|---|
| `<animate>` / `<animateTransform>` | 动画 |
| `<script>` | 运行时脚本 |
| `<foreignObject>` | 嵌套 HTML，行为不可控 |
| `<image href="...">` | 外链资源 |
| `<use href="#...">` 跨文件引用 | 外链资源 |

### 数据图表的真实几何

- 饼图、环形图、进度环**禁止用 `stroke-dasharray` / `stroke-dashoffset` 模拟**。
- 必须使用真实 `<path>` 圆弧或扇区几何。无法可靠计算时**降级为柱状图、条形图、表格或数字卡**。
- `<circle>` **仅用于完整圆点、节点、标记或装饰**；不得用于"描边裁切式"数据图表。

---

## 7. 动画、交互、特效（全部禁用）

| 特性 | 状态 | 替代方案 |
|---|---|---|
| `@keyframes` / `animation` | ❌ | 多导出几张幻灯片表达状态变化 |
| `transition` | ❌ | 只有 resting 状态会被导出，所以不需要 |
| `:hover` / `:focus` / `:active` | ❌ | 只读取 default state |
| `filter: blur(Npx)` | ❌ **禁用** | 即使引擎映射为软边也不要用 |
| `filter: brightness/contrast/saturate/hue-rotate/grayscale/sepia/invert/drop-shadow` | ❌ | 忽略，把效果烘焙进图片 |
| `backdrop-filter` | ❌ | 用半透明覆盖层 div 替代 |
| `clip-path` | ❌ | 用 `border-radius` 或 SVG 几何替代 |
| `mask-image` | ❌ | 同上 |

### `transform` 子项

| 属性 | 状态 | 说明 |
|---|---|---|
| `transform: rotate(Ndeg)` | ⚠️ **限量** | **仅可用于少量倾斜标签或装饰**，不得用于卡片主体、正文、标题 |
| `transform: translate / translateX / translateY` | ❌ | 用 `left` / `top` 代替 |
| `transform: scale` | ❌ | 直接设最终 width/height |
| `transform: skew` / `matrix` | ❌ | |
| `transform-origin` | ❌ | |

居中、缩放、位移必须用 flex/grid 或 `left/top/width/height` 实现，**不得**用 `transform: translate(-50%, -50%)`。

---

## 8. HTML 元素

| 元素 | 状态 | 说明 |
|---|---|---|
| `div` / `span` / `section` / `article` / `header` / `footer` / `figure` / `figcaption` | ✅ | |
| `p` / `h1`-`h6` | ✅ | |
| `ul` / `ol` / `li` | ✅ | |
| `img` / `svg` | ✅ | 见 §5 §6 |
| `a` | ✅ | 渲染为带样式文本 |
| `button` | ✅ | 当作 styled box |
| `input[type=text]` / `textarea` | ⚠️ | 仅取当前 value 为纯文本 |
| `table` / `tr` / `td` / `th` | ✅ | v1.1.6+ 原生 PPTX table |
| `video` / `audio` / `canvas` / `iframe` | ❌ **禁用** | 不捕获，需先截图为 `<img>` |
| `form` / `select` / `option` | ⚠️ | 渲染为 styled box，语义丢失 |

---

## 9. 禁用项总清单（速查）

```
CSS:
  vh / vw / vmin / vmax
  rem / em（用于幻灯片内容）
  clamp() / min() / max()
  transform: translate / scale / skew / matrix
  transform-origin
  backdrop-filter
  conic-gradient
  radial-gradient 复杂用法（ellipse / 偏移圆心 / 尺寸关键字；简单居中圆形 radial-gradient(circle …) 例外可用）
  mix-blend-mode / background-blend-mode
  clip-path / mask-image
  text-shadow
  filter（除非完全不写 filter 属性）
  :hover / :focus / :active / 伪类伪元素交互
  animation / transition / @keyframes
  outline
  box-shadow: inset
  多重 box-shadow（只写一层）

HTML:
  iframe / canvas / video / audio
  外链 CSS（含 Google Fonts、Tailwind CDN 等）
  外链图标字体（FontAwesome / Material 等）
  <img loading="lazy">
  <img srcset> / <picture> / <source>
  相对路径或 file:// 的图片
  无 CORS 头的图片 URL

SVG:
  <animate> / <animateTransform>
  <script>
  <foreignObject>
  <image href="...">（外链）
  跨文件 <use href="...">
  数据图表用 stroke-dasharray / stroke-dashoffset 伪造饼图/环形图/进度环
```

---

## 10. 居中速查表

| 需求 | 推荐做法 | 反例 |
|---|---|---|
| 元素水平+垂直居中 | 父容器 `display: flex; justify-content: center; align-items: center;` | `transform: translate(-50%, -50%)` ❌ |
| 圆形裁剪 | `border-radius: 50%` | `clip-path: circle()` ❌ |
| 圆角图片 | `<img>` 上加 `border-radius` | SVG mask ❌ |
| 半透明面板 | `background: rgba(...)` 或 `opacity` | `backdrop-filter: blur` ❌ |
| 倾斜标签 | `transform: rotate(-3deg)`（仅装饰） | `transform: skew()` ❌ |
| 图层叠放 | 同位置叠 div，靠 DOM 顺序与 `z-index` | `mix-blend-mode` ❌ |
| 卡片与白底页面"看起来不同" | 加 `border: 1px solid #B4B5B5` + 一层 box-shadow，**保持白底** | 方案 A 下改卡片底色为彩色 ❌ |
