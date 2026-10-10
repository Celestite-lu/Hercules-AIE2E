# 06 — Agent-E 解剖对照 + 公开基线与锚点更正

> 档案日期：2026-10-10。核心源：`dev_docs/benchmark/agent-e-anatomy.md`（2026-09-22~24 调研，全篇精读）、`dev_docs/benchmark/baselines.md`（含 2026-09-24 更正行）、`dev_docs/benchmark/round4-report.md` §1 更正注；口径一致性参照：`dev_docs/interview/qa-bank.md` Q14/Q18/Q22/Q32（参照不作为事实源）。代码锚点已于 2026-10-10 在本仓库现场复验；Agent-E 侧结论依据当时的实克隆解剖固化记录（克隆位于 /tmp/agent-e-anatomy，HEAD f218c3c，2025-05-12，现已随 /tmp 清理，本次未重克隆复核）。

## 1. 这一页讲什么

本档案回答三件事：Hercules 与 Agent-E 是什么关系（直系后裔，文件级同源可 diff，13 项机制组件逐项对照后无一项"我方缺了导致丢分"）；81.6% 锚点是怎么回事（归属错误——它属 HxAgent 而非 Agent-E，且四重口径不可比，已按"历史文档不改写、更正以新文档固化"原则更正）；61.6% 凭什么叫"领先"（越出零样本文本-DOM 口径公开可比带 47-60% 的上沿，regime 内无同口径公开 SOTA，表述上从不说"超过某某 SOTA"）。它是面试中所有"对比、外部数字引用、检索素养"类追问的底层证据库。

## 2. 设计决策与方法论

### 2.1 为什么做这次解剖（三个动机）

1. **同源关系必被问**：执行内核 Hercules 是 Agent-E 的直系后裔（qa-bank Q32 出现概率 80%），"为什么不用 Agent-E / 你们和它差多少"必须有比"参考了它"更深的一手答案。
2. **81.6% 锚点出现疑点**：r1 起把"MiniWoB++ 125 instances 81.6%"记为 Agent-E 成绩当冲刺锚点；准备对比材料时需要逐字核对原文，暴露了这个数字经不起一手核验。
3. **r5 吸收决策需要证据**：哪些祖先机制值得回吸、哪些是传说组件、哪些会破坏我方评测口径——需要组件级清单而不是印象。

### 2.2 怎么做的（四步）

1. **论文精读**：Agent-E 论文 arXiv 2407.13032（HTML v1）全文，§2 架构 / §3 评测逐节读，数字逐字摘录。
2. **仓库实解剖**：克隆 EmergenceAI/Agent-E HEAD f218c3c（2025-05-12）至本地，与本仓库 `testzeus_hercules/` 逐文件 diff；同源文件（dom_helper / js_helper / dom_mutation_observer / detect_llm_loops / response_parser 等）全部确认为"同源后分化"（全部 DIFFERS——即两边都改过，不是纯拷贝）。
3. **数字追源**：对 81.6% 做 backwards 检索，定位真实出处为 HxAgent（arXiv 2608.15491，Ho Chi Minh City Univ.，2026-08）Table 4，并逐项摘录其口径。
4. **对照我方现状**：以 r4 后的 `testzeus_hercules/core/` 为准做 13 项组件对照与吸收评估矩阵（A~H 八个候选逐个评级：收益/成本/文化冲突/作弊面）。

### 2.3 基线为什么分层（A/B/C）

外部数字不可直接比的原因是**四个口径变量**：是否零样本（有无演示/训练）、观察模态（文本-DOM vs 截图多模态）、任务集（全量 vs 子集）、判分方式（页面原生奖励 vs 人工/LLM/金标序列）。按这四个变量把公开数字切成三档：

- **A 档 = 零样本/文本-DOM/全任务集**：与我方口径最可比，合成 47-60% 可比带。
- **B 档 = 少样本/带演示**（Synapse、RCI）：不同 regime，只作背景。
- **C 档 = 训练模型**（WebGUM）：能力来源不同，只作背景。

