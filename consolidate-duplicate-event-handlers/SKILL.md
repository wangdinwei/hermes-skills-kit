---
name: consolidate-duplicate-event-handlers
description: 系统性查找和合并 Web 项目中重复的 DOM 事件处理器（如密码 toggle、弹窗关闭等）。当同一 action 有多个实现（app.js、auth.js、base.html inline）且只有部分页面生效时，审计→去重→统一到 base template。
---

# 合并重复的事件处理器

## 触发条件

用户反馈某个 UI 控件（如密码小眼睛、关闭按钮等）「没反应」或「部分页面不生效」时，先怀疑**多套实现冲突**。

## 审计流程

### 1. 找所有实现
搜索 `data-action="xxx"` 和对应的 handler 函数名：

```bash
# 在 static/js/ 中找所有 handler 定义
grep -rn "toggle-pwd\|togglePwd\|toggle.*password" templates/ static/js/ --include="*.html" --include="*.js"

# 在模板中找所有使用点
grep -rn 'data-action="toggle-pwd"' templates/ --include="*.html"
```

### 2. 画冲突矩阵

| 位置 | 实现方式 | 生效范围 | 实际可用 |
|------|----------|----------|----------|
| `app.js` | 函数定义 + switch case | IIFE 闭包内 | ❌ 死代码 |
| `auth.js` | 独立事件委托 | 仅 auth 页 | ❌ 范围窄 |
| `base.html` | inline 事件委托 | 全站 | ❌ 函数未 export |

### 3. 判断根因

三种典型冲突模式：

- **模式A（双发对消）**：两个事件监听器先后触发，第一个做了 A→B，第二个做 B→A，用户感知为「没反应」
- **模式B（作用域断链）**：handler 在 IIFE 闭包内但 base.html 的 fallback 通过 `window[fnName]()` 调用——IIFE 不 export 到 window
- **模式C（脚本未加载）**：handler 在一个专用 script 里但非 auth 页面没加载它

### 4. 修复方案

**原则：事件处理器统一放在 `base.html` 的 inline `<script>` 中**，因为：
- 所有页面都加载 base.html（包括 onboarding/独立页面）
- inline script 不会被 esbuild IIFE 闭包拦截
- `window[fnName]()` fallback 只对显式 export 到 window 的函数有效

**三步改法：**

```javascript
// 1. 从 app.js 删掉死代码函数（如 togglePwd）
// 2. 从 auth.js/专用 JS 删掉对应事件分支
// 3. 加到 base.html 的全局事件委托中，explicit early return
if (action === 'toggle-pwd') {
  var _inp;
  if(el.dataset.target) {
    _inp = document.getElementById(el.dataset.target);
  } else {
    _inp = el.parentElement.querySelector('input[type=password],input[type=text]');
  }
  if(!_inp) return;
  if(_inp.type==='password'){_inp.type='text';el.textContent='🙈'}
  else{_inp.type='password';el.textContent='👁️'}
  return;
}
```

### 5. 验证清单

- [ ] 注册页两个密码输入框都能 toggle
- [ ] 登录页（有 data-target 和无 data-target 两种模式）
- [ ] 忘记密码页
- [ ] Onboarding 引导页
- [ ] 控制台零 JS 错误
- [ ] 点击「确认密码」的 toggle 不会误触「设置密码」的输入框
- [ ] 重建 JS（`node build.js`）+ 重启服务后验证

## 特别陷阱

1. **IIFE 闭包陷阱**：esbuild `format: 'iife'` 把所有函数闭在 IIFE 内，不赋值 `window.xxx` 则 `base.html` 的 `window['funcName']()` 发现函数不存在
2. **base.html 早退顺序**：新加的 handler 必须放在 `switch(action)` fallback 之前（用 `if + return`），否则会被通用 fallback 兜住
3. **data-target vs 自动查找**：有的模板用 `data-target="inputId"` 指定密码框，有的靠 `parentElement.querySelector` 找最近的——统一 handler 必须支持两种模式
4. **esbuild 重建**：改了 `static/js/` 下的文件后必须跑 `node build.js`，改了 `templates/` 下只需要重启服务
