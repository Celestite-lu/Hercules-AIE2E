# 03 — 失败归因器 + 子进程编排 CLI（全档案）

> 档案子代理 C，2026-10-10。必读来源全部读毕：`dev_docs/attributor/{plan,spec,review,test-report}.md`、`dev_docs/cli/{plan,spec,review,test-report}.md`、`record2gherkin/attributor/{evidence,rules,llm_layer,report,api,__init__}.py`、`record2gherkin/{cli,__main__}.py`、`record2gherkin/evaluation/runner.py`、`PLAN.md` §3.3/§3.5、`dev_docs/STATUS.md`（阶段 5/6 + 收尾行）、`dev_docs/reports/final-analysis.md`、`dev_docs/evaluation/experiment-report.md`；引擎锚点逐条打开核对（simple_hercules / junit_helper / click_using_selector / static_ltm / config / playwright_manager / mcp_server）。测试计数为本机 `pytest --collect-only` 实测复核。

## 1. 这一页讲什么（3 句）

失败归因器把一次 Hercules 失败运行变成"可信归类 + 可回查证据引用"：13 条错误串签名规则层命中即定类（零模型、确定性、可机械验证），只有模糊区（断言不匹配、无签名长尾）才交给弱模型 LLM 层，而 LLM 的每条引用都被程序回查——查无此证强制降级"无法定位（inconclusive）"且置信度压 0.30。子进程编排 CLI 用 `record / generate / run / analyze` 四条命令把录制→蒸馏→执行→归因串成一条链：run 走独立 opt 根的 Hercules 子进程（key 只经 env 注入、日志强制脱敏、--dry-run 无 key 可干跑），这条子进程路线第一天就定死，因为进程内有四个实证的坑（StaticLTM 串场、config 单例、时间戳目录混叠、PlaywrightManager 关闭竞态），且上游自己的 MCP server 也用子进程。真实验证做了两条：F5/M4 真实失败格经"S10 签名 + 无 LLM 降级"产出证据直指文案改写的报告，F1 真实链路（record→generate→run passed 62s/7.3 万 token→analyze）全链路验收通过；`--polish`/`--llm` 两个 flag 真机未验收，属如实标注的待验收项。

## 2. 设计决策总表

| # | 决策 | 为什么 | 备选与否决理由 | 出处 |
|---|---|---|---|---|
| A1 | 两层架构：规则层是定类主体，LLM 层只处理 `needs_llm` 路由来的模糊区 | 引擎错误串是源码固定文本，命中即准、确定性、可单测；LLM 只在"product_bug vs test_rot vs agent 点错"分不清的模糊区发言，不做定类初审 | 全 LLM 归因（否：不确定、贵、会幻觉）；全规则（否：断言 EXPECTED/ACTUAL 不匹配的根因多样，规则定不了 product_bug） | attributor plan §2；spec §3.4 |
| A2 | 五档分类 `product_bug / test_rot / environment / agent_limit / inconclusive`（较 PLAN §3.5 三分类扩两档） | `agent_limit` 把"引擎侧硬上限"从 environment 分出（混着会误导排查方向）；`inconclusive` 给"宁可疑而不定"一等公民出口，避免 LLM 被迫瞎猜 | 维持三分类（否：任务指派即总计划修订，plan/spec 以五档为准） | attributor plan §4；spec §1.1（`evidence.py:34-40`） |
| A3 | 证据入口 = JUnit XML properties（内嵌全部证据路径，含上游 typo "netwrok" 键按 `Proofs Base Folder` 前缀匹配） | `utils/junit_helper.py:108-146` 已把 Feature/Planner Thoughts/Proofs 等路径写进每个 testcase，不用自己拼目录 | 扫目录约定（否：脆弱且要重复上游逻辑） | attributor plan §3.1；spec §2.1 |
| A4 | 截图↔planner 轮次只做 v1 近似对齐（文件名形态 `{tool}_{start\|end}_{ns}.png` + 时间序排序 + 末状态对 + thoughts 字面提及建锚），不做 step 精确对齐 | PLAN §3.5 明确 v1 近似即可；锚只认字面提及、无锚不造锚，报告明示"近似对齐非逐轮映射" | step 精确对齐（否：Out of Scope，收益低边界问题） | attributor plan §3.2；spec §2.4 |
| A5 | 签名置信度分层 + conf<0.8 强制附替代假设：not found 类封顶 0.65-0.75 且必带 alt，固定引擎串 0.90-0.95 | "not found ≠ 必然用例过时"（产品 bug 也表现为找不到元素）；低置信度结论必须把别的可能摆上台面 | 单一置信度（否：掩盖歧义） | attributor plan §8；spec §3.2 |
| A6 | S3 做成复合签名：主复合命中后对 THOUGHTS 次级复扫，not found→test_rot 0.75 / 网络→environment 0.75 / 均无→agent_limit 0.90 | "轮次耗尽"是表象不是根因，二跳能把最大失败形态再细分 | S3 只出 agent_limit（否：丢掉最有价值的一跳归因） | attributor spec §3.2；test-report B 组 14-16 |
| A7 | 规则层零模型：`evidence.py/rules.py/report.py` 不 import litellm/requests 等，且有两条机械检查（源码 grep 用例 + 子进程 import 后 sys.modules 检查） | "规则层无 LLM"是可验证属性不是口头承诺 | 只写注释约定（否：不可验） | attributor spec §0/§8-2；test-report B 组 #27 |
| A8 | LLM 输出强制引用编号证据包 + 程序回查 + 查无此证降级 inconclusive 0.30 | "绝不瞎编"是模块核心不变量；LLM 最坏行为的下限是诚实的 inconclusive | 信任 LLM 输出（否：幻觉引用无法发现） | attributor spec §4.4-§4.5；llm_layer.py:390-421 |
| A9 | thoughts 证据 excerpt 必须是 content_text 的纯截断（400 字符，不追加任何装饰文本） | review M2：装饰文本会破坏"摘录是原文子串"回查，把带截图锚轮次的合法 LLM 引用系统性误杀成 inconclusive | excerpt 附截图文件名（否：review 必改项 M2，最值场景反被压成无法定位） | attributor review M2；spec §4.2-2 |
| A10 | S3/S12 签名语料必须含 THOUGHTS | review M1：引擎错误串经 HumanMessage 回灌 planner 会话，规范落点是 `agent_inner_thoughts.json`；FAIL-only 实现会让旗舰签名在真实运行系统性失配，而合成用例全用 FAIL 文本测不出 | 语料 FAIL-only（否：review 必改项 M1） | attributor review M1；spec §3.1 数据流注记 |
| A11 | CLI analyze 默认 `analyzer=None`（纯规则、离线、逐字节确定），`--llm` 才启用 LLM 层，且构造失败自动降级纯规则 exit 0 | 演示链路不应隐性依赖 LLM 配置；attributor 降级语义天然支持 | analyze 默认带 LLM（否：演示脆弱、不确定性） | cli plan D4；cli spec §6.2 |
| C1 | 入口 `uv run python -m record2gherkin <sub>`，不改 pyproject.toml、不新增 console-script | 避免动上游锁定的 pyproject/uv.lock | console-script（否：要改 pyproject，超出二开边界） | cli plan §1；spec §0 |
| C2 | `run` 直接包装 `evaluation/runner.py`（read_api_key/build_run_plan/run_feature），禁止绕过 runner 自建子进程 | runner 已解决 key 文件读取、env 注入、进程组超时击杀、stdout 脱敏、JUnit 两级定位 | 重写执行器（否：复制密钥与进程管理逻辑，双倍泄露面） | cli plan D1；spec §5.1 |
| C3 | P0-2 两段式蒸馏：先 `distill_events(polisher=None)` 出骨架正则扫 `{{TEST_DATA:key}}` 键集，再 `distill_file(test_data_values=needed ∩ provided)` 出终稿 | 骨架纯函数零成本；只蒸馏一次润色（不烧双份 LLM 调用）；`{{TEST_DATA:*}}` 在引擎侧没有机械消费者，填充必须在 generate 期完成 | run 期注入（否：引擎无消费者，phase1-merge-analysis P0-2 实证） | cli plan D3；spec §3.2 |
| C4 | `--test-data` 值契约与 distiller `_clean_test_data_values` 对齐：bool/null/空白视为未提供，object/array 报错 exit 2 | review R3：不一致会"filled=N 虚报但产物仍留占位符"，缺值警告闭环失效 | CLI 侧宽收 bool/空白（否：摘要与产物互相矛盾） | cli review R3；spec §3.1 步骤 4 |
| C5 | `--dry-run`：只 `build_run_plan` 打印 cmd + 经 `env_redacted()` 的五键 env；key 文件缺失特批占位串 `DRYRUN-PLACEHOLDER` 继续构建 | 编排层可无 key 演示/测试；不执行所以无泄露面 | dry-run 也要求 key（否：离线验收做不了） | cli spec §5.4；cli.py:423-497 |
| C6 | run 摘要打印 `failure_message` 前在 CLI 侧再过一次 `mask_secret`（R2 二次脱敏） | review R2：runner 只对子进程 stdout 脱敏，`STATUS_FAILED` 的 failure_message 直接取自 JUnit XML 不经脱敏，是 CLI 日志里唯一未过脱敏的外部文本面 | 信 runner 已脱敏（否：断言失实，密钥红线不容"大概率不会回显"） | cli review R2；cli.py:465-467 |
| C7 | 退出码全命令统一 0/1/2/3/4/130；analyze 出报告即 exit 0（哪怕用例失败——报告产出是 analyze 的职责，用例成败由 run 的退出码承载） | 链路里各命令职责清晰、互不吞并 | analyze 反映用例成败（否：与 run 退出码重复且混乱） | cli spec §1.2/§6.2 |
| C8 | record 可测接缝 `stop_when(page) -> bool`，轮询循环把 page 传给谓词 | review R1：闭包拿 page 才能驱动真实点击写测试；生产谓词（SIGINT Event / --max-duration）忽略该参数 | `stop_when()` 无参（否：测试物理上触达不了页面） | cli review R1；cli.py:123-183 |
| C9 | 子进程模式第一天定死，不许进程内连跑 | 进程内四个实证坑（见 §4 坑 1-4）；上游 MCP server 自己都回避进程内执行 | 进程内连跑省冷启动（否：正确性坑无法工程消除） | PLAN §3.3 |
| C10 | 全部输出走 `logger`（main() 第一步 `configure_logger("INFO")`），禁 print | ruff T20；裸 import 时 logger 无 handler，不装配用户什么也看不到 | print（否：违反仓库规约） | cli plan D6；spec §0 |

