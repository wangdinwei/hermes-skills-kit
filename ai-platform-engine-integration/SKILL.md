---
name: ai-platform-engine-integration
description: Use when 把第三方 AI 生成平台接进自己的产品当引擎。
version: 1.0.0
author: 阿常（feishu_4）
license: internal
metadata:
  hermes:
    tags: [architecture, integration, ai-video, provider-abstraction]
    related_skills: [libtv, video-consistency-audit]
---

# 把第三方 AI 平台接成自己的引擎

> 场景：自己的产品已有生成链路，想接一个更强/更便宜的第三方平台。
> 核心问题不是「接哪家」，是「接它的哪一层」。
> 案例背景：LibTV/liblib.tv（具体参数见 `libtv` 技能）。

## When to Use

- 要给自己的流水线加视频/图片生成后端，在几个平台间选
- 平台同时提供「裸模型 API」和「托管 Agent + 模板」两种入口
- 用户问「把它当模型供应商还是 skill 供应商」

---

## 一、先分清它给了几层，只有一层该接

| 能力层 | 典型入口 | 该不该接 | 理由 |
|---|---|---|---|
| **托管 Agent / 模板层** | `run_agent(skills=[...])`、一句话丢给它自己排 | **不接做主流程** | 黑盒、不可复现、成本不可估、错误不可定位 |
| **模型层** | `generation_submit(model, params)` | **接** | 能指定 model + 完整 params（含 prompt），功能上等价于模型 API |
| **工程能力层** | 画布/节点连线（多参考图、分组约束、节点间接续） | **接（往往这才是它真正独有的）** | 裸模型 API 给不了「一镜挂 3-6 张参考图」这种编排能力 |

### 为什么托管 Agent 层不能当产品引擎

| 问题 | 后果 |
|---|---|
| 黑盒 | 用户付钱要「X 天出片」，你不知道它打算怎么写提示词、怎么切镜、怎么算时长 |
| 不可复现 | 同样输入跑两次结果不一样，出问题无法向用户解释 |
| 成本不可估 | 一开生成就开始扣费，事前算不出这一单花多少 |
| 错误不可定位 | 成品不好，是提示词错、模型错、还是模板编错？查不出来 |

**卖「确定性交付」的产品，不能用「随机创作」的引擎。**

可以留作**溢价选项**（用户主动勾选「交给平台 Agent 自由创作」），但不能是默认路径。

---

## 二、正确接法：抽一层 provider

```python
# services/video_provider.py
class VideoProvider(Protocol):
    name: str
    def submit(self, prompt: str, model: str, refs: list[str],
               duration: int, audio: str | None) -> str: ...   # → task_id
    def poll(self, task_id: str) -> dict: ...                  # → {status, url, error}
    def download(self, task_id: str, out: Path) -> Path: ...
```

每个平台一个实现，平级并存；配置一行切换（`VIDEO_PROVIDER=xxx`）。

**这一步的价值不是「优雅」，是能 A/B 对比** —— 同一批素材分别跑两个 provider，比一致性指标、单集成本、耗时、失败率，用数据决定主力与兜底。

### 接入函数清单（照这个写）

```python
def auth_header()          # token 一般 1h 过期，调用前查 expires_at 提前 5min refresh，对调用方无感
def create_project(name)   # 一任务一项目，方便查体检
def upload_media(path)     # ticket → 分片 PUT → complete → 挂到项目
def create_image_node(prompt, model, refs)
def create_video_node(prompt, model, ratio, refs, audio_node)
def submit(node_id, model, params)
def poll(task_id)
def check_issues(project_id)   # ★ 出片前必查
def export(project_id, clips, audio)
```

---

## 三、四个必坑

| 坑 | 解法 |
|---|---|
| **token 短期过期**（常见 1h） | 每次调用前查 `expires_at`，提前 5 分钟 refresh |
| **没有余额查询接口** | 自己记账：提交前按 `秒数 × 单价` 预估写库，提交后记实扣 |
| **接口通道一律计费**（网页的「0 积分畅用」不适用） | 成本算进定价，别拿网页活动价估接口价 |
| **重试重复扣费** | 强制 `idempotency_key`，同一次重试必须复用同一个 |

---

## 四、接入顺序（反了就是白接）

1. **先数文件，确认产线跑到第几步 —— 不看代码，看产出。**

```bash
# 列出所有集
find <production_root> -maxdepth 5 -type d -name "第*集"
# 逐集数各步骤产出
for d in <...>; do for s in "$d"/L*/; do echo "$(basename $s): $(ls -1 "$s" | wc -l)"; done; done
```

若 N 集全部「第 1 步有产出、第 2 步起全空」，结论就一句：**产线根本没跑通，现在谈接引擎是空转**。先修产线。

