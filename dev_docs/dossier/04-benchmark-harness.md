# 04 — Benchmark 基建与防作弊（面试复习版）

> 覆盖：`record2gherkin/benchmark/`（harness 全套）+ `testzeus_hercules/` 引擎侧 flag 化补丁 + `dev_docs/benchmark/` 全部 25 份文档（plan/spec/review/test-report × r1-r4、八份安全审查、四份 round-report、三份 analysis、baselines）。
> 数字口径：一手报告（round1/2/3/4-report）+ 复审（security-review 各轮 post）为准，两处不一致已逐条对照。审计级精度（行号/字节/哈希/计数过程）已删，业务数字全保留。

## 1. 这一页讲什么 + 本档案复习法

这一页讲三件事：

1. 我把 Farama 的 MiniWoB++（browsergym registry 全量 **125 个任务类**，一类一页浏览器小任务）变成"本地伺服 + 补丁供题 + 页面原生奖励判分"的可复现基准：上游 360 个 HTML 文件整树入库（构建期逐文件 sha256 指纹防漂移），伺服时对 core.js **纯追加**两段补丁——带 seed 自动开局 + 终局同步回传奖励；官方成败以页面 JS 全局 `WOB_RAW_REWARD_GLOBAL > 0` 为唯一权威，**全程无 LLM 裁判**。
2. 围绕"分数诚实"建了**五层防作弊**：判分权威封闭（HUD/重开按钮加固）、收尾五类扫描（只披露不改判）、实验前后双安全审查、口径防灌水（invalid 剔进 clean 口径、480s/多 seed 只进对照臂）、原则性极限披露（奖励端点无防伪造能力）。
3. 四轮迭代 **43.2% → 46.4%（clean 47.8%）→ 47.2% → 61.6%**，每一格都经逐格归因与安全审计背书：r2 修掉自家 5 格判分瑕疵（54→59 回放锁定）、W2 文件工具事件按预登记契约判 10 格无效、r3 堵掉 4 次 scheme 越界与 W2 结构性根因、r4 超额的 +17 格全部有 md 点击日志实证。

**复习法**：先背第 2 节的 60 秒主线，被问"你的评测怎么保证可信"直接整段输出；决策卡记"结论 + 否决理由"，不记实现细节；坑卡按"现象→定位→修复→教训"讲故事；上场前只刷第 5 节必背数字。说不出精确数字就退到口径层面（分母恒 125、单 seed 单次），绝不猜数。

## 2. 60 秒主线

> 被问"你的评测怎么保证可信"，我这么答——可信度建立在三件事上。
> **第一，判分权威封闭。** MiniWoB++ 每个任务页自己会算分，成败只认页面 JS 全局变量 `WOB_RAW_REWARD_GLOBAL > 0`，我不引入 LLM 裁判；agent 看不到分数——计分 HUD 被隐藏、终局后的重开按钮被封死，JUnit 或 planner 自己的判断只记不一致、不改判。
> **第二，口径诚实。** 每轮同表披露四个口径：官方 last-wins（每格只认最后一次提交）作 headline；首实例口径（每格第一条奖励记录）证明重试和重导航没灌水；零重导航子集是诚实性参考（有选择偏差，不上头条）；row-based 披露含重试行的分母。无效格自动移出分子分母形成 clean 口径，r3/r4 官方=clean、invalid=0。
> **第三，五层防作弊。** 判分权威封闭；收尾五类扫描（只披露不改判）；实验前后双安全审查（pre 定级放行、post 全量重放扫描器与三口径复算）；口径防灌水（480s 加时长只进 ablation 对照臂、多 seed 均值与 best-of-3 明确拒绝进 headline）；原则性极限披露（奖励端点没有防伪造能力，如实写进报告）。
> 四轮成绩 43.2%、46.4%、47.2%、61.6%，每一格提升都有日志实证：r2 修掉的是自家 5 格判分瑕疵（54→59 是把口径执行对，不是放宽）；W2 事件按预登记契约主动无效化 10 格；r4 超额的 +17 格全部有 md 点击日志背书。

## 3. 决策速答卡（13 张）

### 卡 1 · 选型：MiniWoB++ 全量 125 格，不挑任务
- 口头答：选 MiniWoB++ 是因为官方 pip 包自带全部 HTML，本地伺服就能跑，页面自己打分免裁判；任务表取 registry 全量 125 类、一个不挑，visual 族也不豁免——挑任务就是口径灌水，"visual 失败就是失败，报告分家族如实呈现"。任务表构建期三重硬校验：125 个、无重复、字典序。
- 否决：WebArena（本机无 docker、要自托管 5 个站点）、WebVoyager（真实外网站点 + GPT-4V 裁判依赖）、Mind2Web（只能测静态子能力）——全部可行性否决；豁免 visual 被明确拒绝。
- 出处：plan.md、spec.md。

### 卡 2 · vendored 供题 + 补丁式伺服
- vendored（把上游文件整树原样复制进仓库）：360 个文件一次性入库，PROVENANCE 记录逐文件 sha256 指纹、单测逐个核对防漂移；许可合规（miniwob 1.1.0，BSD-3-Clause）；miniwob/browsergym-miniwob 不进 pyproject——离线可复现是红线，在线拉取/CDN 被否。
- 补丁式伺服：请求 core.js 时返回"vendored 原文 + 纯追加补丁"，磁盘字节从不改写；纯追加避免文件内替换的锚点漂移，单测逐字节断言前缀一致。页面侧改源码违反 vendored 不改写，被否。
- seed 派生：seed 由 exp_id 与 task_id 拼串取 sha1 派生——同格恒等（可复现）、跨格互异，全表枚举单测锁定；换 exp_id 即换 seed，所以 480s 对照臂复用同 exp_id、只换输出目录来保逐格配对。
- 补丁五段（追加顺序）：原文 → 自动开局 → 奖励钩子 → 终局信号 → 完整性信标。
  - 自动开局补丁（恒开）：50ms 轮询，三条件缺一不可——任务旗标为真、文档加载完成、封面层存在；开局成功才停轮询，失败重试约 200 次（~10s）后显式置开局失败标记；URL 不带 seed 就不自动开局（不干扰人工调试）。
  - 单次开局变体（--single-start，默认 off）：由自动开局补丁程序化派生、恰三处不同；被拦截的加载同样 stub 掉 HUD 刷新与重开入口 + CSS 隐藏，且不产生任何奖励记录。
  - 奖励钩子补丁（恒开）：终局时用同步 XHR 把 {path, seed, reward, raw, done, reason, ts} POST 回伺服器，try 全包"绝不干扰 episode 本身"。
  - 终局信号（--terminal-cue，默认 off）：终局后在页面左下角生成恒为"EPISODE ENDED"的中性文本——无数值无成败、不回传、不用 display:none（文本提取必须可见）。
  - 完整性信标（r4，orchestrator 恒开、无 CLI）：页面加载时发一条只写不读的 fetch——query 没带 seed 记 offseed、带了记 epstart；服务端 append-only 落两份 jsonl，恒返回 204 空 body、零回读。
- golden 纪律：全部 flag 关闭时补丁输出与 r1/r3 版逐字节一致（golden 基准锁定）；r4 信标恒开使 core.js 交付字节与 r3 不同，但判分中立（披露口径）。
- 出处：spec.md、test-report.md、PROVENANCE.md。

