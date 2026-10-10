# 06 — Agent-E 解剖与基线锚点（面试复习版）

## 1. 这一页讲什么 + 本档案复习法

本档案回答三件事，也是面试里所有"对比、外部数字、检索素养"类追问的底牌：

1. **Hercules 和 Agent-E 是什么关系**：直系后裔，文件级同源可 diff；13 项机制组件逐项对照后，没有一项"我方缺了导致丢分"。
2. **81.6% 锚点是怎么回事**：归属错误——它属 HxAgent 而非 Agent-E，且四重口径不可比；已按"历史文档不改写、更正以新文档固化"的原则更正。
3. **61.6% 凭什么叫"领先"**：越出零样本文本-DOM 口径公开可比带 47-60% 的上沿，该 regime 内无同口径公开可比数字；表述上从不说"超过某某 SOTA"。

**复习法**：先背第 2 节 60 秒主线到能脱稿；再用第 3 节速答卡过一遍"被问什么、答什么"；第 4 节两张故事卡各完整讲一遍（第一张计时 90 秒）；最后合上文件，用第 5 节表格自测数字。纪律只有一条：任何外部数字出口前，先过四口径卡。

## 2. 60 秒主线

被问"你们和 Agent-E 什么关系"，一段可背：

"我的执行内核是 Agent-E 的直系后裔——dom_helper、dom_mutation_observer 这些文件两边同名，逐文件 diff 看得到'同源之后各自分化'。但我做的远不止继承。它只有 browser 单域，我扩成七个 helper 的多域编排；它的 AutoGen wrapper 没有 token 核算、没有压缩回退、没有超时兜底，我换 LangGraph 后这些全有；工具是它的超集，22 个核心加 7 个扩展；评测比它严——页面原生 reward 加 own-seed 过滤、off-seed 信标、五类作弊扫描。13 项机制组件逐项对照完，结论是它没有一项'我缺了导致丢分'的机制。另外我要主动更正自己犯过的错：我一度把 81.6% 当成 Agent-E 的 MiniWoB 成绩当冲刺锚点，追了一手出处才发现它从未发表过 MiniWoB 数字，81.6% 真身是 HxAgent，而且四重口径不可比。所以我讲'领先'只有一句：零样本文本-DOM 全任务口径下，越出公开可比带 47-60% 的上沿——不说 SOTA。"

## 3. 决策速答卡

### 卡 1｜为什么做这次解剖（三个动机 + 怎么做的）

动机：

1. **同源关系必被问**：执行内核是 Agent-E 直系后裔（qa-bank Q32，按 80% 出现概率准备），"为什么不用 Agent-E / 差多少"必须有一手答案，不能停在"参考了它"。
2. **81.6% 锚点起疑**：r1 起把"MiniWoB++ 125 instances 81.6%"当 Agent-E 成绩记，准备对比材料逐字核对原文时暴露经不起一手核验。
3. **r5 吸收决策要证据**：哪些祖先机制值得回吸、哪些是传说组件、哪些会破坏我方口径——要组件级清单，不要印象。

方法四步：论文精读（Agent-E，arXiv 2407.13032 HTML v1 全文，架构与评测逐节读、数字逐字摘录）→ 仓库实解剖（克隆 Agent-E 当时最新提交，与本仓库 testzeus_hercules/ 逐文件 diff；dom_helper、js_helper、dom_mutation_observer、detect_llm_loops、response_parser 等同源文件全部 DIFFERS，即两边都改过、非纯拷贝）→ 数字追源（81.6% 反向检索，定位真实出处 HxAgent）→ 对照我方现状做 13 项组件对照与 A~H 八候选吸收评级（收益/成本/文化冲突/作弊面）。

### 卡 2｜13 项组件对照（紧凑版）

判定口径：**继承** = 机制照搬后分化；**改造** = 按 MiniWoB regime 有意反向裁制；**超越** = 我方有而它无或更强。

