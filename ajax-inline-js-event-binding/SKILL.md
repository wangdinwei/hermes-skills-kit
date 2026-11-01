---
name: ajax-inline-js-event-binding
title: AJAX-Injected Inline JS Event Binding
description: 当 Flask/Jinja2 模板通过 AJAX `innerHTML` 注入时，`onclick` 和 `data-action` 按钮容易失效。根因分析与可靠绑定方案。含 JSON 脚本克隆踩坑记录。
---

# AJAX 注入模板的事件绑定

## 问题场景

Flask 项目中，主页通过 AJAX 加载面板内容：

```js
fetch('/api/select-episode', { method: 'POST', body: formData })
  .then(r => r.text())
  .then(html => {
    document.getElementById('action-body').innerHTML = html;
    // 重新执行 inline script — ⚠️ 此处有坑！
    document.querySelectorAll('#action-body script').forEach(old => {
      const ns = document.createElement('script');
      ns.textContent = old.textContent;
      old.parentNode.replaceChild(ns, old);
    });
  });
```

## ⚠️ 重大踩坑：JSON data <script> 会杀死脚本链

**症状**：按钮完全无响应，但元素存在于 DOM。`console` 无 JS 错误（因为是 JSON 解析错误而非 throw，不会显示）。

**根因**：上述克隆代码遍历**所有** `<script>` 元素，包括 `<script id="step-data" type="application/json">`。JSON 数组（`[["L01", "剧本"]]`）作为 JS 执行不报错，但 JSON 对象（`{"L01_剧本": {"status": "done"}}`）在 JS 解析时 `{` 被当作块语句，`"L01_剧本"` 不是有效 label，导致 **SyntaxError**。

**关键**：浏览器中一个 `<script>` 的 SyntaxError **不会阻止后续 `<script>` 执行**，但 JSON data 脚本通常排在主 IIFE 脚本**前面**，且 SyntaxError 发生时会中断**当前**解析器上下文——当克隆代码串行 `forEach` 每个脚本时，前一个 JSON 脚本的 SyntaxError 会导致当前 `replaceChild` 返回异常，`forEach` 链中断，主 IIFE 永远不会执行。

```js
// ❌ 错误：遍历所有 script，JSON 数据杀死了整个链
body.querySelectorAll('script').forEach(old => {
  const ns = document.createElement('script');
  ns.textContent = old.textContent;
  old.parentNode.replaceChild(ns, old);
});
// 主 IIFE 里的 window.fn = ... 和 element.onclick = ... 全部没执行
```

### 修复：跳过 type="application/json"

```js
// ✅ 正确：跳过非 JS 脚本
body.querySelectorAll('script').forEach(old => {
  if (old.type === 'application/json' || old.getAttribute('type') === 'application/json') return;
  const ns = document.createElement('script');
  ns.textContent = old.textContent;
  old.parentNode.replaceChild(ns, old);
});
```

> 注意：`old.type` 属性是只读的规范属性（小写）。用 `getAttribute('type')` 兜底兼容某些服务端渲染的写法。

---

## 三种绑定方式 & 失效原因

### ❌ `onclick="fn()"`（HTML 属性）
- 浏览器解析时 `fn` 从 `window` 查找
- 内联 `<script>` 的 IIFE 确实会执行 `window.fn = function(){}`
- **但** 脚本克隆执行时机与 HTML 解析顺序有微妙差异，函数可能还未挂载就被点击
- ⚠️ `innerHTML` 注入后 `onclick` 立即求值，但 `<script>` 还没执行完

### ❌ `data-action="kebab-case"`（事件委托）
- 依赖 `static/js/app.js` 的全局 click 委托：kebab-case → camelCase → `window[fnName]()`
- 问题1：`app.js` 被 esbuild 打包为 IIFE（`dist/shared.*.js`），函数未挂 `window`
- 问题2：switch 中有显式 case，新 action 可能未注册
- 问题3：委托执行时 `window.fn` 可能还未被 AJAX 脚本定义

### ✅ `element.onclick = handler`（IIFE 内直绑）
- 在 IIFE 内部 DOM 渲染完成后直接绑定
- 不依赖 `window` 全局函数
- 不依赖外部 `.js` 文件
- 100% 可靠