### 卡 3 · goal 预读与 Gherkin 中转
- 口头答：harness 用同一个 seed 先打开同一页面，把任务指令读出来"烤"进 Gherkin——Given=页面 URL、When=指令原文逐字、Then=意图陈述。同 seed 确定性保证预读到的就是执行侧会看到的指令；测的是"执行内核能力"（自然语言→Gherkin→语义执行），Gherkin 中转本身是口径之一、随报告披露。预读页不触发终局、零奖励记录（实测预读前后记录数不变）。
- 否决：把 goal 塞环境变量绕过 Gherkin 被否；任务特定提示（如日期示例值）违反判分中立被否。
- 出处：spec.md、goal_reader.py、plan.md。

### 卡 4 · 判分权威与奖励回传（含 H1 加固）
- 唯一权威：官方成败 = 页面原生 `WOB_RAW_REWARD_GLOBAL`（页面自己算分的 JS 全局变量）> 0，无 LLM 裁判；JUnit/planner 自判不一致只记 disagreement 不改判。页面终态就是任务真值，raw>0 机械可判；引入 GPT-4V 裁判会叠加主观性与外部依赖。
- 判定优先级链（C1a 后固定）：runner_status 为空 → no_goal（没执行，最高优先）；raw>0 → official_passed（页面正奖励压倒 infra 状态）；runner_status 为 timeout/no_junit → 原样保留（可进重试池）；reward 缺失 → no_reward；其余 → official_failed（raw<=0，含页面自己 240s 超时的 raw=-1）。
- H1 加固：开局成功后把原生计分 HUD 隐藏（display:none + stub 掉刷新与重开两个函数）；不能用 remove()——终局还要写 episode-id 子元素，移除会抛 TypeError；加固只封观察/点击路径、不封 JS 执行能力（如实披露）。
- 奖励回传无兜底：同步 XHR 保证页面销毁前送达；取不到（端点 404）如实记 no_reward 官方失败——绝不"重开同 seed 页再读"，重开会重置状态、读到的不是终局值，机制上不可行。页面 240s 自超时会先于引擎 600s 超时回传（raw=-1、reason=timed out）。
- 为什么判分可信（防伪造边界）：奖励端点无防伪造能力是**原则性极限**——任何浏览器侧密钥都会出现在伺服的 core.js 里，机制上不可根除，只能检测+披露；四轮全量 reason 域/done/raw 自洽/孤儿记录/快通检测全零，奖励端点不存在可供页面读回的查询面，引擎也无 JS-eval 工具——伪造的前提（页面侧 JS 执行能力）不存在。
- 出处：spec.md、plan.md、round2 系文档、miniwob_server.py。

### 卡 5 · 三口径与 clean 口径自动化
- 四口径矩阵（每轮同表披露）：

| 口径 | r1 | r2 | r3 | r4 |
|---|---|---|---|---|
| 官方 last-wins（headline） | 54/125=43.2% | 58/125=46.4%（clean 55/115=47.8%） | 59/125=47.2%（=clean） | **77/125=61.6%**（=clean，invalid=0 连续两轮） |
| 首实例（不给重试/重导航任何信用的上界） | 59/125=47.2% | 58/125=46.4%（与官方完全相等，桥接双向为空） | 58/125=46.4%（差 1：click-menu-2 合法 infra 重试翻正） | 76/125=60.8%（差 1：click-dialog-2 重试翻正） |
| 零重导航子集（诚实性参考，有选择偏差不可作 headline） | 37/60=61.7% | 55/102=53.9% | 57/87=65.5% | 74/108=68.5% |
| row-based（披露口径，分母含重试行） | 54/130=41.5% | 58/136=42.6% | 59/137=43.1% | 77/131=58.8% |

- 演进三里程碑：r1 官方比首实例差 5 格（59 − 6 反向自伤 + 1 重试翻正 = 54），其中 5 格是"页面已过被 infra 判负"；r2 C1a 修复归零（回放锁 54→59、+4.0pp 锁定值）；r3 起官方=clean（invalid 格同时移出分子与分母，overall 官方口径零改动），r4 同值。
- clean 自动化（E4）：invalid 由代码产出而非人工剔除（r2 是磁盘取证人工剔除），并补一条——invalid_reason 行不进重试池，防"重试洗干净"弱化 clean。
- 出处：round1-report.md 至 round4-report.md、spec-r3、metrics.py。

### 卡 6 · 收尾五类扫描（+r4 第 6 类）
- 口头答：每格跑完对已落盘日志做五类检查；扫描器无网络无浏览器、可独立复算（复审把四轮全部行 130/136/137/131 重放，零 mismatch）。结论只写行内三键——`task_url_navigations` / `flagged` / `invalid_reason`——**绝不改判**：flagged=披露但保留官方奖励，invalid_reason=该格无效。
- 五类小表：

| # | 检查 | 判定 |
|---|---|---|
| 1 | 重导航/file:// 逃逸（任务开页日志与本格 seed 同行的计数；全文出现 file://） | 计数>1 → flagged；file:// → invalid（安全事件 file_url_navigation） |
| 2 | 沙箱调用（真实调用标记两行 + r3 起被拦标记也算尝试；绝不匹配每 run 必有的工具注册行——匹配注册行即 100% 误报） | 命中 → invalid（sandbox_tool_invoked） |
| 3 | 奖励行数与 reason 域自洽（本格时间窗内记录数超过页面加载数；reason 出合法域；done 非真或 raw 非数字） | 任一 → flagged（启发式非不变量：终局后合法追加可能） |
| 4 | 孤儿记录（记录时间戳落在所有 attempt 窗口外；无时间戳按计入处理，宁多勿漏） | 四轮全零（r2 147 条、r3 163 条全在窗内） |
| 5 | 快通检测（<15s 官方通过）+ 计时矛盾（页 240s/引擎 600s） | 四轮全零 |
| r4 增 | off-seed 信标归因 + 零事件细分：offseed 命中 → off_seed_navigation；零奖励记录 → zero_reward_events，若启动信标也零命中则改标 episode_never_started（替代不叠加）；全程零导航+超时 → flagged | 只披露永不 invalid（flagged-only） |

- 实现要点（也是坑）：基础设施重试复用同一 run 目录与 (path,seed)——append-only 日志上"本格记录"必须按 attempt 时间窗过滤，不带时间窗就会把历史记录误判为异常；无时间戳按计入处理（宁多勿漏）。
- 辅助不变式：flag 原因恒伴 invalid_reason 为空；一格可带多个 flagged 原因（字段级合并）；日志缺失=中性结果（无证据不下结论）。
- 出处：spec.md、test-report.md、orchestrator.py。

### 卡 7 · 双安全审查流程
- 每轮实验前（pre）+ 实验后（post）各一份独立审查，**与实验执行者分离**；pre 定级 GREEN/YELLOW/RED 决定能否开跑。
- 实证：r1-pre YELLOW 拦下一批问题后再跑全量；r2-pre 两个 P0（planner 模型未探测、runbook 命令带残留参数）在开跑前修掉；post 全量重放扫描器与三口径复算。单次审查被否——W2 事件证明 pre 预警可成真，post 必须逐格取证。
- 出处：八份 security-review 文档。