| 组件 | 判定 | 一句话差异 |
|---|---|---|
| 1 分层 planner（nested chat） | 继承+超越 | 它 AutoGen 双 agent（planner 50 轮、每步 fresh 实例化 nav 10 轮）；我 LangGraph（planner 500 轮上限、每步 fresh nav 10 轮）加多域路由、判分语义、token/耗时核算 |
| 2 子目标管理 | 继承（无差距） | 同为 next_step 单元素粒度 + helper 无状态每步带全上下文；我加 EXPLICIT CLOSURE NUDGES |
| 3 DOM 三档蒸馏 | 继承+超越 | 同为 text/input/all 三档；元素标识 mmid 改名 md（机制同一）；r4 `--md-extended` 把"可见但不可寻址"元素纳入终表（+17 格） |
| 4 变化观察 | 继承（无差距） | 同源 MutationObserver + click 回包变化提示；我另有状态刷新标记机制 |
| 5 planner 验证纪律 | 改造 | 它要求每步后验证、终止前验证；我按 regime 反向裁制——明令勿加独立验证步、勿找页面成功指示（这环境没有），诚实性由 verify gate（0 分诚实性组件）+ assert-discipline 承担；回吸会重蹈 r2 前验证空转、240s 里烧钟 |
| 6 持久性/"绝不轻言终止" | 改造（有意收紧） | 它 prompt 要求绝不轻言终止；我允许多次尝试后报告并终止——r3 实证 9 个超时格死因是 canvas 盲区+停顿，不是过早放弃，收紧无收益证据 |
| 7 URL 回溯 | 改造/不适用 | 它 planner 步骤显式带返回 URL；我明确禁止重导航——MiniWoB 单页任务回溯=重置任务，是 r3 专项修掉的 churn |
| 8 self-evaluator / Curious Reflector | 传说组件，无实体可吸 | 论文与仓库均不存在；实际机制 = 自查（#5）+ 变化观察（#4）+ 计划修订（#6）的组合 |
| 9 技能集 | 超越（超集） | 它 click/entertext/openurl/get_dom/pause_flow/get_user_input 等约十项 + 动态加载；我 core/tools 22 个 + extra_tools 7 个 |
| 10 用户 LTM | 继承改用 | 同源机制，它注入用户偏好，我改注入测试数据（basic_test_information） |
| 11 ask-user（get_user_input/pause_flow） | 不适用 | 评测禁人工，对应动作是终止报告 |
| 12 编排基建 | 超越 | 它 AutoGen wrapper 无 token 核算、无压缩、无超时兜底、无重试语义；我 LangGraph 全有 + Portkey 多模型路由 |
| 13 评测 harness | 超越（更严） | 它 JSON 任务 + 人工/LLM 字符串判分；我页面原生 reward + own-seed 过滤 + off-seed 信标 + 五类作弊扫描 + 超额归因逐格实证 |

统计：超越 5（#1/3/9/12/13，其中 #1/3 继承基础上超越）、纯继承无差距 3（#2/4/10）、regime 改造 3（#5/6/7）、不适用 1（#11）、传说无实体 1（#8）。小结：Agent-E 的机制箱里没有一把钥匙对得上我剩余 48 格的结构（官方失败约 39 = 完成幻觉存量 + 能力失败，超时 9）——它的 WebVoyager 成绩靠长程多页导航与人工判分的宽容度，两者都不在我方口径内。

### 卡 3｜领先 Agent-E 的 6 条

1. **判分完整性**：页面原生 reward、own-seed 过滤、off-seed 信标、五类作弊扫描、超额归因逐格实证——它用人工/LLM 字符串判分，无此闭环。
2. **多域编排**：七个 helper（browser/api/sec/sql/mcp/time_keeper/executor）+ target_helper 路由——它只有 browser。
3. **上下文/token/超时治理**：LangGraph 状态机 + assertion/verify 节点、context 超限压缩回退、planner timeout 兜底、token/cost 全程核算、Portkey 多模型 fallback——它的 AutoGen wrapper 均无。
4. **md-extended DOM 覆盖**：把 span 假链接、折叠面板头、hitarea、slider-handle、label 这类"可见但不可寻址"元素纳入终表，+17 格逐格实证——超出它 all_fields 的交互元素白名单。
5. **工具超集**：22 + 7，含 drag_and_drop/dropdown/slider/date-time/upload/geo/captcha/sandbox/clipboard。
6. **任务表示**：Gherkin 中转（Given=URL/When=指令）是双刃剑——转译损耗未量化（没做过有/无中转对照），但它承载"录制即用例"主目标，属战略分化而非落后。

### 卡 4｜唯一硬差距 + 吸收评估（A~H 八候选）

