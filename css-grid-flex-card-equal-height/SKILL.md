---
name: css-grid-flex-card-equal-height
description: 修 CSS Grid 中卡片布局不等高问题——当 grid 子项是 flex 容器（如 Tabler 的 .card）时，min-height 会阻止 grid stretch，导致卡片不等高。系统性的调试和修复方法。
category: software-development
---

# CSS Grid + Flex 卡片不等高修复

## 触发条件

用户反馈多列卡片布局中某张卡片比其他卡片高/低，且反复调整 `min-height`/`padding` 无效。

## 典型症状

- Grid 容器内 4 张卡片中，第 1 张比其余 3 张高 14px（或类似固定差值）
- 给所有卡片设了相同 `min-height` 仍不等高
- 桌面浏览器 DevTools 显示某些卡片 `offsetHeight < grid row height`

## 根因

**CSS Grid `align-items:stretch` 与 flex 容器 `min-height` 的冲突：**

1. 框架（如 **Tabler**）的 `.card` 默认带 `display:flex;flex-direction:column`
2. 项目代码又重复设 inline `display:flex;flex-direction:column`（冗余但无害）
3. 关键：设了 `min-height:140px`（或类似的固定值）
4. Grid 行高被内容最多的卡片撑到 `max(140, 184) = 184px`
5. 其余卡片被自己的 `min-height:140px` 卡住，**无法拉伸**填满 184px 的 grid 行
6. 结果：内容多的卡片 = 内容高度 + min-height 中的较大值，内容少的填不满

## 正确修复方案

### 方案：`height:100%` 替代 `min-height`

```html
<!-- ❌ 错误：min-height 阻止 grid stretch -->
<div class="card" style="padding:24px;min-height:140px;display:flex;flex-direction:column">

<!-- ✅ 正确：用 height:100% 填满 grid cell -->
<div class="card" style="padding:24px;height:100%">
```

### 注意点

1. **不要重复写 `display:flex;flex-direction:column`** — 如果 Tabler 或 Bootstrap 等框架的 `.card` 已经自带，inline 再写多余
2. **保留 `flex:1` 撑杆** — 卡片内部的 `<div style="flex:1"></div>` 保证内容底部对齐
3. **撑杆加 `min-height` 不影响** — `flex:1;min-height:40px` 在卡片内 flex 布局中是安全的，不会阻止外层 grid stretch
4. **`<a>` 标签不能当 grid 子项** — inline 元素作为 grid 直接子项会破坏网格尺寸计算。用 `div` + `onclick` 替代

## 调试步骤

### 第1步：实测卡片渲染高度

用浏览器 console 测量 grid 容器中所有卡片的高度：

```javascript
// 选择 grid 容器（通过子项 class 或 sibling 定位）
let gc = document.querySelector('[style*="grid-template-columns"]');
// 如果无法用属性选择器，利用页面结构定位
let gc = document.querySelector('h2').nextElementSibling.nextElementSibling;
// 输出每张卡片的实际渲染高度
[...gc.children].map((c,i) => `card${i}: ${c.offsetHeight}px`);
```

### 第2步：检查 Grid 布局设置

```javascript
getComputedStyle(gc).display;              // 应是 "grid"
getComputedStyle(gc).alignItems;           // 应是 "normal"(=stretch)
```

### 第3步：检查 `.card` 框架默认样式

```javascript
// 检查框架是否给 .card 带了默认 display:flex
getComputedStyle(gc.children[0]).display;  // 如果框架自带 flex，就不需要 inline 重复设
```

### 第4步：检查每张卡盒模型差异

```javascript
// 比较不同卡片的 border/padding/box-sizing
let c0 = gc.children[0], c1 = gc.children[1];
'c0: ' + getComputedStyle(c0).border + ' | ' + getComputedStyle(c0).padding;
'c1: ' + getComputedStyle(c1).border + ' | ' + getComputedStyle(c1).padding;
```

### 第5步：检查 align-self 是否被覆盖

```javascript
getComputedStyle(gc.children[1]).alignSelf; // 应为 "auto"
```

## 全站审计

如果遇到一张卡片不等高的问题，很有可能其他页面也有同样模式。用子助手批量审计：

