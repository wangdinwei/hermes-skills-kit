---
name: ai-music-two-lane-production
description: Use when 歌曲要伴奏生动/贴切/多元，或要做剧场立体声效果。人声与伴奏分两条腿生成再适配。
category: media
version: 1.0.0
author: Hermes (feishu_4)
license: MIT
metadata:
  hermes:
    tags: [music, mixing, stereo, reverb, demucs, ace-step, mastering]
    related_skills: [diffrhythm-song-generation, ai-music-video-production, local-music-generation]
---

# AI 歌曲「两条腿」制作法：人声与伴奏分开生成再适配

> 2026-10-26 实测跑通：DiffRhythm 出真唱 → htdemucs 分离人声 → ACE-Step 按同 BPM 定制伴奏 → 混音做剧场立体声。

## 一、先过这道门：多数情况不要上两条腿

用户说「伴奏要生动／贴切／多元」「要立体声／剧场效果」时，直觉是拆成两条腿各自做好再适配。**但先量一件事：你的伴奏源干净吗？**

```python
flat = float(np.mean(librosa.feature.spectral_flatness(y=y)))
# < 0.012 干净；> 0.02 噪声感重
```

| 伴奏源 | 平坦度 | 能不能拿来当第二条腿 |
|---|---|---|
| 整曲模型原生输出（MiniMax Music 3 / DiffRhythm） | **0.0050–0.0140** | ✅ 干净，但没必要拆 |
| ACE-Step 生成的伴奏 | **0.0339** | ❌ **不能用**，比原生差 6.8 倍，一换就变怪物 |
| 商业工具官方分轨（MELO 等 48kHz 全轨） | 官方分轨 | ✅ 最理想，不经过 demucs |

**决策顺序：**

1. **首选：不拆。** 直接让整曲模型自己出完整混音，只做极轻母带。伴奏“多元”靠**每段不同 caption + 不同 seed**，不靠拆轨。这是成本最低、听感最保真的路
2. **次选：要真分轨就用商业工具的官方分轨**，不要自己跑 demucs
3. **最后才是两条腿**：只有在「已有一版神级人声、必须保留它、而伴奏确实不行」时才走。且伴奏源必须是干净的

**两条腿真正的代价不是工作量，是人声必然经过处理链。** 这条链上任何一步过度（混响、宽度、压缩）都会把“真人感”拿走 —— 而那通常是用户唯一认可的东西。

### 端到端模型的局限（知道就好，不必都去解）

1. **伴奏不可控** — 想要“这段只有钢琴”做不到，词表里也没 BPM
2. **不能单独做声场** — 整体加混响会把人声也弄糊
3. **同 prompt 多段会出相近伴奏** — 用不同 caption + 不同 seed 解决，不用拆轨

## 二、完整流程（六步）

```
① DiffRhythm 出真唱（抽卡到男声）
② htdemucs 分离 → vocals.wav + no_vocals.wav
③ 测 vocals 的真实 BPM（关键：这是唯一真相）
④ ACE-Step 按该 BPM + 该段情绪 生成**纯伴奏**
⑤ sox 分别加混响（人声轻/伴奏稍大）+ ffmpeg stereotools 展开
⑥ 混音（人声压伴奏，谐波/鼓比 ≈1.4）+ 母带（loudnorm I=-14）+ 编码
```

### ② 分离

```bash
/opt/tts-venv/bin/demucs --two-stems=vocals -n htdemucs -o /tmp/split 输入.wav
# 产出 /tmp/split/htdemucs/<name>/vocals.wav + no_vocals.wav
```

**注意 demucs 装在 `/opt/tts-venv`，不在 lipsync-venv。**

### ③ 测 BPM（不可跳过）

```python
import librosa, numpy as np
y, sr = librosa.load("vocals.wav", sr=22050, mono=True)
bpm = float(np.atleast_1d(librosa.beat.beat_track(y=y, sr=sr)[0])[0])
```

实测：分离出的 vocals / no_vocals / 原混三者 BPM 完全一致（112.3），说明分离不破坏节拍。

### ④ ACE-Step 出纯伴奏

