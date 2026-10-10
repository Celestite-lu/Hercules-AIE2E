# 全档案 01：录制器（recorder）+ 蒸馏器（distiller）

> 覆盖范围：`record2gherkin/recorder/`（注入式 JS 事件录制器 + bookmarklet 打包）、`record2gherkin/distiller/`（事件流 → Gherkin 确定性三层蒸馏），以及两者之间的契约（PLAN.md §3.1 事件 Schema v1）与合并后修复（P0-1 / P0-2 / 快照归因 / P1-4）。
> 材料来源：dev_docs/recorder/{plan,spec,review,test-report}.md、dev_docs/distiller/{plan,spec,review,test-report}.md、dev_docs/reports/phase1-merge-analysis.md、PLAN.md §0–§3.2、record2gherkin 源码、tests/record2gherkin/{recorder,distiller}/、git 历史（4283ebe / 9664cd3 / 0ad2075 / fe11083 / 7ce6c95）。
> 铁律声明：本文所有数字逐字来自源文档；源文档之间存在矛盾的，两说并注出处；无法核实的标注"未核实"。

## 1. 这一页讲什么（3 句）

录制器是一个注入普通同源页面的单文件 JS（`window.R2GRecorder`，bookmarklet / DevTools Snippet 分发），把"人工点一遍页面"忠实捕获为 Schema v1 的 JSON 事件流（navigate/click/input/select/submit 五类），带 target 语义（name 五级优先级 + ordinal 消歧）、密码掩码、去噪与 300ms 延迟快照差集 `assert_texts`（Then 断言的 grounding 素材）。蒸馏器把事件流确定性蒸馏为恒合法的 Gherkin（1 Feature 1 Scenario）：规则模板层零模型依赖出骨架，可选 LLM 润色层做纯加法，事实回查层做硬闸——每个引号字面值必须 ⊆ 录制事实集，任一道闸不过就**整体**回退骨架；骨架自检失败则抛 `DistillError`（唯一主动抛错路径，视作程序 bug 不兜底）。两模块各自 20 条 spec 用例 / 35 条 spec 用例起家，经合并后分析（phase1-merge-analysis）修复 P0-1（click+submit 双事件去重）、P0-2（占位符注入落 CLI）、快照归因缺陷（Then 早于状态变化的假失败）、P1-4（标题文件名清洗）后，定格为 recorder 21 例 + distiller 68 例全绿（全局 492 passed 中的两个组成部分）。

## 2. 设计决策总表