### 卡 8 · 治理包：r2 C 系列 + r3 E 系列
- 终局信号 C2：r1 自造盲区——HUD 被加固隐藏后 agent 找不到成功指示必然空转（multi-layouts 页面过后 46/64 轮空转；23 格平均空转 273s）。修复=终局后注入恒定中性文本"EPISODE ENDED"（无数值无成败、不回传）。原版裸 HUD 数值被否（泄漏分数）；引擎侧 stop-at-terminal 被否（与 C2 重叠且是口径变更：last-wins 会变 first-terminal-wins）。
- 单次开局 C3：sessionStorage 每 tab 只自动开局一次。r1 实证 61/125 格重导航≥2、93%（249/267）走同 tab——重导航=免费重置 240s 页钟+作废失败提交（email-inbox-forward 16 次导航把页钟续到 +584s）。拦截态同样加固，否则 START 可点会重开随机实例经 last-wins 洗白（见坑卡 4）。
- extra_tools 治理：r2 C6 用 --extra-tools 开闸 + 2 格冒烟先行（kill-switch（预先登记的验收/止损判据）=选择器解析成功）；W2 事件后 r3 升级三道防线——E1 子集加载（默认只放 drag_and_drop_tool，persist/recall/augment/clipboard 结构性不可加载；子集注册 43 个工具 vs 全量 54、persist/recall/clipboard 零注册）；E2 调用日志行入扫描（命中即 invalid）；invalid 行不进重试池。r3/r4 实效：文件工具标记 0/137、0/131，盘上零 agent 产物。
- scheme 白名单 E3：只放行任务页 URL 形态；连续两轮 P1 未做后才补上，r3 实拦 4 次（两格 javascript: 伪协议、两格 view-source: 伪协议）。
- 模板上下文 C7：Gherkin 固定 4 行注释（指令同文在页面、单页勿重载、页面无成功提示）+ 终局信号开启时联动第 5 行。动因：r1 有 25 格页面 240s 干等死亡、58/121 格首轮计划含"verify the task completed"空转步。任务特定提示被否（泄漏任务语义）；When 指令原文逐字不动是红线。
- 沙箱关停 E5：两轮合法使用为零、越界尝试 1 格 → 沙箱函数体首行拒绝（在读取租户之前），被拦尝试计入扫描标记。极早崩留痕 E6：预读失败也写 goal_read_error.log 留痕。
- 否决备案：off-seed 的阻断/自动补齐被否（掩盖 planner 缺陷且需逐格传 seed）——只做检测披露。
- 出处：plan-r2、spec-r2、plan-r3、security-review-r2-post。

### 卡 9 · 延迟包与配速熔断
- 延迟包 C4（引擎侧，独立 flag）：状态刷新守卫 markers_only、nav 轮数 50→30、步级预算 120s、prompt 包（撤销"刷新教学"——引擎 prompt 原本在主动教重导航、planner 瘦身、输出克制）、LLM 请求超时 90s + 重试 2。动因 r1 实证：超时格平均 51.3 个 executor 轮、轮间 9.0s、断裂 25.1 次/格、completion p90 1998 tokens；每步 10-40s 的 LLM 延迟是主体。
- 分角色路由 C5：planner/helper 用旗舰模型、nav 用快档——executor 轮是延迟主体，planner 每格仅 3.2 轮非瓶颈（planner 升级被拒，无证据）；key 只走环境变量四键注入（含 planner 专用的 OPENAI_API_KEY）；生成文件永不含 key、写盘函数对含 key 配置直接抛错（红线）。
- planner 双层 150s 超时（R3-2）：25/41 超时格至少 1 次 planner 90s 超时；90s 是双层硬墙（外层 asyncio.wait_for + 内层 provider 层兜底同喂一个 env）——修复是新增 planner 专用超时同源喂两层；r3 实证超时 41→21 的主要来源。全 agent 超时重试被拒（temperature=0 重发同样慢，只把最坏暂停翻倍吃页钟）。
- nav max_tokens 768（R3-1）：r2 实测 10.3% 的轮吃掉 38% executor 墙钟（肥尾 2000-3400 token）；r3 上线后被 post 审查实证 no-op（GLM 端点把 max_tokens 当参考值，见坑卡 16）——r4 headline 摘除、CLI 保留不删。
- 运维三件套（r2 C1b/c/d）：熔断器——attempt 级三条件（infra 状态 + 用时<30s + 402/余额不足/连接类标记）命中即逐出重试池，run 级连续 2 格中止 stage（已写行保留）；1-token 预检硬 gate（key 不通 exit 3，换 provider 前必须过）；attempt 分目录日志（重试不再互相覆盖）。
- 配速与断点续跑：GLM coding plan 5h 窗口 → --max-cells 分段（r4 配速翻倍、单段可 140 格级）+ 重跑同命令自动跳过已有行 + 429 熔断标记（标记串特意不用裸"429"，会误命中时长数字）；results.jsonl 逐行追加落盘、manifest 收尾写。并发/分布式编排被否（Out of Scope）。
- 出处：plan-r2、spec-r2、plan-r3、round3-report.md。

### 卡 10 · 计时与 seed 口径纪律
- 240s 页面计时（原生 10s 的 24× 放宽，参数化）+ 单 seed 单次 + 引擎 600s，三条全部随报告原样披露——10s 对"每步 10-40s LLM 延迟"的 agent 不可用。
- 480s 只进 ablation 臂（只作对照分析、永不进主成绩）：数字永带 480000ms 标注双列呈现、禁止合并分母——改时长就是改任务难度，进 headline 会永久污染跨轮可比性；seed 派生不变可逐格配对归因。r3 把 480s 预期校准为 +1±1，r4 进一步降级为默认不做。
- 多 seed：best-of-3（取最好成绩）被拒（口径掺水）；majority-of-3（三次投票，只治测量）设计好待批。单 seed 是已知口径限制：纯噪声翻转约 12.8%、噪声带 ±10 格随表披露；r4 审计把 +18 格归因到日志实证后，预置的门控未触发。
- 出处：spec.md、plan-r3、plan-r4。

### 卡 11 · 引擎侧修复与扩展包（r3 drag 透传 / r4 三件）
- drag 选择器透传（R3-3）：拖拽家族 r2 全灭、61 次调用 0 次解析成功——包装层"好心规范化"把一切输入强行包成 md 属性选择器，而拖拽源根本不在 md 表里（详见坑卡 18）；r3 改透传 + kill-switch（9/9 格解析成功），实绩 0/9→3/9。放弃案（移除工具）= 8 格天花板永久归零被弃；坐标拖拽（canvas/SVG 无 DOM 锚点）列射程外。
- md-extended（R4-A）：email 家族 10 格两轮 0/20、死法唯一——元素注入了 md 但被终表白名单整批丢弃，agent 自述"Interactive elements list is empty"。修复=终表过滤器扩展：8 个标签（div/span/li/tr/td/th/img/label）+ 4 个角色（row/cell/listitem/img）+ identity 门控（七种标识属性之一，杜绝无标识灌水）+ class 锚 + 150 条硬上限（超限打截断告警、无哨兵节点）。实测注入层已被 cursor:pointer 启发式覆盖（thread/star/trash 各 7 个全带 md）。点击工具选择器透传被拒（爆炸半径=所有任务每次点击，列 r5 候选）；交互判定大改被拒（只授权最小修补且以真浏览器断言失败为前提，实际未触发）。
- verify-before-done（R4-B）：planner 产出"terminate=yes 且 is_passed=true"且未核验过 → 路由进固定核验步（只许取页面文本与交互元素、报告可见状态，禁读 reward 全局——结构面也封死：JS 全局不落 DOM、无 JS-eval 工具），再回 planner 二次裁决，核验轮数硬上限 1。动因：完成幻觉（planner 自认为完成但页面没过）24 格、占官方失败 57%，另有 3 格同病根——prompt 层断言纪律已被 r3 实证不足（假阳性 17→24 不降反升）。实绩 0 分贡献，r5 起定位为诚实性组件。
- 完整性信标（R4-E）：off-seed 探测 + 启动信标，恒开、判分中立、flagged-only——r3 暴露 1 格 off-seed + 6 格零奖励事件，两种零事件形态（"从未启动"vs"已启动无奖励"）需信标细分。seed 值校验缺失成 r5 备忘（wrong-seed 盲区，见坑卡 20）。
- 出处：plan-r3、plan-r4、spec-r4、review-r4。

