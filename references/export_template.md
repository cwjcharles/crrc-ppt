# 导出脚本规范 / export_template

本文件规定 HTML 文件头、Export 按钮与 dom-to-pptx 集成方式。完整可复制骨架见 [../assets/template.html](../assets/template.html)。

## 1. HTML 头部硬约束

- 必含：`<!DOCTYPE html>`、`<html>`、`<head>`、`<meta charset="UTF-8">`、`<title>`。
- **不引入任何外链 CSS / 字体 / 图标库**：
  - ❌ `<link href="https://fonts.googleapis.com/...">`
  - ❌ `<link href="https://cdn.jsdelivr.net/npm/font-awesome/...">`
  - ❌ `<link href="https://cdn.tailwindcss.com">`
- 字体栈直接写 `'Microsoft YaHei', 'Inter', Arial, sans-serif`；系统未装 Inter 时会优雅退化到 Arial。

## 2. 页面 chrome 风格

`<style>` 块**仅**用于页面背景、Export 按钮、reset 与 `.slide-stage` 居中容器。**不得**用于 `.slide` 内部样式。

```css
html, body {
  margin: 0;
  padding: 0;
  background: #FFFFFF;
  font-family: 'Microsoft YaHei', 'Inter', Arial, sans-serif;
}
.slide-stage {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px;
  padding: 40px 0;
  min-width: 1920px;  /* 避免 1920px 幻灯片被预览窗口水平裁切 */
}
```

## 3. dom-to-pptx 加载

**必须**用 CDN：

```html
<script src="https://cdn.jsdelivr.net/npm/dom-to-pptx@latest/dist/dom-to-pptx.bundle.js"></script>
```

不允许自托管、`@1.1.x` 锁定版本、或换其他 CDN。

## 4. Export 按钮

放在右上角（或右下角，但不得遮挡 `.slide`）：

```html
<button id="export-btn" style="
  position: fixed; top: 20px; right: 20px; z-index: 9999;
  padding: 12px 24px;
  background: #C70019; color: #FFFFFF;
  border: none; border-radius: 8px;
  font-size: 16px; font-weight: 700;
  cursor: pointer;
">Export PPTX</button>
```

按钮颜色用中车红 `#C70019`，与品牌色板对齐。

## 5. 导出 API

### 5.1 真实签名（这是唯一正确的写法）

```
exportToPptx(target, options)
```

- **target**（**第一个参数**，必填）：必须是以下三种之一
  - 单个 `HTMLElement`
  - `Array<HTMLElement>`
  - CSS 选择器字符串（如 `'.slide'`、`'#slide'`）
- **options**（第二个参数）：含 `fileName` / `autoEmbedFonts` / `svgAsVector` 等的普通对象

**唯一可靠的模板**（必须按此抄）：

```js
const slides = Array.from(document.querySelectorAll('.slide'));
if (!slides.length) throw new Error('No .slide elements found');
await domToPptx.exportToPptx(slides, {
  fileName: 'presentation.pptx',
  autoEmbedFonts: true,
  svgAsVector: true
});
```

允许先检查 `window.domToPptx` 是否就绪：

```js
if (!window.domToPptx || typeof domToPptx.exportToPptx !== 'function') {
  alert('导出失败：dom-to-pptx 库未加载成功，请检查网络或 CDN 是否可访问。');
  return;
}
```

### 5.2 致命陷阱：不要把 options 当第一个参数

下面这种写法**会导致运行时报错** `root.getBoundingClientRect is not a function`，因为库会把 options 对象当成 target 元素，然后在 options 上调用 `getBoundingClientRect()`：

```js
// ❌ 致命错误：把所有参数堆进一个对象传第一参，options 没了
await domToPptx.exportToPptx({
  slideSelector: '.slide',   // ← 这是不存在的字段
  fileName: 'x.pptx',
  autoEmbedFonts: true,
  svgAsVector: true
});
```

**根因**：`slideSelector` 字段不存在；库只接受 `(target, options)` 两参数形式。出错时浏览器控制台会看到 `TypeError: root.getBoundingClientRect is not a function`。

### 5.3 伪 API 黑名单

调用以下任意一个都是错误，**不要**写：

| 伪 API | 为什么是错的 |
|---|---|
| `new DomToPptx()` | 不存在这个构造函数 |
| `domToPptx({...})` | 没有这种顶层调用 |
| `domToPptx.exportToPptx({slideSelector: ...})` | `slideSelector` 字段不存在，options 错位为 target |
| `domToPptx.exportToPptx({slides: [...]})` | `slides` 字段不存在 |
| `domToPptx.exportToPptx({root: ...})` | `root` 字段不存在 |
| `domToPptx.exportToPptx({target: ...})` | `target` 字段不存在 |
| `pptx.fromContainer()` | 不存在 |
| `file.save()` | 不存在 |
| `html2pptx()` | 不存在 |
| `PptxGenJS.exportSlides()` | 不是这个库 |

**自检方法**：grep 自己生成的代码，确认 `exportToPptx(` **后面紧跟的是变量名（如 `slides`）、`Array.from(...)`、`document.querySelectorAll(...)` 或字符串字面量（如 `'.slide'`），而不是 `{`**。如果紧跟 `{`，就是把 options 错放成第一个参数了。

## 6. 导出 options 三件套

`options` **必须显式包含** 这三项：

```js
{
  fileName: 'presentation.pptx',
  autoEmbedFonts: true,
  svgAsVector: true
}
```

`svgAsVector` 默认 `true`（保留矢量）；**仅当用户明确要求"稳定优先"或"减少兼容风险"时**才改为 `false`。

## 7. 完整 export 调用结构

**不要在此处重复代码骨架**——以 [../assets/template.html](../assets/template.html) 内嵌的 export 脚本为**唯一权威**，直接复制即可。

该骨架包含的必备要素（如有改动，按此清单核对）：

- 库就绪检查：`if (!window.domToPptx || typeof domToPptx.exportToPptx !== 'function')` → alert 并 return
- validator 拦截：`window.validateSlides()` 返回 issues，`console.table` + `confirm` 让用户决定是否继续
- 按钮禁用 + 文案切换：`disabled = true`，`textContent = 'Exporting...'`
- **完整 `try / catch / finally`**（三段都要）
- `try` 中：`Array.from(document.querySelectorAll('.slide'))` → 空数组抛错 → `await domToPptx.exportToPptx(slides, {...})`
- `catch` 中：`console.error(err)` + `alert(err.message)`，并附带"如果错误是 `root.getBoundingClientRect is not a function`，说明 exportToPptx 第一个参数错传了 options 对象"的诊断提示
- `finally` 中：`setTimeout` 2 秒后恢复按钮状态
- options 显式含 `fileName` / `autoEmbedFonts: true` / `svgAsVector: true`

## 8. validator 必备

**必须**定义 `window.validateSlides()`，放在 export 脚本之前。完整可粘贴的脚本与覆盖范围见 [self_check.md](self_check.md)；template.html 内已内嵌。