## 3. 关键实现细节

### 3.1 归因器数据流与五档语义

```
产物目录（output/*.xml + log_files/<stake>/<ts>/agent_inner_thoughts.json + proofs/.../screenshots/*.png）
  → evidence.py 证据加载（JUnit 缺失/不可解析是唯一加载期抛错 AttributionError；其余缺件记 missing 警告容错）
  → rules.py 签名匹配（每个签名按 FAIL → SYSOUT → THOUGHTS 固定序扫描，表序即优先级，首个命中生效）
      命中确定性签名（S1-S9、S12）→ 直接出结果；命中路由签名（S10/S11/S13）或无命中 → needs_llm
  → llm_layer.py（仅 needs_llm）：编号证据包 → 弱模型严格 JSON → 程序回查 → 采纳或降级
  → report.py：AttributionResult → to_json()（schema_version=1，12 键）/ to_markdown()（中文正文、摘录保持原文）
```

五档语义：`product_bug`=页面实际行为与用例预期不符（页面没坏、用例没过时，规则层不下此结论，LLM 层才敢下）；`test_rot`=用例依据的元素/URL/文案与当前站点脱节；`environment`=站点不可达/浏览器/代理等运行环境问题；`agent_limit`=引擎轮次耗尽、超时、坏 JSON 空转、context 超限等硬上限；`inconclusive`=证据不足以定位（诚实输出，LLM 不确定或被降级时的归宿）。规则层内部路由值 `needs_llm` 永不出现在最终结果。

### 3.2 13 条签名逐条表（`rules.py:147-279`，匹配串逐字取自引擎源码锚点）

扫描语料字段：FAIL=failure_message、SYSOUT=system_out、THOUGHTS=全部轮次 content_text；每签名按 FAIL→SYSOUT→THOUGHTS 固定序、命中即停（THOUGHTS 命中记首末两个匹配轮）。表序即优先级：S1/S2 的 assert_summary 也含 EXPECTED/ACTUAL，必须先于 S10（用例 26 `test_precedence_limit_before_mismatch` 钉死）。

