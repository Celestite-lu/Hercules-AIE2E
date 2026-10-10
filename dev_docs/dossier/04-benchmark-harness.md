# 04 — MiniWoB++ Benchmark 基建与防作弊体系全档案

> 覆盖：`record2gherkin/benchmark/`（harness 全套）+ `testzeus_hercules/` 引擎侧 flag 化补丁 + `dev_docs/benchmark/` 全部 25 份文档（plan/spec/review/test-report × r1-r4、八份安全审查、四份 round-report、三份 analysis、baselines）。
> 数字来源标注：一手报告 = round1/2/3/4-report；复审 = security-review-r{N}-post；其余为 plan/spec/test-report。两处不一致时以一手报告+复审为准（本档案已逐条对照）。

## 1. 这一页讲什么（3 句）

1. 这套基建把 Farama-Foundation MiniWoB++（browsergym registry 全量 **125 个任务类**）变成"本地伺服 + 补丁供题 + 页面原生奖励判分"的可复现基准：vendored 360 个 HTML 文件字节原样伺服，`core.js` 纯追加两段补丁（`?r2g_seed=` 自动开局 + `endEpisode` 同步 POST 奖励），官方成败以 `WOB_RAW_REWARD_GLOBAL > 0` 为唯一权威、**无 LLM 裁判**。
2. 围绕"分数诚实"建了五层防作弊体系：判分权威封闭（HUD/重开按钮加固）、cell 收尾五类扫描（只披露不改判）、实验前后双安全审查（与执行者分离）、口径防灌水（invalid 剔除进 `clean` 口径、480s/多 seed 只进 ablation 臂）、原则性极限披露（奖励端点无防伪造能力）。
3. 四轮迭代（43.2% → 46.4%（clean 47.8%）→ 47.2% → **61.6%**）每一格都经逐格归因与安全审计背书：r2 修掉 r1 的 5 格判分瑕疵（54→59 回放锁定）、W2 文件工具事件按预登记契约判 10 格无效、r3 堵掉 4 次 scheme 越界与 W2 结构性根因、r4 超额的 +17 格全部有 md 点击日志实证。

## 2. 设计决策总表

