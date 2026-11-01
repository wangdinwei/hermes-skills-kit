---
name: ai-agent-landscape-research
description: Use when 要 AI 智能体清单或对比（扣子、OpenClaw）。
category: research
version: 1.0.0
author: Hermes (feishu_4)
license: MIT
metadata:
  hermes:
    tags: [ai-agents, research, product-comparison, agent-platforms]
    related_skills: [ai-api-comparison-research, tavily-search, baidu-web-search]
---

# AI 智能体产品调研与选型对比

> 用于回答「目前有哪些 Agent／智能体，对比一下」这类问题。
> 与 `ai-api-comparison-research` 的分工：那个管**模型 API 的计费与接入**，这个管**智能体产品/框架/平台本身**。

## When to Use

触发：用户要「AI Agent／智能体」的清单、排行、横评、选型建议；或多半直接点名 WorkBuddy、扣子 Coze、OpenClaw、Hermes Agent、Manus、Dify、千问办公、豆包工作这类产品。

不适用：只问模型 API 的计费与接入（走 `ai-api-comparison-research`）；只问模型能力排名（走模型测评类调研）。

## 第一铁律：先消歧，别默认

「AI Agent 清单／对比」至少有五个层次，答错层等于白答。用户说「智能体」时，**大部分指的是能装在自己机器上干活的运行时**，不是云端开箱即用的消费产品 —— 别默认成后者。

| 层 | 是什么 | 典型产品 |
|---|---|---|
| **开源自托管框架** | 自己部署，接管你的电脑/服务器，任意接模型 | OpenClaw、Hermes Agent |
| **大厂桌面办公智能体** | 桌面客户端，替你在本机操作文件 | WorkBuddy、百度搭子、千问办公、豆包工作 |
| **低代码搭建平台** | 搭自己的 Bot/工作流，再发布到渠道 | 扣子 Coze、Dify、腾讯元器、千帆 AppBuilder、百炼、腾讯云 ADP |
| **通用交付型 Agent** | 给个 brief，收回成品 | Manus、Claude Cowork、ChatGPT Work、Perplexity |
| **代码编排框架** | 纯写代码的库 | LangGraph、AutoGen、CrewAI、AgentScope |

判断不了就用一句话问清「你要的是能装在自己机器上干活的、云端开箱即用的成品，还是搭 Bot 的平台」，别自己假设。

**落层是开口前的动作，不是被纠正后的补救。** 先判定是哪一层再回答；先甩一份云端消费级清单、等用户回一句「我指的是像 WorkBuddy、扣子、OpenClaw 这样的」，就是一次完整返工。

补充判据：用户报出的名字若**跨层混排**（如同时提 WorkBuddy + 扣子 + OpenClaw + Hermes），说明他要的是**能部署/能自建/能装在自己机器上的那整个谱系**，应按「开源自托管框架 / 大厂桌面办公智能体 / 搭建平台」三层并列给出，而不是只挑一层深讲。

## 第二铁律：接进 IM 是另一个问题，先分清「通道」和「替身」

选型之后用户几乎一定追问「能不能接企业微信／能不能接我的微信」。这一问答错会给出封号级的建议，先分流：

- **通道** = 你在 IM 里给它派活、它回结果。个人微信只有这一种官方途径（微信助理，扫码绑定、零风险）。
- **替身** = 替你收发别人的消息、管群。个人微信**从未开放**消息读取 API，任何方案都是外挂。

**铁律：个人微信只能当遥控器，不能当替身。** 要合规地自动收发客户消息，唯一路径是**企业微信客服 API**（或公众号/小程序客服 API）—— 把私域从个人号迁到企微是行业标准做法。落到具体人身上，最实用的折中是「让 AI 只帮写、不代发，人审后手动发」：效率提升、风险为零。

渠道/版本号/开通步骤/权限模型见 `references/im-channel-integration.md`。

## 调研路径

1. **中文**：百度千帆「百度搜索」（`safe_search: true` 必开）。查询词要分层，一次只搜一层。
2. **英文**：Tavily 走搬瓦工代理。海外这层中文资料滞后严重，必须补英文。
3. **抓正文用 `curl` + 正则清洗**，不要依赖 `web_extract`（keyless 常失败）。具体调用见 `tavily-search` skill。
4. **来源优先级**：官方定价页 > 第三方实测/横评 > 榜单机构（QuestMobile、IDC、易观、GAIA、SWE-bench）> 媒体报道 >> 软文。

## 输出形状

- 开头一句**分水岭式判断**（今年这个品类的形态变了什么），别直接甩清单。
- 每个层次一张表，每行给：**定位 / 价格 / 硬短板**。只写优点的行没有信息量。
- 表格价格**必须标核实月份**（「2026-10 核实」），并提醒会变。
- 单独一段「按你的情况选型」，落到用户手上真正在做的项目，不要泛泛而谈。
- 单独一段「明确别碰的」，给出为什么。

## 坑

1. **产品会死，每次先确认还活着。** 这个品类的下架速度快过价格变化：OpenAI Operator 已关、ChatGPT Atlas 已关、Sora 网页端和 API 已先后停服。仍在推荐已死产品的攻略到处都是，看到「Operator」「Atlas」就要警觉。
2. **中文「选型／对比／榜单」类查询是软文重灾区。** 同一篇稿子会在 ifeng、sina、eastmoney、地方新闻站等 5+ 域名重复发布；且大量是单一厂商买稿（典型句：「最优路径推荐：优先选择 XX」）。做法：按标题去重 → 剔除点名单一厂商的推荐 → 回到一手榜单和实际测评。
3. **星数、月活、访问量是三种口径，不能混。** 引用时标清来源和时点（GitHub star ≠ 注册用户 ≠ 月活 ≠ 月访问量）。
4. **安全记录必须单独查。** 自托管 Agent 的核心风险是第三方技能市场与权限边界，这是选型的一等维度，不是脚注。
5. **别先落地方案。** 先确认层次和选型，用户点头再谈部署或写代码。
6. **「能不能帮我管微信」先问清是哪个微信。** 企业微信有官方开放 API（内部应用/机器人，能读消息、文档、会议、网盘）；个人微信从未开放消息读取。混为一谈必然给错建议 —— 详见 `references/im-channel-integration.md`。

## 参考文件

- `references/agent-landscape-china.md` — 国内生态地图（桌面智能体月活、搭建平台、会员与 Coding Plan 价目）
- `references/agent-landscape-global.md` — 海外生态地图（常驻 Agent 时间线、编程/自动化工具、开源自托管框架、已停服清单）
- `references/im-channel-integration.md` — 智能体接 IM 渠道对照（WorkBuddy×企微融合版与 API 模式、微信助理、ClawBot 插件、个人微信红线、群聊提效动作、管理员管控）