关键纪律：**分层表在实验前就固化在 baselines.md（2026-09-21 初版）**，不是出了分再找参照——这是"检索素养"的题眼（qa-bank Q18）。A 档成员的出处粒度是文献级不是复现级，报告须注明"检索自公开资料，设置细节以原论文为准"。

## 3. 组件对照与基线分层

### 3.1 同源关系：Hercules 是 Agent-E 直系后裔的四类证据

| 证据 | Agent-E 侧 | 我方侧（已现场复验，2026-10-10） |
|---|---|---|
| 文件级同源可 diff | 仓库 HEAD f218c3c：dom_helper、js_helper、dom_mutation_observer、detect_llm_loops、response_parser 等 | `testzeus_hercules/utils/` 下**同名文件全部存在**（dom_helper.py / js_helper.py / dom_mutation_observer.py / detect_llm_loops.py / response_parser.py）；当时逐文件 diff 结论：全部 DIFFERS（同源后各自分化，非纯拷贝） |
| planner→browser-nav 分层 | AutoGen nested chat：PlannerAgent（50 轮）→ 每步 fresh 实例化 BrowserNavAgent（10 轮/步）；planner 输出 JSON `{plan, next_step, terminate, final_response}` | 同构：`core/simple_hercules.py` planner 节点 `planner_max_chat_round=500`（L103）→ executor 节点每步 fresh 调 nav agent（`browser_nav_max_chat_round=10`，L104）→ assertion/verify 节点；planner JSON schema 同形且扩展（`is_assert`/`target_helper` 等，high_level_planner_agent.py L83） |
| md/mmid DOM 蒸馏 | a11y 树注入 `mmid` + `aria-keyshortcuts`（get_detailed_accessibility_tree.py） | **同一机制、属性改名**：`utils/get_detailed_accessibility_tree.py` 注入 `md` + `aria-keyshortcuts` 双写（L50-88，已有 orig-aria-keyshortcuts 保留逻辑）；本仓库 grep "mmid" 零命中属属性改名，非机制缺失 |
| mutation observer 变化观察 | 页面级 MutationObserver + click 技能回包附 "new elements have appeared... needs further interaction"（click_using_selector.py L45-58） | 同源继承：`utils/dom_mutation_observer.py`；click_using_selector.py L129 同款回包文案；另有 `_STATE_REFRESH_MARKERS`（simple_hercules.py L528）触发状态刷新 |

### 3.2 13 项机制组件对照表（逐项）

判定口径：**继承** = 机制照搬后分化；**改造** = 按 MiniWoB regime 有意反向裁制；**超越** = 我方有而 Agent-E 无或更强。