```
delegate_task:
  goal: 全站审计所有 HTML 模板中的 grid + card 布局，找出设了 min-height 或 display:flex 的卡片
  context: 项目目录 /path/to/templates/
  toolsets: [terminal, file, search]
```

审计要点：
- 搜索 `class="card"` 是否与 `min-height` 同时出现
- 搜索 grid 容器（`grid-template-columns`）内的 `.card` 元素
- 检查 JS 动态渲染的卡片模板字符串中是否有 `flex flex-col` 但无 `flex-1` 撑杆
- 检查是否用 `<a>` 标签包裹 grid 子项

## 常见框架的 `.card` 默认样式

| 框架 | `.card` 默认 flex | 说明 |
|------|-------------------|------|
| **Tabler** | ✅ `display:flex;flex-direction:column` | 不需要 inline 重复 |
| **Bootstrap** | ✅ `display:flex;flex-direction:column` | 不需要 inline 重复 |
| **Tailwind** 无框架 | ❌ 无默认 flex | 需要自己加 `flex flex-col` |

## 项目级 CSS 覆盖框架 `.card` 的陷阱

项目中可能有一个独立的 `_shared.css` 或 `base.css` **重新定义了 `.card` 类**，与 Tabler/Bootstrap 框架的 `.card` 类**类名完全相同**，导致双重重叠：

```css
/* _shared.css — 项目自有样式，覆盖了框架的 .card */
.card {
    padding: 20px;          /* 与框架默认 padding 冲突 */
    background: #16161a;    /* 项目自定义背景 */
    overflow: hidden;       /* 可能裁剪内容 */
    border-radius: 8px;     /* 覆盖框架圆角 */
}
```

此外，项目 CSS 可能还有：
```css
.card + .card { margin-top: 14px; }  /* 非 grid 布局中相邻卡片间距 */
.card:hover { transform: translateY(-2px); }  /* hover 偏移可能被误认为"压住" */
```

这些项目级 CSS 与框架 `.card` 默认 flex 叠加，加上 inline 样式，形成**三重样式竞争**。

### 排查方法

```javascript
// 检查项目中是否有重定义 .card 的 CSS 文件
// 用 grep 搜索项目目录
grep -rn "\.card\s*{" static/css/ | grep -v tabler | grep -v bootstrap

// 或在浏览器中检查 .card 类实际的 CSS 规则来源
// 在 DevTools Elements 面板选中卡片，查看 Styles 面板底部的来源文件
```

## 踩坑记录

1. **从修复过程中反复卡住：** 每次改 margin/padding 都以为对了，但测量发现还是差 14px。根因根本不是 padding，而是 `min-height` 阻止了 grid stretch。修布局问题要先用 JS 量实际渲染尺寸再动手。
2. **`<a>` 包裹 grid 子项：** 行内元素不参与 grid 尺寸计算。之前多次改 padding/min-height 都无效，因为根因在 `<a>` 标签。
3. **Tabler `.card` 自带 flex：** 加了 inline `display:flex;flex-direction:column` 看似无害，但会与框架默认样式叠加产生预料之外的优先级冲突。检查框架默认样式后再决定是否需要重复写。
4. **grid `align-items:normal` 不同浏览器行为不同：** Chrome 中 `normal` = `stretch` 对非替换元素成立，但配合 `display:flex` 时行为不确定。明确设 `align-items:stretch` 可避免歧义。
5. **改了 CSS 但浏览器看到的是旧版：** 如果项目的静态文件有版本号（如 `?v={{ STATIC_VERSION }}`），改完 CSS 后需要**同步 bump 版本号**（如 `2.0.10 → 2.0.11`），否则浏览器强制用缓存版本，改了半天看不出效果。排查方法是：
   ```javascript
   // 检查浏览器加载的 CSS 版本
   [...document.querySelectorAll('link[rel=stylesheet]')]
     .find(l => l.href.includes('base.css'))?.href
   // 强制拉取最新版本验证
   fetch('/static/css/base.css?_='+Date.now())
     .then(r=>r.text())
     .then(t=>console.log('has old:', t.includes('旧值')))
   ```