| # | 决策 | 为什么 | 考虑过/否决的备选 | 否决理由 | 出处 |
|---|---|---|---|---|---|
| D1 | 录制形态选**注入式 JS**（bookmarklet 或 DevTools Snippet 的 content script），零安装 | 演示友好：复制一条 `javascript:` 书签即可在任意同源页录制；事件已落 JSON（stop/getJSON 随时可取），新页面重新注入即可续 | 浏览器扩展形态 | 需要 manifest/加载机制与商店审核，注入式已满足零安装演示（recorder spec §9）；跨文档导航杀死注入 JS 是该形态固有边界，如实记已知限制不解决（recorder plan §2/§5） | recorder plan §2；spec §4.6.3/§9；PLAN §3.1 |
| D1a | （附注）**Playwright tracing / codegen 作为录制替代**：源文档（recorder plan/spec/review、PLAN §3.1）**均未记录**该备选的显式取舍——未核实。可核实的相近事实是：评测框架用 Playwright **回放动作**驱动录制器（`record2gherkin/evaluation/record_flows.py`），且 PLAN §3.4 用 Playwright codegen selector 脚本仅作**存活率对照基线**，不是录制手段 | — | — | — | PLAN §3.4；evaluation test-report：113；本档案标注"未核实" |
| D2 | 事件监听用 `document.addEventListener(type, handler, /*capture=*/true)` **事件委托**，单套监听器覆盖全页 | 动态插入的元素天然被覆盖 | 逐元素绑定 + MutationObserver | 需遍历 + MutationObserver，复杂且漏绑（recorder plan §2） | recorder plan §2 |
| D3 | input 用**定稿制**：只听 `change` + `focusout`，按"元素 + 焦点会话"（`focusin` 开新会话）去重，记最终值 | 一个输入框多按键只落一条最终值事件 | 每 keydown 记一条 | 事件爆炸、无意义；change 与 focusout 双保险覆盖两种结束方式（recorder plan §2）。focusout 是捕获阶段安全的 blur 等价实现（recorder review 契约核对表） | recorder plan §2；spec §4.3 |
| D4 | 快照采集：每个有效事件入队后 `setTimeout(..., 300)` 延迟抓可见文本，与上一次快照文本集做**差集**得 `assert_texts` | 延迟让点击引发的 DOM 更新先落地；差集保证 assert_texts 是"这次操作**新出现**的文本"，才是有效断言依据 | 全量文本入流 | 体积大、下游无法区分新旧（recorder plan §2） | recorder plan §2；spec §3 |
| D5 | navigate 捕获：start() 记初始 navigate + patch `history.pushState/replaceState` + 监听 `popstate/hashchange` | 覆盖 SPA 路由；跨文档导航（整页刷新/跳转）使注入 JS 销毁——bookmarklet 形态固有，如实不解决 | — | — | recorder plan §2；spec §4.6 |
| D6 | `target.name` 五级优先级：aria-label → associated label（`label[for]` → `aria-labelledby` → 包裹型 `closest("label")`）→ placeholder → innerText → title | PLAN §3.1 契约顺序不可改；属性集合与 Hercules `getAccessibleName`（get_detailed_accessibility_tree.py:1050-1087）"同源"，保证蒸馏产物与引擎感知口径一致（该"同源"说法的精度争议见踩坑 K16） | 对齐引擎实际手动链（含 alt/value 级） | 契约是唯一权威，录制器/蒸馏器均遵契约；顺序差异是 PLAN 层既定决策（recorder review 备注 a） | recorder spec §2.1.2；recorder review 备注 a |
| D7 | bookmarklet 单行化：python 读 recorder.js → IIFE 包裹 → `urllib.parse.quote(payload, safe="")` **全量 URL 编码** → 拼 `javascript:` 前缀 | 全量编码比手写 minify 稳（不会改坏语义）；Chrome 对 bookmarklet 长度上限宽松（>100KB），无需压缩 | 手写 minify / 正则删注释；npm/webpack/esbuild 工具链 | minify 有改坏语义风险；本机无 Node 且引入工具链违背"小而准"（recorder plan §2；spec §6/§9） | recorder plan §2；spec §6 |
| D8 | 测试形态：pytest + playwright（uv 已有），file:// 打开唯一 fixture → `page.evaluate` 注入 recorder.js 源码 → `page.click/fill` 模拟操作 → `json.loads(getJSON())` 断言 | 与生产注入路径同构（都是 evaluate 注入同一份 JS 源）；录制器不依赖网络/cookie，file:// 下 history.pushState 可用 | 起 HTTP 服务 | 无需（recorder plan §2/§5） | recorder plan §2；spec §7 |
| D9 | `dom_snapshot` 字段**恒存在、恒为对象**（初始 navigate 为 `{"assert_texts": []}`） | review 必改 1：spec 三处定义互相矛盾（示例 null / §4.6.1 null / 字段表恒存在），实现者无法同时满足；统一后与"整字段不省略"、用例 17 键存在性断言自洽 | 初始 navigate 为 null（旧设计） | 是"无基线"两代设计残留，未收敛（recorder review 必改 1） | recorder spec §2/§3；review 必改 1 |
| D10 | 快照差集文本**归给触发时刻最新已入队事件**（追加写入、8 条上限），而非调度该快照的事件 | input 在下一次交互的 focusout 时才定稿入队（先于引发 DOM 变化的 click），若按调度事件归因，Then 会出现在状态变化之前，重放必假失败；归给更晚的事件总是安全的（文本在该事件之后持续可见） | 按"调度该快照的事件"归因（原始设计） | 集成冒烟实证产生"Then 早于状态变化"假失败（recorder spec §3 修订；test-report 补记；提交 0ad2075） | recorder spec §3（2026-09-21 修订）；test-report 补记 |
| D11 | 点空白判定**取严**：target 为 document/html/body 的 click 一律丢弃 | spec §4.2 字面是"且 name 为空才丢弃"，但 body/html 的 innerText 是整页文本，字面实现会把整页文本写成 name，与 §4.7 去噪总表"点空白 → 0 条"矛盾；取最简确定的语义 | 按 spec 字面（name 为空才丢） | 与去噪总表矛盾（recorder test-report 已知问题 1；phase1 P2-3 列为"有意为之的 spec 文字漂移"） | recorder test-report §6.1；recorder.js:351-355 注释 |
| D12 | 蒸馏输出恒为 **1 Feature 1 Scenario**，骨架措辞统一英文（元素录制名保持原语言）；标题做单行化 + 120 字符截断，**标题行不受事实回查约束** | 消费方是 Hercules `split_feature_file`（1 场景一文件）；标题行不是步骤字面值来源，回查约束只作用于步骤行（distiller spec §2.1/§5.2） | 多 Feature 切分、中文/双语模板 | Out of Scope（distiller spec §8） | distiller spec §2.1/§8 |
| D13 | 蒸馏三层架构：**规则模板层（纯规则零模型）→ 可选润色层 → 事实回查层（规则层）**；骨架自带一次回查自检（防御性，应恒过） | 核心目标三条刻死推论：规则层不 import LLM 不发网络；任何输入必产合法 feature 绝不抛异常中断；润色是纯加法，出错一律整体丢弃回退骨架。骨架输出的保证不在 prompt 而在回查层 | 让 LLM 直接从事件流生成用例（玩具级上游 prompt 路线） | 对模型能力零依赖是既定设计：PLAN §3.2"确定性优先"，上游 guided 模式 prompt 是玩具级（utils/test_builder.py:13-28） | distiller plan §1/§2；PLAN §3.2 |
| D14 | 骨架自检失败**不回退、直接抛 `DistillError`**（唯一允许骨架层抛错的场景） | 模板由事实集生成、按构造应恒过；若不过说明校验器/模板有 bug，静默兜底会掩盖 bug（distiller spec §2.5/§5.3）；测试必须覆盖恒过不变量 | 静默回退/兜底 | 会掩盖校验器 bug（distiller spec §5.3） | distiller spec §2.5/§5.3；api.py:39-47 |
| D15 | click+submit 双事件去重放**规则层（templates）**而非润色层，是规则层唯一例外合并 | 去重是确定性规则、零模型依赖；润色白名单只允许合并 input。契约原本悬空：recorder spec §4.2 写"蒸馏器负责合并"，distiller 润色白名单却不含——phase1 判定为"两份 spec 互相指派、无人落地"的唯一跨模块契约断点 | 放润色层合并；或保留 submit 丢 click | 非确定性、不可测；click 带按钮 name hint 重放定位更稳（distiller spec §2.2；test-report 补记；phase1 P0-1） | distiller spec §2.2；test-report 补记；phase1 P0-1 |
| D16 | 润色失败**整体回退**，不做部分采用；骨架稿是唯一回退目标 | 部分采用需要逐行可信度判定，复杂且引入新的出错面；骨架永远在手上，`fallback_reason` 记原因即可 | 部分采用润色稿 | distiller spec §5.3 明文"不存在部分采用" | distiller spec §5.3 |
| D17 | 掩码值：`value == "<masked>"`（实现为 `strip()` 后比较的超集判定）→ 占位符 `{{TEST_DATA:password_{seq}}}`；`test_data_values` 提供真值则填充且真值入事实集；未提供则保留占位符（显式 TODO） | 录制层绝不记录真实密码；占位符不参与事实回查（豁免）；`<masked>` 本身不入事实集（它不是事实） | 把 `<masked>` 当明文写进用例；或把掩码值排除出步骤 | 绝不泄漏敏感值（验收标准 7 机械检查 masked_leak）；`strip()==` 超集是"宁多掩不可漏"的刻意取舍（test-report 已知问题 4） | distiller spec §1.1/§2.3；test-report §4.4 |
| D18 | 模板静态兜底词 `element`/`field`/`dropdown`/`form` **豁免事实回查**（视为模板常量，不参与校验） | review 必改 1：hint 兜底链末端是写死引号字面值，不在事实集也不在豁免清单 → target 缺失时骨架自检必失败抛 DistillError，直接违反"任何输入恒产合法 feature"；不推荐把它们加进事实集（污染 ground truth 语义） | 把四词加入事实集 | 污染 ground truth 语义（distiller review 必改 1） | distiller spec §5.2；review 必改 1 |
| D19 | 结构校验用**依赖无关的正则**实现（venv 无 gherkin 解析器、不新增依赖），且**每行先 strip 再锚定/关键词检查** | PLAN §3.2 说"可用 gherkin_helper 已有解析"是"可用"非"必须"，正则校验 + 用例 34 接 `split_feature_file` 冒烟等价细化；strip 是 review 必改 2：LLM 润色稿普遍带缩进，`clean_gherkin_output` 不做缩进归一化，`^` 锚定会把缩进产出系统性判非法 → 润色层在生产上永远回退 | 用 gherkin 解析库 | venv 无解析器，不新增依赖（distiller spec §5.1；review 必改 2） | distiller spec §5.1；review 必改 2 |
| D20 | 字面值匹配：`norm(L)==norm(F)` **或 `norm(L) ⊆ norm(F)`**（子串方向 L⊆F），空白归一化、**大小写敏感**；转义契约 `escape_literal`（先 `\` 后 `"`）与 `unescape_literal` 互逆 | 允许 L⊆F 是为容忍润色截取（断言长句取短语）；空白归一化与下游 `serialize_feature_file` 的空格折叠兼容（gherkin_helper.py:102-134） | 词序子集匹配（非连续词组也算命中） | test-report 已知问题 1：按 §5.2 严格子串实现；若评测期误杀真实润色稿再考虑放宽（会削弱硬闸，需评审） | distiller spec §5.2/§2.4；test-report §4.1 |
| D21 | 断言存活检查方向固定：**润色字面值 ⊆ 骨架 Then 字面值**（只许缩短不许加长） | review 建议 3：⊆ 方向的唯一合理解读；双向子串会把"润色稿加长断言"也放行，削弱硬闸（factcheck.py dropped_assertions docstring 明注） | 双向子串匹配 | 削弱硬闸（distiller review 建议 3） | distiller review 建议 3；factcheck.py:141-157 |
| D22 | Scenario/Feature 标题做**文件名安全清洗**：仅保留 `[字母数字下划线、CJK、空格、连字符]`，其余连续段折叠为单个 `_`，首尾 `_`/空格剥除，清洗后为空回退默认标题 | 标题会被上游用作 JUnit 文件名（gherkin_helper.py:70-71 只 `replace(' ','_')` 无安全化），`Recorded flow on http://...` 中的 `://` 产生的 `//` 被当作目录分隔符导致写文件失败（pilot002 复盘，阶段 1 分析 P1-4，提交 7ce6c95） | 上游 split 前 sanitize（评测框架做） | 落在蒸馏器源头更稳，规范固化在 distiller spec §2.1（spec 原文） | distiller spec §2.1；templates.py:52-59；phase1 P1-4；7ce6c95 |
| D23 | Polisher 协议：单方法 `async polish(skeleton_text, events) -> str`；`DefaultPolisher` 走 litellm 弱模型档（role key `distiller_polish`）、温度 0、总尝试 ≤2 次、单次超时 60s；异常向上抛由 api 层捕获降级 | 异步便于接 litellm；协议实现自身不做回退决策（职责分离）；弱模型档即可因骨架是规则出的（PLAN §3.3 路由表）；litellm/gherkin_generator 均函数内惰性 import，import 本模块不触网络 | 协议内自带回退 | 违反职责分离（polisher.py docstring） | distiller plan §3；spec §4.1；polisher.py:14-18/119-138 |
| D24 | 润色 prompt 四条白名单 + 硬约束：只许合并"连续且同一目标字段"的 input（保留最终值）、改措辞、改标题；禁止增删 Then、禁止动任何引号字面值、禁止改顺序与关键词语义、恰好 1 Feature 1 Scenario、每行顶格、只输出 feature 本体 | 润色是纯加法，prompt 是第一道软约束，回查是硬闸兜底；"顶格输出"是 review 必改 2 后补的双保险（校验端 strip 才是必须） | 无约束自由润色 | 会丢断言/编字面值（distiller spec §4.2；polisher.py SYSTEM_PROMPT） | distiller spec §4.2；polisher.py:21-35 |
| D25 | 润色产物**四道闸依次执行，任一失败整体丢弃**：① `clean_gherkin_output` 清洗后非空（空 → `polish_empty`）② 结构校验（→ `syntax_invalid`）③ 事实回查（→ `fact_check_failed:<字面值列表>`）④ 断言存活（→ `assertion_dropped:<字面值列表>`）；另有 polish 抛异常 → `polish_error:<ExcType>`；原因中的字面值列表最多列 5 个 | 顺序由廉价到昂贵、由格式到语义；五条失败路径全部收敛为"回退骨架 + fallback_reason"（distiller spec §3/§4.3/§5.3） | — | — | distiller spec §3/§4.3/§5.3；api.py:50-67 |
| D26 | 骨架自检的**检查动作放在 api 层执行**（templates.py 保持纯函数、不 import api 的异常类型） | 避免循环依赖（templates → api）；对外可见行为与 spec 一致（DistillError 仍是唯一主动抛错路径） | templates 里直接抛 DistillError | 会造成 templates→api 循环依赖（test-report 已知问题 4，代码注释标注） | distiller test-report §4.4；api.py:39-47 |
| D27 | 所有字符串字段**空串/全空白一律视为缺失**走 fallback 链（`optional_text`） | review 建议 2：`page_title=""`、`url=""`、`target.name=""` 是"缺失"还是"原样填充"未言明；按建议落地最简确定性行为 | 空串原样填充 | 会产出空字面值步骤（distiller review 建议 2；events.py:25-36 docstring） | distiller review 建议 2；events.py optional_text |
| D28 | P0-2 落地：CLI `generate` **两遍蒸馏**——第一遍 `polisher=None` 出骨架提取所需占位符键集，第二遍只喂 `needed ∩ provided` 的 `test_data_values` 终蒸馏；`--test-data` JSON object，blank/bool/null 视为未提供（与蒸馏器同规则），object/array 值为硬错误；missing 键警告 `missing_test_data:<key>` 并保留占位符提示人工填充重跑，多余键警告 `unused_test_data:<key>` | phase1 P0-2：`{{TEST_DATA:password_<seq>}}` 在 Hercules 侧没有任何机械消费者（全仓 grep 无替换逻辑，static_data_loader 只整块注入文件内容），带密码流程重放成败取决于 LLM 猜测；方案取建议 (a) 蒸馏时经既有 `test_data_values` 参数填真实值 | 方案 (b)：约定命名 test_data 文件靠 LLM 推断 | 不确定性大，不推荐作主路径（phase1 P0-2 建议） | phase1 P0-2；cli.py:303-404；STATUS.md D2 |

## 3. 关键实现细节

### 3.1 录制器（record2gherkin/recorder/）