| # | 组件 | Agent-E 实现（论文+仓库实证） | 我方实现 | 判定 |
|---|---|---|---|---|
| 1 | 分层 planner（nested chat） | AutoGen 双 agent：PlannerAgent 50 轮 → 每步 fresh BrowserNavAgent（10 轮/步）；JSON `{plan, next_step, terminate, final_response}`（ae/core/prompts.py PLANNER_AGENT_PROMPT） | LangGraph（simple_hercules.py）：planner 500 轮上限 → executor 每步 fresh nav（10 轮）→ assertion/verify 节点；schema 加 is_assert/target_helper | **继承+超越**（更结构化：+多域路由、+判分语义、+token/耗时核算） |
| 2 | 子目标管理 | next_step 单元素粒度 + "helper 无状态、每步带全上下文"约定 | 同源：next_step + EXPLICIT CLOSURE NUDGES（high_level_planner_agent.py L144/L185，已复验）+ helper 无状态约定 | **继承（无差距）** |
| 3 | DOM processor（三档蒸馏+mmid） | `get_dom_with_content_type`：text_only / input_fields / all_fields；a11y 注入 mmid | 同源三工具：get_page_text / get_input_fields / get_interactive_elements（md 标识）；r4 `--md-extended` 把"可见但不可寻址"元素纳入终表（+17 格实证） | **继承+超越**（extended 覆盖超出 all_fields 的交互元素白名单） |
| 4 | 变化观察 | MutationObserver + click 回包变化提示 | 同源继承 + `_STATE_REFRESH_MARKERS`（L528） | **继承（无差距）** |
| 5 | planner 验证纪律（每步后验证/终止前验证/勿信 helper） | PLANNER_AGENT_PROMPT Guideline 5 | **按 regime 反向裁制**：prompt 明令"勿加独立验证步、勿找页面成功指示（this environment shows none）"（L205/L275，已复验）；诚实性由 --assert-discipline（headline 已开）+ --verify-before-done（r4-B，0 分诚实性组件）承担 | **改造**（口径性差异，非缺失；回吸会重蹈 r2 前验证空转，240s 预算下烧钟） |
| 6 | 持久性/"绝不轻言终止" | prompt Guideline 7 + "Complexities of web navigation" 7 条 | planner 有 MULTIPLE ATTEMPTS 后报告并终止的指令（L268/L285），整体更收敛 | **改造（有意收紧）**（r3 §2 实证：9 个超时格死因是 canvas 盲区+停顿，非过早放弃；无收益证据） |
| 7 | URL 回溯（论文 §3.5） | planner 步骤显式带返回 URL；GO_BACK prompt 存在但 HEAD 未注册技能 | **明确禁止** re-navigation："Do not use re-navigation to the task URL as a retry strategy"（L111，已复验） | **改造/不适用**（MiniWoB 单页任务，回溯=重置任务，r3 专项修掉的 churn） |
| 8 | self-evaluator / self-corrector / Curious Reflector | **仓库 HEAD 与论文 v1 均不存在**（grep 零命中）；实际机制 = 组件 5+4+6 的组合 | 见上各行 | **传说组件，无实体可吸**（未验证：EmergenceAI 博客/其他分支是否有后续版本） |
| 9 | 技能集 | click、entertext、bulk_enter_text、press_key_combination、openurl、geturl、get_dom、pdf_extractor、pause_flow、get_user_input、ADDITIONAL_SKILL_DIRS 动态加载 | `core/tools/` 22 个 + `core/extra_tools/` 7 个（drag_and_drop、dropdown、slider、date-time、upload、geo、captcha、sandbox、clipboard、pdf 等；文件数已复验） | **超越（超集）** |
| 10 | 用户 LTM（user_preferences.txt 注入） | static_ltm.py → `$basic_user_information` | 同源改用：`basic_test_information`（测试数据 LTM；base_nav_agent.py L58 + 各 nav agent prompt 注入，已复验） | **继承改用** |
| 11 | ask-user 技能（get_user_input/pause_flow） | 登录/captcha 交人工 | 无（自动评测下 = 终止报告） | **不适用**（评测口径下不应有） |
| 12 | 编排基建 | AutoGen wrapper；无 token 核算、无压缩、无超时兜底、无重试语义 | LangGraph + context 超限压缩回退（_is_context_limit_error/_compress_messages，L264-292）+ planner timeout 兜底 + cost 汇总 + Portkey 多模型路由（config_portkey_loader.py） | **超越** |
| 13 | 评测 harness | WebArena/WebVoyager 式：JSON 任务 + 字符串判分（exact/fuzzy/must_include/ua_match，test/evaluators.py） | record2gherkin/benchmark：页面原生 reward + own-seed 过滤 + off-seed 信标 + 五类作弊扫描 + 超额归因逐格实证 | **超越（更严）** |

**判定统计**：超越 5 项（#1/3/9/12/13，其中 #1/3 为继承基础上的超越）；纯继承无差距 3 项（#2/4/10）；regime 改造 3 项（#5/6/7）；不适用 1 项（#11）；传说无实体 1 项（#8）。**小结**：组件清单上 Agent-E 没有一项"我方缺了导致丢分"的机制；剩余 48 格的结构（官方失败 ~39 = 完成幻觉存量 + 能力失败，超时 9）在 Agent-E 的机制箱里找不到对应钥匙——它的 WebVoyager 成绩依赖长程多页导航与人工判分的宽容度，两者都不在我方口径内。

### 3.3 传说组件不存在案