| # | 决策 | 为什么 | 备选与否决理由 | 出处 |
|---|---|---|---|---|
| D1 | 选 **MiniWoB++** 作公开基准 | Farama 官方 pip 包自带全部 HTML，本地伺服即可；页面原生奖励免裁判 | **WebArena**（本机无 docker、需 5 个自托管站点）、**WebVoyager**（真实外网站点 + GPT-4V 裁判依赖）、**Mind2Web**（只能测静态子能力）均淘汰 | plan.md §1 |
| D2 | registry **全量 125 任务类不挑选**，visual 族不豁免 | 挑任务=口径灌水；125 是 browsergym registry 注册数，任务表从 `ALL_MINIWOB_TASKS` 构建且 125/唯一/字典序为构建期硬校验 | 豁免 visual（确定全挂拉低总分）被明确拒绝："visual 失败就是失败，报告分家族如实呈现" | plan.md §1/§6、spec.md §2、build_task_table.py |
| D3 | **vendored 供题**：整树复制 360 文件 + PROVENANCE（sha256 漂移检测） | 确定性 + 许可合规（BSD-3-Clause，`miniwob 1.1.0`，4,988,677 字节）；构建期一次性，`miniwob`/`browsergym-miniwob` 不进 pyproject | 在线拉取/CDN 依赖被否（离线可复现红线）；脚本零内容改写（补丁在伺服期做） | spec.md §1、test-report.md §2、PROVENANCE.md |
| D4 | **补丁式伺服**：`GET /core/core.js` = vendored 原文 + **纯追加**补丁，vendored 字节从不改写 | Hercules 起自己的浏览器跑完即关，奖励状态随浏览器消失；纯追加避免锚点漂移，B 组单测逐字节断言前缀一致 | 文件内替换（锚点漂移风险）被否；页面侧改源码违反 vendored 不改写 | spec.md §3.2、miniwob_server.py `patch_core_js` |
| D5 | seed 派生 `int(sha1(f"{exp_id}:miniwob:{task_id}")[:8], 16)` | 同格恒等可复现；seed 只进 URL query，对 agent 是无意义数 | 随机 seed 被否（破坏复现）；exp_id 变更即换 seed——480s 臂因此复用同 exp_id 换 `--exp-root` 保逐格配对 | spec.md §2.3、tasks.py `derive_seed`、analysis-r1-architecture H7 |
| D6 | goal 预读：harness 用同 seed 打开同 URL 读 `core.getUtterance()` **烤进 Gherkin**（Given=URL / When=指令原文 / Then=意图陈述） | 测的是"执行内核能力"（NL→Gherkin→语义执行）；同 seed 确定性 ⇒ 预读=执行侧指令；预读页不触发 endEpisode、零 reward 记录 | 直接把 goal 塞环境变量绕过 Gherkin 被否（Gherkin 中转是口径之一，须披露）；任务特定提示（日期示例值等）违反判分中立被否 | spec.md §4/§5、goal_reader.py、plan.md §1 口径 1 |
| D7 | 官方成败 = 页面原生 `WOB_RAW_REWARD_GLOBAL > 0` **唯一权威，无 LLM 裁判**；JUnit/planner 自判不一致只记 disagreement 不改判 | 免裁判依赖（WebVoyager 需 GPT-4V 裁判被拒的根因）；BrowserGym 同口径 | LLM 裁判被否（引入主观性与成本）；以 JUnit 为准被否（页面才是任务真值） | spec.md §0 口径 4、plan.md §1 |
| D8 | **240s 页面计时**（原生 10s，24×放宽，参数化 `r2g_ms`）+ **单 seed 单次** + 引擎 600s | 10s 对"每步 10-40s LLM 延迟"的 agent 不可用；沿用 BrowserGym `episode_max_time` 可配做法；三条全部随报告原样披露 | 480s 进 headline 被拒（口径变更永久污染跨轮可比性，只进 ablation 臂）；多 seed 均值被拒（Out of Scope，majority-of-3 待批） | spec.md §0 口径 2/3、plan-r3 R3-4、plan-r4 R4-D |
| D9 | 奖励 POST 用**同步 XHR**；服务端内存 `(path,seed)→record` last-wins + rewards.jsonl 逐行 flush | 页面销毁前必须送达；`/latest` 404=`no_reward` 如实记失败，**无任何"重开同 seed 页"兜底**（`startEpisodeReal` 会重置状态，读到的不是终局值） | 异步 POST 被否（页面销毁丢记录）；重开兜底被否（机制上不可行） | spec.md §3.2/§7.2.3、miniwob_server.py `RewardCollector` |
| D10 | **H1 判分权威封闭**：开局成功后 `#reward-display` 置 `display:none`、stub `core.updateDisplay`/`core.startEpisode` | V1：原生 HUD（`Last reward: 1.00`）落在 `get_page_text` 的 innerText 视角；V2：endEpisode 尾部（core.js:145）重显 START 覆盖层，一次普通 DOM 点击即重开同 (task,seed) 新实例 + 240s 重置（re-roll） | `element.remove()` 被否——endEpisode 仍要写 `#episode-id`，移除会抛 TypeError；加固只封观察/点击路径、不封 JS 执行能力（如实披露） | security-review-r1-pre §2.V1/V2、spec.md §0 口径 5、miniwob_server.py:116-119 |
| D11 | **H2/H3 收尾扫描**：对已落盘 stdout.log/rewards.jsonl 做五类检查，结论只写行内三键（`task_url_navigations`/`flagged`/`invalid_reason`），**绝不改判** | 扫描无网络无浏览器、可独立复算（复审对 130/136/137/131 行重放全部 0 mismatch）；分级语义：flagged=披露但保留、invalid_reason=cell 无效 | 扫描结果改判分被否（口径 4/5：三键不进任何分母与判定） | spec.md §7.6、test-report §11、orchestrator.py `CellScan` |
| D12 | **双安全审查流程**：每轮实验前（pre）+ 实验后（post）各一份独立审查，与实验执行者分离，pre 等级（GREEN/YELLOW/RED）决定能否开跑 | r1-pre YELLOW 拦下 V1-V6 再跑 full；r2-pre W1（glm planner 未探测）+W12（runbook 命令陷阱）在开跑前修掉；post 全量重放扫描器与三口径复算 | 单次审查被否（r2 W2 事件证明 pre 预警可成真，post 必须逐格取证） | 八份 security-review-r*-{pre,post}.md |
| D13 | r2 **C1a 判分修复**：`raw>0` 压倒 infra 状态（timeout/no_junit），附 `runner_status` 审计键与 r1 数据回放锁 | r1 有 5 格"页面已过被引擎判负"（+4.0pp 锁定值 54→59）；修复=把口径 4 执行对，是正确性修复非放宽 | 保持旧行为被否（"保留旧行为无价值"）；`fetch_reward` 仍取 /latest 不新增洗白通道；反例 email-inbox-delete 永久锁死"维持非 official_passed" | plan-r2 C1a、spec-r2 §1、test-report §12.2 |
| D14 | r2 **C2 终局信号**：终局后注入恒定中性文本 `EPISODE ENDED`（无数值无成败、不 POST） | r1 自造盲区：HUD 被我们加固隐藏后 agent 找不存在的成功指示必然空转（multi-layouts 页面过后 46/64 轮空转；23 格 ×均值 273s zombie） | 原版裸 HUD 数值被否（V1 泄漏）；stop-at-terminal 引擎改造被否（与 C2 重叠且是口径变更 last-wins→first-terminal-wins） | plan-r2 C2 与"不采纳"表、spec-r2 §4.3 |
| D15 | r2 **C3 单次开局**：`sessionStorage["r2g_started"]` 每 tab 只自动开局一次；**拦截态同样应用 V2 加固**（stub+CSS） | r1 61/125 格 nav≥2，重导航=免费重置 240s+作废失败提交（email-inbox-forward 16 次导航把页钟续到 +584s）；r1 实证 93% 重导航走同 tab（249/267） | 不治理被否；只封不加固拦截态被 review-r2 M2 否决——拦截态 START 可点会重开无 seed 随机实例并经 last-wins 洗白 | plan-r2 C3、review-r2 M2、spec-r2 §4.2 |
| D16 | r2 **C4 延迟包**（engine，独立 flag+commit）：状态刷新守卫 `markers_only`、轮数 50→30、步级预算 120s、prompt 包（撤销"刷新教学"、planner 瘦身、输出克制）、`LLM_REQUEST_TIMEOUT=90`+`LLM_MAX_RETRIES=2` | r1 实证：超时格均值 51.3 executor 轮、轮间均值 9.0s、断裂均值 25.1 次/格、completion p90 1998 tokens；引擎 prompt 在主动教重导航（browser_nav_agent L55/L116） | 视觉模型接入、planner 模型升级被拒（planner 仅 3.2 轮/格非瓶颈，无证据） | plan-r2 C4、analysis-r1-architecture §0/H3 |
| D17 | r2 **C5 分角色路由**：planner/helper=旗舰、nav=flash 档；key 只走 env（四键注入含 planner 专用的 `OPENAI_API_KEY`） | executor 轮是延迟主体（9.0s×51 轮）；planner 每格仅 3-4 轮 | planner 升级被拒；key 落配置文件被拒（红线：生成文件永不含 `model_api_key`/`sk-`，写盘函数对含 key 配置直接抛错） | plan-r2 C5、spec-r2 §6、review-r2 M1 |
| D18 | r2 **C6 拖拽工具**：`--extra-tools` 开闸 + `--smoke-cells` 2 格冒烟先行，kill-switch 判据=选择器解析成功 | 拖拽家族 14 格全军覆没、agent 只能按键冒充拖拽；先验证"被调用、不崩、无异常面"再算数 | 直接计入预期被否；坐标拖拽（canvas/SVG 无 DOM 锚点）列射程外 | plan-r2 C6、review-r2 M3 |
| D19 | r2 **C7 模板上下文**：固定 4 行注释（指令同文在 #query、单页勿重载、页面无成功提示）+ cue 联动第 5 行（C2 off 时不得出现） | 25 格页面 240s 死亡 + 58/121 格首轮 plan 含"verify the task completed"空转步 | 任务特定提示被否（泄漏任务语义）；When 指令原文逐字不动（红线） | plan-r2 C7、spec-r2 §7.2、goal_reader.py |
| D20 | r2 运维三件套：**C1b 熔断器**（infra 状态+<30s+402/连接标记，连续 2 格熔断 exit 2）、**C1c 1-token 预检**（exit 3 硬 gate）、**C1d attempt 分目录日志** | r1 有 3 行 4-5s 的 402 no_junit 重试白烧预算；key 余额不足会第一格就烧超时；5 个重试格 attempt1 日志被覆盖不可复核 | litellm 探测被 review-r2 M4 否决（裸模型名恒败且不构成引擎路径证明）→ ChatOpenAI 同栈 | plan-r2 C1、spec-r2 §2/§3 |
| D21 | r3 **R3-5 安全必办包 E1-E6**（判分中立却 headline 默认 on） | E1 子集加载结构性消灭 W2 类事件（分母 115→125 回归）；E2 调用日志行封死"静默成功"缺口；E3 scheme 白名单（连续两轮 P1 未做）；E4 clean 口径自动化（invalid 移出分子分母，overall 不动）；E5 沙箱关停（两轮合法使用为零、越界尝试 1 格）；E6 极早崩留痕 | 继续全量 extra_tools 被否；off-seed 自动补齐/阻断型防护被否（掩盖 planner 缺陷且需逐格传 seed）——只做检测披露 | plan-r3 R3-5、security-review-r2-post §9 |
| D22 | r3 **R3-1 nav max_tokens 768**（engine flag）；r4 摘除 | r2 实测肥尾 2000-3400 token 吃墙钟（10.3% turn 吃 38% executor 墙钟）；注入点 `create_chat_model` 兜底段后、env>0 无条件覆盖 | prompt 纪律再加码被拒（C4d 已实证对 GLM-flash 无效）；planner 截断被拒（单块 JSON 断裂→空转循环，风险不对称）；**r3-post 实证 no-op 后 r4 headline 摘除**（`--nav-max-tokens` CLI 保留不删） | plan-r3 R3-1、security-review-r3-post §5、plan-r4 §0.4 |
| D23 | r3 **R3-2 planner 双层 150s 超时**（`LLM_PLANNER_REQUEST_TIMEOUT` 同源喂 wait_for 与 provider 层） | 25/41 超时格发生 ≥1 次 planner 90s 超时；90s 是双层硬墙（外层 `asyncio.wait_for`+内层 ChatOpenAI timeout 兜底），慢生成的单次请求结构上不可能 >90s 完成 | 全 agent 超时重试被拒（temperature=0 重发同样慢，只把最坏暂停翻倍吃掉页钟）；只改一层被 review-r3 M2 否决 | plan-r3 R3-2、review-r3 M2、spec-r3 §2 |
| D24 | r3 **R3-3 drag 选择器透传**（md 包装 bug 修复 + text= 兜底 + kill-switch） | 61 调 0 中全因 `if "md=" not in selector` 把一切 source 包进 `[md='…']`，而拖拽源根本不在 md 表里；拖拽鼠标序列机制本身可行 | 放弃案（移除工具）备选但= 8 格天花板永久归零；坐标拖拽不做（射程外）；实绩 0/9→3/9、kill-switch 达成（9/9 格解析成功） | plan-r3 R3-3、security-review-r3-post §7 |
| D25 | r3 **R3-4 / r4 R4-F：480s 只进 ablation 臂**，数字永带 `episode_max_time_ms=480000` 标注双列呈现，禁止合并分母 | 240s→480s 改变任务难度本身，进 headline 会永久污染 r1/r2/r3 同口径可比性；seed 派生不变可逐格配对归因；r3 分析把 480s 预期校准为 +1±1 后 r4 进一步降级为默认不做 | 进 headline 被否（"50% 主张将被迫携带口径星号"）；best-of-3 被拒（口径掺水） | plan-r3 R3-4、plan-r4 R4-F/R4-D |
| D26 | r4 **R4-A `--md-extended`**：终表过滤器扩展（EXTENDED_TAGS div/span/li/tr/td/th/img/label + EXTENDED_ROLES row/cell/listitem/img + identity 门控 + class 锚 + 150 条硬上限） | email 家族 10 格两轮 0/20、死法唯一（元素注入了 md 但被 flatten 白名单整批丢弃→"Interactive elements list is empty"）；注入层 cursor:pointer 启发式已被实测覆盖（.email-thread×7/.star×7/.trash×7 全带 md） | click 工具选择器透传被拒（爆炸半径=所有任务每次点击，列 r5 候选）；`isInteractiveElement` 大改被拒（只授权最小修补且以 T4 失败为前提，实际未触发） | plan-r4 R4-A、spec-r4 §1、review-r4 V2/V3/V6 |
| D27 | r4 **R4-B `--verify-before-done`**：planner 产出 terminate=yes∧is_passed=true 且未核验过 → 路由到 verify 节点（固定核验步 → 既有 executor 路径 → 回灌 planner 二次裁决），`verify_rounds<1` 硬上限 | 完成幻觉 24 格（junit=true 页面 −1，占官方失败 57%）+ NR 3 格同病根"TERMINATE 无页面证据校验"；**prompt 层已被 r3 实证不足**（断言纪律上线后假阳性 17→24 不降反升） | 读 reward 全局作判据被拒（`_VERIFY_STEP` 显式禁读；结构面也封死：JS 全局不落 DOM+无 JS-eval 工具）；机制化强校验留 r4 即本轮、r5 改定位为诚实性组件（实绩 0 分贡献） | plan-r4 R4-B、spec-r4 §2、security-review-r4-pre §一/R4-B、r4-post §7 |
| D28 | r4 **R4-E 完整性信标**（off-seed 探测 + epstart 启动信标，恒开、判分中立、不带 flag） | r3 暴露 1 格 off-seed（planner 剥 `r2g_seed`）+ 6 格零 reward 事件无 verdict；两种零事件形态（"从未启动"vs"已启动无奖励"）需 epstart 细分；端点恒 204 零回读、flagged-only | 阻断/自动补齐被否（掩盖缺陷）；seed 值校验缺失成为 r5 备忘（wrong-seed 盲区） | plan-r4 R4-E、spec-r4 §3、r4-post §6 |
| D29 | **clean 口径自动化**（E4）：invalid 格同时移出分子与分母；`overall`（官方口径）零改动 | "干净口径"由代码产出而非人工剔除（r2 是磁盘取证人工剔除）；`_rate` 把 invalid 格计入分子属流程风险 | 只人工剔除被否；r3-pre F1 补一行：invalid_reason 行不进重试池（防"重试洗干净"弱化 clean） | spec-r3 §5.4、security-review-r3-pre F1、metrics.py `clean_rate` |
| D30 | 预算护栏：`R2_BUDGET_CAP=144` 硬拦 + manifest `cumulative_hercules_used` 累计入账 + Σ>144 量化披露（纪律条款不阻断） | r1 红线 142 被 r2 的 smoke 机制细化（14+130=144 贴线）；review-r4 MF-3 指出 cap 是单次调用口径、跨调用无工具总账 | 拆 exp-id 分窗绕账被纪律条款明令禁止；结构性最坏 146=16+130 已量化披露（+2 全为 pilot 域重试，≤1.4%） | spec-r2 §0.2、plan-r4 §5、orchestrator.py `_write_manifest` |
| D31 | 配速与断点续跑：GLM coding plan 5h 窗口 → `--max-cells` 分段 + 同命令重跑自动跳过已有行 + 429 熔断标记 | r2 起模型换 GLM 按量计费且限流；results.jsonl 逐行 append+flush、manifest 收尾写 | 并发/分布式编排被否（Out of Scope）；换 provider 前必须过 C1c 预检 | plan-r3 §3/§4、spec-r3 §0、test-report §12/§14/§15 |
| D32 | 每轮预期数字**逐格可推演**（格数×概率来源行号），保守档必须可复算 | review-r3 M3（保守档 −2 格不 reconcile）与 review-r4 MF-2（把 0.7 交互率当通过率凑出 +8）两次拦下不可信预期 | 区间拍脑袋被否——"每个数字诚实可披露"是最高原则 | review-r3 M3、review-r4 MF-2、plan-r4 §6 构成表 |