### 卡 12 · 预算护栏与预期纪律
- 预算：单次调用 cap **144 次**硬拦 + manifest 累计入账（cumulative_hercules_used / cumulative_over_cap）+ Σ>144 量化披露（纪律条款不阻断——护栏强度=单目录自动化审计+跨目录人工纪律）。r1 锚：单价上限 4.7 万 token/$0.13 每次执行、pilot ≤$1.56、full ≤$16.90、红线 142 次/$20。复审指出 cap 是单次调用口径：结构性最坏 146=16+130 已量化披露（+2 全为 pilot 域重试，≤1.4%）；拆 exp-id 分窗绕账被纪律条款明令禁止。
- 预期纪律：每轮预期数字逐格可推演、保守档必须可复算——两次拦下不可信预期（r3 保守档构成列差 2 无注释；r4 把"~0.7 交互正确率"当通过率凑出 +8，法定通过率约 0.4——两个分母混用）。"每个数字诚实可披露"是最高原则，区间拍脑袋被否。
- 出处：spec-r2、plan-r4、review-r3、review-r4。

### 卡 13 · flags 清单与 off 基线纪律
- 纪律：一切行为变更独立 flag、默认 off = 上轮行为逐字节复现（r4 例外口径：信标恒开使 core.js 交付字节与 r3 不同，判分中立，披露）。引擎回归 12 条全绿（改动前后 stash 对照各跑一次一致）；record2gherkin 测试链 75→117→427→456→492 全绿。

| flag | 轮次/用途 | 默认 | 关键实证 |
|---|---|---|---|
| --terminal-cue | r2 终局中性信号 | off | cue 的 300 处提及全部晚于终局记录；失败格中位空转 257s，未借 cue 提前放弃 |
| --single-start | r2 每 tab 单次开局 | off | 拦截态点击零记录；force_new_tab 18 次为残余缺口，计数披露 |
| --role-routing + --nav-model | r2 分角色路由（生成永不含 key 的路由配置） | off / deepseek-flash | 环境变量四键（含 OPENAI_API_KEY planner 专用）精确断言 |
| --extra-tools | r2 开闸；r3 起默认子集 drag_and_drop_tool | off | 子集 43 vs 全量 54，persist/recall/clipboard 零注册 |
| --template-notes | r2 Gherkin 固定注释 4 行（+cue 联动第 5 行） | off | 两变体快照；cue 关时第 5 行不得出现 |
| --latency-env | r2 延迟五键（90/2/markers_only/30/120） | off | 五键值逐字节回归断言（r3/r4 延续） |
| --provider {deepseek,glm} | r2 GLM 接入（glm-5.3-flash / glm-5.3） | deepseek | 默认路径逐字节不变，非默认才追加披露字段 |
| --smoke-cells | r2 pilot 专属冒烟格 | 空 | 不进重试池、计入预算；full+smoke 组合硬报错 |
| --max-cells | r2 5h 窗口配速 | 全量 | 断点续跑=重跑同命令；r4 单段可 140 格级 |
| --nav-max-tokens | r3 收紧 4096→768 | 0=off | r3-post 实证 no-op；r4 headline 摘除（manifest 键缺省） |
| --planner-timeout | r3 planner 150s 双层同源 | 0=跟随 90s | 四断言锁定；超时 41→21 主要来源 |
| --extra-tools-modules | r3 子集白名单 | drag_and_drop_tool | all 须披露记全量；空串直接报错 |
| --disable-sandbox | r3 沙箱关停（函数体首行拒绝） | off | 被拦标记入扫描 → 尝试即 invalid |
| --assert-discipline | r3 断言纪律 prompt | off | 实效不足（17→24），r4 由路由层接管 |
| --md-extended | r4 终表扩展（8 标签+4 角色+identity 门控+class+150 上限） | off | off 档逐字节 golden；真浏览器实测 email-inbox 行/图标入表、star/trash class 互异 |
| --verify-before-done | r4 verify 路由（硬上限 1 轮、核验步禁读 reward） | off | 路由五态 + 全链断言（执行序 planner→verify→executor→planner→END）；off 时图里无 verify 节点 |
| offseed 信标（无 CLI） | r4 orchestrator 恒开 | 恒开 | 关闭分支与 r3 版逐字节相等（golden） |

- 出处：spec-r2、spec-r3、spec-r4、test-report.md。

## 4. 踩坑故事卡（26 张，覆盖原 28 坑）

### 坑 1 · W2：extra_tools 全量加载的静默文件写（r2，本档案第一张完整故事卡）
- **经过**：r2 为救拖拽家族用 --extra-tools 开闸，开的是全量 12 个 extra_tools（含 persist/recall/augment_findings 的任意路径文件读写）。agent **自发**在 10 个格（7.4%）实际调用了 persist_findings，其中 **9 格成功写入 10 个"静默"产物文件**——成功路径零日志行（文件工具三个函数体内没有 logger），扫描器全程零告警；仅 1 格（ascending-numbers）留下可见 traceback。**3 格是通过格**（focus-text、multi-orderings、click-shape）。这正是 r2-pre 预测过的检测缺口（W2）——预警成真。
- **定性（为什么不是作弊）**：写入全是任务诊断笔记；奖励判定权威在伺服器内存，盘上 rewards.jsonl 仅扫描用，机制上无改判通道；产物零密钥材料。
- **题眼——预登记契约**：实验前就登记过的规则"invalid_reason 非 null ⇒ 该格判无效+记安全事件"是 r1 既有约定，事件一成真就机械执行：**10 格判无效，headline 58/125 → clean 55/115=47.8%**，官方数字作披露数并列。事后没有讨论空间——防作弊的底气不是"agent 不作弊"，而是"越界发生时有预登记的处理规则"。
- **r3 结构性根除（三道防线）**：子集加载（persist/recall 结构性不可加载）+ 调用日志行入扫描（命中即 invalid）+ invalid 行不进重试池；r3/r4 该标记全零、分母回归 125。
- **教训**：开一个开关=加载它拖带的整个世界；检测缺口要在事件成真前按"静默成功不可见"的最坏假设评估。
- 出处：security-review-r2-pre/post、round2-report.md、plan-r3。