- **对象**：Curious Reflector / 独立 self-evaluator / 独立 self-corrector。
- **结论**：Agent-E 论文 v1 全文与仓库 HEAD（f218c3c）**均 grep 零命中**（当时解剖记录：`grep -ri "curious\|reflector"` 零命中；论文全文亦无）。本次（2026-10-10）在本仓库代码侧复验：`grep -ri "curious|reflector"` 仅命中讨论其不存在的文档，**代码零命中**。
- **实际机制**：Agent-E 的"自我纠错"是 planner 自查（#5）+ 变化观察（#4）+ 计划修订（#6）三者的组合，没有独立组件。
- **未验证**：EmergenceAI 博客或其他分支是否在后续版本中加入过此类组件。

### 3.4 我方领先 Agent-E 的 6 条（逐条）

1. **判分完整性**：页面原生 reward、own-seed 过滤、off-seed 信标、五类作弊扫描、超额归因逐格实证——Agent-E 用人工/LLM 字符串判分，无此闭环。
2. **多域编排**：七个 helper（browser/api/sec/sql/mcp/time_keeper/executor；目录已复验）+ target_helper 路由（planner schema 允许值 'browser'|'api'|'sec'|'sql'|'time_keeper'|'agent'|'mcp'|'executor'|'Not_Applicable'，L191）；**Agent-E 只有 browser**。
3. **上下文/token/超时治理**：LangGraph 状态机 + assertion/verify 节点、context 超限压缩回退、planner timeout 兜底、token/cost 全程核算、Portkey 多模型 fallback——Agent-E 的 AutoGen wrapper 均无。
4. **md-extended DOM 覆盖**：r4 `--md-extended` 把"可见但不可寻址"元素（span 假链接、折叠面板头、hitarea、slider-handle、label）纳入终表，+17 格逐格实证——超出 Agent-E all_fields 的交互元素白名单。
5. **工具超集**：`core/tools/` 22 + `core/extra_tools/` 7，含 drag_and_drop/dropdown/slider/date-time/upload/geo/captcha/sandbox/clipboard。
6. **任务表示**：Gherkin 中转（Given=URL/When=指令）是双刃剑（转译损耗未量化，见 qa-bank 补充题——没做过有/无中转对照），但它承载"录制即用例"主目标，属**战略分化而非落后**。

### 3.5 差距侧：唯一硬差距 + 吸收评估

**唯一硬差距 = 截图升级观察**（出处是 HxAgent 的 o^img 策略，**Agent-E 无此物**）：DOM 文本优先，出现 ≥2 个候选歧义或超大 DOM 需摘要时才调截图。我方截图仅作 proof 留档（`get_latest_screenshot_stream()` 存在但不上模型；管线缺口在 `_execute_tool_call` 返回 str，simple_hercules.py L637，已复验）。射向视觉/画布/几何族 **~11 格盲区**（r3 §4.4：count-shape、count-sides、click-shape、circle-center、drag-circle、bisect-angle、find-midpoint、draw-* 等）。

**吸收评估矩阵结论（A~H）**：
- **A. 截图升级观察 → 改造后吸收，先 3-5 格试射**（建议 count-shape、count-sides、circle-center、drag-circle、visual-addition——最后一个是 r2 曾过的"非永久盲区"校准点）。实现走合法路径：`EXTRA_TOOLS_MODULES` 注入 observe_screenshot 工具（config.py L542/680/1016 已复验；visual_skill.py L47 有 get_latest_screenshot_stream 先例），不动 testzeus_hercules 核心；若试点通过，"工具回包可携带 image content"作为 PLAN.md 接线点申报。**前置 go/no-go：GLM 端点 image_url content 支持——未验证，是整条建议的生死门**。口径披露：进 headline 需新 regime 标注"文本-DOM+截图升级"并同步修 baselines 分层；试射 0 增益即关闭归档。
- **B. 跨 episode 经验库（HxAgent 长期经验，其消融 −9.4 点）→ 不擅自吸收，口径决策上交**：它把每类任务从零样本变 few-shot，**直接破坏零样本单次口径——成绩增益属协议增益非能力增益**，可比性掺水。若未来采纳，战略载体天然是 record2gherkin（经验→Gherkin 用例库），与项目主目标同构。
- **C~H 不吸收**：C planner 验证纪律（已以 verify+assert-discipline 更适配形式存在）；D "绝不终止"持久性（超时死因非过早放弃，与 240s 冲突）；E URL 回溯（单页 regime 回溯=自毁）；F get_user_input/pause_flow（评测禁人工，pause 已有 time_keeper，pdf 已在 extra_tools）；G SAM 历史截断（无证据我方全历史+压缩受损）；H enter_text_and_click 复合技能（Agent-E 自己都注释掉了，我方 bulk 工具+submit 约定覆盖）。