## 3. 关键实现细节

### 3.1 判分链路图（四轮未变的权威链）

```
vendored 页面（本地 127.0.0.1:8462）
  │ 补丁 A：?r2g_seed= → 等 WOB_TASK_READY && readyState==="complete" && core.cover_div
  │         → Math.seedrandom(seed); EPISODE_MAX_TIME=r2g_ms(默认240000); core.startEpisodeReal()
  │           → H1 加固（HUD display:none + stub updateDisplay/startEpisode）
  │ 任务终局：core.endEpisode(reward, done, reason)（vendored 原文）
  │   ├ EP_TIMER 守卫：终局后重调用 early-return —— WOB_RAW_REWARD_GLOBAL 从此**封存**
  │ └ 补丁 B（REWARD_HOOK）：wrapper 先 orig.apply 再**同步 XHR** POST
  │     {path, seed, reward:<显示参数>, raw:WOB_RAW_REWARD_GLOBAL, done, reason, ts}
  │       → POST /__r2g_reward
  ▼
miniwob_server RewardCollector：内存 (path,seed)→record last-wins + rewards.jsonl append（每行 flush）
  ▼
orchestrator fetch_reward → GET /__r2g_reward/latest?task=<subdomain>&seed=<n>
  │   匹配规则 = path 末段 <subdomain>.html + seed 相等 ⇒ **own-seed 过滤**
  │   （r4 实证：planner 写错 seed 的页面即使做到 raw=1 也不被取回——click-dialog-2 att1）
  ▼
build_result_row 优先级链（C1a，spec-r2 §1.1，逐字）：
  runner_status is None                    → no_goal          （未执行，最高优先）
  raw is not None and raw > 0              → official_passed  （页面正奖励压倒 infra 状态）
  runner_status in (timeout, no_junit)     → 原样保留          （仍可进重试池）
  reward is None                           → no_reward        （disagreement=None）
  else                                     → official_failed  （raw<=0 含页面 timed out raw=-1）
  ▼
results.jsonl（追加式）→ metrics.summarize → manifest.metrics
```

**raw 封存与终局后复触发不翻案（r4 判分链机制新知）**：官方判定读记录的 **`raw` 字段**（`WOB_RAW_REWARD_GLOBAL`），不是 `reward` 字段（endEpisode 的显示参数）。REWARD_HOOK 对被 EP_TIMER 守卫短路的重调用也会 fire——终局后 agent 再点击会 POST 出 `reward=<新参数>` 而 `raw=<封存原值>`。r4 实证 4 条此类记录（book-flight 1、click-menu 1、click-color 多条同值）：rewards.jsonl 中"终局后值翻转"只发生在显示字段，`raw` 一经终局即不可变——**判分在两个方向上都抗终局后翻转**（security-review-r4-post §4；人工审计必读 `raw` 而非 `reward`，否则会把 book-flight/click-menu 误判洗白）。

**无 LLM 裁判的理由**：页面终态是任务真值，`raw>0` 机械可判；引入 GPT-4V 裁判（WebVoyager 路线）会叠加主观性与外部依赖，与"可复现、口径诚实"直接冲突（plan.md §1 选型存档）。

### 3.2 补丁层五段（追加顺序：原文 → A(single) → REWARD_HOOK → TERMINAL_CUE → INTEGRITY_BEACON）

| 补丁 | flag | 内容与行为要点 |
|---|---|---|
| AUTO_START_PATCH（补丁 A） | 恒在 | 50ms 轮询，条件 `WOB_TASK_READY===true && document.readyState==="complete" && core.cover_div`（三条件缺一不可——只等 WOB_TASK_READY 不够，core.js:49 默认即 true，会在重页面抢在 onload 前开局导致 book-flight 抛 `ReferenceError: ui_utils is not defined`，review 必改 1.4）；开局成功才 `clearInterval`，失败重试至 200 次（~10s）后置 `__R2G_START_ERROR`；无 `r2g_seed` 的请求不自动开局（不干扰人工调试） |
| AUTO_START_PATCH_SINGLE | `--single-start`（r2 C3，默认 off） | 由补丁 A 程序化派生、恰三处不同（标记行/拦截 guard/成功登记，T6d 字节断言）：拦截态同样 stub 三函数 + CSS 隐藏 HUD 与 START 覆盖层（M2 加固）；被拦截加载不产生 reward 记录 |
| REWARD_HOOK_PATCH（补丁 B） | 恒在 | 同步 XHR POST（注释原话"确保页面销毁前送达"）；try 全包"绝不干扰 episode 本身"；成功口径=`raw>0`（BrowserGym 同口径） |
| TERMINAL_CUE_PATCH | `--terminal-cue`（r2 C2，默认 off） | endEpisode 后创建 `#r2g-terminal-cue`（fixed 左下角、z-index 2147483647），文本恒 `EPISODE ENDED`——禁止含 raw/reward/done/成败词（T5 断言 endEpisode(1) 与 (-1) 后文本一致）；不用 display:none（innerText 必须可见）；不 POST |
| INTEGRITY_BEACON_PATCH | `--offseed-beacon`（r4 E1/E2；orchestrator 恒传 True） | 页面 load 期一次 fetch POST：query 无 `r2g_seed` → `/__r2g_offseed`；有 → `/__r2g_epstart`（带 seed）。服务端 append-only 写 `offseed.jsonl`/`epstart.jsonl`，**恒 204 无 body**（RFC 7230 合规，零回读信息） |

`patch_core_js(...)` 全 flag off 时输出与 r1/r3 版**逐字节一致**（T8a golden 分支锁定）。披露口径（review-r4 S2）：默认运行的引擎与判分行为 = r3 逐字节；E1/E2 信标恒开使 core.js 交付字节与 r3 不同（无判分影响）。

### 3.3 五类扫描逐条（spec §7.6 = H2+H3；r3 增 ④；r4 增 ①②③ 信标类）

| # | 检查 | 输入 | 判定 | 行内落点 |
|---|---|---|---|---|
| 1 | **重导航/file:// 逃逸**（V4/V5） | 本 attempt stdout.log：`Opening URL:` + `miniwob/<subdomain>.html` + `r2g_seed=<seed>` 三者同现的行数；全文含 `file://` | 行数 >1 → flagged（披露但保留官方奖励）；`file://` 命中 → **invalid**（安全事件 `file_url_navigation`） | `task_url_navigations`、`flagged`、`invalid_reason` |
| 2 | **沙箱调用**（V6） | 调用标记 `Executing Python sandbox:` / `Using sandbox tenant` / r3 起 `[SANDBOX_DISABLED]`（被拦尝试也算尝试） | 命中 → invalid（`sandbox_tool_invoked`）；**绝不匹配注册行**（`[TOOL_DEBUG]...'execute_python_sandbox'`/`Registered tool: ...` 每 run 必有，匹配即 100% 误报） | `flagged` + `invalid_reason` |
| 3 | **奖励行数与 reason 域自洽**（V3/H3） | rewards.jsonl 中 `(path末段==<subdomain>.html, seed 相同)` 且 `received_at ∈ [started_at, finished_at]` 的记录：行数 > max(页面加载数,1)；reason 出域；`done is not True` 或 raw 非数字 | 任一 → flagged（披露启发式非不变量：hook 对每次 endEpisode 都 POST，终局后合法追加可能） | `flagged`、`reward_records` |
| 4 | **孤儿记录**（复审专项） | 全部记录 `received_at` 落在任一 attempt 窗口之外（无 received_at 的行按计入处理，宁多勿漏）；r3 起 `invalid_reason` 行不进重试池（F1 修复） | r1: 0 / r2: 0（147 条全在窗内）/ r3: 0（163 条）/ r4: 0 | 复审表逐轮为零 |
| 5 | **快通检测**（异常 <15s 官方通过）+ 计时矛盾（页 240s / 引擎 600s） | results 行 duration 与 reward 时间戳 | 四轮全零（r1 最快 42.1s terminal；r2 最快 33.1s；r3 最快 30.5s unicode-test；r4 最快 50.9s） | 复审表逐轮为零 |
| 6（r4 增） | **off-seed 信标归因 + 零事件细分**（E3，全部 flagged-only） | `offseed.jsonl` 命中本格任务页前缀+attempt 窗口 → `off_seed_navigation`；`reward_records==0` → `zero_reward_events`，若 epstart 也零命中 → 改标 `episode_never_started`（替代不叠加）；`nav==0 ∧ timeout` → flagged | 只披露永不 invalid | `flag_reasons` |

辅助不变式：`flag_reasons` 恒伴 `invalid_reason=None`（flagged-only）；一格可携多个 flagged 原因（`CellScan.merge` 字段级合并）；日志缺失=中性结果（无证据不下结论）。

### 3.4 r2-r4 引擎/harness flags 清单（每个：作用 | 默认 | golden/锁定）

