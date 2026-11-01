---
name: ai-pipeline-subprocess-debug-checklist
description: 排查 AI 视频/音频流水线中 subprocess 调用的静默失败。涵盖 ffmpeg 多输入音频索引、子进程返回码检查、字幕质量对齐正片规范、BGM 混音参数等踩坑经验。
---

# AI 流水线 subprocess 调试清单

当 AI 流水线（Kling → TTS → ffmpeg 合成）出现视频无 BGM、字幕错乱、子进程静默失败时，按此清单排查。

## 1. ffmpeg 子进程静默失败

**症状**：日志显示「完成」但产物不完整或不存在。

**根因**：`subprocess.run()` 默认不抛异常，即使 ffmpeg 失败返回码非 0。

**修复**：
```python
result = subprocess.run(cmd, capture_output=True, timeout=120)
if result.returncode != 0:
    raise RuntimeError(f"ffmpeg 失败(code={result.returncode}): {result.stderr.decode()[:500]}")
if not os.path.isfile(output_path) or os.path.getsize(output_path) < 1000:
    raise RuntimeError("ffmpeg 未生成有效文件")
```

**注意**：stderr 默认截断太短时看不到真实错误，至少要 500 字符。

## 2. BGM 音频索引混音错误

**症状**：视频有 TTS 配音但无背景音乐。日志无报错。

**根因**：ffmpeg 多输入 `-i` 的音频流索引写错。

**规则**：
```
-i input0.mp4  → 流索引 0 (视频 [0:v], 音频 [0:a])
-i input1.mp3  → 流索引 1 (音频 [1:a])
-i input2.mp3  → 流索引 2 (音频 [2:a])
```

**常见错误**：BGM filter 写成 `[1:a]volume=0.2[bgm]`，但 `[1:a]` 实际上是 TTS！正确写法：
```
[1:a]volume=0.7[voice];[2:a]volume=0.23[bgm];[voice][bgm]amix=inputs=2:duration=first[audio]
```

**正片音量参数（应完全对齐）**：
- TTS：`volume=0.7`
- BGM：`volume=0.23`
- BGM 加淡入淡出：`afade=t=in:st=0:d=2,afade=t=out:st=10:d=2`

## 3. SRT 字幕换行符陷阱

**症状**：字幕显示为 `\n` 字面量而不是换行。

**根因**：Python 字符串中 `"\\n"` 输出字面量 `\n`（两个字符），SRT 需要实际换行符 `\n`（一个字符）。

**修复**：永远用 `"\n".join(lines)` 而不是 `"\\n".join(lines)`。

## 4. 字幕质量对齐正片

正片字幕处理链路：
```
raw_text → clean_punctuation() → split_subtitle_lines(max_lines=2, chars_per_line=24) → SRT/ASS
```

**关键参数**：
- `chars_per_line`：短句用 14（正片多镜头分割场景），单居长句用 24（试玩单句场景）
- `max_lines`：固定 2 行，禁止第 3 行
- 时间分配：按句子字符数加权分配，首尾留 0.3-0.5 秒空白

**正片函数**（可直接复用）：
```python
from app.utils import clean_punctuation, split_subtitle_lines, _to_srt_time
cleaned = clean_punctuation(raw_text)
display_lines = split_subtitle_lines(cleaned, max_lines=2, chars_per_line=24)
```

## 5. Kling API 网络重试

**症状**：Kling 轮询阶段随机出现 `ConnectionResetError`（TLS 断连），导致整个任务失败。

**修复**：在 Kling 轮询 `urlopen` 处加重试：
```python
_retries = 0
_max_retries = 3
for i in range(180):  # 最多等15分钟
    time.sleep(5)
    try:
        resp = json.loads(opener.open(poll_req, timeout=30).read())
    except (urllib.error.URLError, ConnectionError, TimeoutError) as e:
        _retries += 1
        if _retries <= _max_retries:
            continue
        raise
```

## 6. TTS 日志噪音

**症状**：每次 TTS 成功后打印 ERROR 级别 `[json.JSONDecodeError] Extra data`。

**根因**：火山引擎返回 chunked JSON lines，单 JSON 解析预期失败，属于正常流程。

**修复**：降为 DEBUG 级别，标注「正常流程」。

## 7. 测试方法论

1. **先写测试规范文件**：列出所有验收标准、测试类型、容错场景
2. **按标准逐项测**：测一轮→修好→记录→等指示→测下一轮
3. **维修记录写入方案文件**：每个 bug 记（问题 + 根因 + 修复 + 文件 + 行号）
4. **分开测、分开计时**：每测完一轮发一次视频给用户确认，不等同同时提交多个任务