2. **单步手动跑，把报错抓出来（不改逻辑）。** 在哪一步断，就单独跑那一步的 API，看它报什么。被 `except: pass` 吞了就先只把**那一步**改成 `logger.exception`，再跑一次。

3. **再说产线设计**。出片逻辑如果是「一镜一 prompt 一图生视频」，换更强模型照样出「全是怼脸 + 亮度漂移」。**引擎治不了设计问题。**

4. **清错误吞没 —— 只清接入调用链。** 门槛是「新接入路径及其生成／扣费／任务状态／回调／导出路径中，静默吞异常为 0 处」。不要因为存量几百处就不敢开工，也不用一次全清；存量另排。

5. **抽 provider 层**。不动现有逻辑，只加一层。

6. **只接两样**：模型提交接口 + 多参考图编排能力。不接托管 Agent、不照搬对方的工程目录结构。

7. **A/B 对比**，用测量数据决定主力与兜底（怎么量见 `video-consistency-audit`）。切主力的门槛要同时满足多项而不是单项：盲评一致性提升幅度、合格秒成本上限（相对现有链路）、按期交付率、交付时长 P95。任一不达标就维持现有链路。

---

## 五、谈「对方独有」之前，先 grep 自己有没有

**这是最容易丢信任的一步。** 拿着对方的能力清单去说「你有三个缺口」，而自己的代码里其实早就实现了——用户一旦发现，后续所有判断都不作数。

动手前逐个 grep：

```bash
# 参考图／元素挂载是否已支持多个
grep -rn "element_list\|reference_images\|refs\s*=" <services>/
# 是否有「上一镜末帧当首帧」这类接续
grep -rn "lastframe\|last_frame\|first_frame_path" <routes>/ <services>/
# 是否有跨镜语义连贯注入
grep -rn "prev_prompt\|previous_shot" <routes>/
```

**典型误判**：以为「每镜只挂 1 张图」，结果 `lock_elements_for_shot` 的注释写着「角色/场景/道具，上限 7 个」，而 `first_frame_path` 已经拿上一镜的末帧当首帧。**对方能给的编排能力，自己可能已经有一半。**

流程到这一步才该判断：真正的缺口是「7 个参考还不够」还是「cast/资产报名表里压根没登记 element_id」——后者是数据问题，补数据即可，换引擎无用。

---

## 六、单一供应商就是风险

平台改价、封号、宕机都会让业务停摆。provider 抽象不是可选项。
兜底优先级：**已跑通的现有链路 > 第二个云厂商 > 新接入的平台**。

---

## 七、要接的往往是「编排能力」，不是「单价」

把「对方能做什么我做不到」列成表，只接差的那些：

| 能力 | 裸模型 API | 带编排的平台 |
|---|---|---|
| 单镜参考图数 | 1–2 张 | **3–6 张**（脸+场景+道具+上一镜尾帧） |
| 全片共用约束 | 无挂载点 | 分组/文本节点，连到该组所有镜头 |
| 镜头间接续 | 无解 | 上一镜锁下一镜的人物位置 |
| 结构体检 | 无 | `issues` 接口直接报「孤立节点 / 无时间线」 |

**这三四样正好对应「脸漂 / 光漂 / 位置漂」三个顽症。接平台的动机应该是这个，不是便宜几分钱。**

---

## 八、写方案时必须给的六件

用户要的不是「可以接」，是一份能开工的东西：

1. 接哪几层、明确不接哪层，各带一句理由
2. provider 接口签名 + 现有 provider 的改造点
3. 配置项清单与切换方式
4. 四个坑的具体处理（token / 记账 / 计费口径 / 幂等）
5. 分阶段顺序 + 每阶段验收标准（要可测，不能写「跑通」）
6. 回滚方式：一行配置切回现有链路

**不要在这一步夸用户的项目。** 用户明确表示过反感捧场，判断必须由数据支撑，优劣都要说。

### 怎么讲人才能听懂（“没听太懂” = 失败信号）

分层、抽象、引擎、编排——这些词用户不接。他会说「我还是没听太懂」或「到底改哪里」。

**换成这个结构：**

| 步骤 | 现在怎么做 | 改成什么 | 改哪个文件 | 工作量 | 改完怎么验 |
|---|---|---|---|---|---|

\- 先给结论，再给理由；不要先铺垫背景
\- 每一步必须落到「改哪个文件 / 哪一行」，没有落地位置的建议等于没说
\- 用「一条台词 + 一张脸」这种具体描述代替「信息不完整」这种抽象说法
\- 不要把外包模型（如 ChatGPT）的长篇裁决原文转发给用户；先自己消化成表，并明确标出「它纠正了我哪几条」
\- 不要中途汇报进度。用户说过「工作完了再一次性搞定」，分段汇报会被当成没做完