| flag | 引入 | 作用 | 默认 | 锁定/实证 |
|---|---|---|---|---|
| `--terminal-cue` | r2 C2 | 终局中性信号，zombie 回收 | off | T5a-e；r2-post：cue 300 处提及全部晚于终局记录、失败格中位空转 257s（未借 cue 提前放弃） |
| `--single-start` | r2 C3 | 每 tab 单次自动开局，封重导航通道 | off | T6a-e；拦截态点击 cover 零记录（M2 加固）；force_new_tab 18 次为残余缺口，计数披露 |
| `--role-routing` + `--nav-model` | r2 C5 | planner/helper 旗舰、nav 快档；生成 `agents_llm_config.json`（永不含 key） | off / deepseek-flash | T7：env 四键（含 `OPENAI_API_KEY` planner 专用）精确断言 |
| `--extra-tools` | r2 C6 | `LOAD_EXTRA_TOOLS=true`；r3 起默认子集 `drag_and_drop_tool`（E1） | off | r3-pre：子集 43 tools vs 全量 54，persist/recall/clipboard 零注册 |
| `--template-notes` | r2 C7 | Gherkin 固定注释段 4 行（+cue 联动第 5 行） | off | T8 两变体快照；C2 off 时 cue 行不得出现 |
| `--latency-env` | r2 C4f | `LATENCY_ENV_OVERRIDES` 五键（`LLM_REQUEST_TIMEOUT=90`、`LLM_MAX_RETRIES=2`、`BROWSER_STATE_REFRESH_MODE=markers_only`、`BROWSER_NAV_MAX_CHAT_ROUND=30`、`NAV_STEP_TIME_BUDGET_S=120`） | off | T2d：五键值逐字节不变（r3/r4 回归断言） |
| `--provider {deepseek,glm}` | r2 §13 | GLM coding plan 集成（glm-5.3-flash / glm-5.3） | deepseek | 默认路径 env/manifest 逐字节不变（非默认 provider 才追加披露字段） |
| `--smoke-cells` | r2 | pilot 专属追加冒烟格（不进重试池、计入预算；full+smoke 组合硬报错） | 空 | review-r2 M3 机制；orchestrator stage!=pilot → BenchmarkError |
| `--max-cells` | r2 | 5h 窗口配速（r4 单段可 140 格级） | 全量 | 断点续跑=重跑同命令 |
| `--nav-max-tokens` | r3 R3-1 | `NAV_MAX_COMPLETION_TOKENS=768`（收紧 4096 兜底） | 0=off | **r3-post 实证 no-op**（GLM 端点忽略 max_tokens）；r4 headline 摘除、manifest 键缺省（不写 false 也不写 768） |
| `--planner-timeout` | r3 R3-2 | `LLM_PLANNER_REQUEST_TIMEOUT=150` **双层同源**（外层 wait_for + provider 层 ChatOpenAI timeout 兜底） | 0=跟随 90s | T2 四断言；r3 实证超时 41→21 的主要来源 |
| `--extra-tools-modules` | r3 E1 | 子集白名单（`all`=全量须披露记 `["__all__"]`；空串=BenchmarkError） | `drag_and_drop_tool` | T3 subprocess 三态 |
| `--disable-sandbox` | r3 E5 | `SANDBOX_DISABLED=true`（函数体首行拒绝，tenant 读取之前） | off | T6；`[SANDBOX_DISABLED]` 入 `SANDBOX_CALL_MARKERS` → 尝试即无效 |
| `--assert-discipline` | r3 R3-6 | planner `_ASSERT_DISCIPLINE_INSTRUCTION` 前置（terminate=yes∧is_passed=true 须最后一条 helper 观察确认） | off | T11 off 快照与 r2 逐字节；r3 实效不足（假阳性 17→24），r4 由路由层接管 |
| `--md-extended` | r4 R4-A | `MD_INTERACTIVE_EXTENDED=true`：终表扩 div/span/li/tr/td/th/img/label + role 扩 row/cell/listitem/img + identity 门控（name/title/description/text/aria-label/class/id 之一，杜绝无标识灌水）+ **allowed_keys 无条件增 class**（MF-1）+ 150 条上限（超限 `[R2G_MD_TRUNCATED]` warning，无哨兵节点） | off | **T3a/T9a off 档逐字节 golden**（迁移前由 r3 实现对固定合成树产出 golden JSON，迁移后 `json.dumps` 输出逐字节相等且全部条目无 class 键）；T4 真浏览器：email-inbox 行/图标入表、star/trash class 互异 |
| `--verify-before-done` | r4 R4-B | `VERIFY_BEFORE_DONE=true`：`_route_after_planner` 最前插 verify 分支（`verify_rounds<1` 硬上限）；`_VERIFY_STEP` 只许 get_page_text + get_interactive_elements + 报告可见状态（含"禁读 reward 全局"注记） | off | T5 路由五态 + T6 全链（执行序 planner→verify→executor→planner→END，verify_rounds 全程 ≤1）；off 时 simple_hercules 无 verify 节点行为 |
| offseed_beacon（无 CLI flag，orchestrator 恒 True） | r4 E1/E2 | 补丁层 off-seed/epstart 信标（见 §3.2） | 恒开 | T8a：`offseed_beacon=False` 输出与 r3 版 `patch_core_js` 逐字节相等（golden 分支） |

**off 基线纪律**：一切行为变更独立 flag、默认 off = 上轮行为逐字节复现（r4 例外口径：E1/E2 信标恒开使 core.js 交付字节与 r3 不同，判分中立——spec-r4 §0/S2 披露）。引擎回归锚：`tests/test_simple_hercules_langgraph.py` 12 passed（r2/r3 改动前后 stash 对照各跑一次结果一致）。

### 3.5 goal_reader 细节

- 超时常量（代码逐字）：`TASK_READY_TIMEOUT_MS = 15000`、`UTTERANCE_TIMEOUT_MS = 10000`、`DEFAULT_TIMEOUT_S = 30.0`（总预算封顶 navigation + 双 wait）。
- `WOB_TASK_READY` 等待条件：`() => window.WOB_TASK_READY === true`；utterance 等待条件带 try/except 且同时接受 str 与 `{utterance: str}` dict 形态；utterance 超时先查 `__R2G_START_ERROR`（auto-start 失败显式报因）再抛。
- **双类型坑**：`core.getUtterance()` 多数任务返回 str，email-nl 类返回 dict——非 str 取 `["utterance"]` 键（与 browsergym `base.py::_get_goal` 同规则）；读毕立即关浏览器、不触发 endEpisode（预读零 reward 记录，r1 实测预读前后记录数 6→6）。
- `sanitize_goal`：`"`→`'`、换行与连续空白折叠为单空格、strip（纯函数表驱动 7 组单测）。
- 失败 → `GoalReadError` → 该 cell 记 `no_goal`（official_passed=False）、不重试不阻塞；r3 起（E6）同时写 `runs/<run_id>/goal_read_error.log` 留痕。
- r2 附带修复（round2-report §2-7"goal 预读非空等待"）：utterance 等待条件从"存在即返回"改为"非空才返回"，no_goal 从 r1 的 4 格降到 r2 的 1 格。

### 3.6 三口径定义与演进

| 口径 | 定义 | r1 | r2 | r3 | r4 |
|---|---|---|---|---|---|
| **官方 last-wins**（metrics `_rate`，headline 主口径） | 每格取末行，`Σ official_passed / 125`，失败族全计 0、缺行计 0 入 missing | **54/125 = 43.2%** | 58/125 = 46.4%（clean 55/115 = 47.8%） | **59/125 = 47.2%**（=clean，口径首次统一） | **77/125 = 61.6%**（=clean，invalid=0 连续两轮） |
| **首实例**（每格第一条 reward 记录 raw>0，rewards.jsonl 还原） | "不给重试/重导航任何信用"的上界 | 59/125 = 47.2% | 58/125 = 46.4%（**与官方完全相等，桥接双向为空**） | 58/125 = 46.4%（差 1：click-menu-2 合法 infra 重试翻正） | 76/125 = 60.8%（差 1：click-dialog-2 重试翻正） |
| **零重导航子集**（latest 行 nav==1） | 诚实性指标，子集选择偏差（难格更易重导航），**不可作 headline** | 37/60 = 61.7% | 55/102 = 53.9% | 57/87 = 65.5% | 74/108 = 68.5% |
| 官方 row-based（披露口径） | 分母含重试行 | 54/130 = 41.5% | 58/136 = 42.6% | 59/137 = 43.1% | 77/131 = 58.8% |

演进三里程碑：**r1 差 5 格**——59 首实例 −6 反向自伤 +1 重试翻正 = 54 官方，其中 5 格是"页面已过被 infra 状态判负"（login-user-popup/multi-layouts/use-colorwheel-2 被 600s timeout 覆盖、click-pie/click-collapsible-2-nodelay 被重试行 402 崩溃 no_junit 覆盖）；**r2 修复归零**——C1a 回放锁 54→59（+4.0pp 锁定值），r2 实测零"页面 raw>0 被覆盖判负"（2 个 timeout 审计格 + 4 个 junit 审计格见证机制生效）；**r3 起官方=clean**（E4 自动化 + invalid=0），r4 同值。

### 3.7 W2 事件全案（r2-post §2，"本报告最重要发现"）

- **事件**：r2 `--extra-tools` 全量加载 extra_tools（12 工具，含 persist/recall/augment_findings 任意路径文件读写），`persist_findings` 被 agent **自发**在 **10 个格**实际调用（7.4%）；**9 格成功写入 10 个"静默"产物文件**（成功路径零日志行——file_handler_tool 三函数体内无 logger，正是 r2-pre §2.W2 预测的检测缺口），仅 1 格（ascending-numbers）留下可见 traceback；**3 格是通过格**（focus-text、multi-orderings、click-shape）。
- **定性**：非作弊——写入全是任务诊断笔记，奖励判定权威在伺服器内存（/latest），盘上 rewards.jsonl 仅扫描用，机制上无改判通道；10 个产物文件零密钥材料、零写入 dev_runs/（双 key grep 零命中）。
- **处理（预登记契约）**：orchestrator.py:358 契约"invalid_reason 非 null ⇒ 该格判为无效+安全事件"是 r1 既有约定——10 格判无效，**headline 58/125 → clean 55/115 = 47.8%**（官方数字作披露数并列）。
- **r3 治理（三道防线）**：① E1 子集加载（`EXTRA_TOOLS_MODULES=drag_and_drop_tool`，persist/recall/augment/clipboard 结构性不可加载——r3 实测子集 43 tools vs 全量 54、运行日志 `Registered tool` 恰 47 行零基线偏差）；② E2 调用日志行 `[EXTRA_TOOL_CALL] persist/recall/augment_findings` + 扫描标记 `file_tool_invoked`（命中即 invalid）；③ F1：invalid_reason 行不进重试池。r3/r4 实效：文件工具标记 **0/137、0/131**，盘上零 agent 产物。