**交付物形态**：`recorder.js` 单文件 IIFE（test-report 时点 561 行 / 16458 字符；快照归因修复 0ad2075 之后现工作区为 575 行 / 18218 字符），零依赖零构建（验收 `grep -c "require(\|import "` = 0）；`build_bookmarklet.py` 纯标准库 47 行（仅 re/sys/urllib.parse/pathlib），产出 `bookmarklet.txt`（test-report 时点 32293 字符，现 34117 字符，无尾随换行——书签内容需与 payload 逐字节一致）。源码首行即 IIFE，`build_bookmarklet.py` 的 `IIFE_PATTERN`（`^\(\s*function\s*\(`）命中则原样返回，保证"解码后与源码逐字节一致"（用例 20）。（出处：recorder test-report §1/§4.2/§4.4；build_bookmarklet.py:22-29/39-40）

**顶层结构与事件 schema（8 键）**：`getJSON()` 返回 `{session, events}` JSON 字符串（2 空格缩进）。`session = {started_at: ISO 8601 UTC, origin: location.origin}`（均 start() 时刻）。每条事件 8 个契约字段：
- `seq`：int，入队即分配（`(eventSequence += 1)`），从 1 起跨类型严格递增不重置；
- `ts`：int，epoch 毫秒，取**监听器收到事件**的时刻（`Date.now()`），非入队时刻；
- `type`：枚举 `navigate/click/input/select/submit`，不新增；
- `url`：事件时刻 `location.href`（navigate 取跳转后 URL）；
- `page_title`：事件时刻 `document.title`，可为空串；
- `target`：Target 对象或 null（navigate 恒 null；submit 指向 `<form>`）；
- `value`：string/null，仅 input/select 有值（规则见下）；
- `dom_snapshot`：恒存在恒为对象，事件入队时即为 `{"assert_texts": []}`，300ms 后**原地填充**。
（出处：recorder spec §1/§2；recorder.js:308-326；test-report §5）

**Target 对象（7 字段）**：
- `tag`：`tagName.toLowerCase()`；
- `role`：显式 `role` 属性优先 → 隐式映射表（最小集：`a[href]`=link；`button`、`input[type=submit|button|reset]`=button；文本类 input + `textarea`=textbox；checkbox/radio 同名；`select`=combobox；其余=generic）→ `"generic"`。注意 review 建议 3a：该词表与 Hercules `getRole`（:1098-1148）不同（select: combobox vs listbox；search: textbox vs searchbox 等），role 为 ARIA 近似词表，定位以 name/hint 为主；
- `name`：五级优先级（D6），清洗 = 首行、trim、压缩连续空白、≤200 字符（`NAME_MAX_LENGTH=200`）；全部未命中 → 空串（此时 ordinal 必为 null）；
- `testid`：`data-testid`，无则 null；
- `id`：`element.id`，空串视为 null；
- `ordinal`：对当前元素算出 name 后，用 `document.querySelectorAll(tag)` 过滤出"同一套 name 逻辑算出的 name 相同"的元素集合，当前元素在集合中的 1-based 序号；**集合长度 >1 才写**，否则 null；不做 CSS path/nth-of-type（定位交给下游语义引擎）；
- `form_label`：仅 input/textarea/select 有值，关联顺序 `label[for]` → `aria-labelledby`（多 id 取第一个可解析元素）→ 包裹型 `closest("label")`，取不到 null。
（出处：recorder spec §2.1；recorder.js:68-194；review 建议 3a/已知问题 3）

**value 字段规则**：文本类 input/textarea 最终值原样（不 trim 内部空白、去首尾换行——该规则仅在 textarea 可观测，`<input type=text>` 的 value sanitization 本身去换行）、上限 500 字符（`INPUT_VALUE_MAX_LENGTH=500`）；password 恒字面量 `"<masked>"`（`MASKED_VALUE`），绝不记录真实值，但 name/form_label/ordinal 照常采集供下游生成 test_data 引用；checkbox/radio = `"true"/"false"`（checked 状态）；file 恒 `"<file>"`（`FILE_VALUE`，文件名不入流）；select = 被选中 option 的 innerText（与 name 同一套清洗截断，≤200 字符）；navigate/click/submit = null。未定义值规则的类型（range/color/hidden/image 等）不产生 input 事件（已知问题 4，最简确定性行为）。（出处：recorder spec §2.2；recorder.js:53-58/382-419；test-report §6.4/§6.8）

**input 定稿制（focusSessions WeakMap）**：`focusin` 对文本类字段建会话 `{initial: 当前值, recorded: false}`；`change`/`focusout` 触发 `finalizeTextInput`：会话已 recorded → 跳过（同会话只记一条）；值 == focusin 初值 → 不记（未修改的 focus+blur 零事件）；无会话的后备规则（如注入前已聚焦）：值非空记一条、空值不记（已知问题 6）。checkbox/radio 走 `change`（不依赖 focusout）；`<select>` 的 change 走独立 select 分支（互斥，不再产生 input）。（出处：recorder spec §4.3/§4.4；recorder.js:359-423；test-report §6.6）

**dom_snapshot 采集（assert_texts）**：
- 触发：每个有效事件入队后 `setTimeout(..., SNAPSHOT_DELAY_MS=300)`；采集期间又来新事件则各自独立调度；`stop()` 取消未决定时器；
- 归因（2026-09-21 修订，D10）：快照触发时刻，差集文本归给 `events[events.length-1]`（当时最新已入队事件），追加写入、遵守 8 条上限；无新增文本则早退（注意：`collectNewTexts()` 在返回前已把 `lastSnapshotTexts` 滚动更新为当前集合，即基线照常前滚）；
- 可见文本候选元素三条件：①可见（`getBoundingClientRect()` 宽高均 >0，`visibility` 为 visible，自身及祖先 `display` 非 none）②叶子文本元素（直接 TEXT_NODE 拼接 trim 非空，且不含含文本的子元素）③标签不在排除集 `SCRIPT, STYLE, NOSCRIPT, TEMPLATE, OPTION, SELECT, INPUT, TEXTAREA, TITLE, SVG`；
- 清洗与截断：压缩连续空白（含换行）为单空格、trim、≤120 字符（`ASSERT_TEXT_MAX_LENGTH=120`）；
- 差集：`当前集合 − 上次集合`，按 DOM 顺序取**最多 8 条**（`ASSERT_TEXT_MAX_COUNT=8`）；
- 基线：`start()` 时抓一次初始可见文本作基线（不写入任何事件）→ 首个 navigate 的 assert_texts 恒为空数组（加载即存在的文本不构成断言依据）；
- 采集是全量遍历（`querySelectorAll("*")` + 逐元素布局读），无节流/增量（spec §9 Out of Scope，已知问题 10）。
（出处：recorder spec §3；recorder.js:196-304；test-report §6.10）

**去噪总表**：hover/mousemove/scroll/键盘事件不记录；一次输入过程（多按键）→ 1 条 input 仅最终值；未修改值的 focus+blur → 0 条；点空白（body/html/document）→ 0 条；重复 navigate（与上一条 navigate URL 相同则丢弃——实现上不限 spec 字面的"同 tick"，phase1 P2-3 列为有意漂移）→ 1 条；双击 → 2 条 click（忠实记录，蒸馏器负责合并——但注意 phase1 未将其列为必改，双击合并不在已实现范围）；submit 按钮点击 → click + submit 各 1 条（忠实记录）。双击产生 2 条 click 是 Chromium 原生事件流（test-report §5 手工验证项）。（出处：recorder spec §4.7；test-report §5/§6）

**navigate 与 MPA 边界**：start() 立即产生 seq=1 初始 navigate（target/value null，dom_snapshot 为空数组对象）；SPA 路由靠 patch `history.pushState/replaceState`（包装原函数，调用后 URL 变化才产生）+ `popstate/hashchange`；**跨文档导航**（整页跳转/刷新）使注入 JS 随文档销毁——导航后页面不被记录，导航前事件已可 getJSON() 取回，新页面重新注入即可；这是 MPA（多页应用）场景的**干净退出设计**：录制器不尝试跨文档续录（不做会话持久化/心跳），而是"如实记录到导航为止 + 人工重新注入"，作为 bookmarklet 形态固有已知限制写入 plan 风险表与已知问题 9。（出处：recorder spec §4.6；plan §2/§5；test-report §6.9）

**公开 API 与幂等注入**：`window.R2GRecorder.{start, stop, getJSON, status, copy}`；start 已在录制中 → no-op 返回 false；stop 卸载全部监听器（capture=true 配对移除）+ 恢复原生 `pushState/replaceState` + 取消快照定时器，已入队事件保留；status 返回 `{recording, event_count}`；copy 走 `navigator.clipboard`，失败 false（仅返回类型被用例断言，已知问题 7）。幂等注入：IIFE 每次执行先 `stop()` 旧实例（try/catch 包裹，旧实例停止失败不影响新实例），再重定义 `window.R2GRecorder` 为全新停止态实例。补充行为（手工脚本验证，不在 20 条用例内）：getJSON() 在 start() 前返回 `{"session": null, "events": []}`；stop() 未开始时返回 false。所有 handler 只读，绝不 `preventDefault/stopPropagation`。（出处：recorder spec §5；recorder.js:498-575；test-report §5/§6.2/§6.7）

**生产环境约束（测试侧同构）**：测试经 conftest 的 `browser`（session 级）+ `recorder_page`（function 级）fixture 注入；`_helpers.py` 的 `validate_event_stream` 是 schema 校验复用入口（spec §8 验收第 3 条要求手动 bookmarklet 流程复用同一校验函数，实际以等价自动化验证——见 K13）；`wait_for_snapshot` 等 `SNAPSHOT_DELAY_MS + 200 = 500ms` 让快照落地。（出处：recorder spec §7/§8；test-report §4.3；_helpers.py:50-52）

