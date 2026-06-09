# 输出前自检 / self_check

输出最终 HTML 前，**必须内嵌并依赖下方 `window.validateSlides()` 脚本**完成自检；若 validator 报错，**先修正 HTML**，再输出最终版本。

本 validator 是**唯一权威**的自检手段——它逐元素扫描禁用项、字号下限、尺寸、定位、外链 CSS、预览裁切等所有硬约束。所有 markdown 形态的"自检清单"都已下沉到此脚本，不再单独维护，避免漂移。

各项硬约束的**语义解释**见对应文档：
- 容器/定位/单位/禁用项 → [pptx_compat.md](pptx_compat.md)
- 色彩/字号/Bento 间距 → [design_spec.md](design_spec.md)
- Export 按钮 / API 调用形式 → [export_template.md](export_template.md)

## 可粘贴的 `window.validateSlides()` 脚本

放在 export 脚本**之前**：

```html
<script>
(function () {
  const SLIDE_SELECTOR = '.slide';
  const EXPECTED_BG = 'rgb(255, 255, 255)';  // #FFFFFF

  const BAD_TRANSFORM =
    /\b(?:translate|translateX|translateY|translate3d|scale|scaleX|scaleY|scale3d|skew|skewX|skewY|matrix|matrix3d)\s*\(/;
  const BAD_BACKGROUND = /\b(?:radial-gradient|conic-gradient)\s*\(/;
  const VIEWPORT_UNITS = /\b\d*\.?\d+(?:vh|vw|vmin|vmax)\b/;
  const FUNC_LIMITS = /\b(?:clamp|min|max)\s*\(/;
  const ICON_FONT_CLASS = /\b(?:fa-[a-z0-9-]+|material-icons|material-symbols-[a-z]+)\b/;

  const STYLE_ATTR_BLOCKERS = [
    { name: 'backdrop-filter', re: /backdrop-filter\s*:/i },
    { name: 'clip-path', re: /clip-path\s*:/i },
    { name: 'mask-image', re: /mask-image\s*:/i },
    { name: 'mix-blend-mode', re: /mix-blend-mode\s*:/i },
    { name: 'background-blend-mode', re: /background-blend-mode\s*:/i },
    { name: 'text-shadow', re: /text-shadow\s*:/i },
    { name: 'animation', re: /(?:^|[;\s])animation(?:-\w+)?\s*:/i },
    { name: 'transition', re: /(?:^|[;\s])transition(?:-\w+)?\s*:/i },
    { name: 'filter (blur or other)', re: /(?:^|[;\s])filter\s*:/i },
    { name: 'inset box-shadow', re: /box-shadow\s*:[^;]*\binset\b/i },
    { name: 'outline', re: /(?:^|[;\s])outline\s*:/i },
    { name: 'lazy image', re: /loading\s*=\s*["']lazy["']/i },
  ];

  function describe(el) {
    const id = el.id ? '#' + el.id : '';
    const cls =
      el.className && typeof el.className === 'string'
        ? '.' + el.className.trim().split(/\s+/).slice(0, 2).join('.')
        : '';
    return el.tagName.toLowerCase() + id + cls;
  }
  function locate(el, idx) {
    return 'slide ' + (idx + 1) + ' › ' + describe(el);
  }

  function hasTransformedAncestor(slide) {
    let p = slide.parentElement;
    while (p && p !== document.body) {
      const t = getComputedStyle(p).transform;
      if (t && t !== 'none') return p;
      p = p.parentElement;
    }
    return null;
  }

  window.validateSlides = function validateSlides() {
    const slides = Array.from(document.querySelectorAll(SLIDE_SELECTOR));
    const issues = [];

    if (slides.length === 0) {
      issues.push('No elements match ".slide". Add class="slide" to slide roots.');
      return issues;
    }

    // 检查外链 CSS / Google Fonts
    document
      .querySelectorAll('link[rel="stylesheet"][href^="http"]')
      .forEach((link) => {
        issues.push(
          'External stylesheet not allowed: ' +
            link.href +
            ' (crrc-ppt forbids external CSS / Google Fonts)'
        );
      });

    // 检查预览裁切风险
    const stage = document.querySelector('.slide-stage') || document.body;
    const stageW = stage.getBoundingClientRect().width;
    if (stageW < 1920) {
      issues.push(
        '.slide-stage width is ' +
          stageW.toFixed(0) +
          'px < 1920px; preview may horizontally clip slides. Add min-width: 1920px.'
      );
    }

    slides.forEach((slide, idx) => {
      const cs = getComputedStyle(slide);
      const w = parseFloat(cs.width);
      const h = parseFloat(cs.height);

      if (Math.round(w) !== 1920 || Math.round(h) !== 1080) {
        issues.push(
          locate(slide, idx) +
            ' — slide size is ' +
            Math.round(w) +
            '×' +
            Math.round(h) +
            ', expected 1920×1080'
        );
      }
      if (cs.position !== 'relative') {
        issues.push(locate(slide, idx) + ' — slide position is "' + cs.position + '", expected "relative"');
      }
      if (cs.overflow !== 'hidden') {
        issues.push(locate(slide, idx) + ' — slide overflow is "' + cs.overflow + '", expected "hidden"');
      }
      if (cs.backgroundColor !== EXPECTED_BG) {
        issues.push(
          locate(slide, idx) +
            ' — slide background-color is ' +
            cs.backgroundColor +
            ', expected ' +
            EXPECTED_BG +
            ' (#FFFFFF)'
        );
      }
      const ancestor = hasTransformedAncestor(slide);
      if (ancestor) {
        issues.push(
          locate(slide, idx) +
            ' — has a transformed ancestor (' +
            describe(ancestor) +
            '); move slide out of it'
        );
      }

      slide.querySelectorAll('*').forEach((el) => {
        const s = getComputedStyle(el);
        const inline = el.getAttribute('style') || '';
        const klass = (el.getAttribute('class') || '');

        // transform check (only rotate allowed)
        if (inline) {
          const inlineXform = (inline.match(/transform\s*:\s*([^;]+)/i) || [])[1] || '';
          if (BAD_TRANSFORM.test(inlineXform)) {
            issues.push(locate(el, idx) + ' — transform: ' + inlineXform.trim() + ' (only rotate is allowed)');
          }
        }

        // background gradient
        const bg = s.backgroundImage || '';
        if (BAD_BACKGROUND.test(bg)) {
          const type = bg.includes('radial') ? 'radial-gradient' : 'conic-gradient';
          issues.push(locate(el, idx) + ' — uses ' + type + ' (use linear-gradient or stacked divs)');
        }

        // D8: flex/grid 直接子项必须 min-height:0（防纵向 blow-out）
        const parentEl = el.parentElement;
        if (parentEl) {
          const pd = getComputedStyle(parentEl).display;
          if (pd === 'flex' || pd === 'grid' || pd === 'inline-flex' || pd === 'inline-grid') {
            if (s.minHeight !== '0px') {
              issues.push(locate(el, idx) + ' — flex/grid child without min-height:0 (may blow out vertically; add min-height:0; overflow:hidden)');
            }
          }
        }

        // inline blockers
        if (inline) {
          STYLE_ATTR_BLOCKERS.forEach((rule) => {
            if (rule.re.test(inline)) {
              issues.push(locate(el, idx) + ' — uses ' + rule.name + ' (blocked)');
            }
          });
          if (VIEWPORT_UNITS.test(inline)) {
            issues.push(locate(el, idx) + ' — uses viewport units (vh/vw/vmin/vmax); use px');
          }
          if (FUNC_LIMITS.test(inline)) {
            issues.push(locate(el, idx) + ' — uses clamp()/min()/max() (blocked; use fixed px)');
          }
        }

        // icon font class
        if (klass && ICON_FONT_CLASS.test(klass)) {
          issues.push(locate(el, idx) + ' — icon font class "' + klass + '" detected; use inline <svg> instead');
        }

        // image checks
        if (el.tagName === 'IMG') {
          const src = el.getAttribute('src') || '';
          if (!src) {
            issues.push(locate(el, idx) + ' — <img> empty src');
          } else if (!/^(https:\/\/|data:)/i.test(src)) {
            issues.push(locate(el, idx) + ' — <img src="' + src + '"> must be https:// or data:');
          } else if (el.complete && el.naturalWidth === 0) {
            issues.push(locate(el, idx) + ' — <img> failed to load: ' + src);
          }
          if (el.getAttribute('loading') === 'lazy') {
            issues.push(locate(el, idx) + ' — <img loading="lazy"> can race export; remove it');
          }
          if (el.hasAttribute('srcset')) {
            issues.push(locate(el, idx) + ' — <img srcset> not allowed; pin to single URL');
          }
        }

        if (['VIDEO', 'AUDIO', 'IFRAME', 'CANVAS', 'PICTURE', 'SOURCE'].includes(el.tagName)) {
          issues.push(
            locate(el, idx) + ' — <' + el.tagName.toLowerCase() + '> not captured; bake to <img> or SVG'
          );
        }

        // SVG inner <image>
        if (el.tagName.toLowerCase() === 'image' && el.namespaceURI === 'http://www.w3.org/2000/svg') {
          issues.push(locate(el, idx) + ' — SVG <image href> not allowed (no external resources in SVG)');
        }

        // stroke-dasharray on data path/circle
        const da = el.getAttribute && el.getAttribute('stroke-dasharray');
        if (da && (el.tagName.toLowerCase() === 'circle' || el.tagName.toLowerCase() === 'path')) {
          issues.push(
            locate(el, idx) +
              ' — stroke-dasharray on <' +
              el.tagName.toLowerCase() +
              '>; confirm NOT used to fake pie/donut/progress (use real arc geometry)'
          );
        }

        // 字号下限检查（硬约束）：只检查"承载文字内容"的元素
        // 跳过没有直接文字节点的容器，避免误报
        const hasDirectText = Array.from(el.childNodes).some(
          (n) => n.nodeType === 3 && n.textContent.trim().length > 0
        );
        if (hasDirectText) {
          const fontPx = parseFloat(s.fontSize);
          // 硬下限 18px（注释下限）；正文下限 24
          if (fontPx && fontPx < 18) {
            issues.push(
              locate(el, idx) +
                ' — font-size ' +
                fontPx +
                'px below 18px hard floor (annotation min 18, body min 24, card title min 30, page title min 56)'
            );
          } else if (fontPx && fontPx >= 18 && fontPx < 24) {
            // 18-23 区间：可能是注释（18-22 合规）或正文降字号违规
            // 用文本长度做粗判：> 15 字大概率是正文/要点
            const txt = el.textContent.trim();
            if (txt.length > 15) {
              issues.push(
                locate(el, idx) +
                  ' — font-size ' +
                  fontPx +
                  'px with ' +
                  txt.length +
                  ' chars looks like body text below 24px floor (consider split page / merge points / widen container instead of shrinking)'
              );
            }
          }
        }
      });
    });

    return issues;
  };
})();
</script>
```

将此脚本放在 Export 按钮脚本**之前**；在 export 点击处理函数中按 [export_template.md](export_template.md) 调用。完整可运行结构见 [../assets/template.html](../assets/template.html)。