### 3.8 基础设施

- **seed 派生**：`int(sha1(f"{exp_id}:miniwob:{task_id}").hexdigest()[:8], 16)`（tasks.py）——同格恒等、跨格互异（A4 单测全表枚举断言）；换 exp_id 即换 seed（480s 臂保 seed 配对的技巧=复用 exp_id 换 `--exp-root`）。
- **断点续跑**：results.jsonl 逐行 append+flush、manifest 收尾写；重跑同命令时已有 `(task_id, seed)` 行的 cell 自动跳过（`--force` 关闭）；每行结果崩溃安全。
- **配速**：GLM coding plan 5h 窗口 → `--max-cells` 分段（r3 按 5h 配速，r4 配速翻倍单段可 140 格级）；429 熔断标记（`Error code: 429`/`Too Many Requests`/`RateLimitError`/`rate limit`/`限流`——r2-pre 增补时特意不用裸 "429"，会误命中时长数字）。
- **infra 重试池**：`RETRY_BUDGET = {"pilot": 2, "full": 5}`（代码逐字）；重试仅限 `INFRA_STATUSES = {timeout, no_junit, no_reward}`、每 cell 至多 1 次（`INFRA_RETRY_LIMIT_PER_CELL=1`）；用例失败（official_failed）**绝不重跑**；显式排除：熔断行、smoke 格、（r3 F1 起）invalid_reason 行。
- **circuit-break**：attempt 级三条件（status∈{timeout,no_junit} ∧ `duration_s < 30.0` ∧ stdout 含 402/Insufficient Balance/连接类标记）→ `infra_circuit_break=True` 且逐出重试池；run 级连续 2 格 → BenchmarkError 中止 stage（已写行保留）。
- **预算账本**：manifest `budget` 记 `hercules_used/retries_used/cap`；r4 起增 `cumulative_hercules_used`（按带 goal 的行实时累计）+ `cumulative_over_cap`；Σ>144 → warning 日志 + 量化披露（纪律条款，工具不可强制——"护栏强度 = 单 dir 自动化审计 + 跨 dir 人工纪律"，r4-pre V8）。
- **成本实测**：r1 12.9M token / wall ≈336s/格（round1-report §2）；r2 9.08M；r3 12.9M；**r4 12.2M**（round4-report §1）。任务书所写"约 240s/格"：240s 是 episode **页面预算**（agent-e-anatomy 对照表"GLM（240s/格）"即指此口径），实测单格 wall 以 r1 ≈336s 为准——**"240s/格"作为 wall 数字未在文档中出现**，档案按可核实口径记录。
- **r1 预算锚**（plan.md §5）：单价上限取 PLAN §3.3 实证值 4.7 万 token/$0.13 每次；pilot ≤$1.56、full ≤$16.90、红线 142 次/$20。
- **禁代理回环**：harness 对 8462 的 urllib 调用统一 `build_opener(ProxyHandler({}))`（macOS 系统代理会把 127.0.0.1 回环请求走代理返回 502，exp001 已知问题 11 同解）。

## 4. 踩坑全记录（每坑一卡）

> 卡式：现象 | 定位 | 根因 | 修复 | 教训 | 出处

**坑 1 · r1 判分瑕疵：5 格页面已过却被判负**
现象：r1 官方 54 vs 首实例 59，5 格 rewards 里 raw>0 在窗口内到达、官方却 timeout/no_junit。定位：`build_result_row` 把 runner_status==timeout 排在 reward 判定之前。根因：把进程级 infra 事件当成了判定权威，违反口径 4。修复：C1a 优先级重排（raw>0 压倒 infra）+ `runner_status` 审计键 + r1 数据回放锁（5 格翻正、email-inbox-delete 反例维持、54→59 单测锁定）。教训：口径定义与代码实现要逐条对拍——"页面奖励为唯一权威"写进了 spec，实现却没执行对。出处：analysis-r1-architecture H1、review-r2 §5 复算、test-report §12.2。

**坑 2 · email-inbox-delete 反例：重导航 last-wins 自伤**
现象：首实例 raw=1，之后 8 次重导航（nav=8），终局 raw=−1，官方判负。定位：`/latest` last-wins + 补丁 A 对每次 load 都自动开局。根因：重导航重置 240s 并作废失败提交——本例反向作废了已到手的通过。修复：不修判定（该格必须仍判负，作 C1a 正确性的对照锚）；通道级修复是 C3 单次开局。教训：修复类改动必须带反例锁定——"不该被救回的格"比"该被救回的格"更能证明规则正确。出处：plan-r2 C1a、spec-r2 §1.3、security-review-r1-post §3。

**坑 3 · HUD 泄漏（V1）+ START 重开按钮（V2）**
现象：`body.innerText`（get_page_text 提取源）回合内含 `Last reward/Time left/Episodes done`；endEpisode 后 `Last reward: 1.00` 直接可读；错误提交后 START 覆盖层重现，一次普通 DOM 点击即重开同 (task,seed) 新实例（utterance 从 "Joye" 变 "Michel"——同 seed 下 RNG 前进）。定位：vendored core.js DISPLAY_HTML/updateDisplay + endEpisode 尾部（core.js:145）调 startEpisode。根因：Hercules 用自己的 DOM/文本提取，不受 MiniWoB/BrowserGym 的 getDOMInfo 剔除清单约束（BrowserGym 口径下 agent 看不到 HUD，我们口径下看得到→观察空间强于对齐基线）。修复：H1 三行加固（display:none + stub 两函数）；实测后 HUD 零命中、覆盖层 `page.click("#sync-task-cover", timeout=1200)` 超时不可达。教训：换观察框架时，基线的"看不见"保证不随行——要按自己的提取路径重新审计泄漏面。出处：security-review-r1-pre §2.V1/V2、test-report §11.2。

**坑 4 · `display:none` 而非 `remove()`**
现象：（推演）移除 `#reward-display` 会在终局时抛 TypeError。定位：endEpisode 仍要写 `#episode-id` 等子元素。根因：加固点的 DOM 是活依赖。修复：用 `display:none`（innerText 跳过隐藏子树，对 get_page_text 已足够）。教训：最小干预——隐藏语义即可时不要破坏结构依赖。出处：security-review-r1-pre §2.V1 缓解注记、miniwob_server.py:84-87。

**坑 5 · auto-start 只等 WOB_TASK_READY 不够**
现象：book-flight 抛 `ReferenceError: ui_utils is not defined` 后永久失去自动开局。定位：core.js:49 `WOB_TASK_READY` 默认即 true。根因：抢在 onload 前调 `startEpisodeReal()`，依赖此时还未创建的 cover_div/click-canvas。修复：轮询条件改三条件（WOB_TASK_READY && readyState==="complete" && core.cover_div），clearInterval 只在开局成功后执行。教训：布尔旗标的默认值本身就是坑——要等的是"页面就绪"不是"旗标翻转"。出处：spec.md §3.2 行为要点（review 必改 1.4）。

**坑 6 · `WOB_EPISODE_ID` 开局成功后仍为 0**
现象：用 `EPISODE_ID>0` 判断开局会全盘误判。定位：`WOB_EPISODE_ID` 只在 core.endEpisode 内自增。根因：把"回合号"误当"已开局"信号。修复：开局证据改用 `core.ept0 != null` + utterance 非空 + `EPISODE_MAX_TIME===240000` + 无 `__R2G_START_ERROR`。教训：选观测量前先读变量的写点（谁在什么时候写它）。出处：spec.md §9-D16、review（必改 1.6）、test-report §6。

**坑 7 · `browsergym.miniwob.all` 迭代 TypeError**
现象：按 `all` 枚举任务直接 `TypeError: 'module' object is not iterable`。定位：`all` 是子模块不是列表。根因：命名歧义。修复：用包级列表 `ALL_MINIWOB_TASKS`（build_task_table 硬编码 + 注释）。教训：registry API 的"直觉名"要先试；构建脚本带 125/唯一/存在性三重硬失败校验。出处：spec.md §2.1（review 实测确认）、build_task_table.py。

**坑 8 · `core.getUtterance()` 对 email-nl 返回 dict**
现象：按 str 处理 email-inbox-forward-nl 等任务会拿到 dict。定位：上游 getUtterance 双类型。根因：NL 模板任务返回 `{utterance, fields}`。修复：非 str 取 `["utterance"]` 键（与 browsergym `base.py::_get_goal` 同规则）+ 空值即 GoalReadError。教训：预读器要对上游返回类型做枚举处理而非假设。出处：spec.md §4.1、goal_reader.py docstring。

**坑 9 · 沙箱调用标记字符串与注册行混淆（r1 加固的两处字面偏差）**
现象：安全审查报告写的标记若照抄，会 100% 误报。定位：报告写 `Executing execute_python_sandbox`，真实调用标记是 `Executing Python sandbox: file=…`（execute_python_sandbox.py:62）与 `Using sandbox tenant: …`（:70）；而 `execute_python_sandbox` 字样只出现在注册日志（每 run 必有）。根因：审查者按记忆写串未逐字回查。修复：实现用真实调用标记 + 单测含"只有注册行→不 flag"反例；同类偏差还有 reason 合法域（报告给 `{"", "timed out"}`，复核 vendored 树发现 unicode-test.html:53,55 有 `'Cool!'` 与 `'You clicked on '` 前缀——不补会误判 full 的合法行）。教训：检测器的标记串必须逐字来自现场代码/数据，两处偏差都回写了 spec。出处：test-report §11.3、spec.md §7.6。