HxAgent 另三项对照：迭代规划+逐步 State Evaluator 与我方 planner 循环+verify gate **等价**；短期记忆 SAM 无证据我方受损；SoA→Selenium 脚本生成与我方蒸馏器**战略同构**，非 benchmark 补丁。

### 3.6 基线分层 A/B/C 表（实验前固化于 baselines.md，2026-09-21）

| 档位 | 方法 | 分数 | 设置备注 | 为什么不同口径 |
|---|---|---|---|---|
| **A. 零样本/文本-DOM（最可比）** | GPT-4 agent（"The Unsolved Challenges of LLMs as Generalist Web Agents"，NeurIPS 2023，UCLouvain） | ~47% | 全任务集，零样本文本观察 | 同口径成员 |
| A | BrowserGym/WorkArena 系 GPT-4 基线 | ~50-59% | 依赖观察格式，AST 优于裸 DOM 文本 | 同口径成员 |
| ~~A~~ | ~~Agent-E（125 challenging instances）~~ → **归属更正（2026-09-24）**：81.6% 实为 HxAgent（arXiv 2608.15491），四重口径不可比；**Agent-E 从未发表 MiniWoB++ 数字**（唯一数字 WebVoyager 73.2%，GPT-4-Turbo 人工评判） | ~~81.6%~~ | ~~冲刺锚点~~ **降为背景参照，不作对比目标** | 见 §4 |
| B. 少样本/带演示 | Synapse（trajectory-as-exemplar） | ~99.2% | 64 任务子集+示例 | 带演示，不同 regime |
| B | RCI（Reflection-Critique-Improvement，NeurIPS 2023） | 当时 SOTA | 少量演示 | 带演示 |
| C. 训练模型 | WebGUM（Google，多模态） | 94.2% | 专门训练 | 能力来源不同 |
| 参考 | 人类 | ~91%+ | 原论文 | — |

四口径变量：是否零样本 / 观察模态 / 任务集 / 判分方式。出处粒度为文献级非复现级，报告注明"检索自公开资料，设置细节以原论文为准"。

### 3.7 我方 61.6% 的定位表述（逐字口径）

> "零样本文本-DOM 全 125 任务类单次口径下，**无同口径公开 SOTA 可比**；A 档带（47-60%）**已越上沿**；81.6%（HxAgent）与 73.2%（Agent-E/WebVoyager）均不同口径，仅作背景。"

完整口径六要素（qa-bank Q13）：全量 125 任务类（browsergym registry）、零样本、只读文本-DOM（md 蒸馏，≈a11y 档）、GLM 全角色（planner glm-5.3 / nav glm-5.3-flash）、240s/格（原版 10s，放宽已披露）、单 seed 单次、判分=页面原生 `WOB_RAW_REWARD_GLOBAL > 0`；官方=干净口径、零无效格。四轮线 43.2% → 46.4%（clean 47.8%）→ 47.2% → **61.6%（77/125）**。**禁语**：不说"我们是 SOTA"，说"越出公开可比带上沿"；B/C 档（94-99%）不是公平对照，报告只作背景。

## 4. 锚点更正全案（81.6%）

### 4.1 时间线