### 3.2 蒸馏器（record2gherkin/distiller/）

**模块与红线**：`events.py`（277 行，解析归一化 + 事实集）、`templates.py`（156 行，规则模板）、`factcheck.py`（205 行，字面值提取/结构校验/回查/回退判定）、`polisher.py`（159 行，协议 + DefaultPolisher + prompt）、`api.py`（122 行，公开 API + 自检 + 回退编排）、`__init__.py` re-export 六个符号。骨架路径零模型红线：三模块（events/templates/factcheck）不 import litellm/网络模块（验收标准 4 的负向 grep 机械检查，exit=1 无命中；正向 import 证据：events.py 仅 __future__/re/dataclasses/typing，templates.py 另依赖 distiller 内部模块，factcheck.py 同）；templates.py 引 factcheck.escape_literal 是转义契约唯一实现点（§2.4 与 §5.2 互逆），仍是纯规则模块。（出处：distiller test-report §1/§3 命令 B）

**解析归一化（events.py）**：输入为 dict（含 session/events）或裸 list（session 视为 {}）；事件按 `seq` 升序排序（`_sort_events`：带合法 int seq 的升序在前，缺 seq 的保持数组顺序排其后——已知问题 2 的确定性实现）；`ts` 不参与逻辑仅透传；字段访问全部 `dict.get` 宽松；`type` 不在 5 值白名单 → 跳过 + warning `unknown_event_type:<type>`；空事件列表 → 仍产合法空骨架 + warning `empty_events`。warning 词表全集：`empty_events` / `unknown_event_type:<type>` / `missing_url:<seq>` / `empty_value:<seq>` + 4 个 payload warning（`invalid_input_payload` / `invalid_events_payload` / `invalid_session_payload` / `invalid_event`，均为"宁出合法骨架不抛错"的宽松分支，test-report 已知问题 3）。掩码哨兵：`value == "<masked>"` 精确匹配为 spec 语义，实现用 `value.strip() == MASKED_SENTINEL` 超集判定（已知问题 4，防泄漏优先）。（出处：distiller spec §1.1；events.py:14-17/202-277；test-report §4.2/§4.3）

**事实集（Facts，6 组 frozenset）**：模板生成**之前**从原始事件构建，与骨架文本无关——`urls`（每个非空 event.url + session.origin）、`values`（每个非空且 ≠`<masked>` 的 value）、`hints`（每个 target.name / target.form_label 非空者）、`assertions`（每个 assert_texts 非空元素）、`other`（target.ordinal / tag / role / id / page_title 非空字符串化，ordinal 数值入集如 `"2"`）、`caller_values`（test_data_values 的每个值——填充后的真实值属调用方提供的事实）。`literals()` 并集供回查。（出处：distiller spec §1.2；events.py:100-199）

**规则模板层（templates.py）**：
- 命名：`Feature: Recorded flow on <origin>`（origin 缺失 → `Feature: Recorded flow`）；`Scenario: <第一个事件的 page_title>`（缺失 → `Recorded scenario`）；标题单行化 + 120 字符截断（`TITLE_MAX_LENGTH=120`）+ 文件名安全清洗（D22，`_FILENAME_UNSAFE_CHARS = [^\w\- ]+`，`\w` 含 CJK）；
- 映射表（按 seq 顺序，每事件先输出自身步骤再输出其 assert_texts 的 Then；**首个** navigate 用 Given，其余全 When/Then）：
  - navigate（首个）→ `Given I am on the page "{url}"`；navigate（后续）→ `When I navigate to "{url}"`；url 缺失 → `session.origin` → 再缺 → **整条事件（含其 assert_texts）跳过** + warning `missing_url:<seq>`（spec §2.2 只写"跳过该事件"，实现明确含 assert 一起跳过，phase1 P2-4 记录）；
  - click → `When I click on the "{hint}" {role}`，hint 链 `target.name → form_label → id → tag → "element"`，role 链 `role → tag → "element"`（role 槽不加引号、不参与回查）；`ordinal` 存在且 >1 时尾部追加 ` (occurrence {ordinal})`（ordinal=1 不追加——录制器只在 >1 时写 ordinal，但手造数据可能 ordinal=1，蒸馏器防御一致）；
  - input → `When I enter "{value}" in the "{hint}" field`，hint 链 `form_label → name → id → tag → "field"`；
  - select → `When I select "{value}" from the "{hint}" dropdown`，hint 链 `form_label → name → id → "dropdown"`；
  - submit → `When I submit the "{hint}" form`，hint 链 `name → form_label → "form"`（PLAN §3.2 表未覆盖 submit，此为补充定义）；
  - submit（紧邻前驱为 click）→ **不生成自身步骤**（P0-1 去重，`_dedupes_preceding_click`：`index>0 && type=="submit" && 前驱 type=="click"`），但该 submit 的 assert_texts 的 Then **照常生成**，位置在该 click 步骤及其 Then 之后；前驱非 click 的 submit（如 Enter 键直接提交）不去重照常生成；
  - 任意事件后 → `Then I should see "{text}"`，每个非空 text 一行；**跨 Scenario 去重**（同文本只在首次出现处生成，`seen_assertions` 集合，保持首次出现顺序）；
- 值填充：掩码 → 占位符或真值（D17）；转义先 `\` 后 `"`（escape_literal）；换行/制表符折叠单空格、首尾 strip（fold_literal）；空值步骤仍生成（引号内空串）+ warning `empty_value:<seq>`（事实集空串不入集但 §5.2 豁免空串，自检不挂）；
- 规则层不做任何合并/省略/改写（连续 input、重复 click 原样逐步输出），**唯一例外**是 click→submit 去重（D15）。
（出处：distiller spec §2.2/§2.3/§2.4；templates.py 全文；test-report 补记；phase1 P2-4）

**事实回查算法（factcheck.py，精确语义）**：
- 字面值定义：步骤行（strip 后以 Given/When/Then/And/But 开头，忽略大小写）中正则 `"((?:[^"\\]|\\.)*)"` 的捕获内容，提取时反转义（`\\`→`\`、`\"`→`"`；未知转义保留原文）；Feature/Scenario 标题行、注释行不是字面值来源；
- 匹配：字面值 L grounded ⇔ 存在事实 F 使 `norm(L)==norm(F)` 或 `norm(L) ⊆ norm(F)`；`norm(s)` = 反转义后空白归一化（连续空白折叠单空格、strip），**大小写敏感**；空串字面值视为 grounded；
- 豁免三条款：`{{TEST_DATA:*}}` 占位符（`^\{\{TEST_DATA:[^{}]*\}\}$`，匹配整个归一化字面值）；空串；模板静态兜底词 `element`/`field`/`dropdown`/`form`（lower 比对）；
- 结构校验：逐行 strip 后——首个非空非注释行必须以 `Feature:` 开头；全文恰 1 个 `Feature:` 行、恰 1 个 `Scenario:` 行（忽略大小写）；其余每行必须以 7 个关键词之一开头（Feature:/Scenario:/Given /When /Then /And /But ）；忽略空行与 `#` 注释行；
- 断言存活：骨架 Then 步骤的引号字面值，在润色稿中仍以引号字面值形式存在，方向为 润色 ⊆ 骨架（D21）；
- `fallback_reason` 字面值列表最多 5 个（`MAX_REPORTED_LITERALS=5`）。
（出处：distiller spec §5.1/§5.2/§5.3；factcheck.py 全文）

**公开 API（api.py）**：`distill_events(events, polisher=None, test_data_values=None) -> DistillResult`（内部 `asyncio.run` 驱动异步润色，对外同步）；`distill_file(events_path, output_path=None, polisher=None, test_data_values=None)`（默认输出同目录 `<stem>.feature`；输入 JSON 解析失败抛 DistillError——文件级错误不属于"输出永远合法"的范围，后者指生成逻辑对任意合法解析结果产出合法 feature）。`DistillResult` 字段：`feature_text`（最终输出）、`skeleton_text`（纯模板稿，永远存在，供测试/调试/回退）、`output_path`、`used_llm_polish`、`fallback_reason`、`warnings`。执行序：`parse_input` → `build_skeleton` → `validate_skeleton`（不过 → 抛 DistillError，D14/D26）→ polisher None 则直接返回骨架 → 否则 `_apply_polisher` 四道闸（D25）。（出处：distiller spec §3/§2.5；api.py 全文）

**润色层（polisher.py）**：`Polisher` Protocol 单异步方法；`DefaultPolisher(get_model=...)` 经 `get_litellm_chat_model(role)` 取弱模型档（role key `distiller_polish`），钉温度 0.0（有 temperature 属性则设置），总尝试 ≤2（首次+1 重试），单次 `asyncio.wait_for` 超时 60s；单次空输出时 `last_error=None` 并继续重试，重试耗尽后无异常则返回空串（交给 api 层判 `polish_empty`）；任何异常 raise 给 api 层降级。prompt：SYSTEM_PROMPT 固化白名单 + 硬约束（含"每行顶格"双保险）；user 消息 = 骨架全文 + 紧凑事件清单（每行 `seq|type|target.name|value`，value 截 `DIGEST_VALUE_MAX_LENGTH=120`，缺省用 `-`，seq 缺省用数组序号+1）。`clean_polished_output` 复用 `testzeus_hercules.utils.gherkin_generator.clean_gherkin_output`（剥围栏/前导，不做缩进归一化）。所有测试走注入 FakePolisher；DefaultPolisher 无专属在线测试（离线约束），仅离线双打（fake chat model）下有 3 条用例。（出处：distiller spec §4；polisher.py 全文；test-report §2/§1）

