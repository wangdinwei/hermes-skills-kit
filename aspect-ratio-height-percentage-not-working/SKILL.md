---
name: aspect-ratio-height-percentage-not-working
description: 当父容器用 aspect-ratio 撑高度时，子元素 height:100% 不解析，视频/图片不可见。必须用 position:absolute 让子元素填满。
---

# aspect-ratio 容器中 `height: 100%` 不解析

## 问题现象

父容器用 `aspect-ratio: 9/16` 撑高度，子元素（video/img）用 `width:100%;height:100%;object-fit:cover`。
父容器渲染出了正确比例，但子元素**高度为 0，画面不可见**。

视频文件实际已加载完毕（readyState=4，duration 正常），但就是看不见画面。

## 根因

CSS 百分比高度的解析规则：
- **子元素的 `height: 100%` 只有在父元素有"明确高度"时才有效**
- `aspect-ratio` 计算的高度本质上是 `auto`（隐式），不符合"明确高度"条件
- 结果：`height: 100%` 退化为 `height: auto` → 子元素以 intrinsic 高度渲染 → 在非 absolute 定位时缩为 0

## 修复方法

**给子元素加 `position: absolute` 定位填满父容器：**

```css
/* 父容器 */
.parent {
    position: relative;
    width: 100%;
    aspect-ratio: 9/16;
    overflow: hidden;
}

/* 子元素 — 必须 absolute 定位 */
.child {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
}
```

**原理：** Absolutely positioned 元素的百分比尺寸相对于父元素的 padding box 计算。父元素即使由 `aspect-ratio` 驱动高度，其 padding box 尺寸在渲染后是明确的，所以 `width:100%;height:100%` 能正确填满。

## 适用场景

- 视频播放器弹窗（video 在 aspect-ratio 容器内）
- 图片预览（img 在 aspect-ratio 容器内）
- Canvas 元素
- 任何需要子元素填满 aspect-ratio 父容器的场景

## 调试方法

当遇到"数据正确加载但画面不显示"的问题时：

1. 在浏览器控制台检查：
   ```javascript
   var v = document.getElementById('element-id');
   ({readyState: v.readyState, offsetWidth: v.offsetWidth, offsetHeight: v.offsetHeight, position: getComputedStyle(v).position})
   ```
2. 如果 `offsetHeight` 为 0（或远小于预期），而 `readyState` 正常 → 高度解析问题
3. 检查 `position` 是否非 absolute/fixed

## 踩坑记录

- 2026-06-25 L09 预览弹窗：改播放器 CSS 时去掉了 video 的 `position:absolute`，导致视频看不见。折腾了多轮（怀疑编码问题、弹窗大小、z-index、autoplay）才找到根因。
- 备份对比（原始工作版 vs 当前版）是找到这个差异的关键方法。