**唯一硬差距 = 截图升级观察**（出处是 HxAgent 的 o^img 策略，**Agent-E 无此物**）：DOM 文本优先，出现 ≥2 个候选歧义或超大 DOM 需摘要时才调截图。我方截图仅作 proof 留档、不上模型（工具回包返回纯文本，图像传不进模型上下文；get_latest_screenshot_stream 存在但不接模型）。射向视觉/画布/几何族 **约 11 格盲区**（count-shape、count-sides、click-shape、circle-center、drag-circle、bisect-angle、find-midpoint、draw-* 等）。

- **A 截图升级观察 → 改造后吸收，先 3-5 格试射**：建议 count-shape、count-sides、circle-center、drag-circle、visual-addition（最后一个是 r2 曾过的"非永久盲区"校准点）。走合法路径：EXTRA_TOOLS_MODULES 注入 observe_screenshot 工具（get_latest_screenshot_stream 有先例），不动核心；试点通过再把"工具回包可携带 image content"作 PLAN.md 接线点申报。**前置 go/no-go：GLM 端点 image_url 支持——未验证，整条建议的生死门**。口径披露：进 headline 需新 regime 标注"文本-DOM+截图升级"并同步修 baselines 分层；试射 0 增益即关闭归档。
- **B 跨 episode 经验库 → 不擅自吸收，口径决策上交**：HxAgent 长期经验（其消融 −9.4 点）把每类任务从零样本变 few-shot，**直接破坏零样本单次口径——增益属协议增益非能力增益**，可比性掺水。若未来采纳，战略载体天然是 record2gherkin（经验 → Gherkin 用例库），与项目主目标同构。
- **C~H 不吸收一句话**：C planner 验证纪律（已以 verify + assert-discipline 更适配形式存在）；D"绝不终止"持久性（超时死因非过早放弃，与 240s 冲突）；E URL 回溯（单页 regime 回溯=自毁）；F get_user_input/pause_flow（评测禁人工；pause 已有 time_keeper，pdf 已在 extra_tools）；G SAM 历史截断（无证据我方全历史+压缩受损）；H enter_text_and_click 复合技能（Agent-E 自己都注释掉了，我方 bulk 工具+submit 约定覆盖）。

HxAgent 另三项对照：迭代规划 + 逐步 State Evaluator 与我方 planner 循环 + verify gate **等价**；短期记忆 SAM 无证据我方受损；SoA→Selenium 脚本生成与我方蒸馏器**战略同构**，不是 benchmark 补丁。

### 卡 5｜A/B/C 基线分层（实验前固化于 baselines.md，2026-09-21）

| 档位 | 方法 | 分数 | 口径/备注 |
|---|---|---|---|
| **A 零样本/文本-DOM（最可比）** | GPT-4 agent（NeurIPS 2023，UCLouvain） | ~47% | 同口径成员 |
| A | BrowserGym/WorkArena 系 GPT-4 基线 | ~50-59% | 依赖观察格式，AST 优于裸 DOM 文本 |
| ~~A~~ Agent-E（125 challenging instances） | 归属更正：81.6% 实为 HxAgent，四重口径不可比；**Agent-E 从未发表 MiniWoB++ 数字**（唯一数字 WebVoyager 73.2%，人工评判） | ~~81.6%~~ | 降为背景参照，不作对比目标 |
| **B 少样本/带演示** | Synapse（trajectory-as-exemplar） | ~99.2% | 64 任务子集+示例，不同 regime |
| B | RCI（NeurIPS 2023） | 当时 SOTA | 少量演示 |
| **C 训练模型** | WebGUM（多模态） | 94.2% | 能力来源不同 |
| 参考 | 人类 | ~91%+ | — |

纪律：这张表在**实验前**（2026-09-21 初版）就固化在 baselines.md，不是出了分再找参照——这是检索素养的题眼（qa-bank Q18）。A 档成员出处是文献级不是复现级，报告注明"检索自公开资料，设置细节以原论文为准"。

### 卡 6｜61.6% 定位表述（逐字口径）

> "零样本文本-DOM 全 125 任务类单次口径下，**无同口径公开 SOTA 可比**；A 档带（47-60%）**已越上沿**；81.6%（HxAgent）与 73.2%（Agent-E/WebVoyager）均不同口径，仅作背景。"