**测试布局**：conftest 与 fixtures 下沉 `tests/record2gherkin/distiller/`（对 spec §0 的机械适配，录制器并行开发隔离）；生成物一律写 pytest `tmp_path`，fixtures 目录无生成物；fixtures 4 份手写 schema v1 样例：`login_search.json` / `form_submit.json` / `sparse_fields.json` / `empty_session.json`。conftest 的 `pytest_configure` 里 `os.environ.setdefault("ENABLE_TELEMETRY", "0")` 消除上游 config.py import 时初始化 Sentry 并在进程退出尝试外发事件的网络尝试（见 K12）。（出处：distiller test-report §1/§2）

### 3.3 跨模块契约与 CLI（P0-2 落地）

**Schema v1 契约（PLAN §3.1）**：session 2 键 + 事件 8 键 + target 7 键；掩码约定 `<masked>` + Gherkin 引用 test_data；assert_texts 为断言 grounding；ordinal 应对"两个 Delete 按钮"。两份模块 spec 均声明 PLAN §3.1 为唯一权威、不增改字段；phase1 专项核查确认"字段集（8 事件键/7 target 键）、枚举、掩码哨兵、空 target（回退到 element/field 且豁免回查）、缺 dom_snapshot（events.py 宽松处理）、seq 缺失排序、值截断（500/200/120/8 条）"逐一比对无其它"录得出、蒸不对"形态（除 P0-1/P1-1/P1-2）。`session.started_at` 与 `target.testid` 全链路不消费（透传；P2-6 记录 testid 是潜在的低成本可靠性增强点）。（出处：PLAN §3.1；recorder/distiller review 交叉核对表；phase1 专项核查 1/P2-6）

**CLI `generate` 子命令（P0-2）**：见 D28。两遍蒸馏的动机：占位符键集 `password_<seq>` 只有骨架渲染后才知道（掩码事件的 seq 决定键名），第一遍无润色骨架是最廉价的键集来源；第二遍只喂 `needed ∩ provided` 防止 unused 键泄漏进 feature。`_read_test_data` 与蒸馏器 `_clean_test_data_values` 同规则（str/number 且 strip 非空才算提供；bool/null/空白视为未提供），保证 CLI 的 filled 计数与产物永不矛盾；顶层非 object 或含 object/array 值 → `_UsageError` 硬错。missing 时中文提示："以上键无值，feature 中保留 {{TEST_DATA:...}} 占位符（...），run 前需人工填充后重跑 generate"。收尾 log 一行：`output=/ needed=N filled=N used_llm_polish=... fallback_reason=...`。（出处：cli.py:303-404；STATUS.md D2" P0-2 占位符注入落 CLI generate"）

**录制节奏 ≥450ms 约束（评测侧机器回放）**：`record2gherkin/evaluation/record_flows.py:44` 定义 `STEP_WAIT_MS = 450`。由来：recorder 的新增文本快照有 300ms 延迟且归属"触发时刻最新事件"（D10），机器回放必须在动作间停顿——否则多个动作被压缩进最后一个事件，超出其 8 条 assert_texts 上限（评测实测 F3 的"下单成功"会被挤掉）；真人录制天然满足该节奏。这是 D4+D10 两个录制器决策在下游的量化推论：机器动作间隔 450ms = 300ms 快照窗口 + 落地余量。（出处：evaluation test-report 已知边界 6（:113）；record_flows.py:44/66）

## 4. 踩坑全记录

### K1 快照归因缺陷（本模块最核心的坑，面试必讲）
- **现象**：「录制器→蒸馏器」集成冒烟（`dev_runs/integration_smoke.py`，真录→蒸馏→上游 `split_feature_file` 可解析恰 1 场景）发现：蒸馏产物中 `Then` 断言出现在**状态变化之前**——例：fill 邮箱后点"显示结果"，结果文案"订单已提交成功"被写进 **input 事件**的 assert_texts，生成 `When I enter ...` → `Then I should see "订单已提交成功"` → 后面才是 click，重放时 Then 先于揭示文本的 click 执行，必然假失败。
- **怎么定位**：集成冒烟端到端跑真实流程暴露；回看 recorder.js 时序——`scheduleSnapshot` 按"调度该快照的事件"归因，而 input 在下一次交互的 focusout 时才定稿入队（seq 先于引发 DOM 变化的 click），其 300ms 延迟快照在 click 揭示文本之后才拍摄，差集文本被归给先入队的 input。
- **根因**：两个机制叠加：①focusout 先入队（input 定稿制副作用）+ ②延迟快照按调度事件归因。"文本实际由后续事件造成却记在先前事件头上"。
- **修复**（提交 0ad2075，2026-09-21 01:33）：`scheduleSnapshot` 改为把差集文本归给**触发时刻最新已入队事件**（`events[events.length-1]`，追加写入、8 条上限）；归给更晚的事件总是安全的（文本在该事件之后持续可见）。spec §3 同步修订；新增回归用例 `test_assert_texts_attributed_to_latest_event`（fill → click → 断言文本在 click 的 assert_texts 里、不在 input 的）。修复后全量 85 passed（recorder 21 + distiller 64），集成冒烟 9/9 通过。
- **20 条单测为何没拦住**：原用例只测了"点击后**包含**"（用例 14：点显示结果后 assert_texts 含结果文本），未测"点击前**排除**"；且用例均以"单次点击 + 等待"方式编写，操作间隔 > 300ms，压缩不了时序。test-report §6 已知问题 12 其实已预感该机制（"快照文本可能被归到前一个事件上……是 spec 机制的固有属性，不是实现缺陷"）——但集成冒烟证明它**会**造成重放假失败，性质从"固有属性"升级为"必须修的缺陷"。
- **教训/遗留**：端到端冒烟与单测互补——单测验证各自 spec，跨模块时序缺陷只有真集成才暴露；"机制固有属性"不等于"无害"，要以下游可观察后果（重放假失败）定性。下游量化推论：评测机回放必须 `STEP_WAIT_MS=450` 停顿，否则多动作压进最后一个事件、超出 8 条上限（evaluation test-report 已知边界 6）。
- **出处**：recorder test-report 补记 + §6.12；recorder spec §3 修订；recorder.js:278-304；0ad2075；phase1 专项核查 2；evaluation test-report:113。

### K2 P0-1：click+submit 双事件契约悬空，真实站点重放必坏
- **现象**：点 submit 按钮恒产出 `click(seq N)` + `submit(seq N+1)` 两条事件，蒸馏为 `When I click on the "Go" button` + `When I submit the "User Go" form` 两步（探针实测）。重放时 click 已触发提交/跳转，submit 步骤落在新页面上（表单已不存在）→ agent 必然失败或二次提交。几乎所有真实流程（登录/搜索/下单）都含此形态。
- **怎么定位**：phase1-merge-analysis 定位三处：recorder.js handleSubmit 无条件记录 + distiller templates 无合并规则 + polisher SYSTEM_PROMPT 白名单只允许合并 input；另用临时页面（svg 图标按钮 + submit 表单）定向探针实证。
- **根因**：**契约悬空**——recorder spec §4.2 写"两条事件由蒸馏器负责合并"，distiller 润色白名单却只允许合并 input；两份 spec 互相指派、无人落地，是唯一跨模块契约断点。
- **修复**（提交 fe11083，2026-09-21 01:59）：蒸馏器 templates 层新增确定性判定 `_dedupes_preceding_click`：submit 且紧邻前驱（按 seq 排序、过滤未知类型后的流）为 click → 丢弃本 submit **步骤**、保留 click 步骤（带按钮 name hint 定位更稳）、保留该 submit 的 Then（位置在 click 及其 Then 之后）；前驱非 click 的 submit（Enter 键直接提交）不去重。守边界：不改 api/factcheck/polisher、公开 API 不变、不新增 warning。spec 同步：distiller spec §2.2 新增行；recorder spec §4.2/§4.5 注明。
- **测试**：新增 3 条（`test_click_then_submit_dedupes_submit_step` / `test_deduped_submit_keeps_its_assertions_after_click` / `test_standalone_submit_keeps_its_step`），distiller 64→67、全量 85→88；既有断言 `test_end_to_end_form_flow` 的 `'When I submit the "form" form' in feature` 与新规则直接矛盾，改判 `assert "When I submit" not in feature`（该用例因此同时成为 P0-1 端到端回归）。
- **教训/遗留**：跨模块行为必须在**一处** spec 里写死owner，"你负责"式互相指派在并行开发时必然漏落地；规则层去重是零模型依赖红线内的正确落点。
- **出处**：phase1 P0-1；distiller test-report 补记；distiller spec §2.2；fe11083。

### K3 P0-2：`{{TEST_DATA:password_<seq>}}` 无机械消费者
- **现象**：带密码流程（旗舰场景）重放时，agent 只看到字面量 `{{TEST_DATA:password_6}}`，执行成败取决于 LLM 猜测。
- **怎么定位**：phase1 全仓 grep——无占位符替换逻辑；`gherkin_helper.py` 只做拆分/单行化；`static_data_loader.py:70-125` 只把 TEST_DATA_PATH 目录文件内容整块注入 LTM prompt；`password_<seq>` 是合成键，与 test_data 目录文件名无确定性映射。
- **根因**：蒸馏器只负责"产出占位符 + 生成期可选值替换"（distiller spec §8 明确引擎侧 test_data 自动注入 Out of Scope），但编排/评测侧没人接。
- **修复**：落 CLI `generate` 两遍蒸馏 + `--test-data`（D28/K3 前节详述）；STATUS.md D2 记"P0-2 占位符注入落 CLI generate"。测试锚点：`test_masked_input_filled_from_test_data` 早在初版就验证了 API 参数路径。
- **教训/遗留**：合成键（`password_<seq>`）在无消费者时是纯仪式；新增占位符机制必须同步指定"谁在什么时候填"。
- **出处**：phase1 P0-2；cli.py:303-404；STATUS.md:30；distiller spec §2.3/§8。