### 坑 2 · r1 判分瑕疵：5 格页面已过却被判负（审查抓到自家 harness 的错）
- 现象：r1 官方 54 vs 首实例 59——5 格的奖励记录里 raw>0 在时间窗内到达、官方却 timeout/no_junit（login-user-popup/multi-layouts/use-colorwheel-2 被 600s 超时覆盖；click-pie/click-collapsible-2-nodelay 被 402 崩溃 no_junit 覆盖）。
- 根因：判定优先级把进程级 infra 状态排在了页面奖励之前，违反"页面奖励为唯一权威"的口径——口径写进了 spec，实现没执行对。
- 修复：C1a 优先级重排（raw>0 压倒 infra）+ runner_status 审计键 + r1 数据回放锁（5 格翻正、email-inbox-delete 反例维持、54→59 单测锁定）；不新增取数通道（仍取 /latest），不给洗白留口子。
- 教训：口径定义与代码实现要逐条对拍；审查和实现分离+双审是体系骨架——这是"审查发现自己人问题"的第一例。
- 出处：analysis-r1-architecture、review-r2、test-report.md。

### 坑 3 · email 页钟续命与 last-wins 自伤（r1，email-inbox 双案，完整卡）
- 现象 A（页钟续命）：email-inbox-forward 一格 16 次重导航，每次重开都重置 240s 页钟——页钟被续到 +584s，失败提交也被作废，agent 靠重导航"续命"空转。r1 全画像：61/125 格导航≥2（最多 15 次），93%（249/267）走同 tab。
- 现象 B（自伤反例）：email-inbox-delete 首实例已拿 raw=1，之后 8 次重导航，终局 raw=−1——last-wins 认最后一次，官方判负：反向作废了已到手的通过。
- 修复：判定不修——该格必须仍判负，留作判分正确性的对照锚（"不该被救回的格"比"该被救回的格"更能证明规则正确）；通道级修复是 C3 单次开局 + 拦截态加固（见坑 4）。
- 教训：last-wins（每格只认最后一次提交）是双刃剑——既防"刷多次取最好"，也会被重导航利用；通道封堵比结果修补更根本；修复类改动必须带反例锁定。
- 出处：security-review-r1-post、plan-r2、spec-r2。

### 坑 4 · 拦截态的 START re-roll 复活（review-r2 M2）
- 现象：单次开局的早退路径若不加加固，被拦截页面上 START 覆盖层仍可点——点击经页面脚本动态取开局函数，重开一个**没有 seed** 的随机实例并 POST 新记录、经 last-wins 覆盖本格终局（洗白通道）。
- 根因：加固几行原本只写在"开局成功"分支里，漏了"被拦截"这个新分支。
- 修复：拦截分支同样 stub 三函数 + 注入 CSS（双保险，不依赖 onclick 绑定形态）；单测断言拦截态点击零记录。
- 教训：堵通道要堵到"被堵状态"本身——每个新分支都是新的攻击面。
- 出处：review-r2、spec-r2、miniwob_server.py。

### 坑 5 · HUD 泄漏与 START 重开按钮（r1-pre V1/V2，加固的由来）
- 现象：页面文本提取视角里能看到原生 HUD（"Last reward: 1.00"直接可读）；终局后 START 覆盖层重现，一次普通 DOM 点击即重开同 (task,seed) 新实例——同 seed 下随机序列前进，指令人名从"Joye"变"Michel"，等于免费 re-roll。
- 根因：Hercules 用自己的 DOM/文本提取，不受 MiniWoB/BrowserGym 的剔除清单约束——对齐基线"agent 看不到 HUD"的保证在我们口径下不成立（观察空间强于基线）。
- 修复：H1 三行加固（HUD 隐藏 + stub 两函数，用 display:none 而非 remove——终局还要写 episode-id 子元素，移除会抛 TypeError，隐藏语义即够，最小干预）；实测对 START 覆盖层的定点点击 1.2s 超时不可达。
- 教训：换观察框架时，基线的"看不见"保证不随行——要按自己的提取路径重新审计泄漏面；隐藏语义即可时不要破坏结构依赖。
- 出处：security-review-r1-pre、test-report.md。

### 坑 6 · 终局后 endEpisode 复触发的"形似洗白"记录（r4）
- 现象：rewards.jsonl 出现 book-flight、click-menu 的"−1 之后显示分变 +1"记录（r4 实证 4 条，click-color 多条同值），人工审计极易误判洗白。
- 定位：终局后 EP_TIMER 守卫会短路重调用的判分（raw 从此封存），但奖励钩子对这种短路重调用照样 fire——POST 里的 reward 是新实参（显示参数），raw 是封存原值。"翻转"只发生在显示字段。
- 结论：不是 bug——判分在两个方向上都抗终局后翻转；两格官方判负（方向是反洗白）。审计纪律：人工核查必读 raw 字段而非 reward，否则会把这两格误判成洗白。
- 教训：判分链的"权威字段"要写进审计手册。
- 出处：security-review-r4-post。

### 坑 7 · 开局信号两坑：auto-start 条件与观测量（r1）
- auto-start 只等任务旗标不够：该旗标默认值本来就是 true，只等它会在重页面上抢在 onload 前开局——book-flight 抛 ReferenceError 后永久失去自动开局。修复：轮询条件改三条件（旗标 + 文档加载完成 + 封面层存在），且只在开局成功后停止轮询。教训：布尔旗标的默认值本身就是坑——要等的是"页面就绪"不是"旗标翻转"。
- 回合号当开局信号：用"episode id>0"判断开局会全盘误判——该变量只在终局函数里自增，开局后仍为 0。修复：开局证据改用组合观测量（内部起点非空 + utterance 非空 + 页面预算=240000 + 无开局失败标记）。教训：选观测量前先读变量的写点（谁在什么时候写它）。
- 出处：spec.md、review 必改条目、test-report.md。

### 坑 8 · goal 预读类完整卡（双类型 / 非空等待 / 超时预算）
- 双类型：页面取指令接口多数任务返回字符串，email-nl 类返回 dict——按字符串处理会拿到对象。修复：非字符串取 utterance 键（与 browsergym 同规则），空值即预读失败。
- 非空等待（r2 修复）：utterance 等待条件从"存在即返回"改为"非空才返回"，no_goal 格从 r1 的 4 格降到 r2 的 1 格。
- 超时预算：等任务就绪 15s、等 utterance 10s、总预算封顶 30s；utterance 超时先查开局失败标记（显式报因）再抛。
- 其余：goal 做净化（引号替换、空白折叠，表驱动 7 组单测）；预读器读毕立即关浏览器、不触发终局（预读零奖励记录，r1 实测预读前后记录数 6→6）；失败记 no_goal（预读失败、根本没执行）不重试不阻塞，r3 起写 goal_read_error.log 留痕。
- 教训：预读器要对上游返回类型做枚举处理而非假设。
- 出处：spec.md、goal_reader.py、round2-report.md。

### 坑 9 · 极早崩格无日志 + 重试覆盖（r2）
- 现象：r2 一格 0.6s 即崩（no_goal）且盘上无任何可复核证据；更早 r1 还发现重试格 attempt1 日志被 attempt2 覆盖、不可复核。
- 根因：预读失败先于子进程启动，没有自己的留痕；日志按格覆盖不分 attempt。
- 修复：attempt 分目录（每次尝试的 stdout.log 独立留痕）+ 预读失败单独写 goal_read_error.log（写失败仅 warning 不阻断）。
- 教训：可观测性要覆盖"进程都没起来"的最短路径。
- 出处：r2-post、security-review-r2-post、spec-r3。