口径六要素（详见 qa-bank Q13）：全量 125 任务类（browsergym registry）、零样本、只读文本-DOM（md 蒸馏，约 a11y 档）、GLM 全角色（planner glm-5.3 / nav glm-5.3-flash）、240s/格（原版 10s，放宽已披露）、单 seed 单次、判分 = 页面原生 `WOB_RAW_REWARD_GLOBAL > 0`；官方口径 = 干净口径、零无效格。四轮线：43.2% → 46.4%（clean 47.8%）→ 47.2% → **61.6%（77/125）**。禁语：不说"我们是 SOTA"，说"越出公开可比带上沿"；B/C 档（94-99%）不是公平对照，只作背景。

### 卡 7｜四口径变量（可比性裁判）

**是否零样本（有无演示/训练）｜观察模态（文本-DOM vs 截图多模态）｜任务集（全量 vs 子集）｜判分方式（页面原生奖励 vs 人工/LLM/金标序列）**。四项任一不同即不可比，只能作背景；外部数字出口前先过这张卡。

## 4. 踩坑故事卡

### 故事卡一｜81.6% 锚点更正（王牌：检索素养 + 诚信）

**什么时候讲**：被问"你们离 SOTA 多远""和 Agent-E 差多少""你引用的外部数字可靠吗"；也是"你犯过什么错"的最佳素材。

**讲述**（90 秒，平实版）：

"我引错过一个数字，后来追了一手出处把它更正了。9 月 21 号我搭基线表，公开检索时把'MiniWoB++ 125 instances 81.6%'记成了 Agent-E 的成绩当冲刺锚点——当时没做一手核验，二手来源的归属本身就是错的，r1 到 r4 的报告里写着'距同源架构锚点 Agent-E 差 20 个百分点'。准备对比材料时要逐字核对原文，9 月 22 号我把 Agent-E 仓库克隆下来通读论文，发现全篇没有 MiniWoB++ 字样——它唯一的成绩是 WebVoyager 73.2%，GPT-4-Turbo、人工评判。那 81.6% 是谁的？22 到 24 号反向追查，定位到 HxAgent 的 Table 4，把口径逐项抄下来，四重不可比：任务集是 5 个任务族×25 实例的偏科子集，同表基线 Li 9.6、SeeAct 8.0、WALT 0.0，差 70 多点，说明子集高度偏科；观察是 GPT-4o 多模态，DOM 加截图；训练加经验注入，每族 20 个训练实例、最多 8 条经验规则，零样本变 few-shot；判分是动作序列金标 Exact-Match，序列全对才算过，跟页面终态 reward 不可互译。而且 81.6% 自带 ±34.6 的方差，本身就不适合当稳定锚点。9 月 24 号更正做了三件事：baselines.md 该行划线保留原文、注明真实归属、降为背景参照；round4-report 原文保留、追加更正标注；'距 Agent-E 20 个百分点'全面停用，qa-bank 写了防复发条款。历史文档不改写，更正以新文档固化。"

**收尾金句**："引用外部数字必须追一手出处；可比先对齐四口径，任一不同只能作背景——这两条纪律就是那次翻车买来的。"

### 故事卡二｜传说组件不存在

**什么时候讲**：被问"Agent-E 不是有 self-evaluator / Curious Reflector 吗？你们怎么没有？"

**讲述**：

"做吸收评估时我想找它的 reflection 机制回吸，结果论文 v1 全文和仓库当时的最新提交里都查不到 Curious Reflector，也没有独立的 self-evaluator 或 self-corrector。它真正的'自我纠错'是三样东西的组合：planner 自查、变化观察、计划修订——没有独立组件，所以这一项无实体可吸。EmergenceAI 博客或后续分支有没有加过这类组件，我没查实，如实标注未验证。"

**收尾金句**："传说组件经不起对原文——机制组合不等于独立模块。"

## 5. 必背数字

