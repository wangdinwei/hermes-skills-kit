---
name: bootstrap-display-important-override-inline
description: 当 JS 设置 element.style.display='none' 但元素仍然可见时，排查 Bootstrap/框架的 !important utility 类（如 .d-flex）覆盖了行内样式。
---

# Bootstrap `!important` 样式覆盖 JS 行内 display 的排查和修复

## 问题现象
- JS 代码执行了 `el.style.display = 'none'`（或 `hide()`）
- 检查 DOM 能确认行内样式已设为 `display: none`
- **但元素在页面上仍然可见**

## 根因
Bootstrap 的 `.d-flex` 实用类定义如下：
```css
.d-flex { display: flex !important; }
```

`!important` 声明优先级高于行内样式（无论行内样式有没有 `!important`）。所以：
- 虽然 `el.style.display = 'none'` 设置了行内样式
- 但 `.d-flex { display: flex !important }` 胜出
- computed style 仍然是 `flex`

## 诊断方法

### 1. 检查 computed vs inline 不一致
在浏览器控制台执行：
```javascript
var el = document.getElementById('storyboard-loading');
console.log('inline:', el.style.display);
console.log('computed:', window.getComputedStyle(el).display);
console.log('has d-flex:', el.classList.contains('d-flex'));
```

**典型输出 (有 bug)**：
```
inline: none
computed: flex
has d-flex: true
```

**正常输出 (修复后)**：
```
inline: none
computed: none
has d-flex: false
```

### 2. 确认是哪个 CSS 规则覆盖的
```javascript
var el = document.getElementById('storyboard-loading');
for (var s = 0; s < document.styleSheets.length; s++) {
    try {
        var rules = document.styleSheets[s].cssRules;
        for (var r = 0; r < rules.length; r++) {
            if (rules[r].selectorText && el.matches(rules[r].selectorText) 
                && rules[r].style.display === 'flex') {
                console.log('OVERRIDE:', rules[r].cssText);
            }
        }
    } catch(e) {}
}
```

## 修复方法

不在行内样式上纠缠，直接操控 CSS 类：

### 显示时：添加 `d-flex` 类
```javascript
function showLoadingSpinner() {
    const el = document.getElementById('storyboard-loading');
    if (el) {
        el.classList.add('d-flex');
        el.style.display = 'flex';  // 双保险
    }
}
```

### 隐藏时：移除 `d-flex` 类（关键！）
```javascript
function hideLoadingSpinner() {
    const el = document.getElementById('storyboard-loading');
    if (el) {
        el.classList.remove('d-flex');  // 移除 !important 覆盖源
        el.style.display = 'none';
    }
}
```

### 模板默认状态：不要硬编码 `d-flex`
模板中的 HTML 应让元素默认隐藏：
```html
<!-- ❌ 错误：d-flex 的 !important 覆盖了 style="display:none" -->
<div id="storyboard-loading" class="d-flex ..." style="display:none">

<!-- ✅ 正确：由 JS 控制 d-flex 的添加和移除 -->
<div id="storyboard-loading" class="align-items-center gap-2 ..." style="display:none">
```

## 受影响的其他代码
搜索所有对此元素做 `style.display` 操作的地方，一并修复：
- `pageshow` bfcache 恢复处理
- 模板内联的 fallback 兜底脚本
- 外部 JS bundle 中其他隐藏该元素的函数

统统改成 `classList.remove('d-flex') + style.display = 'none'` 双保险。

## 受影响的其他元素
如果页面还有其他通过 `d-flex + style.display` 控制显隐的 UI（如 loading 浮层、弹窗），看是否也需要同样的修复。举一反三。

## Verification
```javascript
var el = document.getElementById('storyboard-loading');
// 验证 fix 后
console.log({
    display: window.getComputedStyle(el).display,  // 应为 'none'
    classListContains_dflex: el.classList.contains('d-flex')  // 应为 false
});
```

## 为什么行内 `!important` 不行？
CSS 中行内样式加 `!important`：
```javascript
el.style.setProperty('display', 'none', 'important');
```
虽然在行内可以生效，但在 Bootstrap 等框架中，`!important` 战争不应继续。更好的方案是操控 CSS 类——移除 `!important` 的源头。
