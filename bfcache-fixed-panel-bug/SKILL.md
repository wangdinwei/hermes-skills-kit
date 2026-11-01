---
name: bfcache-fixed-panel-bug
description: 诊断和修复浏览器 bfcache 导致 position:fixed 面板/弹窗随机出现的 bug。症状：刷新页面或前进后退时，浮动客服面板、弹窗等莫名出现，再刷新又消失。
category: debug
---

# 浏览器 bfcache 导致固定面板残留的 bug

## 症状

- 用户在步骤页打开了客服面板（或其他 fixed 定位弹窗）
- 导航回首页（或浏览器前进/后退），面板偶尔还浮在页面上
- 再刷新一次页面就消失了
- **间歇性**出现，难以稳定复现

## 根因

浏览器 bfcache（Back/Forward Cache）会完整缓存页面的 DOM 状态，包括：
1. JS 添加/移除的 CSS 类（如 `.open`）
2. JS 设置的 inline style（如 `body{position:fixed; overflow:hidden}`）
3. JS 闭包中的模块变量（如 `csOpen = true`）

当用户从步骤页回到首页时，浏览器可能从 bfcache 恢复首页，此时 JS 不重新执行（`script src` 不会重新加载），但 DOM 带着旧状态出现。

## 修复方案（三管齐下）

### 1. HTML 模板：添加 `pageshow` 事件重置 DOM

在两处模板的 `</body>` 前添加：

```html
<script>
window.addEventListener('pageshow', function(e) {
    // bfcache 恢复时强制重置面板
    var panel = document.getElementById('cs-panel');
    if (panel) panel.classList.remove('open');
    // 重置 body 样式（防止 bfcache 残留 fixed/hidden）
    document.body.style.overflow = '';
    document.body.style.position = '';
    document.body.style.width = '';
    document.body.style.top = '';
});
</script>
```

`pageshow` 事件在以下情况都会触发：
- 正常页面加载
- bfcache 恢复
- `event.persisted` 为 `true` 表示来自 bfcache（但我们不管来源，统一重置）

### 2. JS 文件：toggle 前同步 DOM 状态

```javascript
window.toggleCS = function() {
    var panel = document.getElementById('cs-panel');
    // 同步 DOM 状态（防止 bfcache 残留导致变量与 DOM 不一致）
    csOpen = panel.classList.contains('open');
    csOpen = !csOpen;
    panel.classList.toggle('open', csOpen);
    // ...
};
```

关键：在 `csOpen = !csOpen` 之前，先用 DOM 实际状态校正 `csOpen` 变量。否则 bfcache 残留的 `csOpen=true` 但 DOM 已被 pageshow 重置（`.open` 已移除），导致用户需要点击两次才能打开面板。

### 3. 所有使用 `position:fixed` + 类名切换的元素都受影响

不只是客服面板，以下都需要同样处理：
- 弹窗（modal overlay）
- 侧边栏（sidebar overlay）
- 任何通过 JS 添加/移除类来控制显示/隐藏的 fixed 元素

## 验证方法

1. 打开客服面板
2. 导航到另一个页面
3. 浏览器「后退」回首页
4. 面板不应出现
5. 再点客服按钮，应该一次点击就能打开（不需要点两次）

## 注意

- `pageshow` 不是 `DOMContentLoaded`，后者在 bfcache 恢复时**不触发**
- 不要用 `beforeunload` 清理（bfcache 不会触发 unload）
- `pagehide` 可以用于保存状态，但不适合清理 DOM（会阻止 bfcache）