| 数字 | 白话含义 | 什么时候会被问 |
|---|---|---|
| 61.6% = 77/125 | 我方 headline；四轮线 43.2 → 46.4（clean 47.8）→ 47.2 → 61.6 | 成绩、涨分曲线类问题 |
| 口径六要素 | 240s/格（原版 10s）、零样本、只读文本-DOM、全量 125 任务类、planner glm-5.3 / nav glm-5.3-flash、单 seed 单次、`WOB_RAW_REWARD_GLOBAL > 0` | "你这数字怎么来的""和谁能比" |
| 47-60% | A 档可比带（UCLouvain ~47%；BrowserGym/WorkArena 系 50-59%），61.6% 越其上沿 | "你说领先，超过谁了" |
| WebVoyager 73.2%（Table 3 写 73.1） | Agent-E（arXiv 2407.13032）唯一公开成绩：WebVoyager（He et al., 2024）15 个真实网站 643 任务；GPT-4-Turbo（planner/nav 同款）；5 名标注员（IST 办公时间）人工评判、"完整完成才算过"；静态日期任务平移 8 个月保可做；实测成功均值 150s / 失败 220s、平均 25 次 LLM 调用/任务；站点间 27.3%（Booking）~95.7%（WolframAlpha）；对 text SOTA +21%、多模态 +16%；无消融表 | "Agent-E 多少分""你们比它强吗" |
| 81.6% EM ±34.6 | HxAgent（arXiv 2608.15491）Table 4：5 任务族×25 实例偏科子集（choose-date、book-flight、flight.AA、flight.Alaska、flight.Alaska-auto）；同表基线 Li 9.6 / SeeAct 8.0 / WALT 0.0（差 70+ 点=子集高度偏科） | "81.6% 不是比你们高吗" |
| 975 实例 97.4% EM / 99% PM | HxAgent 完整集（39 任务×25 实例）成绩；每族 20 训练实例 + 最多 8 条 few-shot 经验规则；GPT-4o temperature=0；判分=动作序列金标 Exact-Match（(op, element, value) 全对才算过） | "那它 97% 怎么说" |
| 消融 88 / 75 / 47（%） | HxAgent 去经验 88%、去迭代规划 75%、去 SAM+经验 47%；经验库单项 −9.4 点 | "经验库到底值多少分" |
| 48 格 ≈ 官方失败 ~39 + 超时 9 | 我方剩余丢格结构；超时从 21 修到 9；方差 +7 格（±10 格噪声带/±8%；16.8% 翻转 / 12.8% 纯噪声 / 26-34% 边缘格） | "剩下的分为什么拿不到" |
| +17 格 | md-extended 收益：24 个新通过格中 17 格有点击日志实证（email 4/4 + find-greatest + 12 格"可见但不可寻址"元素） | "DOM 蒸馏值多少分" |
| verify gate 0 分 / 103 轮 | 诚实性组件 0 分、103 轮全观察性 | "怎么防作弊/防幻觉完成" |
| ~11 格 | 视觉/画布/几何盲区（count-shape、click-shape、bisect-angle、draw-* 等）；试射建议 3-5 格，含 visual-addition（r2 曾过的非永久盲区校准点） | "你们还差什么" |
| 工具 22 + 7 | core/tools 22 个 + extra_tools 7 个 | "工具能力对比" |
| 七个 helper | browser/api/sec/sql/mcp/time_keeper/executor 多域编排 | "和 Agent-E 架构差异" |
| 50/10 vs 500/10 轮 | planner/nav 轮数：Agent-E planner 50 轮、每步 fresh nav 10 轮；我方 planner 500 轮上限、nav 10 轮 | "架构参数级对比" |
| 13 项判定统计 | 超越 5、纯继承 3、改造 3、不适用 1、传说 1 | "逐项比过吗" |
| Synapse 99.2% / WebGUM 94.2% / 人类 ~91%+ | B 档（Synapse 64 任务子集+演示；RCI 当时 SOTA）、C 档（训练模型）、人类参考 | "为什么别人 90 多你们 61" |
| arXiv 号 | Agent-E 2407.13032；HxAgent 2608.15491（2026-08，胡志明市高校团队；其 OpenReview 页引 2508.10833，对不上，哪个是正式号没查实） | 被要求给出处 |

未验证事项（被追问时的诚实答案）：GLM 端点图像输入支持（吸收方案 A 的 go/no-go）；EmergenceAI 后续版本是否发过 MiniWoB 数字；HxAgent 步数上限 MAX_t 值与 GPT-4o 快照版本；HxAgent 正式 arXiv 号。

## 6. 联动

- **qa-bank Q32**（和 Agent-E 什么关系/为什么不用它）→ 本档案第 2 节主线 + 卡 2。
- **qa-bank Q18**（检索素养/基线何时固化）→ 卡 5 纪律 + 故事卡一。
- **qa-bank Q14 / Q22**（外部数字口径类）→ 卡 6 逐字口径 + 卡 7 四口径 + 第 5 节必背数字。
- 事实底稿（需溯源时再读，不在面试中展开）：dev_docs/benchmark/agent-e-anatomy.md、dev_docs/benchmark/baselines.md、dev_docs/benchmark/round4-report.md。