```bash
curl -s -X POST http://127.0.0.1:8001/v1/chat/completions \
  -H "Content-Type: application/json" -d '{
  "model": "acestep/acestep-v15-turbo",
  "messages": [{"role":"user","content":"<prompt>Chinese rap instrumental, drum machine, deep bass, no vocals, melancholic</prompt>"}],
  "audio_config": {"duration": 95, "bpm": 112, "format": "wav", "vocal_language": "zh", "key_scale": "D major"}
}'
```

- **参数必须包在 `audio_config` 里**，平铺顶层静默失效
- 传 `bpm: 112` 实测出 **112.3**，与人声自动对齐 —— **BPM 不会错位，这是整套方法能成立的基础**
- 输出是 48kHz 立体声 wav，时长精确到设定值

**取音频的路径（容易踩）：** 响应里 `choices[0].message.audio` 是 **list**，元素是 `{"type":"audio_url", "audio_url":{"url":"data:audio/wav;base64,..."}}`（约 24MB 字符串）。

```python
url = d['choices'][0]['message']['audio'][0]['audio_url']['url']
raw = url.split(",", 1)[1] if url.startswith("data:") else url
open("inst.wav","wb").write(base64.b64decode(raw))
```

### ⑤ 空间感：默认**什么都不加**（这一节最容易做错）

**先看实测，再决定要不要碰混响：**

| 处理 | 平坦度 |
|---|---|
| 分离后的人声 | 0.0217 |
| **人声加 sox reverb（wet 28）** | **0.1288（恶化 6 倍，已接近噪声）** |

正常音乐 0.005–0.02，**0.13 就是噪声**。用户听到的就是“变成怪物了，没有真人感”。

| 想做的效果 | 正确做法 | 不要做 |
|---|---|---|
| 空间感 | 什么都不加；要氛围只能在**伴奏轨**加 wet ≤ 10 | ❌ 人声轨加 sox reverb |
| 立体声宽度 | 编曲本身的声部摆位 | ❌ `stereotools phase`（相位偏移会把声音搞散，丢中心感） |
| 人声突出 | 只做 EQ + 轻压缩（ratio ≤ 2） | ❌ ratio 3+、makeup 1.5+（人声动态本来就小，压很了会僵） |

**保真母带模板（实测平坦度 0.0055，与原生 0.0050 几乎无差）：**

```bash
ffmpeg -y -i 输入.wav -filter_complex \
"[0:a]equalizer=f=300:t=q:w=1.2:g=-1.2,\
equalizer=f=3200:t=q:w=1.5:g=1.2,\
acompressor=threshold=-20dB:ratio=2:attack=15:release=250:makeup=1.1,\
alimiter=0.97,loudnorm=I=-14:TP=-1.5:LRA=11[a]" \
-map "[a]" -c:a libmp3lame -b:a 192k out.mp3
```

**不加混响、不加宽度、不分离 —— 只做 EQ + 轻压缩 + 响度标准化。** 人声一个插件都不挂。

### ⑥ 混音 + 母带（只在真两条腿时）

```bash
ffmpeg -y -i v_fx.wav -i i_fx.wav -filter_complex \
"[0:a]equalizer=f=300:t=q:w=1:g=-2.5,equalizer=f=3200:t=q:w=1.5:g=2,\
     acompressor=threshold=-18dB:ratio=3:attack=10:release=200:makeup=1.5[v];\
 [1:a]volume=0.50[bg];\
 [v][bg]amix=inputs=2:duration=first:dropout_transition=0,\
     stereotools=mlev=1.12:slev=1.30:phase=6,\
     alimiter=0.95,loudnorm=I=-14:TP=-1.5:LRA=10[a]" \
-map "[a]" -c:a libmp3lame -b:a 192k out.mp3
```

- `slev`（side level）比 `mlev` 大 → 拉宽声场
- `phase=6` 轻微相位偏移制造深度感
- 伴奏 volume 0.50 是起点，以「谐波/鼓比」校准

## 三、⚠️ 这套方案在什么情况下不成立（重要）

### 先讲方法：诊断必须逐级量同一指标

用户说「变成怪物了」时，很容易抓一个可疑环节就开始改。实测连续归错因**两次**（先怪混响+分离 → 再怪其他处理链 → 最后才量出来是伴奏源本身）。

**正确做法：把链上每一级产物都拿同一指标量一遍，先找突变点，再动手：**

```
原生整轨 → 分离的人声 → 分离的伴奏(或外部伴奏源) → 加完效果的轨 → 混音成品
        每一级都算 spectral_flatness + 频谱重心，突变那一级就是元凶
```

