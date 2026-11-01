---
name: ai-multi-recommendation-with-direction-rotation
category: software-development
description: Make AI recommendation features produce diverse results (not the same every time) by rotating creative directions via seed parameter, returning multiple options, and showing selectable cards in the frontend.
---

# AI Multi-Recommendation with Direction Rotation

When building AI-powered recommendation features (project names, titles, style suggestions, etc.), use this pattern to solve 3 common problems:
1. **Same output every time** — seed parameter is mentioned in prompt text but not used for actual variation
2. **Only 1 result** — user can't compare and choose
3. **Same for all users** — no per-user differentiation

## Architecture

### 1. Backend: Define Rotating Creative Directions

Instead of asking the LLM the same question every time, rotate between distinct creative angles:

```python
def api_ai_recommend():
    seed = request.args.get("seed", "1")
    
    directions = ["复仇反转", "情感冲突", "悬疑切入", "现实议题", "高概念设定", "黑色幽默"]
    direction = directions[int(seed) % len(directions)]
    
    angle_hint = {
        0: '以「大快人心」的复仇爽感为核心，突出反转和碾压对手的场面',
        1: '以「情感纠葛」为核心，突出信任背叛和人物关系变化',
        2: '以「悬疑秘密」切入，开篇留悬念引导观众追剧',
        3: '以「社会议题」为底色，突出人性和道德困境',
        4: '以「高概念设定」为亮点——穿越、重生、系统、预知未来等',
        5: '以「黑色幽默」为基调，用荒诞讽刺的方式讲严肃故事',
    }
```

### 2. Backend: Request Multiple Results in One API Call

Change the prompt to output an array of results. Use `f-string` with escaped braces (`{{ }}`) for JSON:

```python
system_prompt = f"""你是一个策划高手。根据故事大纲，**一次推荐3个不同的名字**，各有不同切入角度。
...
第{seed}批推荐，核心切入角度：**{direction}**
{angle_hint.get(int(seed) % 6, angle_hint[0])}

请按以下 JSON 格式回复（只输出 JSON）：
{{
  "recommendations": [
    {{
      "project_name": "名字1（3-8字，从这个方向衍生）",
      "num_episodes": 推荐集数,
      "duration_minutes": 每集时长,
      "note": "一句话风格备注（10字内）"
    }},
    ...
  ]
}}"""
```

`temperature=0.8` alone is NOT enough — rotating the prompt content is the reliable way to get diverse results.

### 3. Frontend: Selectable Recommendation Cards

Display as clickable cards. Click fills the form:

```javascript
var _seed = Date.now() % 1000;  // random start, unique per user

function aiRecommend(retry) {
    if (retry) _seed++;  // "换一批" → different direction
    fetch('/api/ai/recommend?...&seed=' + _seed).then(r=>r.json()).then(d => {
        if (d.ok && d.recommendations) {
            window._lastRecs = d.recommendations;
            // Render cards...
            var html = d.recommendations.map((r, i) => {
                return '<div class="rec-card" onclick="pickRec('+i+')">'
                     + r.project_name + '</div>';
            }).join('');
            // Default select first
            pickRec(0);
        }
    });
}

function pickRec(idx) {
    var r = window._lastRecs[idx];
    document.getElementById('projName').value = r.project_name;
    // Also fill num_episodes, duration_minutes
}
```

### 4. Per-User Variation

- Seed initialized to a unique value per session (`Date.now() % 1000` or `user_id * 100 + batch`)
- Incremented on "换一批" → maps to a different direction
- Different users with different seeds get different creative angles

## Pitfalls

### 🔴 Chinese Quotation Marks in Python Strings
Chinese-style double quotes ("") look identical to ASCII `"` in many code editors. When used inside Python strings delimited by `"`, they can cause syntax errors:

```python
# WRONG — Python sees string ending at the second "
hint = {0: "以"大快人心"的复仇爽感为核心"}

# RIGHT — use single quotes
hint = {0: '以「大快人心」的复仇爽感为核心'}
# Or use 「」 brackets instead
```

### 🔴 F-string JSON Escaping
When outputting JSON in an f-string, double all braces `{{ }}` to escape:

```python
# {seed} is f-string interpolation, {{ is literal {
f"""第{seed}批... {{"key": "value"}}"""
```

### 🔴 API Reachability in China
Don't hardcode `api.deepseek.com` in Flask apps on Chinese servers. Use DashScope compatible API:
`https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions`

Add `proxies={"http": None, "https": None}` to bypass system proxy.

### 🔴 Temperature is Not Enough
`temperature=0.8` with identical prompts still converges to near-identical outputs on repeated calls. **Change the prompt content** (different directions per seed) for reliable diversity.