| 阶段 | 时间 | 事件 |
|---|---|---|
| 旧叙事形成 | 2026-09-21 | baselines.md 初版（公开检索）把"Agent-E（125 challenging instances）81.6%"列为冲刺锚点——**当时未做一手核验，检索来源的二手归属即错** |
| 旧叙事固化 | r1~r4 期间 | round4-report.md §1 写下"距同源架构锚点 Agent-E（81.6%）差 20 个百分点" |
| 疑点 | 2026-09-22 | 为准备对比材料克隆 Agent-E 仓库逐字核对论文——发现**全篇没有 MiniWoB++ 字样** |
| 核实 | 2026-09-22~24 | ① 确认 Agent-E 唯一成绩是 WebVoyager 73.2%；② 反向追查 81.6%，定位真实出处 HxAgent（arXiv 2608.15491）Table 4；③ 逐项摘录其口径，确认四重不可比 |
| 更正动作 | 2026-09-24 | 三件（见 4.4） |
| 固化 | 2026-09-24 起 | 权威更正 = baselines.md 更正行 + agent-e-anatomy.md §1；历史文档不改写 |

### 4.2 核实结论一：Agent-E 真实成绩（arXiv 2407.13032 v1 §3）

- 基准：**仅 WebVoyager**（He et al. 2024），15 个真实网站 643 任务；MiniWoB++ **全篇未出现**。人工把静态日期任务平移 8 个月保可做。
- 观察：纯文本 DOM（"text-only web agent"；三档 content type：text_only / input_fields / all_fields）。模型：**GPT-4-Turbo**（planner 与 browser nav 同款），非 GPT-4o。
- 判分：5 名人工标注员（IST 办公时间），"完整完成才算过"；ask-user 技能禁用。尝试次数/限时论文未报告；实测均值成功 150s / 失败 220s，平均 25 次 LLM 调用/任务。
- 成绩：**总 73.2%**（Table 3 为 73.1）；站点间 27.3%（Booking）~95.7%（WolframAlpha）；对 text SOTA +21%、多模态 SOTA +16%。**无消融表**。
- → 与我方 MiniWoB++ 口径**没有交集**；"距 Agent-E 20 个百分点"的叙事失去事实基础。

### 4.3 核实结论二：81.6% 真身 = HxAgent（arXiv 2608.15491，Table 4），四重口径不可比

1. **任务集偏科**：5 个任务 × 25 实例 = 125（choose-date、book-flight、flight.AA、flight.Alaska、flight.Alaska-auto）——恰是其经验规则最密的 flight/日历系；我方是全量 125 任务类。同表基线 Li et al. 9.6、SeeAct 8.0、**WALT 0.0**——差 70+ 点的"基线"说明子集高度偏科，不是全任务能力的度量。
2. **观察模态**：GPT-4o 多模态三元组（DOM 文本, 截图, 可交互元素），文本先行、≥2 候选歧义才调截图；temperature=0；我方是零样本文本-DOM。
3. **训练+经验注入**：每族 20 训练实例 + 最多 8 条 few-shot 经验规则（完整集 39 任务 × 25 实例 = 975 实例上 97.4% EM / 99% PM；消融：去经验 88%、去迭代规划 75%、去 SAM+经验 47%）；我方零样本。训练-评测分离 = 同任务族的变相多 attempt。
4. **判分方式**：**动作序列金标 Exact-Match**——生成的 (op, element, value) 序列与金标全对才算过；是序列对齐不是页面终态 reward，与我方 `WOB_RAW_REWARD_GLOBAL > 0` **不可互译**（序列全对≠终态成功，反之亦然）。步数上限 MAX_t 值未披露、无时间限制表述（"每任务 50 实例"为 anatomy 原文记录；与 25 实例/族的关系未核实）。
5. **方差佐证**：81.6% 的 **±34.6** 波动本身说明该数字不适合当稳定锚点。
- 附注：HxAgent 的 OpenReview 页引用 arXiv:2508.10833，与 2608.15491 不一致——**未验证哪个是正式号**；GPT-4o 快照版本未核实。

### 4.4 更正动作（三件，"历史文档不改写"原则）