**坑 10 · H3 窗口过滤：重试复用同一 (path, seed)**
现象：按整文件计数会把每个重试 cell 的历史记录误判为异常。定位：基础设施重试复用 run 目录与 (path,seed)。根因：rewards.jsonl 是跨 attempt 追加的。修复：只统计 `received_at ∈ [started_at, finished_at]` 的本轮记录；无 received_at 按计入处理（宁多勿漏）。教训：append-only 日志上的"本格记录"必须带时间窗语义。出处：spec.md §7.6、orchestrator.py `_in_window`。

**坑 11 · W2：extra_tools 全量加载的静默文件写（10 格事件）**
现象：9 格静默成功写入 10 个诊断文件、扫描器零告警（成功路径无日志行）；3 格是通过格。定位：`--extra-tools` 开的是全量 12 工具面（非"仅拖拽"），file_handler 三函数无 logger。根因：开闸只看主功能（drag），没审计同闸加载的附带工具面。修复：按预登记契约 10 格判无效（headline 58/125→55/115）；r3 三道防线（子集加载/调用日志行/invalid 不进重试池），r3/r4 该标记全零。教训：开一个开关=加载它拖带的整个世界；检测缺口要在事件成真前按"静默成功不可见"最坏假设评估。出处：security-review-r2-pre W2、r2-post §2、round2-report §3、plan-r3 R3-5。

**坑 12 · C1c 预检 transport（litellm 恒败）**
现象：按 spec 字面用 litellm 裸模型名探测，可用 key 会被误判为失败并 exit 3。定位：litellm 对自定义端点需 provider 前缀（openai/...）+ api_base。根因：探测栈与引擎栈不同。修复：钉死同栈 `ChatOpenAI(model, api_key, base_url, max_tokens=1, max_retries=0)`。教训：gate 的探测路径必须与被 gate 的生产路径同栈，否则 gate 本身成为事故源。出处：review-r2 M4、spec-r2 §3、preflight.py。

**坑 13 · C5 planner key 断链**
现象：`--role-routing` headline 在引擎启动即崩、每格 no_junit。定位：PlannerAgent 裸构造 ChatOpenAI（不经 create_chat_model），langchain-openai≥1.1.14 只认显式参数或 `OPENAI_API_KEY` env，不读 `MODEL_API_KEY`；且配置文件加载成功后 env provider 完全不注册（File>Env 是源级覆盖）。根因：key 回退链覆盖面被高估。修复：child env 四键注入（`AGENTS_LLM_CONFIG_FILE`、`REF_KEY`、`MODEL_API_KEY`、`OPENAI_API_KEY` 后两者同值）。教训：多配置源叠加时，"谁在什么条件下还读 env"要追到构造函数级。出处：review-r2 M1、spec-r2 §6.2。

**坑 14 · C3 拦截态的 START re-roll 复活（review-r2 M2）**
现象：sessionStorage 早退路径若不加加固，被拦截页面上 START 覆盖层可点——onclick 点击期经 `core.startEpisodeReal()` 动态取函数（core.js:78-80），会重开**无 seedrandom** 的随机实例并 POST 新记录、经 last-wins 覆盖本格终局。根因：加固四行原本只在开局成功分支内执行。修复：拦截分支同样 stub 三函数 + 注入 CSS（双保险不依赖 onclick 绑定形态），T6e 断言拦截态点击零记录。教训：堵通道要堵到"被堵状态"本身——每个新分支都是新的攻击面。出处：review-r2 M2、spec-r2 §4.2、miniwob_server.py `_SINGLE_START_GUARD_BLOCK`。

**坑 15 · runbook 陷阱：GLM 下残留 `--nav-model deepseek-flash`（W12）**
现象：照旧命令跑会把 deepseek 模型名路由到 bigmodel 端点 → 每格 nav 调用失败（全量报废级操作失误）。定位：test-report §12.5 命令写于 provider 切换之前，orchestrator 不做 provider 校验。根因：文档与配置漂移。修复：r2-pre §6 给出正确命令（GLM 下不传 `--nav-model`，默认 glm-5.3-flash）；与 W1（glm+role_routing 时 planner glm-5.3 未探测，probed=[glm-5.3-flash]）一并作为开跑前 P0 加固。教训：实验命令是活文档，provider 切换时逐参数过一遍。出处：security-review-r2-pre §2.W1/W12。

**坑 16 · R3-1 补丁是死代码 + "r2 无上界"立论不成立（review-r3 M1）**
现象：按 spec 原文 `if kwargs.get("max_tokens") is None` 实现，768 上限永不生效（headline 会在杠杆静默失效下跑完）。定位：benchmark 必经路径上 `adapt_llm_params_for_model` 被调两次（SimpleHercules.create + create_chat_model），GLM 模型名落入 model_utils "other models" 分支注入 `max_tokens=4096`——kwargs 恒带 4096，`is None` 恒 False。根因：立论证据链错误（"从未被消费"没追 adapt 注入链）。修复：env>0 无条件覆盖；T1 断言改为 env 未设→4096（真实基线）、env=768→768；语义口径改写为"把 4096 收紧到 768"。教训：改上限类参数前先证明"现状值是什么"——死代码比没有代码更危险（测试绿而生产死）。出处：review-r3 M1、spec-r3 §1.1。

**坑 17 · R3-2 只改一层超时，150s 兑现不了（review-r3 M2）**
现象：只改 `_llm_ainvoke` 的 wait_for，>90s 的单次 planner 生成仍被 provider 层 90s 掐断（随后 SDK 重试链），"150s 内完成即活"不成立。根因：90s 是双层墙（外层 wait_for + ChatOpenAI 自身 timeout 兜底同喂 `LLM_REQUEST_TIMEOUT`）。修复：新增 `LLM_PLANNER_REQUEST_TIMEOUT` **同源喂两层**；off 时两 helper 同值返回= r2 逐字节。教训：超时链要沿调用栈数层数，改外层不改内层=白改。出处：review-r3 M2、spec-r3 §2.3。r3 实效：超时 41→21 的主要来源（round3-report §2）。

**坑 18 · R3-1 实证 no-op：GLM 端点忽略 max_tokens**
现象：flag 声明 `nav_max_tokens=768` 与运行时行为不符。定位：live 实验——引擎注入链正常（model.max_tokens=768 确认），但 768 限制下模型产出 ~13.9k 字符内容、usage 报 19,349 completion tokens；数据面 4,213 个 executor 轮 0 轮恰等于 768、215 轮 >768（max 3,918），与 r2（250/3,721 >768、max 3,402）分布同质。根因：GLM coding 端点把 max_tokens 当参考值（思考 token 另计、内容不截断）。修复：r3 报告与 manifest 修正为 no-op、任何 r2→r3 差异不得归因于 R3-1；r4 headline 摘除该 flag（manifest 键缺省），r5 前置=换 provider 时先做 max_tokens 尊重性 live 预检。教训：干预是否生效要用端到端证据关账，flag 在位≠flag 生效。出处：security-review-r3-post §5、round3-report §2、plan-r4 §0.4。

**坑 19 · drag_and_drop md 包装 bug：61 调 0 中**
现象：拖拽家族 r2 全灭，61 次 drag_and_drop 调用 0 次解析成功。定位：`if "md=" not in selector: selector = f"[md='{selector}']"`（drag_and_drop_tool.py:29-31，子串误判）。根因：拖拽源（drag-handle/SVG shape）根本不在 md 交互元素表里，agent 无正确答案可传，一切输入被错误包装。修复：r3 选择器透传（css=/xpath=/text=/#/. 等显式形态原样传，裸数字才包 md，未知形态三候选兜底）+ description 更新；kill-switch 判据=`Found source element using selector` ≥1。实绩 0/9→3/9、9/9 格解析成功。教训：工具包装层"好心规范化"是最大的静默故障源。出处：plan-r3 R3-3、security-review-r3-pre §一/R3-3、r3-post §7。

**坑 20 · 极早崩格无日志（r2 email-inbox-forward-nl 0.6s no_goal 零 stdout.log）**
现象：格 0.6s 即崩且盘上无任何可复核证据。定位：GoalReadError 路径先于子进程启动，无 stdout.log 落盘；重试格 attempt1 日志还曾被 attempt2 覆盖（r1 后附注）。根因：留痕依赖子进程，预读失败段没有自己的留痕。修复：r2 C1d 按 attempt 分目录（`runs/<run_id>/attempt<N>/stdout.log`）；r3 E6 写 `goal_read_error.log`（mkdir+写失败仅 warning）。教训：可观测性要覆盖"进程都没起来"的最短路径。出处：r2-post §3.2、security-review-r2-post §9-P2、spec-r3 §5.6、orchestrator.py:1176-1185。

**坑 21 · compact 剥 class → star/trash 同形不可辨（review-r4 MF-1）**
现象：extended 收录的图标节点经 compact 后 `{"md":11,"tag":"span","name":"Caralie"}` 与 trash 完全同形——star-reply 类任务只能盲试，T4 断言按字面实现必然失败。定位：`compact_interactive_node` 的 allowed_keys 不含 class/id，而 spec 同时要求"compact 逐字节保留"。根因：图标无文本，class 是唯一可辨别语义锚，两层规格自相矛盾。修复：allowed_keys **无条件**增 class（off 模式抓取属性表无 class，off golden 逐字节不受影响，T3a 断言 `all("class" not in entry)` 锁定）。教训：上层扩展与下层压缩的字段契约要一张表对齐；审查用真浏览器实测（email-inbox?r2g_seed=42）拿到的同形证据比读代码有力。出处：review-r4 MF-1、spec-r4 §1.2、test-report §15.2。