### K4 兜底词未 grounding：缺 target 输入必触发 DistillError（distiller review 必改 1）
- **现象**（spec 层，实现前被 review 拦下）：事件 target 缺失/全空时，hint 兜底链末端写死的 `"element"/"field"/"dropdown"/"form"` 是引号字面值——不在事实集也不在豁免清单 → 骨架自检失败 → **抛 DistillError**，直接违反核心目标推论 2（任何输入含缺失字段必须产出合法 feature 绝不抛异常）。
- **怎么定位**：distiller review 契约核对——§2.2 各 fallback 链末端 vs §5.2 豁免清单只有占位符与空串 vs §1.2 事实集只收录制事实。
- **根因**：兜底词是**模板常量不是录制事实**，却被当字面值参与回查；§2.5 自检本意是"校验器 bug 才触发"，该规则使它在合法输入上必然触发，自检不再是纯 bug 探测器。真实可达路径：schema 漂移、手写 fixtures（`"target": null` 的 click 很自然）、submit 事件 target=null（录制器 submit name 通常空串，仅因 tag="form" 恰好入事实集才在正常录制下侥幸通过）。
- **修复**：§5.2 豁免清单增加第三条款（D18）；review 明确不推荐把四词加进事实集（污染 ground truth）。改后不变量成立：模板全部引号字面值 = 事实集成员或豁免项，"任意输入恒过"对任意输入成立。落地后另有专属用例 `test_template_fallback_words_are_exempt`。
- **教训/遗留**：新增模板常量时必须同步过一遍回查不变量（"字面值 ⊆ 事实集 ∪ 豁免集"）；豁免清单是"任意输入恒过"不变量的组成部分，不是事后补丁。
- **出处**：distiller review 必改 1；distiller spec §5.2；factcheck.py:26-27/79-86。

### K5 缩进误杀：LLM 润色稿被系统性判非法（distiller review 必改 2）
- **现象**（spec 层，实现前被 review 拦下）：§5.1 结构校验按 `^` 行首锚定、未规定先 strip 行首空白。LLM 润色稿普遍给 Scenario/步骤行加缩进 → Scenario 计数为 0 或关键词检查失败 → 判 `syntax_invalid` → 整体回退。
- **怎么定位**：review 核读 `clean_gherkin_output`（utils/gherkin_generator.py:11-22）——只剥围栏和 Feature: 前导，**不做内部缩进归一化**，救不了场。
- **根因**：校验端锚定方式与被校验对象的实际形态（LLM 缩进产出）不匹配。后果最阴险：离线测试全绿（FakePolisher 手写、天然顶格），生产 DefaultPolisher 却几乎每次被回退——润色层形同虚设，且不会被 35 条用例暴露。
- **修复**：§5.1 加"所有行先 strip 行首（及行尾）空白后再做锚定与关键词检查（Feature:/Scenario: 计数同理）"；双保险：SYSTEM_PROMPT 硬约束加"每行顶格（column 0）"。落地后有专属用例 `test_indented_text_is_structurally_valid`。
- **教训/遗留**：校验器的措辞性误杀与"润色层永远可用"的架构保证直接矛盾——硬闸要先证明"对合法形态不误杀"，再谈"对非法形态拦得住"。
- **出处**：distiller review 必改 2；distiller spec §5.1；polisher.py:33；factcheck.py:106-128。

### K6 标题的 `://` 当目录分隔符（P1-4，pilot002 复盘）
- **现象**：真实 pilot（pilot002）中，`Feature: Recorded flow on http://...` 的标题被上游 `split_feature_file` 用作 JUnit 文件名（`gherkin_helper.py:70-71` 只 `scenario_title.replace(' ', '_') + ".feature"`，无文件名安全化），`://` 产生的 `//` 被当作目录分隔符 → 写文件 ENOENT/路径穿越，阻塞"真录→蒸馏→Hercules 子进程"拆分环节。
- **怎么定位**：阶段 1 分析定为 P1-4；位置链 templates.py 标题源头（首事件 page_title / origin）→ gherkin_helper 文件名。
- **根因**：蒸馏器标题直通下游文件名，两端都没做安全化。
- **修复**（提交 7ce6c95，2026-09-21 03:05）：蒸馏器源头清洗（D22）：`[^\w\- ]+` → `_`、连续 `_` 折叠、strip `"_ "`、空则回退默认标题；保留 Feature 标题行语义（清洗只作用于标题行文本）。同步改 2 条既有断言（`test_end_to_end_sample_flow`、`test_hercules_helper_integration` 期望 `Recorded flow on https_staging_example_com`）+ 新增回归 `test_title_sanitized_to_filename_safe`（distiller 67→68 的 +1）。
- **教训/遗留**：任何会被下游当文件名/路径的字符串，生产点就应文件名安全化，不赌下游行为；page_title 含 `/` 在真实站点常见。
- **出处**：distiller spec §2.1；phase1 P1-4；7ce6c95 diff；templates.py:52-59。

### K7 初始 navigate 的 dom_snapshot 三处定义互相矛盾（recorder review 必改 1）
- **现象**（spec 层，实现前被 review 拦下）：§2 示例 JSON 写 `"dom_snapshot": null`、§4.6.1 写"snapshot 为 null"、§2 字段表写"恒存在，采集失败 `{"assert_texts": []}`"、§3 写"首个 navigate 的 assert_texts 为空数组"——null 派与对象派是两代设计残留未收敛。
- **根因**：spec 迭代时旧设计（无基线）没删干净；直接违背 spec 头部"实现者读完不再需要做设计决策"，且用例 1/17 的断言值形态取决于这个未决决策。
- **修复**：统一为"字段恒存在、恒为对象；初始 navigate = `{"assert_texts": []}`"；同步改 §2 示例与 §4.6.1 两处；下游蒸馏器宽松解析两种都能消化（不破契约），但录制器 spec 必须唯一确定。
- **教训/遗留**：同一 spec 内同一字段的定义要全局 grep 一遍再定稿；"宽松消费方"不豁免上游契约的唯一性。
- **出处**：recorder review 必改 1；recorder spec §2/§3/§4.6.1。

### K8 spec 用例 20 示例与 §5.2 规则不自洽（`order placed` 非连续子串）
- **现象**：distiller spec §6 用例 20 示例称字面值 `order placed` ⊆ 事实 `Your order has been placed successfully` 应通过，但按 §5.2 规范算法（连续子串）它**不是**连续子串，应判不 grounded。
- **怎么定位**：实现期对照 §5.2 精确语义发现。
- **根因**：spec 示例按"词序子集"直觉书写，与规范算法（严格连续子串）不一致。
- **修复/取舍**：按 §5.2 规范实现（严格子串），用例内改判为两种情形都断言——连续片段（`order has been placed`）通过、非连续词组（`order placed`）不通过；若评测期发现真实润色稿因此被误杀，再考虑放宽为词序子集匹配（会削弱硬闸，需评审）。
- **教训/遗留**：spec 里的示例本身就是规范——示例与规则文字冲突时实现者必须二选一，写 spec 时应对每个示例跑一遍规则。
- **出处**：distiller test-report 已知问题 1；distiller spec §5.2/§6 用例 20；phase1 P2-3。

### K9 点空白判定：spec §4.2 字面与 §4.7 去噪总表矛盾
- **现象/根因**：§4.2 字面是"target 为 document/html/body **且 name 为空**才丢弃"，但 body/html 的 innerText 是整页文本，字面实现会把整页文本写成 name，与 §4.7"点空白 → 0 条"矛盾。
- **修复/取舍**：实现取严——documentElement/body 一律丢弃（recorder.js:351-355 注释明示理由），记为已知问题 1；phase1 P2-3 列为"有意为之的 spec 文字漂移（双方记录在案，无动作）"。
- **教训/遗留**：当 spec 两条规则冲突时，取"最简且确定"的一侧并在代码注释与 test-report 双处留痕。
- **出处**：recorder test-report §6.1；recorder.js:351-355；phase1 P2-3。

### K10 submit 的 target.name 取到 form innerText 首行
- **现象**：探针实测 submit 事件 name = `"User Go"`（form innerText 首行）而非 spec §4.5 预期的"通常取不到为空串"——`<form>` 有 innerText（整块表单文本），按 §2.1.2 会取到截 200 字符的文本块。
- **根因**：recorder spec §4.5 预测不准（recorder review 建议 3b 已指出）。
- **处置**：不修（P2-1，"无意义 hint，agent 可容忍"；可在 recorder 对 tag=form 跳过 innerText，记录即可）；但它正是 K4 中"submit 仅因 tag=form 恰好入事实集才侥幸通过"的另一半。
- **教训/遗留**：对"某属性通常取不到"的预测要拿真实 DOM 验证。
- **出处**：recorder review 建议 3b；phase1 P2-1；recorder spec §4.5。

### K11 测试环境前置：playwright 浏览器二进制缺失
- **现象**：recorder review 实测当前 `.venv` 的 playwright（pin ≤1.49）要求 `chromium_headless_shell-1148`，本机缓存只有 `chromium-1243`（其他 playwright 版本装入），首次跑报 `Executable doesn't exist`。
- **处置**：D1 开工前 `uv run playwright install chromium`（一次性，需网络）；plan"uv 环境已有 playwright"仅对 Python 包成立。同时 D1 冒烟先验证 file:// 下 `history.pushState`（Chromium 对 file:// 同文档 pushState 既有行为可用，且 spec 同时监听 hashchange 兜底）。
- **教训/遗留**：pin 住 playwright 版本意味着浏览器二进制也要对号入座。
- **出处**：recorder review 建议 1；recorder plan §2/§5。