| # | id | 签名特征（ci=忽略大小写） | 语料 | 分类 | 置信度 | 替代假设 | 源码锚点 |
|---|---|---|---|---|---|---|---|
| S1 | planner_max_rounds | ci 子串 `max planner rounds exceeded` | 全部 | agent_limit | 0.95 | — | simple_hercules.py:409 |
| S2 | planner_timeout | ci 子串 `llm call timed out after` 或 `planner receives a timely model response` | FAIL∪SYSOUT | agent_limit | 0.90 | — | simple_hercules.py:320,357-359 |
| S3 | nav_max_rounds（复合） | 同一字段内 ci 子串 `reached before ##TERMINATE TASK##` **且** `max nav rounds (`；命中后对 THOUGHTS 次级复扫：(a) S8 模式命中→test_rot 0.75（引该轮+alt）；(b) S5∪S6∪S7 命中→environment 0.75；(c) 均无→agent_limit 0.90 | 全部＋THOUGHTS 复扫 | (a) test_rot / (b) environment / (c) agent_limit | 0.75/0.75/0.90 | (a)"轮次耗尽可能因引擎定位能力不足，而非用例过时" | simple_hercules.py:1004-1007 |
| S4 | llm_context_limit | ci 正则四选一：`context_length_exceeded` / `maximum context length` / `context window.{0,40}exceed` / `prompt is too long` | 全部 | agent_limit | 0.85 | — | 通用 LLM 报错固定串 |
| S5 | network_dns | ci 正则五选一：`net::ERR_[A-Z_]+` / `getaddrinfo` / `Temporary failure in name resolution` / `Name or service not known` / `NS_ERROR_UNKNOWN_HOST` | 全部 | environment | 0.90 | — | Chromium 固定串 |
| S6 | browser_disconnected | ci 正则五选一：`Target (page, context or browser )?closed` / `browser has been closed` / `Playwright connection (closed\|error)` / `Target crashed` / `Session with given id not found` | 全部 | environment | 0.80 | — | Playwright 固定串 |
| S7 | tool_error_network | 同一命中行内：ci 子串 `[tool error]`/`[mcp tool error]` **且** ci 正则网络词 `timed out\|timeout\|connection\|econn\|unreachable\|refused\|reset by peer` | 全部 | environment | 0.75 | "也可能是被测站点自身故障被记为工具错误" | simple_hercules.py:682,697；mcp_help.py:113 |
| S8 | element_not_found | ci 正则三选一：`element with selector[^\n]{0,80}not found` / `since the selector is invalid` / `(element\|node\|option\|button\|field\|link\|form)[^\n]{0,60}\bnot found\b`；best-effort：命中片段引号内选择器名出现在 feature_text → 追加 `feature:<行号>` 引用 | 全部 | test_rot | 0.70 | "元素缺失也可能是产品缺陷或 agent 误定位（静默点错）" | click_using_selector.py:157-159,306 |
| S9 | http_404 | ci 正则：`\b404\b[^0-9\n]{0,40}not found` 或 `\bstatus["']?[:= ]+ ?404\b` | 全部 | test_rot | 0.65 | "404 也可能因环境路由/代理而非用例过时" | — |
| S10 | assertion_mismatch（路由） | FAIL 同字段内 ci `\bexpected\b` **且** `\bactual\b`（覆盖 `EXPECTED RESULT/ACTUAL RESULT` 与 `EXPECTED:/ACTUAL:` 两变体） | FAIL | **needs_llm**（三假设并列：产品缺陷 / 用例过时 / agent 误操作静默点错） | — | 三假设 | high_level_planner_agent.py:183 |
| S11 | helper_uncertain（路由） | 同一字段内 ci 子串 `##TERMINATE TASK##` **且** ci 正则 `\b(uncertain\|incomplete\|contradict\w*)\b` | 全部 | **needs_llm** | — | — | simple_hercules.py:110-124（failure markers 子集） |
| S12 | llm_call_error | ci 正则 `\[error\] [a-z_ ]+ llm error:` | 全部（规范落点 THOUGHTS 回灌行） | agent_limit | 0.80 | — | simple_hercules.py:917 |
| S13 | generic_tool_error（路由） | ci 子串 `[tool error]`/`[mcp tool error]`（S7 未接管的非网络形态） | 全部 | **needs_llm** | — | — | 同 S7 |
| — | fallback | 以上全不命中 | — | **needs_llm**（模糊区长尾） | — | — | — |

规则层固定建议模板 5 条（agent_limit/test_rot/environment/product_bug/inconclusive 各一条中文文案，`rules.py:284-290`），确定性拼装。

### 3.3 LLM 层：证据包 / 回查 / 降级（`llm_layer.py`）

**证据包结构（确定性，编号 E1..En）**：
1. 种子：规则层命中全部 evidence + FAIL 原文（kind=junit_failure）+ final_response（kind=junit_sysout）——**永不被上限丢弃**。
2. 上下文：THOUGHTS 中规则命中轮 + 末 6 轮（`CONTEXT_ROUNDS=6`），每轮一条 `thoughts:<index>`，excerpt=content_text **纯截断 400 字符**（追加装饰文本会破坏子串回查——review M2 修订的落点）。
3. 截图：轮次锚截图 + 末状态对截图各自独立打包为 `screenshot:<filename>`（excerpt=文件名；`[FINAL]` 标记落在 `screenshot_names` 与报告附录，不进 excerpt）。
4. 总条目上限 30（`MAX_EVIDENCE_ITEMS`）：超限先丢上下文轮（按 index 旧→新），再丢非末状态截图（按时间序），种子与末状态对永不丢。
5. user 消息另含 scenario / failure_message（截 2000）/ final_response（截 2000）/ feature 摘录（截 2000）/ 截图文件名时间序清单（上限 20，末状态对标 `[FINAL]`），并明示"截图与轮次仅近似对齐"。

