---
name: ai-pipeline-rogue-data-defense
description: 多层防御模式 — 防止 AI 生成的脏数据（非角色名等）进入管线。包含 regex 过滤、白名单匹配、patch 工具反斜杠转义踩坑等。
---

# AI 管线脏数据防御 — 多层防御模式

## 适用场景
AI 管线中，数据从 AI 生成文本 → regex 提取 → 入库/落盘，regex 提取了不该提取的内容（角色 title、舞台指示、括号残片等），导致脏数据在后续环节被处理（建文件夹、生图、生视频）。

## 典型链路
```
AI 写剧本 → 剧本里有 "法官：xxx" "（第一幕：xxx" "（语音继续）：xxx"
         ↓
parse_script_metadata() 用正则 r'^([^：\n]{2,6})[：:]' 盲抓角色名
         ↓
正则截到 "法官" / "（第一幕"（strip→"第一幕"） / "（语音继续）"（strip→"语音继续）")
         ↓
script.json.roles 存了脏数据
         ↓
资产生成管线读 roles → 为脏名字建文件夹/生图
```

## 三层防御

### 第1层：提取时过滤（regex 层）

在所有的 `角色名：对白` 正则提取(fallback)处，加 2 个过滤：

```python
for m in _re.finditer(r'^([^：\n]{2,6})[：:]', content, _re.MULTILINE):
    n = m.group(1).strip().strip('（(')
    raw = m.group(1).strip()
    
    # ❌ 跳过以（开头的（舞台指示如"（第一幕"）
    if raw.startswith('（') or raw.startswith('('):
        continue
    
    # ❌ 跳过括号不匹配的残片（如"语音继续）"）
    if '）' in n and '（' not in n:
        continue
    
    if n and len(n) >= 2 and len(n) <= 6 and ...:
        names.add(n)
```

**同一模式通常散布在多个文件中**，必须全量 grep 确认：
```bash
grep -rn "r'^\(\[^：\\\\n\]" --include="*.py" app/ | grep -v ".pyc"
```

### 第2层：管线入口白名单过滤

在资产生成/入库的入口处，对比项目已知注册名单，只放行已知角色：

```python
# 读取 assets/characters.json 的项目已知角色
_known_chars_path = os.path.join(root_prod, "assets", "characters.json")
if os.path.exists(_known_chars_path):
    with open(_known_chars_path, "r") as _f:
        _kc = json.load(_f)
    _known_names = set()
    for _ch in _kc.get("characters", {}).values():
        _n = _ch.get("name", "") if isinstance(_ch, dict) else str(_ch)
        if _n:
            _known_names.add(_n)
    # 也加入 cast.json 中的临时角色
    if cast_characters:
        _known_names.update(cast_characters.keys())
    if _known_names:
        _extra = all_names - _known_names
        if _extra:
            logger.warning(f"[资产生成] 过滤掉非注册角色名: {sorted(_extra)}")
        all_names &= _known_names
```

### 第3层：前端的进度轮询超时报错

后台任务可能因重启等原因丢失，前端轮询必须检测：

```javascript
safeFetch('/api/progress/' + taskId).then(function(prog) {
    // ❌ 错误：只检查 !prog（后端返回 404 时 prog 是 {success:false} 对象，永远 truthy）
    if (!prog) { ... return; }
    
    // ✅ 正确：检查 prog.success
    if (!prog || !prog.success) {
        clearInterval(pollTimer);
        lh_showError(prog?.error || '任务已中断，请重试',
            {onRetry: function() { location.reload(); }});
        return;
    }
}).catch(function() {
    // 网络错误也要处理
    _pollErrCount++;
    if (_pollErrCount >= 4) {
        clearInterval(pollTimer);
        lh_showError('网络连接异常', {onRetry: function() { location.reload(); }});
    }
});
```

## 🚨 踩坑：patch 工具会双转义反斜杠

`patch(old, new)` 在替换包含反斜杠的代码时，会把 `\d` 转义为 `\\d`，`\n` 转义为 `\\n`。

### 现象
```python
# 修复前（被 patch 工具破坏了）
_re.match(r'^场景\\\\d*$', n)   # ← \\d 变成了 \\\\d，regex 匹配不到数字

# 修复后（正确的）
_re.match(r'^场景\\d*$', n)     # ← \d 才是数字类
```

### 正确的修复方法
**不要用 patch 工具**改包含反斜杠的 regex 行。改用 Python 字节替换：

```python
# 用 Python bytes 操作直接替换字节
with open('file.py', 'rb') as f:
    content = f.read()

# 替换 \\d (两个反斜杠 + d) 为 \d (一个反斜杠 + d)
# 在源文件中，正确的 \d 是 0x5c 0x64
old = bytes([0x5c, 0x5c, 0x64])  # \\d
new = bytes([0x5c, 0x64])         # \d
content = content.replace(old, new)

with open('file.py', 'wb') as f:
    f.write(content)
```

### 验证
```bash
# 查看源文件中的实际字节
python3 -c "
with open('file.py', 'rb') as f:
    for i, line in enumerate(f.read().split(b'\n')):
        if b'场景' in line:
            print(f'Line {i+1}: {line}')
            print(f'Hex: {line.hex()}')
"
# \d 的正确 hex：5c 64（一个反斜杠）
# \\d 的错误 hex：5c 5c 64（两个反斜杠）
```

## 排查清单

当发现管线中出现脏数据时：

1. grep 所有 regex 提取角色名的地方：
   ```bash
   grep -rn "r'^\(\[^：\\\\n\]" --include="*.py" . | grep -v ".pyc"
   grep -rn "场景\\\\d" --include="*.py" . | grep -v ".pyc"
   ```
2. 在每个 regex 处加 `（` 开头过滤和括号不匹配过滤
3. 在管线入口加白名单对比（assets/characters.json）
4. 前端轮询加 `prog.success` 检测
5. 检查 patch 工具是否破坏了反斜杠转义