**坑 22 · 终局后 endEpisode 复触发的"形似洗白"记录**
现象：r4 rewards.jsonl 出现 book-flight（−1→display+1）、click-menu（−1,−1→display+1）形态，人工审计易误判洗白。定位：REWARD_HOOK 对被 EP_TIMER 守卫短路的重调用也 fire，POST 的 `reward` 是新实参、`raw` 是封存原值。根因：payload 双字段语义不同（显示参数 vs 权威全局）。修复：不是 bug——机制本身抗翻转；审计纪律=必读 `raw` 字段。两格官方判负（方向反洗白）。教训：判分链的"权威字段"要写进审计手册。出处：security-review-r4-post §3/§4（P3-2）。

**坑 23 · wrong-seed 盲区（E1 只验 seed 存在性）**
现象：r4 实发 2 格 3 次 wrong-seed 加载（click-dialog-2 att1 seed=78425566≠本格 786425566；count-sides seed=999/1），E1 信标对错值静默。定位：offseed 探测只判 query 有无 `r2g_seed`，不比对 harness 派生值。根因：检测器粒度不足。兜底：own-seed 过滤生效——click-dialog-2 att1 的错 seed 页面被 agent 做到 raw=1，但 fetch_reward 按 (path, seed) 匹配取不到，**未计入分值**；r5 建议服务端比对 seed 值不等即 flagged（一行判断）。教训：过滤与检测要分开评估——判分没被骗过（过滤有效）不等于检测完备。出处：security-review-r4-post §1/§3/§6（P3-1）、round4-report §3。

**坑 24 · E4 clean 口径在"无效格被重试"下弱化（r3-pre F1）**
现象：无效事件发生在 att1、格因 infra 状态进重试、att2 干净 → latest 行 invalid_reason=None → 该格从 clean 剔除中消失（r2 数据回放 clean=57/124 ≠ 手工 55/115，差 2 格定位到 ascending-numbers/click-pie）。根因：last-wins 只看末行。修复：一行——`_retry_infrastructure_failures` 选格循环加 `if row.get("invalid_reason"): continue`（orchestrator.py:1395-1398）。教训：自动化口径要拿历史人工口径做差分回归，差格要逐行归因。出处：security-review-r3-pre F1、r3-post §6。

**坑 25 · 预算账面"144 恰满"与代码不符（review-r4 MF-3）**
现象：plan 写 14+130=144 ✓，但 `RETRY_BUDGET pilot=2` 使 M1 最坏 16，结构性最坏 146。定位：cap 强制是单次 orchestrator 调用口径（`hercules_runs` 每次 run() 重置）。修复：预算表改为 M1=14–16、M2=130/调用；全局累计护栏（manifest cumulative_* + Σ>144 量化披露 + 禁拆 exp-id 分窗纪律）。教训：护栏数字要按代码的计数口径算，不能按愿景算。出处：review-r4 MF-3、plan-r4 §5、orchestrator.py `_write_manifest`。

**坑 26 · 预期数字不可推演（review-r3 M3 / review-r4 MF-2）**
现象：r3 保守档表写 63 与构成 58+5+1+1=65 差 2 无注释；r4 把分析给的"~0.7 能正确交互"当通过率凑出 A 上界 +8（法定通过概率 ~0.4）。根因：构成列不可复算、两类概率混用。修复：r3 改 65/125=52.0%；r4 重推为逐格构成表（A1 email 10×0.3/0.4、A2 单格赋档、B1 24×0.15/0.2、B2 ⌊2×0.5⌋ → 增量 7~10、66/125=52.8% ~ 69/125=55.2%），交互率不入通过算术。教训：预期表必须"读者可复算"，交互率和通过率是两个分母。出处：review-r3 M3、review-r4 MF-2、plan-r4 §6。

**坑 27 · 文档间小矛盾备案（review-r2 §4-b）**
现象：multi-layouts 页面通过时刻 failures 写 83s vs architecture 写"4.3 分钟"；email-inbox-forward 导航次数 16 vs 15。定位：两份分析文档内部矛盾。核实：经 rewards.jsonl 差分，**83s 正确**；nav 字段值 15（16 条 Opening URL 中 15 条命中本格 URL，第 16 条是 about:blank 新标签机制，良性）。教训：引用前向原始数据回查；分歧数字要落备案防后人踩错源。出处：review-r2 §4-b、security-review-r1-post §4。

**坑 28 · 杂项工程坑（各一处，合并一卡）**
① playwright sync API 同线程不允许两个并发上下文 → 预读顺序调用、浏览器 fixture function scope，并行化必须改多进程（test-report 已知问题 1）。② macOS 系统代理劫持 127.0.0.1 → `ProxyHandler({})` 禁代理（exp001 已知问题 11 同解）。③ Python 默认参数在定义期求值 → `browser_nav_max_chat_round` 缺省改在 `__init__` 函数体内读 config（review-r2 建议c）。④ spec T8 "overall = 1/3" 系算术勘误，在 `_rate` 零改动约束下实为 2/3（invalid-but-passed 格照计 1，r2 官方口径本就如此）——测试锁定 2/3 并注明理由（test-report §14.4-1）。⑤ `"Its Empty, try something else"` 分支是不可达死代码：`json.dumps` 恒返回非空串，空表实际返回 `"[]"`；r4 原样保留未触碰，T9 锁定（test-report §15.4-4）。⑥ registry 中唯一 visual 任务是 `visual-addition`（不存在 `visual-maze`，pip 包 html 树亦无该文件）——pilot 子集若含不存在任务则启动即报错（spec §2.3）。⑦ `localhost:PORT` 形态被 scheme 白名单拒绝（`urlsplit` 解析出 scheme="localhost"）——任务 URL 恒带 `http://127.0.0.1:` 不受影响，行为收紧非漏洞，勘误落档（security-review-r3-pre F4、test-report §14.4-3）。⑧ 5 个 vendored 页面未被 registry 引用（如 book-flight-nodelay.html）——任务表只做存在性校验，不影响口径（test-report 已知问题 6）。

## 5. 面试深挖要点

| 追问 | 一句话答 | 指向 |
|---|---|---|
| 为什么选 MiniWoB++ 而不是 WebArena/WebVoyager？ | MiniWoB++ 官方 pip 包自带全部 HTML 本地伺服即可、页面原生奖励免裁判；WebArena 要 5 个自托管站点+docker、WebVoyager 要外网+GPT-4V 裁判、Mind2Web 只测静态子能力，全被可行性否决 | §2-D1 |
| 怎么保证 360 个 HTML 没被人改过？ | 构建期整树复制 + PROVENANCE 记录 360 行逐文件 sha256 与总字节数 4,988,677，单测逐个核对；补丁只在伺服期纯追加、vendored 字节从不改写（B 组单测断言前缀逐字节一致） | §2-D3/D4 |
| 判分为什么可信？agent 能不能伪造奖励？ | 唯一权威是页面 JS 全局 `WOB_RAW_REWARD_GLOBAL>0`，端点无防伪造是**原则性极限**（任何浏览器侧密钥都会出现在伺服的 core.js 里），机制上不可根除只能检测+披露——四轮全量 reason 域/done/raw 自洽/孤儿记录/快通全部为零，且 GET 端点本体 404、无 JS-eval 工具使伪造前提（页面 JS 执行能力）不存在 | §3.1、§2-D7、security-review-r1-pre V3 |
| 防作弊具体做了哪几层？ | 五层：判分权威封闭（HUD display:none+stub、START 覆盖层不可达、sessionStorage 单次开局且拦截态同层加固）；五类收尾扫描只披露不改判；双安全审查（pre 定级 gate、post 全量重放扫描器与三口径复算）；口径防灌水（invalid 出分子分母、480s/多 seed 只进 ablation 臂、best-of-3 明确拒绝）；原则性极限披露（口径 5/6/7 三条基准限制随报告原样转载） | §3.3、§2-D10~D12/D29 |
| 四轮分数各自怎么来的？有灌水吗？ | 43.2→46.4（判分修复 54→59 是执行口径非放宽 + provider 换 GLM 混杂披露）→47.2（planner 双层 150s 实证救超时 41→21、drag 0/9→3/9；nav 768 实证 no-op 不记功）→61.6（md-extended +17 格逐格 md 点击日志实证 + verify 0 分诚实记录 + 方差 +7 在 ±10 噪声带内披露）；每轮首实例口径同表披露证明重试/重导航零灌水 | §3.6、round4-report §2 |
| r2 的干净口径为什么是 55/115 不是 58/125？ | W2 事件：agent 自发调用文件写工具 10 格（9 格静默成功绕过扫描），按 r1 预登记契约（invalid_reason 非 null ⇒ 无效）剔除，3 格是通过格——r3 用子集加载结构性根除，r3/r4 该标记全零、分母回 125 | §3.7 |
| 引擎改动怎么保证不污染默认行为？ | 一切行为变更独立 flag 默认 off；off 档 golden 逐字节锁定（flatten 过滤器迁移前固化 golden JSON、patch_core_js(offseed_beacon=False) 与 r3 版逐字节相等、planner prompt off 快照）；引擎回归 12 条 + record2gherkin 492 条全绿 | §3.4 |
| 你们的成本和速度？ | r1 12.9M token / wall≈336s/格（deepseek）；r2 9.08M（GLM 5h 窗口分段+429 熔断）；r3 12.9M；r4 12.2M；episode 页面预算恒 240s（原版 10s 的 24×放宽，随报告披露） | §3.8 |
| 单 seed 单次是不是太弱？ | 是已知口径限制、随报告披露并给三口径对照；单 seed 纯噪声翻转 ~12.8%、噪声带 ±10 格随表披露；majority-of-3（只治测量）设计好待批、best-of-3（口径掺水）明确拒绝；r4 审计把 +18 格归因到日志实证后 M4 门控未触发 | §2-D25、r4-post §9-5 |
| 审查发现过自己人的问题吗？ | 有且不止一次：r1 判分瑕疵 5 格（自己 harness 把口径执行错了）；R3-1 按原 spec 实现是死代码（审查用 adapt 注入链证伪"r2 无上界"立论）；review-r4 MF-1 用真浏览器实测发现 compact 剥 class 使 star/trash 同形；R3-1 上线后被 post 审查实证为 no-op 并在 r4 摘除——审查和实现分离+pre/post 双审是这个体系的骨架 | §4 坑 1/16/21/18 |
| 邮件家族两轮全灭怎么破的？ | 死因定位到 `flatten_elements` 过滤器：行/图标（div/span）注入了 md 但不在 role/tag 白名单被整批丢弃（off 模式 email-inbox 终表 0 条，复现 executor 自述"Interactive elements list is empty"）；r4 扩表后 4/4 精确兑现 + 12 格同构外溢（span.alink/accordion 头/hitarea/slider-handle/label）——教训：机制改动的受益面按机制算不按家族算 | §2-D26、r4-post §5 |
| 奖励 POST 丢了怎么办？ | 无兜底（机制上重开同 seed 页读到的不是终局值）；页面 240s 自超时会先于 600s harness 超时 POST（raw=−1, reason='timed out'）；no_reward 如实记官方失败、限 infra 重试 1 次——r4 借 verify+epstart 信标做到四轮首个零"无页面裁决"终局行 | §2-D9、round4-report §1 |