### K12 上游 Sentry 遥测污染离线测试
- **现象**：pytest 结束输出出现 `Sentry is attempting to send 2 pending events`——上游 `testzeus_hercules/config.py` 在 import 时初始化 Sentry 客户端并在进程退出时尝试外发事件，违背"全程离线"验收。
- **修复**：`tests/record2gherkin/distiller/conftest.py` 的 `pytest_configure` 里 `os.environ.setdefault("ENABLE_TELEMETRY", "0")`（用 setdefault，显式设 1 时仍以环境为准）；外发尝试消失。
- **教训/遗留**：import 副作用（遥测/配置单例）要在测试入口最早处关断。
- **出处**：distiller test-report §2。

### K13 验收口径小偏差："首 10 字符" vs 实际 11 字符前缀；MVP 验收的等价自动化
- **现象**：recorder spec §8.2 写"首 10 字符为 `javascript:`"，实际 `javascript:` 是 11 字符；spec §8.3 要求手动用 bookmarklet 走通 MVP 流程，无头环境无法点击真实书签栏。
- **处置**：用例 20 断言 `startswith("javascript:")` 且长度校验用完整前缀（test-report 明注差异）；§8.3 以等价自动化验证——同一份产物 `unquote` 后 `page.evaluate`（与点击书签完全同一条代码路径）+ 同一 `validate_event_stream` 校验，人工书签栏点击建议演示前做一次。
- **教训/遗留**：验收标准的字面细节（字符数、手动步骤）要可机械执行，或显式声明等价路径。
- **出处**：recorder test-report §4.2/§4.3。

### K14 P1-1：无语义 click（图标按钮）产出垃圾步骤——实现-spec 漂移影响最大的一条（范围外相邻，简要附注）
- recorder spec §4.2 本写"目标向上取最近的有语义祖先"，实现未落地：`aria-label="Close dialog"` 的纯图标按钮点击，event.target 落在 svg 上，录制为 `tag=svg name=''`，蒸馏产物 `When I click on the "svg" generic`——aria-label 全丢且回查不拦（"svg" 在事实集内）。phase1 列为建议下一阶段项（recorder 加具名祖先探测或 distiller 记 warning），未在本模块修复。**出处：phase1 P1-1。**
### K15 P1-2：checkbox/radio 双事件执行歧义（范围外相邻，简要附注）
- 物理点击 checkbox 产 click + change→input("true") 两条，蒸馏出 `When I click on the "★ 订阅邮件" checkbox` + `When I enter "true" in ... field` 并存——重放时 click 已切换状态，fill 对 checkbox 执行语义未定义（Playwright fill 会报错）；勾又取消会膨胀成 4 步。建议蒸馏规则丢弃/改 check/uncheck 措辞，未在本模块修复。**出处：phase1 P1-2。**
### K16 P1-3：录制 name 与引擎可访问名"同源"声明不成立（范围外相邻，简要附注）
- 三档口径并录：①recorder plan/spec 称属性集合与 `getAccessibleName`（get_detailed_accessibility_tree.py:1050-1087）同源；②recorder review 备注 a 实测引擎手动链为 aria-label → aria-labelledby → **alt → title → placeholder** → input value → innerText，与契约顺序不同且多 alt/value 两级（契约是唯一权威，不算录制器漂移）；③phase1 P1-3 进一步指出引擎手动链**不含任何 label 关联**（checkbox 落到 element.value="on"；`getComputedAccessibleNode` 返回 Promise 未 await 实际恒走手工回退）——`When I click on the "邮箱地址" field` 在引擎树上可能对不上同名节点，全靠 LLM 模糊匹配。评测期建议加"name 命中率"统计，不预先改。**出处：recorder review 备注 a；phase1 P1-3。**

### 其余已记录的刻意取舍（P2 档，一览）
- P2-3 五项"有意为之的 spec 文字漂移"：点空白取严（K9）；navigate 去重不限"同 tick"（实现按 `lastNavigateUrl === location.href` 判定）；掩码判定 `strip()==` 超集（宁多掩不漏）；骨架自检移 api 层（D26）；用例 20 示例不自洽（K8）。
- P2-2：`"<file>"` 哨兵按普通值进步骤（`When I enter "<file>" ...`），不可执行，评测数据出现该步骤判 N/A；distiller review 建议 1 曾提"应写死处置"，实现按普通值透传（蒸馏器 review 判定输出合法不阻塞）。
- P2-4：navigate 缺 url 被跳过时其 assert_texts 一并丢弃（实现比 spec 字面更严）；recorder 恒有 location.href 实际不可触发，仅手造数据命中。
- P2-5：`_apply_polisher` 捕获一切异常，包括已运行事件循环内调 `asyncio.run` 的 RuntimeError——被静默降级为 polish_error 而非暴露调用方式错误；评测走同步子进程不受影响。
- P2-6：target.testid 全链路采集但零消费（重放最稳定位线索，评测期低成本增强点，需动 spec，先记录）。
（出处：phase1 P2-1..P2-6；distiller review 建议 1）

## 5. 面试深挖要点

| 预期追问 | 一句话答 | 详见 |
|---|---|---|
| 为什么用注入式 JS，不用浏览器扩展或 Playwright 录制？ | 扩展需要 manifest/加载/商店审核，注入式零安装、复制一条书签即可演示，且跨文档导航等形态边界如实写进已知限制；Playwright tracing 备选源文档未记录（未核实），可说的是评测框架用 Playwright 只是**回放动作驱动录制器**、codegen 只作存活率对照基线 | §2 D1/D1a；§3.1 navigate 与 MPA 边界 |
| bookmarklet 怎么打包的？为什么不做 minify？ | python 纯标准库读源码 → IIFE 包裹（源码已是 IIFE 则原样）→ `urllib.parse.quote(safe="")` 全量 URL 编码 → 拼 `javascript:` 前缀，产物与源码 `unquote` 后逐字节一致；minify 有改坏语义风险不值当，Chrome bookmarklet 上限 >100KB 无需压缩，且本机无 Node 引入工具链违背小而准 | §2 D7 |
| 密码是怎么处理的？在哪一层掩码？ | 录制层在**采集时**就写死字面量 `<masked>`（绝不记录真实值，name/label 照常采集），蒸馏层精确识别后换成 `{{TEST_DATA:password_<seq>}}` 占位符、`test_data_values` 可填真值且真值入事实集、CLI `--test-data` 填充并对 missing 键告警保留占位符；实现用 `strip()` 超集判定是宁多掩不漏 | §2 D17/D28；§3.2/§3.3 |
| assert_texts 为什么是差集？300ms 怎么定的？ | 延迟 300ms 让点击引发的 DOM 更新先落地；与上次快照做差集保证记录的是"这次操作**新出现**的文本"——加载即存在的文本（start 基线）不构成断言依据；上限 8 条/条 120 字符兜体积，轮播图/计时器噪声 MVP 接受、由蒸馏器事实回查兜底 | §2 D4；§3.1 dom_snapshot |
| 讲一个你最有代表性的 bug | 快照归因缺陷：input 在下一次交互的 focusout 才定稿入队（先于引发 DOM 变化的 click），300ms 快照在 click 揭示文本之后拍摄，按调度事件归因就把新文本记给了 input——蒸馏出 Then 早于状态变化、重放必假失败；20 条单测只测"点击后包含"没测"点击前排除"所以没拦住，集成冒烟暴露后改为归给触发时刻最新已入队事件，补排除性回归用例 | §4 K1 |
| 点提交按钮为什么录两条事件？怎么不变成两步？ | 忠实记录（click+submit 都是真实 DOM 事件），去重放蒸馏器规则层：submit 紧邻前驱是 click 就丢弃 submit 步骤保 click 步骤（带按钮 name hint 定位更稳）、submit 的 Then 保留且位置在后；Enter 键直接提交（前驱非 click）不去重照常生成——这是规则层唯一例外合并，零模型依赖可机械测试 | §2 D15；§4 K2 |
| 为什么骨架自检失败要抛错而不是回退？ | 模板由事实集生成、按构造应恒过，自检失败只可能是校验器/模板的 bug，静默兜底会掩盖 bug——这是全链路唯一主动抛错路径（DistillError），且测试必须覆盖"任意输入恒过"不变量；为了让 templates 保持纯函数，检查动作放 api 层避免循环依赖 | §2 D14/D26 |
| 润色层怎么保证不出错？ | 保证不在 prompt 而在回查层：四道闸（输出清洗非空 → 正则结构校验 → 事实回查 → 断言存活）任一失败**整体**回退骨架并记 fallback_reason（polish_error/syntax_invalid/fact_check_failed/assertion_dropped/polish_empty 五条路径），不做部分采用——骨架永远在手上 | §2 D25/D16 |
| 事实回查的匹配规则精确是什么？为什么允许子串、方向朝哪边？ | 字面值 L grounded ⇔ `norm(L)==norm(F)` 或 `norm(L) ⊆ norm(F)`（norm=空白归一化，大小写敏感）；允许 L⊆F 是容忍润色截取（长断言取短语）；断言存活检查方向固定"润色 ⊆ 骨架"（只许缩短不许加长，双向会放行加长断言削弱硬闸）；豁免三条款：占位符、空串、模板兜底词 element/field/dropdown/form | §2 D20/D21/D18 |
| 为什么 element/field 这些词要豁免？不豁免会怎样？ | 它们是模板 fallback 链末端的写死常量、不是录制事实；不豁免的话 target 缺失的合法输入必然自检失败抛 DistillError——违反"任何输入恒产合法 feature"的核心推论，自检也从 bug 探测器退化成合法输入杀手 | §4 K4 |
| 润色稿被误杀的风险有哪些，怎么防的？ | 两个系统性误杀点都被 review 拦下：缩进（LLM 普遍缩进而 clean_gherkin_output 不归一化 → 校验端每行先 strip + prompt 要求顶格双保险）；严格子串对非连续词组的误杀（保留严格子串，评测期有误杀再评审放宽） | §4 K5/K8 |
| `{{TEST_DATA}}` 占位符谁来填？ | 生成期 CLI `generate` 两遍蒸馏：先无润色骨架提取所需键集，`--test-data` JSON 提供真值（blank/bool/null 视为未提供、与蒸馏器同规则），只喂 needed∩provided 防泄漏；missing 键 `missing_test_data:<key>` 警告并保留占位符提示人工填充重跑——Hercules 引擎侧没有机械消费者，这是 phase1 P0-2 定的编排决策 | §2 D28；§4 K3 |
| 录制器对 UI 变异的自愈怎么保证？产物凭什么能被引擎执行？ | 定位 hint 直接写录制时的 name/label（与引擎感知属性"同源"的契约口径），`(occurrence n)` 消歧同名元素；不持久化任何 selector，每次执行引擎重新感知 DOM；"同源"的精度争议（引擎无 label 关联）如实承认，评测期用 name 命中率数据决定补哪边 | §2 D6；§4 K16 |
| 为什么整机回放录制时动作间要停 450ms？ | 快照 300ms 延迟 + 归给最新事件 + 单事件 8 条 assert_texts 上限，三个参数合起来意味着机器连续动作会把多条 DOM 变化压进最后一个事件、挤掉前面的断言（实测 F3"下单成功"被挤掉）；真人手速天然满足，机器回放 `STEP_WAIT_MS=450` | §3.3 录制节奏 |
| 测试怎么做到全离线又高覆盖？ | recorder：pytest+playwright 以 evaluate 注入同一份 JS 源（与生产同构），21 例全绿；distiller：FakePolisher 注入 + DefaultPolisher 仅离线双打（fake chat model）+ `ENABLE_TELEMETRY=0` 关 Sentry 外发，骨架三模块负向 grep 证明零模型依赖，68 例 2-3 秒跑完 | §3.1/§3.2；§4 K12 |
| 测试数量怎么长起来的？ | recorder 20 条 spec 用例 + 1 条归因回归 = 21；distiller 35 条 spec 用例落地为 64（参数化），P0-1 修复 +3 = 67，P1-4 标题清洗 +1 = 68（+1 来自提交 7ce6c95 的 `test_title_sanitized_to_filename_safe`）；合并态 85 → P0-1 后 88 → 全局五模块 492 | §6 数字对照表 |
| 哪些是你明确不做的？ | recorder：跨域 iframe、shadow DOM（含 open——open 的合成事件 target 被重定向语义失真）、canvas、多 tab、扩展形态、回放编辑 UI、性能优化、断言自动生成；distiller：多 Scenario、中文模板、复杂断言推理、数据抽取、引擎侧 test_data 自动注入、模型路由调优——边界全部写进两份 spec 的 Out of Scope | §2 D1/D12；recorder spec §9；distiller spec §8 |