1. **baselines.md**：A 档该行划掉（删除线保留原文），注明真实归属 HxAgent（arXiv 2608.15491）与四重口径差，"降为背景参照，不作对比目标"；解读第 2 条同步更正；来源清单加 HxAgent 条目。
2. **round4-report.md §1**：原文保留，追加更正标注（"更正 2026-09-24：81.6% 经核实为 HxAgent……本行叙事废弃"）；§4-1 同步加注。
3. **叙事废弃**："距 Agent-E 20 个百分点"全面停用；权威更正固化在 baselines.md 更正行 + agent-e-anatomy.md §1。qa-bank Q32 设了防复发条款："严禁再引用距 Agent-E 20 个百分点旧叙事"。

### 4.5 教训（两条，均有制度性产出）

1. **引用外部数字必须追一手出处**（克隆仓库/核对原文）——这条纪律本身就是这次翻车的产出；此前"检索自公开资料"的免责声明不能替代核验。
2. **数字可比先对齐四要素**（零样本与否/观察模态/任务集/判分方式）——四要素任一不同即不可比，只能作背景。

## 5. 面试深挖要点

| 追问 | 一句话答 | 指向 |
|---|---|---|
| "和 Agent-E 什么关系？为什么不用它？" | 它是我的执行内核的直系祖先——文件级同源可 diff（dom_helper/dom_mutation_observer 等），我就在这条技术线上，做的是它没有的录制/蒸馏/归因/评测；13 项组件对照后它没有我缺的机制。 | §3.1/3.2 |
| "听说你们距 Agent-E 差 20 个点？" | 那是我自己已更正的错误——81.6% 从来不是 Agent-E 的数字，它从未发表过 MiniWoB 成绩，该叙事已废弃。 | §4.1/4.4 |
| "那 81.6% 到底是谁的？" | HxAgent 的 Table 4，四重口径不可比：5 任务族偏科子集、GPT-4o 多模态、20 训练实例+8 条经验注入、动作序列金标判分，方差 ±34.6。 | §4.3 |
| "你说'领先'的依据？你们是 SOTA？" | 不是 SOTA——准确说法是"零样本文本-DOM 全任务口径下越出公开可比带（47-60%）上沿"，regime 内无同口径公开可比数字。 | §3.6/3.7 |
| "Agent-E 不是有 self-evaluator/Curious Reflector 吗？" | 传说组件：论文 v1 和仓库 HEAD 双双 grep 零命中，实际是 planner 自查+变化观察+计划修订的组合，无实体可吸。 | §3.3 |
| "你们还差它什么？" | 唯一硬差距是 DOM 失明时升级看截图（还是 HxAgent 的策略，Agent-E 无此物），射向 ~11 格视觉/画布盲区，按"试射 3-5 格→GLM 视觉裁决"两步走，GLM 视觉支持未验证是 go/no-go。 | §3.5 |
| "为什么不抄它的经验库涨分？" | 跨 episode 经验库破坏零样本单次口径——那是协议增益不是能力增益，可比性掺水；口径决策上交，战略载体天然是 record2gherkin。 | §3.5 |
| "分层表什么时候定的？不会是出了分再找的吧？" | 实验前（2026-09-21）固化在 baselines.md，按四个口径变量切 A/B/C 档；A 档成员出处是文献级粒度，已注明。 | §2.3/3.6 |

## 6. 数字与出处对照表

