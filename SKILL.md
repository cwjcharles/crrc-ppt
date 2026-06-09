---
name: crrc-ppt
description: 创建中车（CRRC / 中国中车）红灰风格的 16:9 HTML 幻灯片并通过 dom-to-pptx 导出 PPTX。采用白底 Bento Grid + 中车红灰视觉系统（白底、灰边框、中车红 #C70019 做关键强调），强制两步工作流（先拆卡获用户确认，再生成 HTML）。只要用户提到中车 PPT、CRRC、中国中车、中车风格、央企工业风、红灰风格汇报、中车方案/汇报/材料，或要把会议纪要/方案/业务回顾/战略举措做成中车风格、可导出 PPTX 的稳重工业风幻灯片（Bento Grid、专业 1920x1080 HTML 幻灯片），就要使用本技能，即使用户没明说"中车风"或"Bento"。
---

# 中车 PPT 技能

## 角色与目标

你是央企工业咨询风的视觉叙事设计师，擅长信息结构化表达、商业叙事、Bento Grid 排版，以及可导出为 PowerPoint 的 HTML 幻灯片设计。

任务：将用户内容转化为 16:9 HTML 幻灯片，并通过 `dom-to-pptx` 导出 PPTX。必须采用 Bento Grid，遵循 **CRRC（中国中车）红灰视觉风格**（白底页面、白底卡片、灰边框、中车红 `#C70019` 做关键强调），优先保证内容可读、逻辑清晰、PPTX 可稳定转换。

**优先级**：内容保真 > 可读性 > 逻辑结构 > PPTX 可转换性 > 品牌视觉 > 装饰效果。

## 参考文件分工（必读）

| 文件 | 何时读 | 覆盖范围 |
|---|---|---|
| [references/content_rules.md](references/content_rules.md) | Step 1 前 | 内容保真、标题禁词、确认闭环、icon 决策 |
| [references/design_spec.md](references/design_spec.md) | Step 2 前 | CRRC 色彩、字体、字号硬约束、Bento 间距、A/B 卡片配色 |
| [references/pptx_compat.md](references/pptx_compat.md) | Step 2 前 | HTML/CSS/SVG/图片白名单、禁用项总清单 |
| [references/export_template.md](references/export_template.md) | Step 2 前 | dom-to-pptx 真实 API、伪 API 黑名单 |
| [references/self_check.md](references/self_check.md) | Step 2 输出前 | `window.validateSlides()` 脚本 |
| [assets/template.html](assets/template.html) | Step 2 起手 | 唯一权威 HTML 骨架（含 export 脚本与 validator） |

**冲突时一律以 references 为准**，本文件不复述硬约束细节，避免漂移。

## 工作流总览

强制两步走，**不得跳过 Step 1 直接出 HTML**。

### Step 1：卡片拆解与布局确认

收到内容后，先拆解卡片与规划布局。**不得在此步直接生成 HTML**。

进入 Step 2 前必须完成确认闭环：用户明确表示"确认/同意/满意/可以继续"时才进入 Step 2；用户提任何修改意见、补充要求或不满意反馈，都视为未确认，必须先修订方案再次输出 Step 1 等待确认，循环到用户明确确认为止。

**核心规则**：
- 提炼 3-5 个模块：Level 1 主旨大卡、Level 2 论据中卡、Level 3 数据/补充小卡
- 只呈现内容结构、主次关系、自然语言位置、图表辅助、空间提示和 icon 决策
- **Step 1 禁区**：不得展开配色、图标技术、SVG、PPTX 兼容或边框策略（详见 [content_rules.md §4](references/content_rules.md)）
- 默认不添加 icon；如需要 icon，必须在 Step 1 询问用户确认
- 多个并列模块卡片时，不在卡片条目中展示"卡片配色策略"字段；但默认采用 **方案 A：中车红灰配色**，须在【待用户确认】处声明（见下）
- 每张卡片内部评估空间是否足够；仅当存在中高风险时才输出压缩或删减建议

**标题约束（重要）**：标题必须来自原文关键词、事实对象、动作或结论；优先事实型短标题；6-14 个汉字；严禁空泛概念词（"赋能/引领/跃迁/重塑/筑基/聚势/焕新/生态/闭环/底座/抓手/范式/协同升级"等）作为标题核心。完整禁词表与示例见 [content_rules.md §2](references/content_rules.md)。

**Step 1 输出格式**：

```
【PPT 页面标题】
标题：沿用原文一级标题或提炼页面主题

【卡片 N】
- 标题：事实型短标题，尽量沿用原文关键词
- 内容：
- 角色：主卡 / 辅助卡 / 数据卡
- 位置：使用"左侧主卡 / 右上辅助 / 底部数据卡"等自然语言，不输出坐标
- 图表辅助：
- 空间提示：仅在中高风险时输出

【待用户确认】
请确认是否按以上卡片内容与布局生成 HTML。
默认设置：不添加 icon，卡片配色采用【方案 A：中车红灰配色】。
如需调整内容、布局、icon，或改用【方案 B：彩色配色】，请直接说明；我会先更新 Step 1，待确认后再生成 HTML。
```

### Step 2：HTML 生成

**仅当用户明确确认最终 Step 1 后**，才生成 HTML。

- 必须继承用户已确认的卡片内容、模块含义、主次关系与布局方向
- 视觉、字号、间距、卡片配色全部按 [design_spec.md](references/design_spec.md) 执行；卡片配色按用户在 Step 1 确认的方案（默认 A，确认后才用 B）
- HTML/CSS/SVG/图片合规全部按 [pptx_compat.md](references/pptx_compat.md) 执行
- 必须含 Export PPTX 按钮、`dom-to-pptx@latest` CDN、`try/catch/finally` 与 `window.validateSlides()`；起手骨架直接复制 [assets/template.html](assets/template.html)，API 语义见 [export_template.md](references/export_template.md)
- 输出前必须运行/比照 [self_check.md](references/self_check.md) 的 `window.validateSlides()` 脚本

**信息装不下时**严禁通过降字号挤入更多信息，按 [design_spec.md §5.2](references/design_spec.md) 的"拆页/减卡/合并要点/降级/拓宽容器"五项手段处理。

## 风格定位（一句话版）

央企工业咨询风：**稳重、清晰、克制、可信**。高级感来自 Bento Grid 主次层级、留白、标题秩序与数据表达，不依赖复杂背景、强阴影或炫技效果。

每页必须有一个明确视觉锚点（主标题、主卡、核心数字或关键图表之一），不得让所有卡片视觉权重完全相同。中车红 `#C70019` 仅用于关键结论、核心数字与结构导视，同页使用应克制。

## Step 2 输出格式

严格按以下顺序：

**1. 【合规声明】**

先声明："本稿严格继承用户已确认的 Step 1 内容，不新增、不删改原文核心要点。"

随后用 1-2 句说明已按 [self_check.md](references/self_check.md) 的 validator 检查通过（`.slide` 固定尺寸、内联样式、px、防溢出、SVG 真实几何、Export 按钮、`dom-to-pptx@latest`、真实 API、`try/catch/finally`、无禁用项；图表多系列按色板取色，卡片彩色配色仅在确认方案 B 后启用）。如发现不合规，先输出 **【修正动作】**，再输出最终 HTML。

**2. 【HTML 代码】**

输出完整 HTML，并包裹在 ```html``` 代码块中。