**输出校验与回查（§4.4，程序性五步）**：
1. 提取：剥 ```` ```json ```` 围栏 → 取首个 `{` 至末个 `}` → `json.loads`。
2. Schema：五键齐全、category ∈ 五类、confidence 数值（clamp [0.05, 0.95] 后四舍五入 2 位）、summary 非空（截 2000）、evidence_refs 字符串列表。
3. **逐条引用回查**：引用号在包内不存在 → drop（`unknown_ref`）；kind=screenshot → 存在即通过；其余 kind 按定位符语法**重新解析** referent——`thoughts:<index>` 须界内并取该轮 content_text、`feature:<line>` 1-based 界内取该行、`junit_failure`/`sysout:final_response` 取对应原文、`property:<name>` 取 properties 值——解析失败 → drop（`unresolvable`）；解析成功后校验 `norm(excerpt)` 是 `norm(referent)` 子串（norm=连续空白折叠+strip，大小写敏感），不成立 → drop（`excerpt_mismatch`）。
4. 结论级约束：非 inconclusive 结论须存活引用 ≥1 且**至少 1 条文本类**（thoughts/junit_*/feature_file；仅凭截图文件名不支撑任何结论——本层不读像素），否则本次尝试失败（`insufficient_evidence`）。
5. 重试与降级：总尝试 ≤2（`MAX_ATTEMPTS=2`）；第 1 次失败把违反项以机器文本（`violation: unknown_ref E99` 这类）作为追加 user 消息重试一次；LLM 调用异常计一次尝试、不重试解析。

**四条降级路径（唯一出口 inconclusive）**：analyzer 为 None 或抛 AttributionError → warning `llm_not_configured`、confidence=0.30；异常×2 → `llm_error:<ExcType>`、0.30；校验失败×2 → `llm_output_invalid`、0.30；回查后存活引用不足 → `evidence_backcheck_failed:<dropped refs 逗号列表，最多 5 个>`、confidence=min(原值, 0.30)。降级产物 `decided_by="degraded"`、**保留规则层种子证据**（inconclusive 也有可查证据）与规则层替代假设；建议回填 inconclusive 模板。`DEGRADED_CONFIDENCE=0.30`（llm_layer.py:74）。部分引用被丢但存活引用足够时仍采纳，附 `evidence_ref_dropped:<violations>` warning。

**"宁可疑而不定"prompt 纪律**（SYSTEM_PROMPT 固化要点，llm_layer.py:80-100）：角色限定为只归类与引用、不修 bug；五类判定标准各一句；硬约束逐条——(a) evidence_refs 只许填请求中的 ref id，禁止编造文件名/轮次/引用；(b) **Prefer inconclusive over guessing. inconclusive is a legal and encouraged answer.**（宁可疑而不定，不许猜）；(c) 非 inconclusive 必须引用 ≥1 条文本类证据，仅截图文件名不足以支撑结论（看不到像素）；(d) EXPECTED/ACTUAL 形态要权衡产品 bug、用例过时、agent 点错三假设；(e) 只输出严格 JSON 无围栏无解释。温度 0；弱模型档经 `get_litellm_chat_model("attributor_analyze")`，模型**懒构建**（构造 DefaultAttributorAnalyzer 不触网不读配置，缺 `attributor_analyze` 键的错误在首次 analyze 时抛 AttributionError、被上层按"analyzer 不可用"降级）。

### 3.4 真实验证（两条）与 --polish/--llm 如实状态

**F5/M4（归因器真实验证）**：exp001 六十格中唯一失败格 F5/M4（按钮文案改写"去下单"→"马上下单"），失败运行喂给归因器：规则层 S10 签名命中路由 + analyzer=None 降级，产出报告 category=inconclusive、decided_by=degraded、rule_signature=assertion_mismatch（S10），**种子证据保留且直指文案改写（`Missing: '去下单' (found '马上下单')`）**，替代假设含"用例过时"——诚实降级设计在真实失败上成立（experiment-report.md:31）。精确语义注意：S10 是路由签名，"无 LLM"下不定类为 test_rot，而是降级 inconclusive；报告的"正确"体现在证据与替代假设把根因摆给了人。

**F1 真实链路验收（CLI 收尾验收）**：record 复用 F1 录制（MiniShop 搜索流程：打开 `/` → 输入"台灯" → 点"搜索" → 断言"找到 1 件商品"）→ generate → **run 真实执行 passed（62s / 7.3 万 token）** → analyze（STATUS.md 阶段 6 行与收尾行；演示视频镜 8 字幕注明"JUnit tests=1 errors=0"；固化提交 `ef25a9c`）。注意 run 的 62s 用的是引擎自身 LLM（deepseek-v4-pro 全角色），与下述两个 flag 无关。

**`--polish` / `--llm` 真机未验收（如实状态）**：离线测试只覆盖注入 fake（用例 15/16、30）与降级路径；真实润色可用性取决于 agents_llm_config 的 `distiller_polish` 档位，真实 LLM 归因取决于 `attributor_analyze` 档位（cli test-report 已知#6/#7）。原因：离线测试纪律禁止在测试里触发真实 litellm 调用，真机验收依赖 key 与模型档配置，且两者失败都有契约内出口（--polish 失败回退骨架 exit 0 + `polish_error:*`；--llm 构造失败降级纯规则 exit 0）。终审口径（final-analysis R2）：演示不开这两个 flag；被问就答"润色与 LLM 归因有完整的降级与回查测试，真实 LLM 归因验收只做了规则层（F5/M4），LLM 层标为待验收"。**不能宣传"LLM 归因生产可用"**。

### 3.5 CLI 四命令参数与产物（`cli.py:584-618`）

| 子命令 | 参数（类型/默认） | 产物与行为 | 退出码 |
|---|---|---|---|
| `record` | `url`（位置，--manual 时忽略）；`--manual`；`--out PATH`（默认 `dev_runs/recordings/<YYYYmmdd-HHMMSS>-events.json`）；`--headless`（默认关=有头）；`--max-duration SECONDS`（默认 0.0=不限时） | sync playwright 起 chromium → `page.goto(url, timeout=30_000)` → 注入 `recorder.js`（`R2GRecorder.start()` 必须 `is True`）→ 0.2s 轮询（stop_when / SIGINT / 到时；页面被关或录制器丢失=跨文档导航 → exit 2 不落盘）→ `stop()`+`getJSON()` 校验 `session`/`events` 键 → 事件 JSON 落盘；`len(events) <= 1`（只录到初始 navigate）→ `empty_recording` warning 照常保存。`--manual` 不启浏览器，打印 bookmarklet 用法四步 + "只注入一次"警示 | 成功 0；启动/跳转/注入/页面丢失/URL 不可达 2；第二次 Ctrl+C 强制中止 130 |
| `generate` | `events`（位置）；`--out PATH`（默认 `distiller.default_output_path` = 事件同目录 `<stem>.feature`）；`--polish`（默认关=纯模板零模型）；`--test-data PATH`（JSON：`{"password_<seq>": "值"}`） | 两段式：骨架预蒸馏 → 正则扫 `{{TEST_DATA:([^{}]+)\}\}` 得 needed（保序去重）→ 读 --test-data（值契约见 C4）→ 终稿蒸馏只传 needed ∩ provided → 逐条 warning（`missing_test_data:<k>`/`unused_test_data:<k>`/占位符保留说明）→ info 摘要 output/needed/filled/used_llm_polish/fallback_reason | 成功 0（含占位符保留、润色回退）；events 缺失/JSON 非法/test-data 形态错 2；DistillError（骨架自检失败，bug 级）1 |
| `run` | `feature`（位置）；`--out-dir PATH`（默认 `dev_runs/cli_runs/<run_id>`，`run_id=cli_<时间戳>_<stem>`，冲突追加 `_1/_2`）；`--timeout SECONDS`（默认 `DEFAULT_TIMEOUT_S=900`）；`--key-file PATH`（默认仓库根 `LLM-Key.txt`）；`--dry-run` | 包装 runner：独立 opt 根（`<out-dir>/opt` 为 Hercules 项目根，runner 建 `input/`/`output/`）→ 子进程 `python -u -m testzeus_hercules --input-file … --project-base … --output-path …`（key 只经 env）→ 摘要 status/passed/duration_s/cost_usd/total_tokens（None 打 `-`）+ junit_xml + 同 stem .html + proofs/log_files（存在才打）+ failure_message（截 500，**CLI 侧 mask_secret 二次脱敏**）+ 末行 analyze 提示 | passed/dry-run 0；JUnit 判失败 1；子进程超时 3；JUnit 缺失/不可读 4；feature/key 文件缺失等输入错 2；Ctrl+C 130（warning 子进程可能残留） |
| `analyze` | `run_dir`（位置）；`--junit PATH`（默认 attribute_run 自行 glob `**/*.xml` 取 mtime 最新）；`--llm`（默认关=纯规则）；`--out-dir PATH`（默认 `<run_dir>/analysis`） | 包装 `attribute_run`（analyzer=None 或 --llm 惰性构造 DefaultAttributorAnalyzer，失败 warning 降级）→ 按失败用例序写 `case-<i>.json` + `case-<i>.md` → 摘要表每行 `case-<i> \| <scenario 截 60> \| <category> \| conf=<x.xx> \| by=<decided_by> \| sig=<rule_signature 或 ->` | 出报告 0（哪怕用例失败）；全过 0（不建目录）；run_dir 不存在 2；AttributionError（JUnit 缺失/不可解析）3 |

无子命令 → argparse 退出 2；未捕获意外异常 → 最外层兜底 error 日志（带类型名）→ exit 2，不裸 traceback。

### 3.6 --dry-run 设计

不调 `run_feature`，直接 `build_run_plan(feature, project_root, api_key=key)` 并打印：`plan.cmd`（空格连接一行，含 `python -u -m testzeus_hercules --input-file …`，argv 中无 key）+ 经 `plan.env_redacted()` 的**五键**（`LLM_MODEL_NAME`、`LLM_MODEL_BASE_URL`、`HEADLESS`、`ENABLE_TELEMETRY`、`LLM_MODEL_API_KEY`，cli.py:94）。key 显示为 `***REDACTED***`（`REDACTED` 常量，runner.py:36）。**`--key-file` 不存在也能干跑**：`_load_api_key` 在 dry-run 分支对 RunnerError/空文件 warning 后用占位串 `"DRYRUN-PLACEHOLDER"` 充当 api_key 继续构建（不会执行，无泄露面）；非 dry-run 缺失 → exit 2 不启动任何子进程。实测（cli test-report §3-4）：dry-run 日志 `grep -c dummy-key-0123456789` = **0**，`LLM_MODEL_API_KEY=***REDACTED***` 命中 1 行；四连冒烟第 4 步用 `--key-file "$WORK/no-such-key.txt"` 实证干跑可用。

### 3.7 密钥纪律全链

1. **读取**：`read_api_key(path)`（runner.py:150-172）——两种格式：单行文件（strip 即 key，历史行为）或 KV 格式（`#` 注释行 + `KEY=`/`BASE_URL=`/`MODEL=` 三行，见到 `KEY=` 行按 KV 解析；`KEY=` 为空视为不可用凭据；MODEL/BASE_URL 行忽略，model 来自 provider 常量）。测试一律假 key（`dummy-key-0123456789`）。
2. **传输**：key 只在内存 → `run_feature(api_key=…)` → `build_child_env` 把 `LLM_MODEL_API_KEY` 放进子进程 `env=`。`build_command` 的 argv 不含 key。同 env 固定注入 `LLM_MODEL_NAME=deepseek-v4-pro`、`LLM_MODEL_BASE_URL=https://api.deepseek.com`、`ENABLE_TELEMETRY=0`（否则 config import 初始化 Sentry）、`HEADLESS=true`、`ENABLE_UBLOCK_EXTENSION=false`（本机直连 GitHub releases 不稳，扩展下载超时会让浏览器初始化直接失败）、`CHROMIUM_PROXY_BYPASS_LOOPBACK=1`（系统代理劫持 127.0.0.1 的门控）。
3. **脱敏**：`mask_secret(text, secret)`（runner.py:175-189）——整串替换 + 当 secret ≥16 字符时头尾各 8 字符片段（`MASK_FRAGMENT_LEN=8`）也替换，全部换成 `***REDACTED***`；空 key 原样返回。三个落点：子进程 stdout+stderr 合流后写 `stdout.log` 前脱敏；CLI run 摘要对 JUnit 来的 failure_message 二次脱敏（R2）；dry-run 打印走 `env_redacted()`。
4. **验证**：CLI 侧——演示脚本实测 dry-run 日志假 key 零命中、写成文件的工作目录全量 grep 仅命中 key 文件自身（cli test-report §3-4）；用例 20/23 断言日志全文假 key 明文零命中（含 JUnit failure_message 路径）。全项目侧——benchmark 安全审查 r2/r3 post 各做"**双 key 全目录 grep 零命中**"：`LLM-Key.txt`（35 字符值）与 `GLM-Key.txt`（KEY 49 字符）逐值对 `dev_runs/` 全目录 grep，KEY 值全部零命中（security-review-r2-post.md §2.3、security-review-r3-post.md §2）。
5. **密码层**（上游配合）：录制层即掩码 → `{{TEST_DATA:password_<seq>}}` 占位符 → generate `--test-data` 填充真实值（P0-2），key/密码永不进 feature 原文之外的人工可读面。

### 3.8 测试布局与覆盖（本机 --collect-only 实测复核）

**attributor 81 例**（`tests/record2gherkin/attributor/`，纯离线：合成产物目录 + FakeAnalyzer + 0 字节 PNG）：
- `test_evidence.py` 13：A 组用例 1-11（全套产物加载、缺 thoughts/截图容错、content dict/字符串双形态、截图名解析与非标准名、末状态对、字面锚、failure/final_response 提取、feature 载入与缺失、路径失效 glob 找回）+ 附加（JUnit 缺失/损坏抛 AttributionError、fixture 自包含与 PNG 0 字节）。
- `test_rules.py` 27：B 组用例 12-27（S1-S13 每签名 ≥1 正例断言 category/confidence/证据位置；S3 三分支 14/15/16；S10/S11 路由与种子；表序优先；零 LLM import grep）+ 附加 10（S3 分支顺序、S5 首末轮双引用、S7 无网络词落 S13、S8 优先于 S13、主命中 FAIL 而信号在 THOUGHTS 的数据流注记、fallback、5 类模板、excerpt 截 500、`test_package_import_is_offline` 子进程 import 后 sys.modules 无 litellm/langchain/openai/httpx/urllib3）。
- `test_llm_layer.py` 29：C 组用例 28-36（合法采纳、未知 ref 降级、仅截图引用降级、非法→合法重试、两次非法、异常×2、无 analyzer、inconclusive 合法、回查定位符语义）+ 附加（`resolve_reference_boundaries` 参数化 14 实例、部分丢弃仍采纳、配置异常不耗尝试预算、空白归一化边界、30 条上限丢弃顺序、excerpt 纯截断、DefaultAnalyzer stub 注入）。
- `test_report.py` 5 + `test_api_e2e.py` 7：D 组用例 37-42（Markdown 必备节、JSON schema 12 键、P0-1 形态 e2e test_rot、静默点错+fake LLM product_bug、无 LLM 逐字节确定性、过滤通过用例）+ 附加（LLM 节条件渲染、渲染幂等、附录保留定位符、`not_a_failure` 短路、`--junit` 覆盖、最新 XML 自动发现）。e2e 里机械校验 `assert_citations_resolvable`：每条结论证据都能按定位符回查到 referent 且 excerpt 是子串。

**cli 54 例**（`tests/record2gherkin/cli/`，离线：run_feature 全 monkeypatch、--polish/--llm 走接缝注 fake、record 用真实 headless Chromium 开本地 file:// fixture）：
- `test_parser.py` 8（help 四子命令、无子命令 exit 2、configure_logger 被调、`python -m` 真子进程入口、未知子命令、record 缺 url、默认路径仓库锚定、异常兜底）
- `test_record.py` 6（--manual 文案、fixture 录制往返 stop_when 驱动 2 次点击、空录制 warning、不可达 URL exit 2、--max-duration 保存、页面中途消失 exit 2）
- `test_generate.py` 13（默认路径、--out、占位符填充、缺值/多余键双警告、不给 --test-data、形态错+bool/空白值、events 非法、polish 回退/采纳、逐字节确定、DistillError exit 1、只传交集）
- `test_run.py` 17（dry-run exit 0 / 无 key 干跑 / key 掩码零命中 / feature 缺失 / key 缺失 / 四状态退出码+二次脱敏 / 摘要产物路径 / analyze 提示行 / 空 key 文件 / None 指标打 - / 默认布局 / run_id 冲突后缀 / Ctrl+C 130）
- `test_analyze.py` 10（纯规则归因、全过不建目录、run_dir 不存在、无 JUnit、--llm 降级、--junit 选定、摘要表行、--llm 且规则已判定不触模型、--out-dir 覆盖、逐字节确定）

断言有效性做过变异测试（cli test-report §2.1）：去掉 run 摘要的 mask_secret → 4 条红；放宽 test-data 值契约 → 1 条红（filled 虚报）。两模块评审均为 REVISE 后修订通过：attributor 2 条必改（M1/M2），cli 3 条必改（R1/R2/R3）。提交：attributor `742a442`（合入时全局 238 passed）、cli `d3aefbb`（合入时全局 292 passed）、收尾固化 `ef25a9c`。

## 4. 踩坑全记录（每坑一卡）

### 坑 1｜进程内连跑：StaticLTM 单例串场
- **现象（实证，非推测）**：`StaticLTM.__new__` 用类属性 `_instance` 只初始化一次（`core/memory/static_ltm.py:13-33`），进程内跑第二个 feature 会拿到第一个 feature 的测试数据（串场）。
- **后果**：同进程连跑多 feature 的结果互相污染，且极难察觉。
- **解法**：每个 feature 一个子进程（PLAN §3.3 第一天定死）；run 的 project_root 也是每 run 独立（`dev_runs/cli_runs/<run_id>/opt`）。
- **出处**：PLAN.md §3.3；dev_docs/interview/qa-bank.md:67。

### 坑 2｜进程内连跑：config 单例重置丢内存态
- **现象**：`SingletonConfigManager.instance()/reset_instance()`（`config.py:1270-1285`）——同进程内重置 config 单例会丢内存态，reset 后再 instance 的行为不等于全新进程。
- **解法**：子进程天然全新 config。
- **出处**：PLAN.md §3.3。

### 坑 3｜进程内连跑：import 时时间戳目录混叠
- **现象**：`config.py:14` 的 `TS = get_timestamp_str()` 在**模块 import 时**求值一次，`config.py:60` 的 `self.timestamp = TS` 把它带进每个实例——同进程连跑共享同一 import 时时间戳，输出目录（`output/run_<TS>` 一族）证据混叠。
- **解法**：子进程各自的 import 时刻即各自的目录命名空间。
- **出处**：PLAN.md §3.3；config.py:14,60（本卡为档案核对时实读验证）。

### 坑 4｜进程内连跑：PlaywrightManager fire-and-forget 关闭竞态
- **现象**：`PlaywrightManager.close_instance` 用 `asyncio.create_task(instance.stop_playwright())` 发后不管（`core/playwright_manager.py:125-142`），关闭与下一个场景的浏览器启动存在竞态。
- **先例佐证**：上游自己的 MCP server 也回避进程内执行——`mcp_server.py:161-172` 用 `asyncio.create_subprocess_exec` 起 `python -m testzeus_hercules` 子进程、300s 超时 kill。"上游自己都这么做"是照抄成熟先例的立论。
- **解法**：子进程 + `start_new_session=True` 独立进程组；超时 `os.killpg(SIGKILL)` 连浏览器子进程一起杀（runner.py:353-391）。
- **出处**：PLAN.md §3.3；mcp_server.py 实读。

### 坑 5｜子进程模式代价（白送三样的对价）
- **现象**：每个场景多一次浏览器/引擎冷启动，单场景约 60 秒起；token 成本显著——F1 简单流程实测 7.3 万 token，exp001 三十次执行共 4,109,386 token（均值 136,980/次）。
- **换来的**：超时可杀（进程组 SIGKILL）、崩溃不传染（子进程崩不伤编排层）、天然可并行（每 feature 独立进程与项目根）。
- **口径**：PLAN §3.3 预算"代码内实证单场景约 4.7 万 token / $0.13（junit_helper.py:322-341 示例数据 0.129235），README 上限 $0.20/复杂场景，全实验 $10-20"；实测口径以 experiment-report 为准（deepseek 前缀缓存显著折扣实际费用，litellm 无该型号价格表 → cost_unavailable，以 token 计量）。
- **出处**：PLAN §3.3；experiment-report.md:25；qa-bank.md:68。

### 坑 6｜review M1：S3 主扫描语料二义会让旗舰签名在真实运行系统性失配
- **现象**：spec 初稿 S3 语料列写"FAIL（主）"，但源码数据流是 nav 轮次耗尽串经 `_executor_node` 的 `HumanMessage` 回灌 planner 会话、存进 `agent_inner_thoughts.json`（simple_hercules.py:805-808 → runner.py:125-135）——**规范落点是 THOUGHTS**；进 FAIL/SYSOUT 仅当 planner 恰好复述（LLM 行为不可依赖）。按 FAIL-only 实现，S3 在真实运行大概率失配，而验收用例 14-16 全用合成 FAIL 文本，测不出此偏差。
- **解法**：spec 钉死"S3 主复合匹配按默认三字段顺序扫（两子串须同字段），S12 语料并入 THOUGHTS"，并在 §3.1 加数据流注记；测试补"主命中在 FAIL 而信号在 THOUGHTS"用例。
- **教训**：签名表的"准"不只在匹配串逐字对，还在语料范围要跟数据流走。
- **出处**：attributor review.md M1；spec §3.1/§3.2。

### 坑 7｜review M2：excerpt 追加装饰文本会误杀合法 LLM 引用
- **现象**：初稿 §4.2-2 让带轮次锚的截图文件名"随该轮 excerpt 附文件名"，但回查要求 excerpt 是 referent（该轮 content_text）的子串——追加装饰文本必判 `excerpt_mismatch`，凡带锚轮次的合法引用被系统性误杀 → 存活引用不足 → 最有归因价值的场景（not found 命中轮 + 对应截图）反而被压成 inconclusive。
- **解法**：截图单独打包为 kind=screenshot 的 EvidenceItem（回查走"存在即通过"分支）；thoughts excerpt 定为纯截断，测试断言 `excerpt == content_text[:400]`。
- **教训**：回查不变量（子串校验）要反过来约束上游打包格式，两处规格必须同句措辞。
- **出处**：attributor review.md M2；spec §4.2-2/§4.4-3。

### 坑 8｜cli review R1：`events=[]` 不可达 + 测试接缝拿不到 page
- **现象**：spec 初稿让"stop_when 立即 True → events=[]"，但 `recorder.js` 的 `start()` 固定 push 一条初始 navigate 事件（recorder.js:518），playwright 模式事件数下限为 1——用例 6 物理跑不通；接缝签名 `stop_when: Callable[[], bool]` 让测试闭包拿不到 page 句柄，无法驱动点击，用例 5 同样跑不通。
- **解法**：接缝改 `stop_when(page)`；空录制语义改"`len(events) <= 1` 即 warning 照常保存"。
- **教训**：审查纪律"用例清单里物理上跑不通的条目按必改计"拦住了两条会让实现者自行改设计的规格。
- **出处**：cli review.md R1；cli.py:87-88（`EMPTY_RECORDING_MAX_EVENTS=1` 注释明写"初始 navigate 由 start() push"）。

### 坑 9｜cli review R2：failure_message 的 JUnit 路径不经 runner 脱敏
- **现象**：runner 只对子进程 stdout 过 `mask_secret`（runner.py:305 写 stdout.log）；`STATUS_FAILED` 的 failure_message 直接取自 JUnit XML（runner.py `_collect_result` → `parse_junit_xml`），不经任何脱敏——spec 初稿"内容已被 runner 脱敏"失实，且原用例清单没有覆盖它。这是 CLI 日志里唯一未过脱敏的外部文本面。
- **解法**：CLI 打印前自行 `mask_secret(result.failure_message or "", key)` 兜底；用例 23 用内嵌假 key 的 failure_message 假 RunResult 断言日志明文零命中；变异测试验证去掉这行 → 4 条红。
- **教训**：密钥红线不能靠"引擎大概率不会回显 key"成立，外部文本进日志前一律过 mask。
- **出处**：cli review.md R2；cli.py:465-467。

### 坑 10｜cli review R3：test-data 值契约与 distiller 不一致会虚报 filled
- **现象**：spec 初稿接受 `bool` 值且未规定空白串，但 distiller `_clean_test_data_values` 丢弃 bool/null/空白（events.py:146-159）——按初稿实现会"摘要 filled=1 但产物仍留占位符且无 missing 警告"，P0-2 缺值警告闭环在该输入类上失效。
- **解法**：CLI 值集与 distiller 对齐（bool/null/空白=未提供，object/array=exit 2 列坏键）；变异测试验证放宽契约 → 红且 filled 虚报。
- **教训**：跨模块值契约以消费方为准逐字对齐，"宽收"制造摘要与产物的矛盾。
- **出处**：cli review.md R3；cli.py:312-340。

### 坑 11｜fixture 乱序：recorder 与 cli 子集连跑会 6 红（终审新发现）
- **现象**：`pytest tests/record2gherkin/recorder tests/record2gherkin/cli`（recorder 在前）挂 cli 的 6 条 record 用例——recorder conftest 的 **session 级 `browser` fixture 全程持有 `sync_playwright()`**，与 `cli.py:160` 的第二个 sync 实例冲突；反序（cli 在前）75 例全过。终审实测复现。
- **定性**：测试夹具的 session 级资源隔离问题，不是产品缺陷；官方入口全量 `uv run pytest tests/record2gherkin -q`（默认字母序 cli 在前）恒绿（292/492 passed）。
- **处置**：终审不改代码（修复属一行 fixture scope 调整）；README 写明"跑测试只用一条命令"。
- **出处**：final-analysis.md §4 R4；record2gherkin/README.md 演示与测试注意。

### 坑 12｜record 跨文档导航即中止（exit 2 不落盘）
- **现象**：注入的 recorder.js 随文档销毁，MPA 站点一跳转 `R2GRecorder.status()` evaluate 就抛错——CLI 检测后 error 日志（error 文案内嵌"recorder spec §4.6.3 已知限制"）并 exit 2，不保存、不产脏数据。
- **取舍**：重注入 + 多段合并明确 Out of Scope；演示与评测选 SPA/file:// 页面；`--manual`（bookmarklet）作为分页录制的替代路径。
- **出处**：cli spec §2.1 步骤 4 / §10；cli test-report 已知#1；final-analysis R1。

### 坑 13｜run 中途 Ctrl+C 遗留 Hercules 子进程
- **现象**：`run_feature` 未暴露协作式中断；run 期间 Ctrl+C 抛 KeyboardInterrupt 且子进程组独立存活。
- **处置**：CLI 捕获后 warning"子进程可能仍在后台运行，请等待其自然结束或手动 pkill"并 exit 130；不为 CLI 单独改 runner（Out of Scope）。第二次 Ctrl+C 的 record 强制中止路径同理无自动化测试（SIGINT 需落进特定窗口才可复现，代码路径存在）。
- **出处**：cli spec §5.5/§7；cli test-report 已知#2/#3。

## 5. 面试深挖要点（追问 → 一句话答 → 指向哪节）

| 追问 | 一句话答 | 指向 |
|---|---|---|
| "LLM 归因不就是猜？" | 规则层先定类（引擎固定错误串命中即准），LLM 只碰模糊区且每条引用被程序回查——引用号要存在、轮次要界内、摘录必须是原文子串，查无此证强制降级"无法定位"压 0.30 | §3.3 |
| "规则层凭什么说零模型？" | evidence/rules/report 三文件不 import 任何 LLM/网络模块，两条机械测试：源码 grep + 子进程真实 import 后查 sys.modules | §3.8（B 组 #27） |
| "五档怎么来的？和 PLAN 三分类什么关系？" | 任务指派即修订：agent_limit 从 environment 分出避免误导排查方向，inconclusive 给"宁可疑而不定"一等公民出口 | §2 A2 |
| "置信度 0.95/0.30 这些数怎么定的？" | 按签名文本与根因的确定性分层：固定引擎串 0.90-0.95、浏览器/工具错误 0.75-0.85、语义歧义的 not found/404 封顶 0.65-0.75 且必附替代假设；一切降级 0.30 | §3.2/§3.3 |
| "S3 为什么搞复合签名？" | 轮次耗尽是表象：主命中后对 thoughts 二跳复扫，not found→test_rot 0.75、网络→environment 0.75、都没有→agent_limit 0.90 | §3.2 S3 行 |
| "真实失败验证做了吗？" | 两条：F5/M4 唯一失败格 S10 路由+无 LLM 降级，报告证据直指 `Missing: '去下单' (found '马上下单')`；F1 全链路真实验收 run passed 62s/7.3 万 token | §3.4 |
| "LLM 层现在生产可用吗？" | 如实：--llm 真机未验收，但降级与回查路径全覆盖测试、规则层真实验收过——话术是"LLM 层标为待验收"，绝不说生产可用 | §3.4 |
| "进程内跑不行吗？为什么非子进程？" | 四个实证坑：StaticLTM 单例串场、config 单例重置丢内存态、import 时时间戳目录混叠、PlaywrightManager 关闭是发后不管；而且上游自己的 MCP server 就用子进程（300s 超时 kill），照抄成熟先例 | §4 坑 1-4 |
| "子进程的代价？" | 每场景一次冷启动约 60s 起、简单流程 7.3 万 token；换来超时可杀、崩溃不传染、天然可并行 | §4 坑 5 |
| "key 怎么保证不泄漏？" | 全链四环：文件读取（KV 解析）→ 只经 env 注入（argv 干净）→ 所有外部文本进日志前 mask_secret（CLI 对 JUnit failure_message 二次脱敏是审查抓出来的缺口）→ 双 key 全目录 grep 零命中实证 | §3.7 / §4 坑 9 |
| "dry-run 为什么不需要 key？" | 只 build_run_plan 打印不执行；无 key 特批 DRYRUN-PLACEHOLDER 占位串；env 打印经 env_redacted 只列五键且 key 显示 ***REDACTED*** | §3.6 |
| "135 条测试质量如何？" | 不是凑数：合成 fixtures 全离线、降级路径逐条覆盖、e2e 机械回查每条引用、还做了变异测试（去掉二次脱敏 4 条红、放宽值契约 1 条红） | §3.8 |
| "全量跑绿，子集跑会红？" | recorder 的 session 级 browser fixture 与 cli 的 sync_playwright 实例冲突，特定顺序 6 红；官方入口全量恒绿，定性为夹具隔离问题非产品缺陷 | §4 坑 11 |
| "占位符密码怎么落地的？" | `{{TEST_DATA:*}}` 引擎侧没有机械消费者（phase1 分析 P0-2 实证），所以 generate 期两段式：骨架扫键 → 交集填充 → 缺值保留占位符并逐键警告 | §2 C3/C4 |

## 6. 数字与出处对照表

| 数字 | 含义 | 出处（文件:行/节） | 复核 |
|---|---|---|---|
| 13 | 规则签名条数（S1-S13） | rules.py:147-279；attributor spec §3.2；final-analysis §2 表 | ✅ 实读代码 |
| 5 | 分类档数（product_bug/test_rot/environment/agent_limit/inconclusive） | evidence.py:34-37；attributor spec §1.1 | ✅ 实读代码 |
| 0.95 / 0.90 / 0.85 / 0.80 / 0.75 / 0.70 / 0.65 | S1/S2·S3c·S5/S4·S6·S12/S7·S3ab/S8/S9 置信度 | rules.py:147-279 | ✅ 实读代码 |
| 0.30 | DEGRADED_CONFIDENCE（降级置信度） | llm_layer.py:74；attributor spec §4.5 | ✅ 实读代码 |
| 2 / 60s | MAX_ATTEMPTS（总尝试）/ 单次调用超时 | llm_layer.py:62,59 | ✅ 实读代码 |
| 30 / 400 / 20 / 6 / 2000 | 证据包上限 / excerpt 截断 / 截图名上限 / 上下文轮数 / 各 2000 字符截断 | llm_layer.py:64-71 | ✅ 实读代码 |
| [0.05, 0.95] | confidence clamp 区间 | llm_layer.py:72-73 | ✅ 实读代码 |
| 81 | attributor 测试数（evidence 13 + rules 27 + llm_layer 29 + report 5 + api_e2e 7） | attributor test-report §1；STATUS.md L14 | ✅ 本机 collect 实测 |
| 54 | cli 测试数（parser 8 + record 6 + generate 13 + run 17 + analyze 10） | cli test-report §1；STATUS.md L15 | ✅ 本机 collect 实测 |
| 42 / 32 | attributor spec 用例数 / cli spec 用例数（实现展开为 81/54） | attributor spec §7；cli test-report §1（32→37+17） | ✅ 文档 |
| 2 / 3 | attributor review 必改（M1/M2）/ cli review 必改（R1/R2/R3） | 两份 review.md 结论行 | ✅ 文档 |
| 23 / 11 | attributor 已知问题 / cli 已知问题条数 | attributor test-report §5；cli test-report §6 | ✅ 文档 |
| 62s / 7.3 万 token | F1 真实链路验收（run 真实执行 passed） | STATUS.md L15、收尾行；experiment-report.md:25；demo-video-report.md:60 | ✅ 三处一致 |
| 900 | DEFAULT_TIMEOUT_S（run 子进程默认超时） | runner.py:35；cli spec §1.1 | ✅ 实读代码 |
| 0/1/2/3/4/130 | CLI 退出码全表 | cli.py:57-62；cli spec §1.2 | ✅ 实读代码 |
| 5 | dry-run 打印的 env 键数 | cli.py:94 | ✅ 实读代码 |
| 500 / 60 | failure_message 日志截断 / analyze 摘要 scenario 截断（字符） | cli.py:99-101 | ✅ 实读代码 |
| 0.2s / 30_000ms | record 轮询间隔 / goto 超时 | cli.py:85-86 | ✅ 实读代码 |
| 300s / 60s | 上游 MCP server 子进程超时（run_tests / 另一处） | mcp_server.py:169,119 | ✅ 实读代码 |
| 8（×2=16） | MASK_FRAGMENT_LEN：secret ≥16 字符时头尾各 8 字符片段也脱敏 | runner.py:38,186-189 | ✅ 实读代码 |
| 0 | dry-run 日志假 key grep 命中数（`grep -c dummy-key-0123456789`）；占位符检查 `{{TEST_DATA` 命中 0 | cli test-report §3-4/§5 | ✅ 文档（实测记录） |
| 35 / 49 | 双 key grep 零命中验证：LLM-Key.txt 35 字符值、GLM-Key.txt KEY 49 字符 | security-review-r2-post.md §2.3；security-review-r3-post.md §2 | ✅ 文档 |
| 4,109,386 / 136,980 | exp001 三十次执行总 token / 均值 | experiment-report.md:25 | ✅ 文档（终审曾逐行复算） |
| 约 60s / 7.3 万 | 子进程冷启动代价量级 / 单场景 token | qa-bank.md:68 | ✅ 文档 |
| 4.7 万 / $0.13 / $0.20 | PLAN 预算口径：代码内实证单场景 token / 成本 / README 上限 | PLAN.md §3.3；junit_helper.py:322-341（示例 0.129235） | ✅ 实读代码 |
| 238 / 292 / 492 | attributor 合入时全局 passed / cli 合入时全局 passed / 最终全局基线 | git log（742a442/d3aefbb 实查）；STATUS.md L20 | ✅ git 实查 |
| 742a442 / d3aefbb / ef25a9c | attributor 合并 / cli 合并 / 收尾固化（实验报告+README）提交 | STATUS.md L14-15、收尾行；git log 实查 | ✅ git 实查 |
| `Missing: '去下单' (found '马上下单')` | F5/M4 失败格根因证据原文 | experiment-report.md:31；resume.md:140 | ✅ 文档 |
| 1 / 0 | F5/M4 报告语义：证据命中文案改写 1 处直指根因；LLM 调用 0 次（analyzer=None 降级） | experiment-report.md:31 + attributor spec §4.5 推导 | ✅ 推导自代码契约 |
| 6 / 75 | fixture 乱序红条数 / 反序全过条数 | final-analysis.md §4 R4 | ✅ 文档（终审实测） |