### 坑 10 · 任务表枚举与幽灵页面（registry 类）
- 按 `all` 枚举任务直接 TypeError——`all` 是子模块不是列表；改用包级任务列表，构建脚本带 125/唯一/存在性三重硬失败校验。
- registry 里唯一 visual 任务是 visual-addition（并不存在 visual-maze，HTML 树里也没有该文件）——子集若含不存在任务启动即报错；另 5 个 vendored 页面未被 registry 引用（如 book-flight-nodelay.html），任务表只做存在性校验、不影响口径。
- 教训：registry API 的"直觉名"要先试；构建期硬校验是免死金牌。
- 出处：spec.md、test-report.md。

### 坑 11 · 沙箱检测标记串与合法域漏项（r1 加固的两处字面偏差）
- 现象：安全审查报告写的调用标记若照抄实现，会 100% 误报——报告写的串与代码真实调用标记不一致，而工具名字样只出现在每 run 必有的注册日志里（匹配注册行即全误报）。
- 同类偏差：报告给的 reason 合法域漏了两类——复核 vendored 树发现 unicode-test 页面有 'Cool!' 和 'You clicked on ' 前缀两类合法 reason，不补会把全量里的合法行误判。
- 修复：实现用逐字回查的真实调用标记 + 单测含"只有注册行→不 flag"反例；reason 合法域补全（空串、timed out、Cool!，及 You clicked on 前缀）；两处偏差都回写 spec。
- 教训：检测器的标记串必须逐字来自现场代码/数据，不能按记忆转写。
- 出处：test-report.md、spec.md。

### 坑 12 · 预检 transport 与生产不同栈（r2 C1c）
- 现象：按 spec 字面用 litellm 裸模型名探测，可用 key 会被误判失败并 exit 3——litellm 对自定义端点要 provider 前缀 + api_base。
- 修复：钉死与引擎同栈的 ChatOpenAI（max_tokens=1、不重试）做 1-token 预检。
- 教训：gate 的探测路径必须与被 gate 的生产路径同栈，否则 gate 本身成为事故源。
- 出处：review-r2、spec-r2、preflight.py。

### 坑 13 · planner key 断链（r2 C5 上线即崩）
- 现象：分角色路由 headline 在引擎启动即崩、每格 no_junit。
- 根因：PlannerAgent 裸构造 LLM 客户端（不经统一入口），langchain-openai 新版（≥1.1.14）只认显式参数或 OPENAI_API_KEY、不读我们的 MODEL_API_KEY；且配置文件加载成功后 env provider 完全不注册（File>Env 是源级覆盖）——key 回退链覆盖面被高估。
- 修复：child 进程四键注入（配置路径、REF_KEY、MODEL_API_KEY、OPENAI_API_KEY 后两者同值）。
- 教训：多配置源叠加时，"谁在什么条件下还读 env"要追到构造函数级。
- 出处：review-r2、spec-r2。

### 坑 14 · runbook 陷阱：provider 切换后残留旧参数（r2-pre W12/W1，两个 P0）
- 现象：照旧命令跑会把 deepseek 模型名路由到 bigmodel 端点 → 每格 nav 调用失败，全量报废级失误；另一 P0：GLM+分角色路由时 planner 大模型未过预检（probed 列表里只有 glm-5.3-flash，planner 的 glm-5.3 没探到）。
- 修复：r2-pre 给出正确命令（GLM 下不传 nav-model，默认 glm-5.3-flash）+ 预检补探测，开跑前修掉。
- 教训：实验命令是活文档，provider 切换时逐参数过一遍。
- 出处：security-review-r2-pre。

### 坑 15 · max_tokens 双坑：死代码 + no-op（R3-1 全案）
- 第一坑（死代码，review-r3 M1）：按 spec 原文"仅当未设置时注入"实现，768 上限永不生效——引擎链路上参数会被两段 adapt 调用注入 4096 兜底，"未设置"恒不成立，headline 会在杠杆静默失效下跑完；审查用注入链证伪了"r2 无上界"立论。修复：显式设置时无条件覆盖，断言改为"未设→4096（真实基线）、设 768→768"，语义改写为"把 4096 收紧到 768"。教训：死代码比没有代码更危险（测试绿而生产死）；改上限类参数前先证明"现状值是什么"。
- 第二坑（no-op 实证，r3-post）：live 实验发现 GLM coding 端点把 max_tokens 当参考值——768 限制下单轮仍产出约 13.9k 字符、usage 报 **19,349** completion tokens；数据面 4,213 个 executor 轮 **0 轮恰等于 768**、215 轮超 768（max 3,918），与 r2 分布（250/3,721 超 768、max 3,402）同质。r3 报告与 manifest 修正为 no-op、r2→r3 差异不得归因于它；r4 headline 摘除该 flag，r5 前置=换 provider 先做 max_tokens 尊重性 live 预检。教训：flag 在位≠flag 生效，干预生效要用端到端证据关账。
- 出处：review-r3、security-review-r3-post、round3-report.md、plan-r4。

### 坑 16 · planner 超时只改一层，150s 兑现不了（review-r3 M2）
- 现象：只改外层 wait_for，>90s 的单次 planner 生成仍被 provider 层 90s 掐断（随后触发 SDK 重试链），"150s 内完成即活"不成立。
- 根因：90s 是双层墙（外层 asyncio.wait_for + ChatOpenAI 自身 timeout 兜底同喂一个 env）。
- 修复：新增 planner 专用超时同源喂两层；off 时两 helper 同值返回= r2 逐字节。r3 实效：超时格 41→21 的主要来源。
- 教训：超时链要沿调用栈数层数，改外层不改内层=白改。
- 出处：review-r3、spec-r3、round3-report.md。

### 坑 17 · drag md 包装 bug：61 调 0 中（R3-3）
- 现象：拖拽家族 r2 全灭——61 次 drag_and_drop 调用 0 次解析成功。
- 根因：包装层"好心规范化"用子串判断选择器形态、判断失误，把一切输入强行包成 md 属性选择器，而拖拽源（drag-handle/SVG shape）根本不在 md 交互元素表里——agent 无正确答案可传。
- 修复：r3 选择器透传（css=/xpath=/text= 等显式形态原样传，裸数字才包 md，未知形态三候选兜底）+ 工具描述更新；kill-switch 判据=源元素解析成功≥1，实绩 0/9→3/9、9/9 格解析成功（三格耗时 147s/125s/505s）。
- 教训：工具包装层"好心规范化"是最大的静默故障源。
- 出处：plan-r3、security-review-r3-pre/post。

### 坑 18 · compact 剥 class → star/trash 同形不可辨（review-r4 MF-1）
- 现象：extended 收录的图标节点经 compact 后与 trash 完全同形（只剩 md 号、标签、name），star-reply 类任务只能盲试——按字面写的断言必然失败。
- 根因：compact 的字段白名单不含 class/id，而 spec 同时要求"compact 逐字节保留"——图标无文本、class 是唯一可辨别语义锚，两层规格自相矛盾。
- 修复：字段白名单无条件增 class（off 模式抓取的属性表本就无 class，off golden 逐字节不受影响，断言锁定）。
- 教训：上层扩展与下层压缩的字段契约要一张表对齐；审查用真浏览器实测（email-inbox 带 seed 实开）拿到的同形证据比读代码有力。
- 出处：review-r4、spec-r4、test-report.md。

