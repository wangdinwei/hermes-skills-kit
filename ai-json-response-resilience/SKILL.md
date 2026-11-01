---
name: ai-json-response-resilience
description: 多层容错策略解析 AI 模型返回的 JSON — 标准解析 → 正则提取 → 自动修复 → AI 自修复重试。适用于所有 AI 调用后需解析 JSON 的场景（大纲生成、分镜、旁白等）。
---

# AI JSON 响应容错解析

## 触发条件

AI 返回内容包含 JSON，但标准 `json.loads()` 失败，报错如：
- `Expecting ',' delimiter`
- `Expecting value`
- `Expecting property name enclosed in double quotes`
- 前端显示"AI 返回格式异常"

## 标准三层容错模式

### 第0层：直接解析

```python
raw = ai_response.strip().lstrip("```json").rstrip("```").strip()
try:
    result = json.loads(raw)
except:
    pass
```

### 第1层：正则提取 + 直接解析

AI 有时会在 JSON 前后加解释文字。用 `r'\{.*\}'` + `DOTALL` 抓取最外层 `{}` 内的内容：

```python
import re
m = re.search(r'\{.*\}', raw, re.DOTALL)
if m:
    try:
        result = json.loads(m.group())
    except Exception:
        logger.exception("[模块名] JSON 解析失败")
```

### 第2层：自动修复常见问题

```python
if not result:
    m = re.search(r'\{[^]*\}', raw, re.DOTALL)
    if m:
        text = m.group()
        # 修复尾部逗号：移除 `]` 或 `}` 前的逗号
        text = re.sub(r',\s*([}\]])', r'\1', text)
        # 修复单引号 → 双引号
        text = re.sub(r"(?<!\\)'", '"', text)
        try:
            result = json.loads(text)
        except:
            # 第二次尝试：处理转义引号冲突
            try:
                result = json.loads(text.replace('\\"', "'").replace("\\'", "'"))
            except:
                pass
```

### 第3层：AI 自修复重试

自动修复也失败时，把坏 JSON 发给 AI 让它自己修正：

```python
if not result:
    logger.warning("[模块名] JSON 修复失败，向 AI 请求修正")
    fix_prompt = (
        "你输出的 JSON 格式不合法，请仅输出修正后的 JSON，不要任何其他文字。\n"
        "不要用 ```json 包裹。确保：\n"
        "1. 所有键名用双引号\n"
        "2. 没有多余逗号\n"
        f"3. 合法 JSON\n\n原文：\n{ai_response[:3000]}"
    )
    r2 = ai_call(fix_prompt, "", model="deepseek-v4-flash", max_tokens=4096, timeout=30)
    if r2:
        c2 = r2.strip().lstrip("```json").rstrip("```").strip()
        try:
            result = json.loads(c2)
        except:
            m = re.search(r'\{.*\}', c2, re.DOTALL)
            if m:
                try:
                    result = json.loads(m.group())
                except:
                    logger.exception("[模块名] 重试后 JSON 仍失败")
```

## 注意

1. **第3层会增加一次 API 调用** — 增加延迟（~5-10s）和成本。只在非重试不可的场景使用。
2. **第2层不会增加额外调用** — 纯本地正则替换，推荐在所有 AI→JSON 端点都加上。
3. **`r'\{[^]*\}'` vs `r'\{.*\}'`** — 前者匹配时 `[^]` 等效于 `.` 加 `DOTALL` 但无需显式传 flag；后者需要 `re.DOTALL`。
4. **第2层尾部逗号修复的正则**：`re.sub(r',\s*([}\]])', r'\1', text)` — 移除最后一个逗号。
5. **单引号修复要小心**：简单全局替换 `'` → `"` 会破坏 JSON 字符串内部的合法单引号。上述正则 `(?<!\\)'` 不替换转义后的单引号 `\'`，但仍是粗糙做法。如果数据里有包含 `'` 的字符串（如中文文案中的单引号），建议加白名单过滤。

## 适用模块

已在以下位置应用此模式：
- `app.py:3398-3440` — 大纲生成（api_ai_generate_outline）
- `app/routes/l05.py:174` — 对白 JSON 解析
- `app/routes/l05.py:274` — 分镜 JSON 解析
- `app/routes/l03.py:571` — 分镜/旁白 JSON 解析

## 历史踩坑

### 自动修复的单引号陷阱

`re.sub(r"(?<!\\\\)'", '"', text)` 虽然跳过了转义单引号，但会破坏 JSON 字符串内容中的合法中文单引号（如描述文字中的 `'`）。如果后续遇到此类数据损失，改用更保守的策略：先尝试 `json5` 或 `demjson3` 库（`pip install json5`），它们对单引号的处理更准确。
