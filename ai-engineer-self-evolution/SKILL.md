---
name: ai-engineer-self-evolution
description: 构建 AI 助手的「工程师自进化体系」——利用 OB bug 记录作为知识库，建立自动触发、预检、复盘、技能提取的闭环机制。适用于非技术用户（甲方/产品）与技术 AI 的长期协作。
---

# AI 工程师自进化体系

## 适用场景

用户是**非技术产品方**（不会写代码、不懂术语、纯甲方视角），而我作为 AI 工程师需要：
1. 记住所有踩过的坑
2. 不重复犯同样的错
3. 用户一句话说现象，我自动调出历史记录

## 核心机制：记忆触发

```mermaid
graph TD
    A[用户说现象] --> B{记忆触发规则}\n(每次对话自动注入)
    B --> C[查 OB 知识库]\n(对应步骤的 bug 记录)
    B --> D[查 session_search]\n(最近讨论过的问题)
    C --> E[出方案前自检]
    D --> E
    E --> F[用户确认]
    F --> G[动手 -> 预检脚本 -> 验证]
    G --> H[复盘 -> 提取 skill]
    H --> B
```

## 步骤

### 1. 建立 OB 知识库索引

全量读取所有 Obsidian bug 修复记录，按步骤（L01~L11）和问题类型（JS编译 / CSS布局 / 后端拆分 / 数据流等）分类。

**产出：** 一份带索引的目录文件 + 步骤→记录的映射关系。

### 2. 固化记忆触发规则

把以下规则写入 `memory`（长期记忆，每次对话自动注入）：

> 「用户说任何关于步骤的问题/bug/页面异常/功能需求时，必须先查 OB 目录中相关记录再回。具体到 L01~L11 的，先搜文件名含该步骤的 md 文件读完再答。记不清的用 session_search。」

**这是整套系统的命脉**——没有这条记忆注入，其他所有机制都失效，因为没人提醒我去查。

### 3. 提取重复模式为 skill

遍历 OB 记录，找**出现 2 次以上的相同根因模式**：

| 模式 | 提取时机 |
|------|---------|
| const/let 重复声明 | 已有 skill `js-const-duplicate-declaration` |
| Tabler .grid vs Tailwind | 已有 `tabler-tailwind-grid-conflict` |
| Flask 模块拆分漏导入 | 已有 `flask-monolith-module-extraction-recovery` |
| data-action 事件双发 | 已有 `data-action-double-firing-debug` |
| CSS `<style>` 标签包裹遗漏 | 新建 skill |
| .env 被工具破坏 | 新建关联 `flask-api-key-centralization` |

每条 skill 记录：触发条件 + 排查步骤 + 修复方案 + 验证方法。

### 4. 出方案前预检

非 trivial 改动前自检三个问题：
1. 这次改造涉及的路由/模板/JS 有没有类似的 bug 历史？（查 OB）
2. 最可能出问题的地方在哪？
3. 如果我的理解是错的，会在哪暴露？

### 5. 修复后复盘

复杂问题（3轮以上交互或用户纠正过）解决后走复盘：
- 我当时漏了什么信号？
- 如果再来一次，第一步应该做什么？
- 这个问题的「一眼直觉」是什么？

发现可重复模式 → 创建/更新 skill。发现了新薄弱领域 → 追记到知识缺口.md。

### 6. 用户沟通铁律

**用户不需要知道任何术语。** 他们只需要：
- 说现象（「这个按钮没反应」「加载不出来」「字太小了」）
- 发截图
- 说想达成什么效果（「帮我加个一键XX的功能」）

剩下的**全是我查**——查 OB、查 session、查代码。

## 边界条件

- **不依赖 cron 也能跑**：记忆触发是基础，cron（盲点扫描/巡查）只是增强
- **不需要用户编程**：用户只需写 OB 记录（已有习惯），其余由 AI 完成
- **用户无需「教我」**：这是和「用户出需求 AI 就从零摸索」模式的最大区别

## 关键文件

- OB 知识存储：`/path/to/workspace
- 实施方案备份：`/path/to/workspace
- 薄弱领域追踪：`/path/to/workspace

## 陷阱

1. **记忆空间有限（10KB）**：每条记忆要精炼，定期合并老旧条目腾空间
2. **不能查太多次**：先搜文件名含步骤的 md 文件是最快的；第一次查了，同一话题后续回复不必重复查
3. **用户不会提醒我**：如果没触发，用户会直接认为 AI 不行——所以触发规则必须写得足够宽泛，覆盖所有「用户说问题」的场景

## 致命错误模式清单（2026-06-23 审计更新）

### 🚨 模式1：数据格式无契约
跨模块传参必须有类型断言（`isinstance(x, float)`）。Chatwoot `created_at` 是整型时间戳不是ISO字符串，`previous_status` 是 `'solved'` 不是 `'resolved'`。双轨存储（progress.json + DB）清一个没清另一个。

### 🚨 模式2：bfcache 生命周期误用
`load` 事件在 bfcache 恢复时不触发。必须用 `pageshow + event.persisted`。`setInterval` 被冻结不自动重启。计时从 `Date.now()` 开始 → bfcache 归零。

### 🚨 模式3：except: pass 吞错误（最高危）
绝对禁止 `except: pass`。所有 except 块必须至少打 `logger.exception()`。检测：`grep -rn "except.*pass" app/`

### 🚨 模式4：data-action kebab-case 静默失败
`app.js` 将 kebab-case 转 camelCase 查 `window[fnName]()`。函数名不匹配时静默跳过。esbuild `format:iife` 必须 `window.xxx = xxx` 导出。

### 🚨 模式5：两套实现路径打架
加新路径时未删除旧路径。典型：`auto_generate.py` 有独立 `_step_lXX`。预防：统一接口 → **删除旧实现**。

### 工程纪律
1. 所有 `except` 块必须打 `logger.exception()`
2. 改 JS 后必须 `node build.js` 重编译
3. 改前必须先查 OB Bug 记录
4. Ctrl+Shift+R 强刷后再验收视觉改动
4. **不要在回复中贴链接列表**：用户不关心你查了什么，他只关心结果