### 坑 19 · wrong-seed 盲区：过滤有效 ≠ 检测完备（r4）
- 现象：r4 实发 2 格 3 次 wrong-seed 加载（click-dialog-2 att1 的 seed=78425566≠本格 786425566；count-sides 的 seed=999/1），off-seed 信标对"错值"静默——它只判 query 有无 r2g_seed，不比对派生值。
- 兜底：own-seed 过滤（取奖励时按任务页+seed 双匹配）生效——click-dialog-2 att1 的错 seed 页面即使被 agent 做到 raw=1 也取不回、**未计入分值**。
- r5 建议：服务端比对 seed 值不等即 flagged（一行判断）。
- 教训：过滤与检测要分开评估——判分没被骗过（过滤有效）不等于检测完备。
- 出处：security-review-r4-post、round4-report.md。

### 坑 20 · E4 clean 口径在"无效格被重试"下弱化（r3-pre F1）
- 现象：无效事件发生在 att1、格因 infra 状态进重试、att2 干净 → last-wins 只看末行、invalid_reason 为空 → 该格从 clean 剔除中消失（r2 数据回放 clean=57/124 ≠ 手工 55/115，差 2 格逐行归因到 ascending-numbers/click-pie）。
- 修复：一行——重试选格循环跳过带 invalid_reason 的行。
- 教训：自动化口径要拿历史人工口径做差分回归，差格要逐行归因。
- 出处：security-review-r3-pre、r3-post。

### 坑 21 · 预算账面"144 恰满"与代码不符（review-r4 MF-3）
- 现象：plan 写 14+130=144 恰满，但重试预算 pilot=2 使 M1 最坏 16——结构性最坏 146=16+130。
- 根因：cap 强制是单次 orchestrator 调用口径（每次 run() 重置），跨调用无工具总账。
- 修复：预算表改为 M1=14–16、M2=130/调用；全局累计护栏（manifest cumulative_* + Σ>144 量化披露 + 禁拆 exp-id 分窗纪律）；+2 全为 pilot 域重试（≤1.4%）量化披露。
- 教训：护栏数字要按代码的计数口径算，不能按愿景算。
- 出处：review-r4、plan-r4、orchestrator.py。

### 坑 22 · 预期数字不可推演（review-r3 M3 / review-r4 MF-2）
- 现象：r3 保守档写 63 但构成列 58+5+1+1=65，差 2 无注释；r4 把分析给的"~0.7 能正确交互"当通过率凑出 A 上界 +8（法定通过概率约 0.4）——交互率和通过率是两个分母。
- 修复：r3 改 65/125=52.0%；r4 重推为逐格构成表（email 10×0.3/0.4、单格赋档、幻觉族 24×0.15/0.2、拖拽 ⌊2×0.5⌋ → 增量 7~10、区间 52.8%~55.2%），交互率不入通过算术。
- 教训：预期表必须"读者可复算"。
- 出处：review-r3、review-r4、plan-r4。

### 坑 23 · 文档间小矛盾备案（r2）
- 现象：multi-layouts 页面通过时刻两份文档分别写 83s 与"4.3 分钟"；email-inbox-forward 导航次数一处 16 一处 15。
- 核实：经 rewards.jsonl 差分，**83s 正确**；nav 字段值 **15**（16 条开页日志中 15 条命中本格 URL，第 16 条是 about:blank 新标签机制，良性）。
- 教训：引用前向原始数据回查；分歧数字要落备案防后人踩错源。
- 出处：review-r2、security-review-r1-post。

### 坑 24 · 杂项工程坑（六条合一卡）
- playwright sync API 同线程不允许两个并发上下文 → 预读顺序调用、浏览器 fixture function scope，并行化必须改多进程。
- macOS 系统代理劫持 127.0.0.1 回环（返回 502）→ harness 对伺服端口的请求统一禁代理 opener（exp001 已知问题同解）。
- Python 默认参数在定义期求值 → nav 轮数缺省改在构造函数体内读 config。
- spec 写"overall=1/3"是算术勘误——在官方口径零改动约束下实为 2/3（invalid-but-passed 格照计 1），测试锁定 2/3 并注明理由。
- "Its Empty, try something else" 分支是不可达死代码（序列化恒返回非空串，空表实际返回 "[]"）；r4 原样保留未触碰、测试锁定。
- localhost:PORT 形态被 scheme 白名单拒绝（URL 解析把 localhost 误判成协议）——任务 URL 恒带 http://127.0.0.1: 不受影响，行为收紧非漏洞，勘误落档。
- 出处：test-report.md、review-r2、security-review-r3-pre。

## 5. 必背数字