## 6. 数字与出处对照表

| # | 数字 | 含义 | 出处 |
|---|---|---|---|
| 1 | 125 / 109 family / 1 visual（visual-addition） | 任务表规模（browsergym-miniwob 0.14.3 registry） | test-report.md §2、tasks.json |
| 2 | 360 文件 / 4,988,677 字节 / 360 行 sha256 / miniwob 1.1.0 / BSD-3-Clause | vendored 供题 | test-report.md §2、PROVENANCE.md |
| 3 | 240000（默认 r2g_ms）/ 10s 原生 / 600s 引擎 timeout / 端口 8462 | 计时与服务参数 | tasks.py、orchestrator.py、miniwob_server.py |
| 4 | r1 54/125=43.2% · 59/125=47.2% · 37/60=61.7%；130 行/227 rewards/126 logs；12.9M token；wall≈336s/格 | r1 三口径与规模 | round1-report §1-2、security-review-r1-post §2 |
| 5 | 54 = 59 − 6 + 1；5 格判分瑕疵名单；email-inbox-delete raw=1→nav=8→raw=−1 | r1 桥接与瑕疵 | security-review-r1-post §2/§3 |
| 6 | 61/125 格 nav≥2（最多 15）；87/125 flagged（130 行中 88 行：纯导航 58/纯 H3 26/双触发 4）；249/267=93% 同 tab | r1 重导航画像 | r1-post §1、plan-r2 C3 |
| 7 | 42.1s（r1 最快通过 terminal）；use-colorwheel-2 raw=0.5372549019607844（=137/255）；click-checkboxes raw=−1/3；29 timeout 行 600.04-600.05s | r1 复核抽样 | r1-post §4/§6 |
| 8 | 54→59（+4.0pp 锁定值） | C1a 回放（fixture 130 行去重 125） | plan-r2 C1a、test-report §12.2 |
| 9 | r2 58/125=46.4% · 首实例=官方 · 零重导航 55/102=53.9% · clean 55/115=47.8%；136 行/147 rewards/135 logs；9.08M token | r2 成绩 | round2-report §1、r2-post §5 |
| 10 | W2：10 格（7.4%）/9 格静默成功写 10 文件/3 格通过格；click-shape 沙箱 8 次 File not found 零执行（raw=1@t+91s 早于 t+233s）；3 格 javascript: 尝试；4 格 21 次外联（terminal 11/login-user 8/email-inbox-delete 1/use-autocomplete 1）；21 组陈旧重发 | r2 安全事件全景 | security-review-r2-post §2/§3/§7 |
| 11 | C1a 审计：2 timeout 格（click-shape raw@91s、odd-or-even raw@62s）+ 4 junit 格；多导航格 61→16；flagged 88→29 行 | r2 机制生效证据 | r2-post §5/§6、round2-report §2 |
| 12 | r3 59/125=47.2%（=clean）· 首实例 58/125=46.4%（click-menu-2 51.7s 翻正）· 57/87=65.5%；137 行/163 rewards/137 logs；12.9M token；超时 41→21 | r3 成绩 | round3-report §1、r3-post §4 |
| 13 | R3-1 no-op：live 768 下产出 ~13.9k 字符/19,349 completion tokens；4,213 轮中 0 轮==768、215 轮>768、max 3,918（r2 对照 250/3,721、max 3,402） | flag 失真实证 | security-review-r3-post §5 |
| 14 | scheme 白名单实拦 4 次（visual-addition att2 / click-pie-nodelay att2 `javascript:void(0)`；count-sides att1 / bisect-angle att2 `view-source:`）； Registered tool 恰 47 行零偏差；drag 0/9→3/9（147s/125s/505s，kill-switch 9/9 解析成功） | r3 治理实效 | r3-post §3/§7 |
| 15 | r3 flagged 55 行=44 nav>1 + 10 记录数>加载数 + 1 view-source 披露；invalid=0；reason 域 ''×103+timed out×59+Cool!×1 | r3 扫描构成 | r3-post §2/§3 |
| 16 | r4 77/125=61.6%（=clean）· 首实例 76/125=60.8% · 74/108=68.5% · row-based 77/131=58.8%；131 行/152 rewards/285 epstart/2 offseed/131 logs；12.2M token；超时 21→9 | r4 成绩 | round4-report §1、security-review-r4-post §1/§3 |
| 17 | +18 = 新通过 24 − 回退 6；A +17（email 4/4 + find-greatest 1 + 机制外溢 12）+ 方差 +7；verify 103/131 行恰 1 次、0 分贡献；md 表长 p50=4/p90=36/max=129、`[R2G_MD_TRUNCATED]` 0 次 | r4 归因 | r4-post §5/§6/§7 |
| 18 | circle-center 0.939 部分分计通过；click-dialog-2 att1 错 seed 78425566≠786425566 被own-seed 过滤拒计；39 official_failed 全带 raw=−1、9 timeout 全带 timed out | r4 完整性 | r4-post §3/§6/§7 |
| 19 | R2_BUDGET_CAP=144；RETRY_BUDGET {"pilot":2,"full":5}；smoke 14=10+2+2、headline 130；结构性最坏 146；r1 红线 142/$20、$0.13/次、4.7 万 token | 预算体系 | orchestrator.py、spec-r2 §0.2、plan.md §5、plan-r4 §5 |
| 20 | CIRCUIT_BREAK_DURATION_S=30.0 / RUN_LIMIT=2 / INFRA_RETRY_LIMIT_PER_CELL=1 / INFRA_STATUSES={timeout,no_junit,no_reward} | 熔断与重试池 | orchestrator.py |
| 21 | TASK_READY_TIMEOUT_MS=15000 / UTTERANCE_TIMEOUT_MS=10000 / DEFAULT_TIMEOUT_S=30.0 | goal_reader 超时 | goal_reader.py |
| 22 | 轮询 50ms / tries>200（~10s）/ 同步 XHR / z-index 2147483647 | 补丁层参数 | miniwob_server.py、spec.md §3.2 |
| 23 | 合法 reason 域 `{"", "timed out", "Cool!"}` ∪ 前缀 `"You clicked on "`（unicode-test.html:53,55 唯一第三参任务页） | H3 reason 域 | spec.md §7.6、test-report §11.3 |
| 24 | EXTENDED_TAGS {div,span,li,tr,td,th,img,label} / EXTENDED_ROLES {row,cell,listitem,img} / max_nodes=150 / identity 7 键 | r4 扩表定义 | spec-r4 §1.2 |
| 25 | LATENCY_ENV_OVERRIDES：LLM_REQUEST_TIMEOUT=90（默认60）、LLM_MAX_RETRIES=2（默认1）、markers_only、NAV_MAX_CHAT_ROUND=30（默认50）、NAV_STEP_TIME_BUDGET_S=120（默认0）；planner-timeout=150 | 延迟包键值 | orchestrator.py LATENCY_ENV_OVERRIDES、spec-r2 §5.3 |
| 26 | 测试基线链：75→117→427→456→492 passed（record2gherkin）；引擎回归 12 passed；pilot 11 次执行=10+1 重试 overall 8/10 | 测试规模 | test-report §1/§12.1/§13.2/§14.1/§15.1/§11 |
| 27 | r1 planner 3.2 轮/格 vs executor 51.3 轮（超时格）/轮间均值 9.0s/断裂 25.1 次/格/completion p90 1998 tokens；58/121 格含 verify 步 | r1 延迟画像 | analysis-r1-architecture §0 |
| 28 | 公开可比带 A 档 47-60%（零样本文本-DOM GPT-4 级）；81.6% 实为 HxAgent 且四重口径不可比、Agent-E 从未发表 MiniWoB++ 数字 | 基线对照与更正 | baselines.md、agent-e-anatomy.md |
| 29 | r2 首例 wrong 预检：probed=[glm-5.3-flash] 而 planner=glm-5.3 未探；W12 命令带 `--nav-model deepseek-flash` | r2-pre 两个 P0 | security-review-r2-pre §2.W1/W12 |
| 30 | 子集加载 43 tools vs 全量 54（r3-pre 实测）；email-inbox off 终表 0 条 / extended 47 条（thread/star/trash 各 7 全带 md）；find-greatest 6 卡带 md；8 页 extended 计数 4~47 无一触 150 | 注入层实测 | security-review-r3-pre §一/E1、review-r4 §3/V6 |

**未核实/勘误标注**：① 任务书所写"单轮约 240s/格"——240s 是 episode 页面预算（agent-e-anatomy.md 对照表口径），实测单格 wall 仅见于 r1 ≈336s/格（round1-report §2），r2-r4 无逐轮 wall 均值披露，按"未核实"处理；② browser 工具集件数 r4-pre 写 15 件、r4-post 写 16 件，本档案取"无 JS-eval 工具"这一两处一致的结论；③ r4 的 email 家族 4/10（round4-report）与 A1 法定预期 3~4 格（plan-r4 §6）为"通过数/家族总数"与"预期区间"两个口径，不矛盾。
