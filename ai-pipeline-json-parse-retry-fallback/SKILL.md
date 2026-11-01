---
name: ai-pipeline-json-parse-retry-fallback
title: AI 管线 JSON 解析失败的三级递进修复模式
description: 当 AI 管线步骤调用 LLM 后需要解析 JSON 结果时，加入三级递进机制：标准 prompt → 简化 prompt → 纯代码兜底。防止 AI 返回空/无效 JSON 导致管线静默失败。
trigger: 步骤页显示"操作成功"但无产出；AI JSON 解析报毒
---

## 适用场景

AI 管线步骤中，调用 LLM 返回结果后需要 `json.loads()` 解析，但：
- AI 返回了空字符串或纯文本（非 JSON）
- AI 返回了 markdown 代码块包裹的 JSON（需要剥离）
- AI 返回了不完整的 JSON（被截断）
- 当前代码在 JSON 解析失败后直接 `return` 不保存任何结果

**典型症状：** 前端显示"操作成功"，但后台步骤输出为空（0段/0字/0文件），日志显示 `JSONDecodeError: Expecting value: line 1 column 1 (char 0)`。

## 🚨 铁律：不能降级简化（2026-07-04 用户明确要求）

**第2级（简化版prompt）已被用户否决，禁止使用。** 原因：

1. **格式不同导致下游崩溃** — 简化版prompt返回纯数组 `[{...}]`，但下游代码用 `.get("segments", [])` 在数组上直接崩溃
2. **产出质量差** — 少了剧本上下文、评定规则，AI只会机械切段
3. **本质是蒙混** — 第一题不会做就把题改成"随便写点啥"

改为：
```
第1次 → 标准 prompt（带完整指令和示例）
第2次 → 标准 prompt 重试（同等品质，不降级）
第3次 → 纯代码兜底（按规则/正则/段落拆分，不用AI）
```

代码模板应改为：

```python
for attempt in range(3):
    if attempt < 2:
        raw = _try_parse_with_ai(data, context)  # 只有标准prompt，无简化版
    else:
        segments_result = _fallback_parse(data)
        break
```

同时，简化版prompt返回的`list`格式必须在入口处加防御，防止意外抵达下游：

```python
# 任何可能来自AI的narration_data都要加
if isinstance(narration_data, list):
    narration_data = {"segments": narration_data}
```

## 三级递进修复方案

### 核心思路

不直接放弃，而是三级尝试：

```
第1级 → 标准 prompt（带完整指令和示例）
第2级 → 简化 prompt（最小化指令，降低AI出错概率）
第3级 → 纯代码兜底（按规则/正则/段落拆分，不用AI）
```

### 代码模板

```python
def _try_parse_with_ai(full_text, context_text, retry_simple=False):
    """尝试 AI 解析，retry_simple=True 时用简化版 prompt"""
    if retry_simple:
        sp = "简化版 prompt（只保留核心指令，不写复杂约束）"
    else:
        sp = "标准版 prompt（带详细规则和示例）"
    return _call_ai(sp, f"输入内容：\n{full_text}\n\n上下文：\n{context_text}")

def _fallback_parse(text):
    """兜底：纯代码解析，不用 AI"""
    # 按换行/句号/分号等拆分
    lines = [l.strip() for l in text.split('\n') if l.strip()]
    if len(lines) < 2:
        import re
        lines = [s.strip() + '。' for s in re.split(r'(?<=[。！？])', text) if len(s.strip()) > 10]
    segments = []
    for i, line in enumerate(lines):
        if len(line) < 5:
            continue
        # 自动补充句尾标点
        if not line.endswith(('。', '！', '？', '；', '…', '"', '"')):
            line += '。'
        segments.append({
            "scene_context": f"段落 {i + 1}",
            "text": line,
            "tone": "平静",
            "estimated_sec": round(len(line) / chars_per_sec, 1),
        })
    return segments

# ─── 三级递进 ─────────
segments_result = []
for attempt in range(3):
    if attempt == 0:
        raw = _try_parse_with_ai(data, context, retry_simple=False)
    elif attempt == 1:
        raw = _try_parse_with_ai(data, context, retry_simple=True)
    else:
        # 兜底：纯代码处理
        segments_result = _fallback_parse(data)
        logger.info(f"AI 解析失败3次，使用兜底: {len(segments_result)}条")
        break

    if not raw:
        continue

    try:
        parsed = _parse_narration_json(raw)  # 需要能处理数组和对象
        segments_result = parsed.get("segments", parsed if isinstance(parsed, list) else [])
    except json.JSONDecodeError as e:
        logger.warning(f"第{attempt+1}次JSON解析失败: {e}")
        continue

    if segments_result:
        logger.info(f"第{attempt+1}次成功: {len(segments_result)}条")
        break

if not segments_result:
    # 全部失败：写错误状态，提供重试入口
    _write_step_progress(l03_dir, "error", round_num, "解析失败：多次尝试均未生成有效结果")
    return
```

### 关键细节

1. **变量初始化前置**：`segments_result = []` 在循环前声明，避免 `NameError`
2. **JSON 解析要兼容两种格式**：`{"segments": [...]}` 和 `[...]` 数组
3. **兜底路径输出足够信息**：日志记下走了兜底，方便后续调优 prompt
4. **失败时写 error 状态**：让前端显示"出错"弹窗，用户点"重试"能再跑一次

## 附加优化

也可以在 `_parse_narration_json` 函数中加入更鲁棒的处理：

```python
def _parse_narration_json(raw):
    clean = raw.strip()
    if clean.startswith("```"):
        clean = clean.split("\n", 1)[1] if "\n" in clean else clean[3:]
    if clean.endswith("```"):
        clean = clean[:-3]
    if not clean.strip():
        raise json.JSONDecodeError("Empty content", "", 0)
    return json.loads(clean.strip())
```

加上空内容检查，让异常信息更明确，便于排查。

## 避免的陷阱

- ❌ JSON 解析失败直接 `return` → 前端显示"成功"但无产出
- ❌ 不初始化变量 → `NameError` 导致更意外的静默失败
- ❌ 第三级兜底用 AI 再试一次 → 同样的问题大概率重现，要换纯代码