| 数字 | 白话含义 | 什么时候会被问 |
|---|---|---|
| 125 任务类（109 family、唯一 visual=visual-addition；browsergym-miniwob 0.14.3） | 分母恒定、不挑任务 | "怎么保证可比" |
| 360 个 HTML 文件 vendored 入库（逐文件 sha256 指纹；miniwob 1.1.0、BSD-3-Clause） | 供题确定性+许可合规 | "任务页没被改过吧" |
| 240s 页面预算（原生 10s 的 24×，默认 240000ms）/ 600s 引擎超时 / 480s 只进 ablation / 端口 8462 | 计时口径 | "一轮跑多久、超时怎么定" |
| 四轮官方成绩 43.2% → 46.4%（clean 47.8%）→ 47.2% → **61.6%（77/125）** | headline 主线 | 必被问 |
| r1 全套：官方 54/125=43.2%、首实例 59/125=47.2%、零重导航 37/60=61.7%、row-based 54/130=41.5%；130 行/227 条奖励/126 份日志；12.9M token；wall≈336s/格 | r1 三口径与规模 | 深挖 r1 |
| 54 = 59 − 6 + 1；5 格判分瑕疵名单；**+4.0pp 锁定值** | 判分修复账 | "涨分是不是放宽口径" |
| r1 重导航画像：61/125 格 nav≥2（最多 15）；87/125 格 flagged（130 行中 88 行=纯导航 58+纯 H3 26+双触发 4）；93%（249/267）同 tab；email-inbox-forward 16 次导航页钟 +584s | C3 单次开局动因 | "为什么要封重导航" |
| multi-layouts 页面过后 46/64 轮空转；23 格平均空转 273s | zombie 实证 | "终局信号是干嘛的" |
| r1 抽样：最快通过 42.1s；use-colorwheel-2 raw=0.537（=137/255 合法部分分）；click-checkboxes raw=−1/3；29 行 timeout 全落在 600.04-600.05s（引擎 600s 超时刚过一线） | 数据真实性复核 | "分数怎么复核的" |
| 四轮最快通过：r1 42.1s、r2 33.1s、r3 30.5s（unicode-test）、r4 50.9s——快通检测（<15s 判通过）四轮全零的依据 | 快通检测依据 | "有没有秒过作弊" |
| r1 延迟画像：planner 3.2 轮/格 vs executor 51.3 轮（超时格）、轮间 9.0s、断裂 25.1 次/格、completion p90 1998 tokens；58/121 格首轮含 verify 空转步；25 格页面 240s 干等死亡 | 延迟包/模板动因 | "慢在哪、做了什么" |
| r2 全套：58/125=46.4%（**clean 55/115=47.8%**）、首实例=官方（桥接双向为空）、零重导航 55/102=53.9%、row-based 58/136=42.6%；136 行/147 条奖励/135 份日志；9.08M token | r2 成绩 | 深挖 r2 |
| W2：10 格（7.4%）/9 格静默写 10 个文件/3 格通过格；r3/r4 该标记 **0/137、0/131** | 干净口径由来 | "55/115 怎么来的" |
| r2 安全全景：click-shape 沙箱 8 次 File not found 零执行（raw=1 在 +91s、尝试在 +233s——通过在先）；3 格 javascript: 尝试；4 格 21 次外联（terminal 11/login-user 8/email-inbox-delete 1/use-autocomplete 1）；21 组陈旧重发 | 攻击面盘点 | "还有别的异常吗" |
| C1a 见证：2 个 timeout 格（click-shape raw@+91s、odd-or-even raw@+62s）+ 4 个 junit 格；多导航格 **61→16**；flagged 88→29 行 | 机制生效证据 | "修了真有用吗" |
| r3 全套：59/125=47.2%（=clean）、首实例 58/125=46.4%（click-menu-2 在 51.7s 合法翻正）、零重导航 57/87=65.5%、row-based 59/137=43.1%；137 行/163 条奖励/137 份日志；12.9M token；超时 **41→21** | r3 成绩 | 深挖 r3 |
| R3-1 no-op：768 下单轮 ~13.9k 字符/**19,349** completion tokens；4,213 轮 0 轮==768、215 轮>768、max 3,918（r2 对照 250/3,721、max 3,402）；肥尾 2000-3400 token 占 10.3% 轮吃 38% 墙钟 | flag 失真实证 | "干预都生效吗" |
| scheme 白名单**实拦 4 次**（visual-addition att2、click-pie-nodelay att2 的 javascript:；count-sides att1、bisect-angle att2 的 view-source:） | 治理实效 | "防越界真拦过吗" |
| drag 修复实绩：61 调 0 中 → **0/9→3/9**（三格 147s/125s/505s）、kill-switch 9/9 格解析成功 | 工具修复 | "拖拽怎么救的" |
| r3 扫描构成：flagged 55 行=44 nav>1 + 10 记录数>加载数 + 1 view-source；invalid=0；reason 域 空串×103 + timed out×59 + Cool!×1 | 扫描输出样貌 | "扫描器报过什么" |
| r4 全套：**77/125=61.6%**（=clean）、首实例 76/125=60.8%、零重导航 74/108=68.5%、row-based 77/131=58.8%；131 行/152 条奖励/**285 条 epstart 信标/2 条 offseed**/131 份日志；**12.2M** token；超时 21→9 | r4 成绩 | 必被问 |
| +18 = 新通过 24 − 回退 6；A +17（email 4/4 + find-greatest 1 + 机制外溢 12）+ 方差 +7（±10 噪声带内）；r3 信标动因=1 格 off-seed + 6 格零奖励事件 | 涨分归因 | "61.6% 怎么来的" |
| verify 实绩：103/131 行恰 1 次、0 分贡献；r3 假阳性 17→24；完成幻觉 24 格占官方失败 57% | 诚实记录 | "有没有把功劳算在没用的东西上" |
| md 终表：off 档 email-inbox 0 条 → extended 47 条（thread/star/trash 各 7 全带 md）；表长 p50=4 / p90=36 / **max 129（上限 150，未触发）**、截断告警 0 次；8 页计数 4~47 无一触 150；find-greatest 6 卡带 md | r4 机制证据 | "+17 格凭什么信" |
| 12 格机制外溢（span.alink/accordion 头/hitarea/slider-handle/label） | 受益面按机制算不按家族算 | 追问外溢 |
| r4 完整性：39 个 official_failed 全带 raw=−1、9 个 timeout 全带 timed out；circle-center 0.939 部分分计通过；错 seed 78425566≠786425566 被拒计（count-sides seed=999/1）；四轮首个零"无页面裁决"终局行 | 数据完整性 | "会不会有格没裁决" |
| 预算：单次 cap **144**；smoke 段 14=10+2+2、full 130；重试预算 pilot 2/full 5、每格至多 1 次；结构性最坏 **146=16+130**（+2≤1.4%）；r1 红线 142 次/$20、$0.13/次、4.7 万 token/次、pilot ≤$1.56、full ≤$16.90 | 预算纪律 | "花了多少、怎么控" |
| 熔断与重试：attempt 级 <30s + 402/余额/连接标记；run 级连续 2 格中止；重试仅限 timeout/no_junit/no_reward 三态 | 熔断机制 | "跑挂了怎么办" |
| 延迟五键：LLM 超时 90（默认 60）/重试 2（默认 1）/markers_only/nav 轮 30（默认 50）/步预算 120s（默认 0）；planner 专超 150s（0=跟随 90） | 引擎调参 | "做了哪些性能工程" |
| 补丁层参数：50ms 轮询、失败重试 200 次（~10s）、同步 XHR、cue 层 z-index 2147483647 | 伺服细节 | 深挖实现 |
| goal 预读：等就绪 15s / 等 utterance 10s / 总预算 30s；no_goal 4→1；预读前后记录数 6→6 | 预读预算 | "goal 怎么来的" |
| 测试链：75→117→427→456→492 passed（record2gherkin）+ 引擎回归 12 passed；pilot 11 次执行=10+1 重试、overall 8/10；overall 2/3 勘误案例 | 质量门 | "回归怎么保证" |
| 基线：公开可比带 **A 档 47-60%**（零样本文本-DOM GPT-4 级）；81.6% 实为 HxAgent 且四重口径不可比；Agent-E 从未发表 MiniWoB++ 数字 | 外部对照 | "61.6% 什么水平" |
| 单 seed 噪声：纯噪声翻转 ~12.8%、噪声带 ±10 格；majority-of-3 待批、best-of-3 拒绝；480s 预期校准 +1±1 后 r4 降级默认不做 | 口径限制 | "单 seed 可信吗、为何不加时长" |
| extra_tools 面：全量 12 个工具（含 3 个文件读写类）；子集注册 43 vs 全量 54 | W2 根因与治理 | "怎么根除的" |
| r2-pre 两 P0：probed 只有 glm-5.3-flash（planner glm-5.3 未探）；runbook 残留 --nav-model deepseek-flash | 审查前置价值 | "pre 审查拦过什么" |

**未核实/勘误备案**：任务书所写"单轮约 240s/格"——240s 是 episode 页面预算，实测单格 wall 仅见于 r1 ≈336s/格，r2-r4 无逐轮 wall 均值披露，按"未核实"处理；browser 工具集件数 r4-pre 写 15 件、r4-post 写 16 件，取两处一致的结论"无 JS-eval 工具"；r4 的 email 家族 4/10（通过数/家族总数）与 A1 法定预期 3~4 格（预期区间）是两个口径，不矛盾。

## 6. 联动

- qa-bank：Q10（工具怎么搭、125 格怎么可比可复现）→ 本档案第 1/2 节 + 决策卡 1/2/5；Q21（怎么防作弊刷分）→ 第 2 节五层 + 决策卡 6/7 + 坑卡 1；Q23（发现成绩有水分怎么办）→ 坑卡 1（预登记契约是标准答案）+ 坑卡 2（自查判分瑕疵）；Q24（页面判分可信吗、能否骗过）→ 决策卡 4 + 坑卡 6（raw 封存）+ 原则性极限披露。
- 与 05 档案分工：04 管"怎么测得可信"（harness 机制、判分权威、防作弊、口径），05 管"四轮发生了什么"（逐轮设定、改动、逐格归因与涨分史）。被问机制答 04，被问"每轮做了什么、为什么涨"答 05。
