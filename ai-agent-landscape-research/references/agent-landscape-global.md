# 海外 AI 智能体生态地图（2026-10 快照）

> 价格核实于 2026-10，仅供参考，引用前重核。

## 一、分水岭：2026 年从「任务型」转向「常驻个人 Agent」

给 AI 在云端配一台独立虚拟电脑、长期记忆、自己判断该干什么、关掉 App 还在跑。

| 发布 | 产品 | 母体 | 价格 | 关键限制 |
|---|---|---|---|---|
| 2026-05-19 | Gemini Spark（I/O） | Google | $19.99（AI Pro） | EEA/UK/瑞士/尼日利亚不可用 |
| 2026-08-11 | Grok Bot（beta） | xAI | 打包在 $120–300/mo 档 | 无免费档 |
| 2026-09-08 | Meta Muse | Meta | 免费至 1 亿 token/周，$20/$100 | 仅美国 |
| 2026-09-25 宣布 | M365 Copilot Cowork | Microsoft | $30/user/mo + 用量 | 绑 M365 |
| 2026-09-29 | OpenAI dots | OpenAI | ChatGPT Pro 起 $100/mo | 无免费档，GPT-6 Astra |
| 2026-10-21 | Claude Duet | Anthropic | 仅 Pro/Max | 研究预览，美国排队，锁 Opus 5.5 |

## 二、通用交付型

| 产品 | 强项 | 价格 | 硬短板 |
|---|---|---|---|
| ChatGPT Work | 长周期办公任务，产出表格/幻灯片/文档/Web App | 免费/$8/$20/$100/$200 | **读不了本地磁盘文件** |
| Claude Cowork | **能真碰本地文件系统**、整理文件夹、定时任务；已上网页+移动 | $20 Pro | Agent 模式烧 token 快，重度用几天顶到 Pro 上限 → 实际要 $100 Max |
| Manus 1.6 | 最接近「描述完就走，回来收货」 | 免费 300 积分/日，$20 起 | 积分不滚存、成本难预测；**Meta 收购 2026-04 被中国监管否掉，现独立运营于新加坡** |
| Perplexity / Computer | 带引用的研究、事实核查 | 免费/$20/$200 | 不产文件、不做流程自动化 |

## 三、编程 Agent（成熟度最高的一类）

| 产品 | 特点 | 价格 |
|---|---|---|
| Claude Code | 终端主力，公开 GitHub 提交占比曾达 4% | $20/$100/$200 |
| **OpenHands** | **最强免费方案**，MIT，SWE-bench Verified 72%，可全本地 BYOK | 自托管免费 |
| Cursor | 2026-10 改 The X Subscription | 免费/$8（仅 Grok）/$30 Premium/$100/$200 |
| GitHub Copilot | 改 AI Credits 制（$0.01/个），旧「premium request」已废 | 免费 2000 补全/$10/$39/$100 |
| Devin | ACU 计费已废改纯美元；Windsurf 已改名 Devin Desktop | $20/$200；Teams $80+$40/人 |
| Google Antigravity 2.0 | 免费多 Agent 编排，还能跑 Claude/GPT-OSS 模型 | 免费（周限额不公开） |
| Grok Build | 终端原生，token 便宜 | 含在 The X Subscription |

## 四、自动化 / 工作流

- **n8n** — 自托管免费，最强。**注意它不是开源**，是 Sustainable Use License（fair-code），公司有 license 政策要留意
- **Make** — $9 起，多分支逻辑最顺，性价比最高
- **Zapier Agents** — 最易上手，9000+ 集成，量大后最贵
- **Relay.app** — human-in-the-loop 强，$19 / 团队 $59 含 10 人
- **Gumloop / Lindy** — 无代码 Agent 搭建

## 五、开源自托管框架（核心对比）

| | OpenClaw | Hermes Agent |
|---|---|---|
| 出品 | Peter Steinberger（OpenAI 技术团队成员） | Nous Research（本机在跑的运行时） |
| 语言/形态 | Node.js，**多 Agent 协作**，靠改配置文件拼装 | Python，**单 Agent + 内置学习循环** |
| 核心卖点 | 生态最大、技能市场最丰富、上限最高；GitHub 三个月 24 万星 | 技能自沉淀，能力随使用时长增强；约 4 万星 |
| 上手难度 | 高（配 Node 环境、跨系统文件权限、通信通道能劝退非技术用户） | 中 |
| 致命问题 | 缺乏限制时死循环烧 token、误删本地文件；创始人自曝个人月 token 账单约千万美元量级 | 同样有本地文件权限风险，需沙箱 |

**OpenClaw 的安全记录是选型一等维度**：第三方 marketplace 中查出过 824 个恶意 skill，历史上出过 9.9 严重级漏洞，微软明确建议不要在存放敏感数据的设备上运行。微软 Build 2026 基于它做了 Microsoft Scout（企业级），并出了 Windows 版预览；NVIDIA 2026-09 底发布开源智能体安全平台 **OpenShell**（100+ 机构采用），自托管 Agent 的权限边界值得跟。

## 六、已停服清单（别照旧攻略推荐）

- OpenAI **Operator** — 2025-08-31 关闭
- OpenAI **ChatGPT Atlas** 浏览器 — 2026-08-09 关闭，能力并入 ChatGPT Work + Chrome 扩展
- **Sora** — 网页/App 2026-04 关闭，API 2026-09 关闭