| 数字/事实 | 值 | 出处 | 核验状态（2026-10-10） |
|---|---|---|---|
| 我方 headline | 61.6% = 77/125；四轮 43.2→46.4（clean 47.8）→47.2→61.6 | round4-report §1；qa-bank Q13 | 一手报告 |
| 我方口径 | 240s/格（原版 10s）、单 seed 单次、WOB_RAW_REWARD_GLOBAL > 0、browsergym 全量 125 类、planner glm-5.3 / nav glm-5.3-flash | baselines.md 口径节；qa-bank Q13 | 一手报告 |
| A 档带 | 47-60%（UCLouvain ~47%；BrowserGym/WorkArena 系 50-59%） | baselines.md A 档 | 文献级，公开检索 |
| B/C 档 | Synapse 99.2%（64 任务子集+示例）、RCI、WebGUM 94.2%、人类 ~91%+ | baselines.md B/C 档 | 文献级 |
| Agent-E 论文 | arXiv 2407.13032；WebVoyager 73.2%（Table 3 为 73.1）；643 任务/15 网站；GPT-4-Turbo；5 名标注员人工评判；日期平移 8 个月；实测 150s 成功/220s 失败、25 次 LLM 调用/任务；站点 27.3%~95.7%；对 text SOTA +21%/多模态 +16%；无消融 | agent-e-anatomy §1.1 | 固化解剖记录（论文 HTML v1 全文精读；本次未重读原文） |
| Agent-E 仓库 | HEAD f218c3c（2025-05-12）；PlannerAgent 50 轮 / BrowserNav 10 轮/步；Curious Reflector / self-evaluator grep 零命中 | agent-e-anatomy 头注/§2 | 固化记录；克隆已随 /tmp 清理，本次未重克隆 |
| HxAgent | arXiv 2608.15491（OpenReview 引 2508.10833，不一致——未验证）；81.6% EM ±34.6（Table 4）；5 任务×25 实例（choose-date/book-flight/flight.AA/flight.Alaska/flight.Alaska-auto）；基线 Li et al. 9.6 / SeeAct 8.0 / WALT 0.0；975 实例 97.4% EM / 99% PM；20 训练实例+最多 8 条经验；消融 88%/75%/47%；GPT-4o temperature=0；动作序列金标 Exact-Match；MAX_t 未披露 | agent-e-anatomy §1.2/§2b | 固化解剖记录（论文 HTML v1 全文精读；快照版本未核实） |
| md-extended 收益 | +17 格（24 新通过格中 17 格有点击日志实证；email 4/4 + find-greatest + 12 格"可见但不可寻址"元素） | round4-report §2 | 一手报告 |
| verify gate | 0 分（诚实性组件）；103 轮全观察性 | round4-report §2 | 一手报告 |
| 超时与方差 | 超时 21→9；方差 +7 格（±10 格噪声带/±8%；16.8% 翻转/12.8% 纯噪声/26-34% 边缘格） | round4-report §1/§2；R3F §3.2 | 一手报告 |
| 视觉盲区 | ~11 格（count-shape、count-sides、click-shape、circle-center、drag-circle、bisect-angle、find-midpoint、draw-* 等）；试射建议 5 格含 visual-addition（r2 曾过的"非永久盲区"校准点） | agent-e-anatomy §3-A/§4；R3F §4.4 | 一手报告+固化记录 |
| 代码锚点 | planner 500 轮（simple_hercules.py L103）、nav 10 轮（L104）、_STATE_REFRESH_MARKERS（L528）、_compress_messages（L264-292）、_execute_tool_call→str（L637）、click 回包文案（click_using_selector.py L129）、md+aria-keyshortcuts 注入（get_detailed_accessibility_tree.py L50-88）、basic_test_information（base_nav_agent.py L58）、target_helper 允许值（high_level_planner_agent.py L191）、禁止重导航（L111）、禁验证步/禁成功指示（L205/L275）、EXPLICIT CLOSURE NUDGES（L144/L185）、EXTRA_TOOLS_MODULES（config.py L542/680/1016）、get_latest_screenshot_stream（playwright_manager.py L1135） | 本仓库 | **已现场复验（grep/读行号）** |
| 工具计数 | core/tools/ 22 个 + core/extra_tools/ 7 个（文件级；22 含 tool_registry.py） | agent-e-anatomy §2-9 | 已现场复验（ls 计数） |
| 未验证清单 | GLM 图像输入支持；EmergenceAI 博客/分支是否发过 MiniWoB 数字；HxAgent MAX_t 值与 GPT-4o 快照；HxAgent 正式 arXiv 号；Agent-E 侧 grep 的本次现场重验（依据固化记录） | agent-e-anatomy §6 | 未验证（逐项标注） |