**别只看成品和原生的差距** —— 那只能告诉你“有问题”，告诉不了你“哪一步”。

### 实测结果

| 轨 | 平坦度 | 结论 |
|---|---|---|
| DiffRhythm 原生整轨 | **0.0050** | 音乐性好 |
| htdemucs / ft 人声 | 0.0221 | 人声本身特性（去低频后高频占比自然升高），**不是 artifacts** |
| 两轨重建（人声+伴奏相加） | **0.0052** | **证明分离不损失音质** |
| **ACE-Step 伴奏** | **0.0339** | **噪声感重，比原生差 6.8 倍** |

**核心结论：ACE-Step 生成的伴奏本身就是“脏”的**（合成器/鼓机高频噪声）。拿它替换 DiffRhythm 那个干净伴奏，声音必然变怪物。

**所以：**

- ✅ **可以用两条腿**：前提是伴奏源质量够高（不是 ACE-Step）
- ❌ **不能用 ACE-Step 换伴奏**：它的平坦度 0.0339，注定“脏”
- ✅ **DiffRhythm 自己出不同伴奏**：平坦度 0.0050，不同 prompt 能出不同配器，这是正确路径

### 还有一个隐藏坑：DiffRhythm 本身质量随机

同样 prompt、同样 LRC，平坦度实测 **0.0050 / 0.0086 / 0.0163 / 0.0333 / 0.0396**（差 8 倍）。

**所以抽卡必须双指标：F0 < 155Hz（男声）且 平坦度 < 0.012（干净）。** 只靠 F0 筛性别不够——“怪物感”往往从生成那一步就埋下了，后面怎么混都救不回。

## 四、验收指标（用数字，不靠耳朵）

**首要指标是频谱平坦度，不是宽度。** 平坦度超过 0.01 就说明已经 audible 失真了。

```python
flat = float(np.mean(librosa.feature.spectral_flatness(y=y)))   # 0.005–0.02 正常
```

| 版本 | 谐波/鼓 | 立体声宽度 |
|---|---|---|
| DiffRhythm 原生混音 | **4.30**（人声太突出，伴奏被压扁） | 0.17 |
| 两条腿·干净版 | 1.42 | 0.170 |
| **两条腿·剧场版** | **1.64** | **0.231** |
| 参考爆款原曲 | 1.33 | 0.212 |

**目标区间：谐波/鼓 1.3–1.7、宽度 0.20–0.24。**
原生 4.30 → 两条腿 1.4，是这套方法最直接的价值证明。

## 五、踩坑

| 坑 | 症状 | 解法 |
|---|---|---|
| **人声轨加 sox reverb** | 用户说「变成怪物了，没有真人感」；平坦度 0.0217 → **0.1288** | **人声一点混响都不要加**。要空间只在伴奏轨加 wet ≤ 10。这条是本 skill 最贵的教训 |
| **sox 没有 `stereo` 效果** | `sox FAIL formats: can't open input file 'stereo'`（把它当输入文件） | 宽度交给 ffmpeg `stereotools`，且 `phase` 保持 0 |
| ACE-Step 输出路径 | 找不到音频文件 | 在 `choices[0].message.audio[0].audio_url.url`，是 base64 data URL |
| ACE-Step 参数平铺 | 传了 bpm/duration 但没生效 | 必须包进 `audio_config` |
| demucs 找不到 | lipsync-venv 里 ModuleNotFoundError | 用 `/opt/tts-venv/bin/demucs` |

## 六、分段做法（要“多元”时）

**首选：不拆轨，每段用不同的 caption + seed 直接让整曲模型出。** 下面是整曲模型的 caption 差异表（不是拆轨配器表）：

| 情绪 | caption 里的配器词 |
|---|---|
| 压迫/不服 | `Bass Drums` |
| 悔/急 | `Drums Bass` |
| 体面/克制 | `Piano Strings Drums` |
| 无力/飘 | `Piano Acoustic Guitar` |
| 疲惫/冤 | `Piano Bass Drums` |

都保留 Drums/Bass 作统一基底，否则拼起来像五首不同的歌。

**如果已经拆了轨**，每段可以用不同 ACE-Step prompt —— 但先回到第一节确认伴奏源够干净。
