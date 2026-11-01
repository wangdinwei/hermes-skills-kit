---
name: ai-content-labeling-cn
description: Use when 在国内平台发 AI 生成内容（抖音/小红书/B站）要合规标注。
version: 1.0.0
author: 小常（feishu_4）
license: internal
metadata:
  hermes:
    tags: [compliance, douyin, aigc-label, watermark]
    related_skills: [douyin-publish-wine, mv-hard-rules-dingwei]
---

# AI 生成内容标注（国内合规）

## 法规依据

- 《**人工智能生成合成内容标识办法**》国信办通字〔2025〕2号，四部门联合发布，**2025-09-01 施行**
- 配套国标：**GB 45438—2025**《网络安全技术 人工智能生成合成内容标识方法》

两类标识，**两个都要做**：

| 类型 | 定义 | 视频怎么落地 |
|---|---|---|
| **显式标识** | 用户能直接感知（文字/语音/图形） | 画面内文字「AI生成」 |
| **隐式标识** | 写在文件元数据里，肉眼看不到 | mp4 metadata / XMP 里写 AIGC 字段 |

## 做法（照拄）

### 1. 画面上标

- 片头：前 2 秒在画面中部偏下加一行「本视频由 AI 生成」
- 全片：右下角或左上角常驻小字「AI生成」，字号不要大到破坏画面，但不能被平台 UI 完全遮住
- 海报/封面：左上或右下角同样加「AI生成」（文字与背景对比度要够）

ffmpeg 叠加（竖屏 720×1280 为例）：
```bash
ffmpeg -i in.mp4 -vf "drawtext=fontfile=/usr/share/fonts/truetype/wqy/wqy-zenhei.ttc:\
text='AI生成':x=w-tw-24:y=h-th-150:fontsize=26:fontcolor=white@0.72:box=1:boxcolor=black@0.28:boxborderw=8" \
  -c:a copy out.mp4
```
- `y=h-th-150` 是为了避开抖音底部 UI 区，**别直接贴最底**

### 2. 元数据里写隐式标识

```bash
ffmpeg -i in.mp4 -c copy \
  -metadata title="标题" \
  -metadata comment="本视频由人工智能生成合成" \
  -metadata description="AIGC:AI-generated-video; producer=示例公司音乐" \
  -metadata "AIGC=AI-generated" out.mp4
```

### 3. 平台发布时声明

抖音/小红书发布页都有「内容由 AI 生成」声明开关 —— **必须勾上**，且与片内文字标识一致。

## 自检清单

- [ ] 片头出现过「AI 生成」字样
- [ ] 全片有常驻小标，且没压到平台底部 UI
- [ ] 海报/封面也有标
- [ ] mp4 metadata 里有 AIGC 字段（`ffprobe -show_entries format_tags` 验）
- [ ] 发布时勾了平台 AI 声明
- [ ] 文案里不写「真人实拍」类误导表述