### ✅ 文档级事件委托（最可靠）
- 使用 `document.addEventListener('click', ...)` + `e.target.closest(...)`
- 只需注册一次（用 `window._flag = true` 防重复）
- 无论 DOM 如何重新注入，事件始终有效
- 适合弹窗等 AJAX 频繁刷新的场景

```js
if (!window._myDelegated) {
  window._myDelegated = true;
  document.addEventListener('click', function(e) {
    var btn = e.target.closest('#change-btn');
    if (btn) {
      // handle click
      return;
    }
    var modal = document.getElementById('my-modal');
    if (!modal || modal.style.display === 'none') return;
    if (e.target.id === 'modal-backdrop' || e.target.id === 'modal-close') {
      modal.style.display = 'none';
    }
    var card = e.target.closest('.card');
    if (card && card.dataset.mode) {
      handleChange(card.dataset.mode);
    }
  });
}
```

## 正确方案总览

### HTML — 只用 ID/data 属性，不用 onclick

```html
<button id="change-btn">✏️ 更换</button>

<div id="my-modal" style="display:none">
  <div id="modal-backdrop"></div>
  <button id="modal-close">×</button>
  <div class="card" data-mode="narration">卡片A</div>
</div>
```

### JS — 文档级事件委托（推荐）+ IIFE 内直绑（备选）

```js
(function() {
  // 文档级委托（一次注册永久有效）
  if (!window._changeHandlers) {
    window._changeHandlers = true;
    document.addEventListener('click', function(e) {
      // ... handler logic
    });
  }
  // IIFE 内直绑（每次刷新重新绑定）
  var btn = document.getElementById('change-btn');
  if (btn) btn.onclick = function() { ... };
})();
```

## 核心原则

| 方法 | 可靠性 | 原因 |
|------|--------|------|
| `onclick="fn()"` | ❌ 低 | 依赖 window 函数挂载时机 |
| `data-action` | ❌ 中 | 依赖外部 JS 委托 + window 函数 |
| `element.onclick`（IIFE内） | ✅ 高 | IIFE 内直绑，不依赖外部作用域 |
| 文档级事件委托 | ✅✅ 最高 | 一次注册，始终有效，不受 innerHTML 影响 |

## 双保险绑定模式（推荐）

对于 AJAX 注入页面，推荐同时使用两种绑定：

```js
(function() {
  // 保险1：文档级委托（一次注册，永久有效）
  if (!window._myDelegated) {
    window._myDelegated = true;
    document.addEventListener('click', function(e) {
      var btn = e.target.closest('#action-btn');
      if (btn) { handleClick(); return; }
      // ... 弹窗关闭、卡片选择等逻辑
    });
  }
  
  // 保险2：元素直绑（兼容移动端 touch 场景）
  // 用 _bound 标志避免脚本重执行时重复绑定
  var btn = document.getElementById('action-btn');
  if (btn && !btn._bound) {
    btn._bound = true;
    btn.onclick = function() { handleClick(); };
  }
  var backdrop = document.getElementById('modal-backdrop');
  if (backdrop && !backdrop._bound) {
    backdrop._bound = true;
    backdrop.onclick = function() { closeModal(); };
  }
})();
```

> 为什么需要双保险？移动端 Safari 上，某些 touch→click 事件链可能不会触发 document 级委托，但会触发元素级 onclick。`_bound` 标志防止每次脚本重执行时重复绑定。

## 调试方法论：判断脚本是否执行

当 AJAX 注入后按钮无响应时，三步排查：

```js
// 1. 检查元素是否存在
document.getElementById('my-btn') !== null

// 2. 检查事件委托是否注册（检查守卫标志）
window._changeModeDelegated // true=已注册, undefined=脚本未执行

// 3. 检查 window 函数是否挂载（如果用了 window 函数模式）
typeof window.myFunction === 'function'
```

如果 `_flag` 是 `undefined`，说明 IIFE 内的脚本从未执行——先检查 JSON 数据脚本拦截问题。

## 验证时间线

- 2026-06-08：发现 JSON data `<script>` 导致脚本链断裂的根因
- 修复：`index.html` 的 `loadActions()` + `action_panel.html` 中 3 处克隆代码全部加 JSON 过滤
- 引入双保险绑定模式（文档级委托 + 元素直绑），兼容移动端 touch 场景
- 浏览器实测全链路通过：按钮点击 → 弹窗 → 遮罩关闭 → 卡片选择