## 6. 数字与出处对照表

| 数字/事实 | 值 | 出处（文件:位置） |
|---|---|---|
| 快照延迟 | `SNAPSHOT_DELAY_MS = 300` | recorder spec §3；recorder.js:13 |
| assert_texts 上限 | 最多 8 条、每条 ≤120 字符 | recorder spec §3；recorder.js:16-17（`ASSERT_TEXT_MAX_LENGTH=120` / `ASSERT_TEXT_MAX_COUNT=8`） |
| name 上限 | ≤200 字符（首行/trim/压缩空白） | recorder spec §2.1.2；recorder.js:14 |
| input value 上限 | ≤500 字符（去首尾换行） | recorder spec §2.2；recorder.js:15 |
| select value 上限 | ≤200 字符（与 name 同一套清洗） | recorder spec §2.2；recorder.js:400-404 注释 |
| 事件字段 | 8 事件键（seq/ts/type/url/page_title/target/value/dom_snapshot）+ 7 target 键 | recorder spec §2/§2.1；phase1 专项核查 1 |
| 事件枚举 | navigate/click/input/select/submit 共 5 值，不新增 | PLAN §3.1；recorder spec §2；events.py:14 |
| recorder spec 用例数 | 20 条（§7 清单），实际交付 20/20 通过 → 快照修复后 21 例（+1 回归） | recorder spec §7/§8；test-report §2/补记；test_snapshot.py:44 |
| recorder 测试耗时 | 20 passed in 9.97s / 9.82s / 9.88s（连续 3 次） | recorder test-report §2 |
| recorder.js 体量 | test-report 时点 561 行 / 16458 字符；现工作区（0ad2075 后）575 行 / 18218 字符 | recorder test-report §1；实测 `wc` |
| bookmarklet.txt 体量 | test-report 时点 32293 字符；现 34117 字符；前缀 `javascript:` 实为 11 字符（spec 误写"首 10 字符"） | recorder test-report §1/§4.2；实测 `wc -c` |
| 蒸馏三层 | 规则模板（零 LLM）→ 可选润色 → 事实回查 | distiller plan §2 |
| distiller spec 用例数 | 35 条（§6：A 17 / B 6 / C 7 / D 5），落地 64 条（22+25+11+6） | distiller spec §6；test-report §2 |
| distiller 用例增长 | 64 →（P0-1 +3，fe11083）67 →（P1-4 +1，7ce6c95 的 `test_title_sanitized_to_filename_safe`）68 | distiller test-report 补记；git show 7ce6c95；STATUS.md:20 |
| 合并态全量 | 85 passed = recorder 21 + distiller 64（main 4f0df82）；P0-1 后 88 passed（21+67）；集成冒烟 9/9 | phase1-merge-analysis.md:4；distiller test-report 补记；recorder test-report 补记 |
| 全局基线 | 492 passed = recorder 21 + distiller 68 + evaluation 77 + attributor 81 + cli 54 + benchmark 191（2026-09-24 --collect-only 逐模块实测） | STATUS.md:20 |
| 当前收集数 | recorder 21 + distiller 68 = 89 collected（实测 `pytest --collect-only`） | 本档案核实（2026-10-10） |
| 标题截断 | `TITLE_MAX_LENGTH = 120`；文件名安全正则 `[^\w\- ]+` → `_` 折叠 | distiller spec §2.1；templates.py:22/52 |
| fallback_reason 字面值上限 | 最多列 5 个（`MAX_REPORTED_LITERALS = 5`） | distiller spec §5.3；factcheck.py:30 |
| DefaultPolisher 参数 | role key `distiller_polish`；温度 0.0；总尝试 ≤2；单次超时 60s；digest value 截 120 | distiller spec §4.1；polisher.py:14-19 |
| 占位符格式 | `{{TEST_DATA:password_{seq}}}`（`password_<seq>`；无 seq 用排序后数组序号） | distiller spec §2.3；templates.py:34 |
| 字面值正则 | `"((?:[^"\\]|\\.)*)"`；占位符 `^\{\{TEST_DATA:[^{}]*\}\}$` | distiller spec §5.2；factcheck.py:21/24 |
| 豁免常量 | `element` / `field` / `dropdown` / `form` 四词 | distiller spec §5.2；factcheck.py:27 |
| CLI P0-2 | 两遍蒸馏；missing → `missing_test_data:<key>` 保留占位符；unused → `unused_test_data:<key>` | cli.py:303-404；STATUS.md:30 |
| 录制节奏约束 | `STEP_WAIT_MS = 450`（evaluation record_flows.py:44；300ms 快照 + 归因 + 8 条上限的推论） | evaluation test-report:113 |
| 关键提交 | 4283ebe（recorder 初版 20 例）→ 0ad2075（快照归因修复，2026-09-21 01:33）→ 9664cd3（distiller 初版 64 例）→ 4f0df82（main 合并，85 passed）→ fe11083（P0-1，2026-09-21 01:59，88 passed）→ 7ce6c95（P1-4，2026-09-21 03:05，distiller 68） | git log/show |
| 日期矛盾注 | distiller test-report 补记标注 P0-1 为"2026-09-20 合并后修复"，git 提交 fe11083 实为 2026-09-21 01:59；且其"总数 85→88"要求 recorder 已是 21 例（快照修复在前），按 git 时间轴顺序为：快照修复 01:33 → P0-1 01:59 → P1-4 03:05。两说并注，以 git 为准 | distiller test-report 补记；git show |
| Chrome bookmarklet 上限 | 宽松（>100KB），无需压缩 | recorder plan §2 |
| 蒸馏器 assert 警戒线 | >20 条/事件的截断策略留评测数据决定（当前不去量只去重） | distiller spec §9；recorder review 交叉核对表（"蒸馏器 20 条警戒线 > 录制器 8 条上限"） |
| 引擎锚点 | getAccessibleName get_detailed_accessibility_tree.py:1050-1087（行号核实无误）；getRole :1098-1148；gherkin_helper.py:102-134（serialize 压缩）；gherkin_generator.py:11-22（clean_gherkin_output）；gherkin_helper.py:70-71（文件名 replace） | recorder review 备注 a；distiller review 交叉核对表；phase1 P1-3/P1-4 |
| fixtures | recorder 1 份（demo_form.html）；distiller 4 份（login_search/form_submit/sparse_fields/empty_session.json） | test-report §1（两份） |
| Sentry 关断 | conftest `ENABLE_TELEMETRY=0`（setdefault）消除 `Sentry is attempting to send 2 pending events` | distiller test-report §2 |
| black/isort | line-length 200；`12 files would be left unchanged`（distiller+tests） | distiller test-report §3 命令 C |
