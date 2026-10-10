# 02 — 评测框架与 exp001 变异实验（dossier）

> 生成：2026-10-10，dossier 子代理 B。范围：`record2gherkin/evaluation/` 全部代码 + `dev_docs/evaluation/` 四件套与 experiment-report + `dev_docs/reports/final-analysis.md` 60 格复核 + STATUS 阶段 3/4 + pilot001-003 原始产物（`dev_runs/experiments/pilot001..003/`）+ exp001 原始数据（`dev_runs/experiments/exp001/`）。
> 核验方式：60 格存活率、token 总量、逐格失败清单均由 `results.jsonl` 逐行**重新复算**（非转抄）；pilot 三轮逐行读 `results.jsonl` 与 `stdout.log` 原始日志；uBlock/代理旁路根因读到上游 `playwright_manager.py` 源码与修复提交 `7ce6c95`/`77a76ec` 的 diff。
> 出处约定：路径相对仓库根；一手事实以 `dev_docs/evaluation/` 与 `dev_runs/experiments/` 为准；与其他 dossier 数字不一致时以一手报告为准。

## 1. 这一页讲什么（3 句）

这一页回答「简历硬数字是怎么产出来的」：一个自建 MiniShop demo + 5 类参数化 UI 变异 + 60 格矩阵（6 流程 × 5 变异 × 2 方法），对比「录制→蒸馏→语义执行」与「selector 式 Playwright 基线」的变异存活率、首轮通过率与 token 成本——主指标 1.00 vs 0.667，M3 标识符随机化 6/6 vs 0/6，M4 文案改写 5/6。它覆盖全部设计决策（为什么这 5 类变异、为什么基线长这样、为什么 60 格、为什么这个预算护栏）、执行器双 provider 工程（DeepSeek/GLM、env 注入与 key 脱敏）、pilot001-003 三轮迭代与全部环境坑（uBlock/系统代理/NLTK/402/GLM 403）。它同时固化结论边界：单 demo、单模型、6 格样本不给概率点估计——这些限定词是口径的一部分，不是附注。

## 2. 设计决策总表（决策 | 为什么 | 备选与否决理由 | 出处）

| # | 决策 | 为什么 | 备选与否决理由 | 出处 |
|---|---|---|---|---|
| D1 | 实验只回答三个可量化问题：首轮通过率 / 变异存活率 / 成本 | 「产出一组可信的对比数据」是唯一核心目标；失败运行如实记为失败数据，不做「重跑到好看为止」 | 不设定性评价、不设「演示好看」类目标 | `dev_docs/evaluation/plan.md` §1 |
| D2 | demo 应用 = 自建 MiniShop：单 HTML 文档 SPA + hash 路由两视图（商品列表 `#/` / 下单表单 `#/order`），约 60 行标准库 HTTP 服务承载，无后端状态 | 无外网依赖、无构建链、变异完全可控；**单文档 SPA 是硬约束**——录制器跨文档导航即失效（recorder 已知问题 9），SPA+pushState 是唯一能让 6 条流程各在一次会话内录完的形态；hash 直达 `#/order` 必须可落地（蒸馏产物会出现 `navigate "...#/order"`） | 否决多页跳转站（录制断流）；否决前端框架（违背克制原则）；否决真实外网站点（变异不可控、不可重复） | plan §2；spec §1.1；`record2gherkin/evaluation/demo_app.py` |
| D3 | 变异引擎 = **参数化渲染**：变异是渲染参数 `(mutation, seed)`，变换作用在结构层（元素注册表），序列化成 HTML 前完成 | 同 seed 字节级可重复；纯函数可离线单测（不需要起浏览器）；「动态 id」语义天然成立（每次加载按 seed 生成） | 否决「DOM 操作脚本注入」（时序与执行环境耦合、难单测）；否决「预生成多版本 HTML 文件」（M3 的"每次都不同"要用 N 份文件模拟，源码漂移易失控）；experiment-report §1 也显式声明「对渲染参数施加，非事后改 DOM」 | plan §2；spec §2.1；`demo_app.py` 模块 docstring |
| D4 | 变异集固定 5 类：M0 对照 / M1 字段组顺序调换 / M2 布局与 DOM 嵌套微调 / M3 id+class+testid 按 seed 随机化（**主指标**）/ M4 按钮文案改写（副指标单列） | 与总计划 PLAN §3.4 一致，**不增不减**；M3 是与属性 selector 基线正交对撞的区分度主指标；M1/M2 是「结构性对照」；M4 是文献与阶段 1 源码审查均预测的「概率性」边界（PLAN §4.1：字段/布局/动态 id 是机制性自愈 ≈8/10，文案改写是纯赌 LLM 语义映射 ≈4/10 且失败静默）——**主指标只选机制性存活的变异** | 不加复合变异（M1+M3，Out of Scope）；不加更多变异类别（预算与周期约束） | PLAN.md §3.4/§4.1（本地文件）；plan §2；`record2gherkin/evaluation/metrics.py` HEADLINE_MUTATIONS |
| D5 | 主指标存活率 = mean(Surv M1, M2, M3)，M4 单列不并入 | M1/M2/M3 是「结构性」变异：改变的是页面结构与标识符，考的是定位与感知机制；M4 是「概率性」变异：两边都靠运气（生成侧赌 LLM 语义映射、失败可能静默；基线侧属性 selector 天然不受文案影响），且 6 格样本给不出点估计——并入会污染主指标的机制性含义 | 不把 M4 塞进主指标凑「4/5 全赢」的话术 | spec §6.2；plan §5；experiment-report §3-2；`metrics.py:29-30` |
| D6 | 基线 = 属性 selector 式 Playwright 脚本：`#id` → `[data-testid=…]` → `.class` 优先级，**不含** `get_by_role`/`get_by_label`/文本定位；模拟经典「录制回放/codegen 产物」的维护形态 | 这正是 PLAN「selector 式基线」的定义与真实世界「selector 腐烂」痛点的载体（id/class/testid 是工程里最常用的持久定位锚）；口径显式声明避免夸大 | 否决语义定位基线（get_by_role）——那会让对比变成「生成 vs 另一种语义方案」，偏离立论点；绝对 XPath 未被单独固化讨论（推论：它比属性 selector 更脆，M1/M2 布局一动就挂，用它反而是更弱的稻草人——选属性 selector 是**对基线更有利**的形态）；自愈库同理属「另一种语义方案」 | plan §2 基线口径行；spec §5；review 事实核对表「非稻草人」判定；experiment-report §1 方法 B |
| D7 | 基线非稻草人的论证是**预注册的**：plan §5 事先写明 M1/M2 基线预期存活、M3 基线预期全灭、M4 各凭运气 | 防「数据不好看就改口径」：预期结果在实验前固化，M1/M2 下基线 6/6 与生成侧并列呈现反而是「基线公允」的加分项；M3 全灭是属性 selector 录制回放的机制性后果，恰是项目立论点 | —— | plan §5；review 事实核对表；final-analysis §3-1（「主动并列 M1/M2 双免疫」） |
| D8 | 流程选取 F1-F6，每条 4-8 个录制事件，形态各异：F1 搜索（3 事件）/ F2 加入清单·同名按钮（3）/ F3 下单表单·hash 导航+checkbox（11）/ F4 筛选（4）/ F5 视图往返（5）/ F6 数量下单（8） | 覆盖输入/点击/select/checkbox/hash 导航/计数断言全部交互形态；约束：全部无密码字段（绕开 `{{TEST_DATA:}}` 注入链路）、断言只引用固定文案（不与 M4 改写表冲突）、checkbox 仅出现在 F3（单点观察） | F3 实际 11 事件超出「4-8」——hash navigate×2 + checkbox click+input + 提交 click+submit 双事件所致，录制器忠实记录、不规避（test-report 已知 #5） | spec §1.3；test-report §3、§6-5 |
| D9 | 执行编排：每 feature 一次子进程 `python -m testzeus_hercules --input-file … --project-base … --output-path …`，编排/超时/杀进程在外层 | 照抄上游 `mcp_server.py:161-172` 的子进程模板；PLAN §3.3 第一天定死不许进程内连跑（StaticLTM 单例串数据、config 单例 reset 丢态、输出目录共享 import 时时间戳、PlaywrightManager 关闭 fire-and-forget 四实证） | 否决进程内连跑；key 不进 argv（ps 可见） | PLAN §3.3；plan §2 执行编排行；`runner.py:226-242` |
| D10 | 每 run 独立 `--project-base`（`runs/<run_id>/opt`） | 规避上游 config 输出目录 import 时时间戳别名（`config.py:14,60`）与证据混叠；`--project-base` 是**唯一有效**的 run 隔离手段（`--output-path` 不影响 JUnit 实际落盘 getter，见坑 E7） | —— | spec §4.2；test-report §6-17/§6-18 |
| D11 | 指标采集从 JUnit XML 解析：通过 = `<failure>`/`<error>` 子元素缺席；cost/token 取 properties 内 litellm 展平键 | `junit_helper.py:108-124,144-152` 已内嵌全部证据路径与成本，不重复造轮子 | 否决解析 Hercules stdout（格式不稳）；否决 hook 进 runner（侵入上游） | plan §2 指标采集行；`runner.py:453-497` |
| D12 | LLM 配置只经 subprocess `env=` 注入（LLM_MODEL_NAME/BASE_URL/API_TYPE/API_KEY 四键 + 运维键），key 从仓库根 `LLM-Key.txt` 读取 | legacy 直连路径零额外配置文件（config 的 env 映射 + `llm_helper` 消费 `model_api_key`）；key 不进 argv、不进日志（stdout 先 `mask_secret` 再落盘）、不进 manifest | 否决 `agents_llm_config.json` 文件路由（evaluation 阶段不需要；且该路径有 key 必须经文件落盘的问题——benchmark C5 的四键故事是另一条线，见 §3.6） | plan §2 LLM 注入行；spec §4.3；`runner.py:192-218`；`testzeus_hercules/config.py:154-156`；`testzeus_hercules/utils/llm_helper.py:103` |
| D13 | seed 从不含 seed 的格坐标派生：`seed = int(sha1("{exp_id}:{method}:{flow}:{mutation}").hexdigest()[:8],16)`，run_id 在 seed 定后拼装 `<method>__<flow>__<mutation>__s<seed>` | 初稿 `seed=f(exp_id:run_id)` 且 `run_id` 内嵌 `s<seed>` 是**循环引用**，按字面写不出代码（review 必改 1）；从格坐标派生后同格重跑同页面字节、同 exp 内任意两格 seed 互异（M3「动态」语义），且 pilot001/002/003 三个 exp_id 同格 seed 全不同（实测 s2224749585/s1949068991/s1848509261）——exp_id 进哈希让每轮实验天然隔离 | 否决「随机取 seed 再手抄进配置」（不可复现） | review 必改 1；spec §2.3；`sweep.py:104-112`；pilot 目录实测 |
| D14 | pilot 先行：D2 先跑 F3×{M0,M3} + F2×M0（+≤1 次基础设施重试），pilot 暴露的边界回写 spec 后才允许 sweep | pilot 的存在意义是吸收执行侧未知（hash URL 行为、checkbox 双事件、`(occurrence N)` 同名按钮步骤、M3 页面自身 JS、模型串组合、单 run 时长）；F2×M0 是 review 必改 4 专门加的——`(occurrence N)` 语法执行侧从未验证过，若失效会在 5 个变异格系统性假失败且 D3 预算无缓冲 | 否决直接全量 sweep（风险后置到预算锁死后） | plan §2 实验节奏行；review 必改 4；spec §12；`sweep.py:56` |
| D15 | 预算护栏：pilot 4 + full 30 + 基础设施重试 5 = 39 ≤ 40（红线），计划期 `assert_budget` + 运行期每 run 前再拦一次；单价上限 $0.13/次（上游 gpt-4o 实证 46945 tokens/$0.129），合计上限 $5.07 ≤ 预算红线 $10 | 钱与墙钟是实验硬约束；护栏落在单测里（test_matrix F30） | 重试**仅限** timeout/no_junit/服务崩溃——用例失败绝不重跑（防「重跑到好看」） | spec §7；plan §4；`sweep.py:58-63,115-126,332-333`；review 事实核对表 |
| D16 | 内联 JS 与 HTML 同源：客户端行为的元素绑定由 render_page 从**变异后**注册表内插生成（`IDS` map），JS 源码禁现受 M3 影响的定位属性字面量 | review 必改 2：若 JS 硬编码 `getElementById("search_input")`，M3 页面自身行为失效（搜索不执行），F1/F4 会双双假失败——「引擎语义失败」被「页面坏了」污染，主指标整体不可信；且字符串级单测防不住它（A8 对硬编码 JS 同样通过），必须专设 A12 单测 + 浏览器级验证 | —— | review 必改 2；spec §2.1；`demo_app.py:303-318,455-568`；`test_demo_app.py` A12；`test_demo_browser.py` |
| D17 | token 键选取与上游 `junit_helper.py:154-158` 同规则：优先 `usage_including_cached_inference.` 前缀且 `.total_tokens` 结尾的全部键求和，仅无此前缀键才回落其余 `*.total_tokens` | 展平后同时存在 usage_including 与 usage_excluding 两组键，字面全取会**双计、token 虚高一倍**（review 必改 3）；suite 级属性是字符串需类型转换 | 无任何键时为 None（禁止记 0） | review 必改 3；spec §6.3；`runner.py:554-579` |
| D18 | 元素注册表是单一事实来源；review 必改 5 补齐 `cart_count` | M3 变换范围、A2 完整性单测、baseline `expected_selectors()` 覆盖面全由注册表定义；缺行会让「是否参与 M3」处于未定义状态 | 基线 selector 从 M0 渲染产物注册表生成（`selector_for`），**不手抄页面源码**，防漂移 | review 必改 5；spec §1.2/§5；`demo_app.py:113-160`；`baseline.py:51-53` |
| D19 | FirstPass 口径：以正式 sweep（exp001）的 generated×M0 首跑为准；pilot 用独立 exp 目录、feature 文件零修改 | sweep 全量中 F3×M0 并非 F3 字面上的「第一次执行」（pilot 跑过）——pilot 不入正式表 + feature 零改动，实质公允（review 建议不阻塞 1） | pilot 数据不入正式表 | review 建议 1；test-report §5；experiment-report §2 |
| D20 | r2 起双 provider：`LLMProviderConfig(name, key_path, model, base_url)` + `PROVIDERS={"deepseek","glm"}` + `resolve_provider`（None/非法名有明确语义）；默认 deepseek **逐字节不变** | deepseek 余额耗尽后 r2 实验切 GLM（coding plan，glm-5.3-flash）；provider 做成开关让「现状口径」与「新 provider」可干净对比，key 文件各用各的（LLM-Key.txt 单行 / GLM-Key.txt KV 三行） | 不硬编码换模型——改模型必须改代码常量，`GLM-Key.txt` 里的 `MODEL=`/`BASE_URL=` 行只是人工参考、运行时不读（防 key 文件内容漂移影响实验口径） | `runner.py:82-108`；`dev_docs/benchmark/test-report.md` §13.1-13.2、注 2；STATUS 7.6 行 |
| D21 | M2 横幅文案「全场满 99 元包邮，次日达限时 8 折」为固定串 | 与 §1.3 全部断言文本不冲突——M2 不能借横幅制造假差异 | —— | spec §2.2 M2；`demo_app.py:49` |
| D22 | 成本口径 cost_unavailable：litellm 价格表无 deepseek-v4-pro，USD 字段不可得，**以 token 计量**；30 行 cost_usd 解析值为 0.0 被按「不可得」解读而非「免费」 | spec 的 None 防线（缺失记 None 禁记 0）没拦住这种情况——litellm 对无价目型号**显式回 0.0** 而非缺键；报告层把它定性为 cost_unavailable 并以 token 为主计量，同时注明「DeepSeek 前缀缓存会显著折扣实际费用」 | 不虚构 USD 数字 | experiment-report §2；exp001 results.jsonl 实测（30 行 cost_usd 全为 0.0）；spec §6.3 |
| D23 | 失败也是数据：每格只跑一次，timeout/no_junit/断言失败全部进分母按 0 计，manifest.metrics.failures 列 run_id+摘要；agent 非确定性用「seed 与参数全记 manifest、可复现即可信」应对 | 同格重跑结果翻转是确定风险，重跑会变成择优采样 | 不重跑用例失败；不做统计显著性检验（Out of Scope） | plan §6 风险表；spec §6.4/§11；`metrics.py:144-156,231-247` |

## 3. 关键实现细节（逐条）

### 3.1 demo 应用 MiniShop（`record2gherkin/evaluation/demo_app.py`，569 行）

1. **元素注册表 `REGISTRY`**：单一事实来源，每个可交互/可断言元素一条 `Element(key, view, group, tag, text, attrs, options, label_for, hidden, contains)`。视图分 `list`（nav_order 去下单、list_status、search_input/search_button、category_select/max_price_input/filter_button、product_1..3 卡片——名称/价格/加入清单按钮，同名×3）、`order`（nav_home 返回列表、姓名/邮箱/地址 input+label、shipping_select、promo_checkbox、qty_plus/minus/display、submit_button、order_result 隐藏 div）、`common`（cart_count 头部清单计数，review 必改 5 补入）。3 个商品：无线耳机 Pro ¥299（数码）/ 桌面台灯 ¥129（家居）/ 陶瓷马克杯 ¥39（家居）。
2. **固定文案常量**：`共 {n} 件商品`、`找到 {n} 件商品`、`筛选后共 {n} 件商品`、`清单（{n}）`、`数量：{n}`、`下单成功，感谢您的购买！`、`订单号：D20260920-001`——同一常量被渲染器、内联 JS 模板、录制脚本与基线脚本四方引用，杜绝口径漂移（`SAMPLE_NAME="张三"`、`SEARCH_QUERY="台灯"`、`MAX_PRICE_INPUT="300"` 同理）。
3. **客户端行为全部同步 DOM 更新、确定性**：搜索按名称子串过滤；筛选按类别+价格；加清单计数 +1；数量 +/− 限幅 [1,99]；提交不做必填校验、任何提交都成功（避免半失败状态引入非确定性）；切视图 pushState + 按 hash 渲染。
4. **M0 id/testid/class 生成规则**：`stable_id = key.replace("_","-")`（如 `product_1_add` → `product-1-add`），testid 同拼写，class 加 `ms-` 前缀且每元素唯一（保证 M3 替换无歧义）。每个注册元素都有 id，所以基线选择链实际退化为 `#id`。
5. **渲染纯净性**：`render_page(mutation, seed)` 不读时钟/环境/`random` 模块，需要随机处一律 `hashlib.sha256`；同参数两次渲染逐字节相同（单测 A1/A11 锁定）。

### 3.2 变异引擎的精确规则（`demo_app.py:261-321`）

6. **M0 恒等**：按注册表原样渲染，无 seed 依赖。
7. **M1 字段调换**（结构层分组重排，确定性、忽略 seed）：order 表单字段组 `G=[g_name,g_email,g_address,g_shipping,g_promo,g_qty,g_submit]` 反转 `G[:-1]`、g_submit 固定最后 → `[g_qty,g_promo,g_shipping,g_address,g_email,g_name,g_submit]`；list 视图搜索组与筛选组互换；组内元素与全部文本不变。模拟「产品迭代把表单字段挪了位置」。
8. **M2 布局/嵌套**：(a) 每个字段分组包一层 `<div class="m2-card" style="border:1px solid #ddd;padding:12px;margin:8px 0">`——卡片总数 **9**（list 搜索组/筛选组 2 + order 字段组 7，test-report 已知 #9 的「表单字段组」最简解释）；(b) 表单/筛选容器加 `m2-flex`（`display:flex;flex-wrap:wrap`，2 处）；(c) 表单前插固定横幅 `m2-banner`。不改任何既有元素文本/属性。模拟「改版加卡片和横幅」。
9. **M3 标识符随机化**（主指标）：注册表**全部**元素 `id = "m3-" + sha256("{seed}:{key}")[:8]`、`class = "c-" + sha256("{seed}:{key}:class")[:8]`、`data-testid = "m3-" + sha256("{seed}:{key}:tid")[:8]`；**不**改 aria-label/placeholder/label 关联文本/innerText。同 seed 幂等、异 seed 必不同（单测 A5/A6）；label 的 `for` 属性同步换成变异后目标 id（`element_html` 内处理）。模拟「构建管线给类名加 hash、前端框架动态 id」——真实世界 selector 腐烂的头号来源。
10. **M4 文案改写**：改写表 7 键（search_button→[查一下,搜索一下]、filter_button→[筛选结果,开始筛选]、submit_button→[确认下单,立即提交]、nav_order→[马上下单,前往下单]、product_1/2/3_add→[添加到清单,放入清单]），按 `idx = int(sha256("{seed}:{key}"),16) % len(table)` 概率选取，同时作用于 innerText 与 aria-label（M0/M3 下这些按钮无 aria-label，M4 下两者一致——test-report 已知 #10）；qty_plus/minus（符号非文案）、nav_home、全部断言文案不动。同 seed 确定、异 seed 组合不同，「概率性」由 seed 枚举体现（单测 A10：8 个 seed 至少 2 种组合）。
11. **不变量**（所有 M 共同保证）：可访问名文本（除 M4 按其规则改写按钮名）、order_result 等断言文案、option 文本、页面标题不变——这是「蒸馏产物 feature 字面值不受变异破坏、事实回查不受影响」的前提（review 事实核对：feature 只含可访问名/URL/值、无 id）。
12. **JS 同源机制**：`inline_script(mutation, seed)` 从变异后注册表生成 `IDS` map（key→变异后 id）注入脚本模板，运行时 `byId(key)` 经 map 解析——M3 打不断页面自身行为；A12 单测机械断言「M3 产物中内联 JS 引用的每个 id 字面量均存在于该产物 DOM」。

### 3.3 demo 服务（`demo_server.py`）

13. `ThreadingHTTPServer` 绑定 `127.0.0.1:8461`（默认；端口是实验参数——录制产物把 URL 烘进 Given 步骤，**被占硬报错，绝不自动换端口**）。端点：`GET /`（任意非保留路径）→ 当前 `(mutation, seed)` 整文档，`Cache-Control: no-store`；`GET /healthz` → 状态 JSON；`POST /__control` → 更新状态，非法 mutation/seed 400 且状态不变。进程内单值状态带锁，默认 M0/0。
14. 生命周期：sweep 以 `subprocess.Popen([sys.executable, "-m", ...demo_server])` 启停；探活 15s 超时；每格执行前 POST `/__control` 再 GET `/healthz` 校验生效（防状态漂移污染下一格）。

### 3.4 录制与蒸馏（`record_flows.py` / `distill_batch.py`）

15. 录制：本地起 demo 服务（M0/seed0）→ 注入 `recorder.js` → 按流程表机器回放 → 校验每条流程的 §1.3 断言文本确实被 `assert_texts` 捕获（丢了 Then 在此就大声失败，不毒害下游）→ 落 `dev_runs/experiments/recordings/F*.json`（schema v1：navigate/click/input/select/submit）。
16. **录制节奏是工程参数**：`SNAPSHOT_WAIT_MS=900`（等 300ms 延迟快照落地）、`STEP_WAIT_MS=450`（动作间停顿——recorder 的延迟快照归属「触发时刻最新事件」，回放过快会把多个动作压进最后一个事件的 8 条 assert_texts 上限，实测 F3 的「下单成功」会被挤掉）。真人手速天然满足（test-report 已知 #6）。
17. 蒸馏：纯模板 `polisher=None`（**零 LLM**）；守卫：恰 1 Feature/1 Scenario、上游 `split_feature_file` 可解析、禁含 `{{TEST_DATA:` 与 `<masked>`、禁用 LLM 润色（违反即 DistillError）。
18. F3 的 11 事件中，提交按钮的 click+submit 双事件被蒸馏器**确定性去重**（保 click 弃 submit、Then 存活——`fe11083`，PLAN P0-1）；checkbox 的 click+`enter "true"` 双事件**保留原样**进入 feature，作为实验观察点。
19. F5 的 Then 断言偏多（16 条）：视图切换最多带出 8 条新文本（recorder 快照窗口），`list_status`（共 3 件商品）被设计在 DOM 前部让指定断言落进窗口（test-report 已知 #7）。

### 3.5 执行器 runner（`runner.py`，666 行）

20. **命令与隔离**：`[sys.executable, "-u", "-m", "testzeus_hercules", "--input-file", <abs>, "--project-base", <runs/<run_id>/opt>, "--output-path", <opt/output>]`，cwd=仓库根，key 不进 argv。run 目录自包含：`opt/{input,output,proofs,log_files,gherkin_files}` + `stdout.log`（已脱敏）。
21. **状态机**：`passed` / `failed` / `timeout`（900s 默认，`start_new_session=True` + `os.killpg(SIGKILL)` 连浏览器子进程一起杀）/ `no_junit`（进程正常退出但产物缺失，或 JUnit 解析抛 `JUnitParseError`——**绝不静默当通过**）/ `dry_run`（只构造计划，绝不写 results.jsonl）。timeout/no_junit 一律 `passed=False` 进分母。
22. **JUnit 两级定位**：先查直连路径 `<output-path>/<feature>_result.xml`，再 glob `<output-path>/*/`（上游 `get_junit_xml_base_path()` 实际返回 `<PROJECT_SOURCE_ROOT>/output/run_<TS>/` 时间戳子目录，`--output-path` 只落配置项不影响 getter）；同 run 目录隔离下至多一个产物，`newest mtime` 作确定性兜底（坑 E7）。
23. **解析规则**：`passed` = 无 `<failure>`/`<error>` 子元素（error 与 failure 同等视为失败，防御）；`failure_message` 取第一条；`cost_usd` = testcase 级 `usage_including_cached_inference.total_cost`（唯一取数键）→ 缺回落 suite 级 `total_execution_cost`（字符串需 float() 转换）→ 再缺为 None（禁记 0）；`total_tokens` 按前缀规则求和（§2 表 D17）；`duration_s` = testcase `time` 求和，缺回落根节点 time；畸形 XML/无 testcase → `JUnitParseError`。
24. **脱敏**：`mask_secret` 替换 key 全值**及 8 字符前/后缀碎片**（`MASK_FRAGMENT_LEN=8`，短于 16 的 key 不切碎片防把日志打成筛子）；子进程 stdout+stderr 合并捕获后先 mask 再落盘；`RunPlan.env_redacted()` 供日志/manifest 使用（`LLM_MODEL_API_KEY=***REDACTED***`）。空 key 拒绝注入（RunnerError）。
25. **失败诊断面**：timeout/no_junit 的 failure_message 附 stdout 尾部 4000 字符（review 建议 3 落地），归因器可直接消费。

### 3.6 双 provider 配置（r2 增补，evaluation 与 benchmark 共用 runner 层）

26. `LLMProviderConfig` 四字段 + 两个现成实例：`DEEPSEEK`（LLM-Key.txt 单行 / `deepseek-v4-pro` / `https://api.deepseek.com`）与 `GLM`（GLM-Key.txt KV / `glm-5.3-flash` / `https://open.bigmodel.cn/api/coding/paas/v4`）；`resolve_provider(None)="deepseek 现状"`、大小写空白不敏感、非法名 RunnerError、配置对象透传。
27. **KV 解析读 key**：`read_api_key` 文件含 `KEY=` 行时按 KV 解析（忽略 `#` 注释与空行；空 `KEY=` 明确报错）；否则维持单行 `strip()` 历史行为。`MODEL=`/`BASE_URL=` 行**运行时不读**——model/base_url 恒取 provider 代码常量，改模型改代码，防 key 文件漂移影响口径。
28. **child env 全表**（`build_child_env`，现状 8 键）：LLM 四键 `LLM_MODEL_NAME` / `LLM_MODEL_BASE_URL` / `LLM_MODEL_API_TYPE`（恒 `openai`，两家端点都是 OpenAI 兼容）/ `LLM_MODEL_API_KEY`，加运维四键 `ENABLE_TELEMETRY="0"`（上游 config **import 时**初始化 Sentry）、`HEADLESS="true"`、`ENABLE_UBLOCK_EXTENSION="false"`（坑 E1）、`CHROMIUM_PROXY_BYPASS_LOOPBACK="1"`（坑 E6）。model/base_url 可被 provider 覆盖。
29. **四键为什么够（evaluation 侧）**：四键走上游 legacy 直连路径——`config.py` 把 `LLM_MODEL_API_KEY/BASE_URL/TYPE/NAME` env 映射进配置（config.py:154-156、194-196 附近；plan 考据时的行号为 :826-832，上游文件后移），`llm_helper.create_chat_model` 消费 `model_api_key`（llm_helper.py:103，缺 key 抛错）。**不需要 `OPENAI_API_KEY`**。
30. **`OPENAI_API_KEY` 为什么也要（benchmark C5 的另一条线，面试必被追问）**：r2 的 C5 角色路由改用 `agents_llm_config.json` 文件路径后，配置管理器 `initialize()` 文件加载成功即 return（File > Env 源级覆盖，env provider 不注册），文件省略 key 时 planner 是**裸 `ChatOpenAI(**filtered)`** 构造——langchain-openai 的 ChatOpenAI 只认显式参数或 `OPENAI_API_KEY` env，**不读 `MODEL_API_KEY`**。所以 C5 开启时 child env 必须恰注四键：`AGENTS_LLM_CONFIG_FILE` / `AGENTS_LLM_CONFIG_FILE_REF_KEY` / `MODEL_API_KEY` / `OPENAI_API_KEY`（后两者同值，单测断言「多一个键都不行」）。这是「声明在位≠生效」的 env 按键生效路径问题，与 evaluation 的四键是两套消费链（出处：`dev_docs/benchmark/review-r2.md` M1、`spec-r2.md` §6.2/T7；dossier 07 §A5）。

### 3.7 基线（`baseline.py`）

31. **口径声明（最终报告须原样转载）**：模拟经典录制回放 UI 自动化产物，仅属性 selector `#id` → `[data-testid=…]` → `.class`，不用 get_by_role/get_by_label/文本定位。6 条流程函数与 §1.3 步骤逐步镜像；断言失败/selector 失配抛 `BaselineError` → `passed=False`。
32. 每条流程独立 fresh browser context（headless chromium）；动作超时 `DEFAULT_ACTION_TIMEOUT_MS=2000`——M3 格（全部 selector 失配）每流程约 2.3s 就失败，30 格总墙钟 <30s（smoke-baseline 22 秒完成）。
33. `expected_selectors()` 从注册表生成（`selector_for(key, mutation, seed)`），与 demo 页面同源不漂移；`run_flow` 永不抛出，异常摘要（首行、500 字符内）进 failure_message——「断言失败即失败数据」。
34. 基线行 `cost/tokens/model/junit_xml` 恒 null（不经 Hercules、无 LLM、无 JUnit）。

### 3.8 指标（`metrics.py`）

35. 公式：`FirstPass = Σ pass(i,M0,generated)/6`；`Surv(method,M) = Σ pass/6`；主指标 = mean(Surv M1,M2,M3)（M4 单列）；`AvgCost/TotalCost` 只聚合非 None 并报 `cost_missing_count`；`pass` 对失败/timeout/no_junit 一律 0。
36. 防御语义：`latest_cell_rows` 同格多行**最后一行胜**（基础设施重试替代旧行，重复行计数披露）；某格没有结果行按 0 计并列 missing（半途而废的 sweep 抬不高比率）；失败行单列（run_id+摘要 240 字符截断）；`load_rows` 跳过空行/坏行（撕裂的尾行不杀死报告）。

### 3.9 sweep 编排（`sweep.py`）

37. 矩阵 `matrix()` = 2 方法 × 6 流程 × 5 变异 = 60 格；stage：pilot（PILOT_CELLS 3 格）/ full（30 格 generated）/ baseline（30 格）；`--dry-run` 打印每格 seed/run_id/命令不执行。
38. 预算双拦截：计划期 `assert_budget(planned+retry)`、运行期 `hercules_runs >= CAP(40)` 时拒绝再跑；基础设施重试只在阶段末尾扫描 `results.jsonl` 中 timeout/no_junit 的格，缓冲内重跑（pilot 限 1 次、full 限 5 次、baseline 0），用例失败永不重跑。
39. `manifest.json`：exp_id/stage/git_rev（**只读 `.git` 文件解析，从不执行 git 命令**）/model_name/llm_base_url/demo_port/features_dir/timeout_s/budget/cells（每格 run_id+seed）/metrics/时间戳——不含 key 也不含 key 前缀。
40. `results.jsonl` 行：run_id/method/flow/mutation/seed/status/passed/duration_s/cost_usd/total_tokens/junit_xml/failure_message/started_at/finished_at/model/attempt。

### 3.10 测试：evaluation 77 例测什么（`tests/record2gherkin/evaluation/`，实测 `--collect-only` 77 collected）

| 文件 | 例数 | 测什么 |
|---|---|---|
| `test_demo_app.py` | 16 | spec §9 A1-A12：M0 确定性/完整性、M1 反转序、M2 卡片数与横幅、M3 幂等/异 seed/格式/文本不变、M4 改写表与概率性、渲染纯净性、**A12 JS 绑定完整性**（M3 打不断页面自身行为）+ 注册表与 selector 优先级补充 |
| `test_demo_server.py` | 7 | B13-B15（healthz/control/400）+ 端口占用硬失败 + 状态校验补充 |
| `test_demo_browser.py` | 6 | **真浏览器（回环）**：M0/M3（seed 42）下搜索/筛选/清单/下单行为真生效、`#/order` 直达——review 必改 2 的浏览器级证据 |
| `test_runner_parse.py` | 18 | C16-C21（JUnit 解析：通过/失败/无 cost 兜底/畸形 XML/多 testcase/token 前缀规则）+ E26-E29（cmd 三参数、key 不进 cmd、env 全键、mask_secret、timeout 路径）+ 命令模板/行结构/JUnit 时间戳子目录定位补充；env 断言锁 `deepseek-v4-pro` 与 `ENABLE_UBLOCK_EXTENSION=false`（pilot002/pilot001 复盘注释） |
| `test_metrics.py` | 5 | D22-D25（FirstPass/Surv 分组/成本 None 排除/timeout 进分母）+ results.jsonl 容错读取 |
| `test_matrix.py` | 4 | F30-F31（60 格、39≤40 护栏、seed 同格恒等同 exp 互异）+ run_id 格式 + 阶段覆盖 |
| `test_pipeline_d1.py` | 12 | 2 函数 × 6 流程参数化：D1 录制产物 schema/断言表 + 蒸馏 1F/1S 可解析；产物缺失时整组 skip 并给重建命令（新克隆环境不掉坑） |
| `test_llm_provider.py` | 9 | r2 增补：KV 解析（注释行/缺 MODEL 回退/空 KEY 报错/单行向后兼容）、resolve_provider、child env 覆盖、**默认 env 零变化**、glm dry-run 四键、provider 自带 tmp key 文件（绝不读真实 key 文件） |

合计 16+7+6+18+5+4+12+9 = **77**。实现收口时为 68（test-report §1，8.3s 全绿离线），r2 GLM provider 增 9 例至 77（STATUS 全局基线 492 = 21/68/**77**/81/54/191）。全程无真实 LLM、无外网（浏览器组仅 127.0.0.1 回环）。

### 3.11 exp001 执行实况（原始数据复算）

41. **pilot001**（exp_id=pilot001，02:45-02:46）：4 行（3 格 + F3×M0 重试 1 次）全部 `no_junit`，时长 8-16s；stdout 全部命中 `Downloading extension from https://github.com/gorhill/uBlock/releases/download/1.61.0/uBlock0_1.61.0.chromium.zip` → `httpx.ConnectTimeout`——浏览器初始化死于 uBlock 下载。
42. **pilot002**（02:48-02:49）：uBlock 已被禁用（日志 `uBlock extension is disabled in config`），浏览器初始化过了，暴露下一层：`openai.BadRequestError: 400 - The supported API model names are deepseek-flash, deepseek-v4-pro, but you passed openai/deepseek-chat.`；且进程收尾时 `FileNotFoundError: .../Recorded_flow_on_http//127.0.0.18461_MiniShop_商城_results.xml`——蒸馏标题里的 `://` 清洗后残留 `//` 被当目录分隔符，JUnit 写盘失败。4 行全部 no_junit，时长 3-6s。
43. **三修复落码**（`7ce6c95` fix(distiller,evaluation)，03:05）：① runner 模型串 `openai/deepseek-chat` → `deepseek-v4-pro`；② build_child_env 注入 `ENABLE_UBLOCK_EXTENSION="false"`（根因注解：playwright_manager.py:349,369 httpx.ConnectTimeout）；③ distiller 标题清洗（只留[字母数字下划线 CJK 空格连字符]、其余折叠为单 `_`，P1-4，distiller spec.md:61 + test_templates.py 回归）。
44. **pilot003**：3/3 passed——F3×M0 = 423,880 token / 285.66s；F3×M3 = 349,797 / 383.49s；F2×M0 = 62,903 / 59.76s。pilot 验证点全部闭合：hash URL 直达可用（F3/F5/F6 全过）、checkbox 双事件未阻碍 F3、`(occurrence 3)` 同名按钮可用（F2 过）、M3 页面自身 JS 行为完好（F3×M3 过）、cost/token 可解析、单 run 时长实测反哺 sweep 墙钟预估。
45. **exp001 full**（30 格 generated，2026-09-20T19:55Z 起，串行）：60 行 results.jsonl = 30 generated + 30 baseline；generated 全列 M0-M3 6/6、M4 5/6；baseline M0/M1/M2/M4 6/6、M3 0/6。generated 总时长（duration_s 求和，派生值）≈3677s ≈ 61 分钟；baseline 阶段 23.8s。
46. **exp001 baseline**：6 个失败格全部是 M3（每格 2s 动作超时形态：`TimeoutError: Locator.click/fill/select_option: Timeout 2000ms exceeded`），M4 6/6（属性选择器不看文案）。
47. **唯一失败格 generated__F5__M4__s1324679310**（seed 1324679310）：JUnit failure 原文 `EXPECTED RESULT: Home page displays '去下单', '共 3 件商品', … ACTUAL RESULT: Present: '共 3 件商品', … Missing: '去下单' (found '马上下单'), '搜索' (found '查一下'), '应用筛选' (found '开始筛选'), '加入清单' (found '放入清单'/'添加到清单').`——F5 的 16 条 Then 断言绑定录制时原文案，M4 改写了导航与按钮文案，断言如实变红；**操作步骤全部语义映射成功**（页面导航回列表、状态文案都在）。归因器（规则层 S10 断言不匹配签名 + 无 LLM 诚实降级）产出报告：根因证据直指文案改写，替代假设含「用例过时」（experiment-report §3-2/3-3；STATUS 阶段 5 以该真实失败验证归因器）。
48. **成本**：30 次执行合计 **4,109,386 token**（均值 **136,980**/次，实测 136,979.53）；F3 类复杂表单单次可达 42 万（pilot003 F3×M0 = 423,880），F1 类简单流程约 7.3 万（CLI 验收 F1 真实执行 62s/7.3 万；exp001 内 F1 各格 49,688-99,108）。cost_usd 全部 0.0（坑 E10 的 cost_unavailable 口径）。

## 4. 踩坑全记录（每坑一卡：现象 | 定位 | 根因 | 修复 | 教训 | 出处）

### E1 uBlock 扩展下载卡死 → 浏览器初始化全灭（pilot001）
- **现象**：pilot001 全部 4 次执行 8-16 秒即死，状态 `no_junit`；stdout 显示每次都在下载 uBlock 扩展后 `httpx.ConnectTimeout`。
- **定位**：pilot 复盘逐行读 stdout：`Downloading extension from https://github.com/gorhill/uBlock/releases/download/1.61.0/uBlock0_1.61.0.chromium.zip` → ConnectTimeout（上游 `playwright_manager.py:349,369`）。
- **根因**：上游引擎默认启用 uBlock 扩展，浏览器启动时要从 GitHub releases 联网下载 zip；本机直连 GitHub 大流量不稳（用户级 AGENTS.md 记载的同款网络环境），下载超时直接打断浏览器初始化。
- **修复**：child env 固定注入 `ENABLE_UBLOCK_EXTENSION="false"`——demo 站无广告，禁用无副作用；此后成为所有执行链路的标准 env（r2 安全审查把它列入外呼面收敛：harness 侧外联只剩 bigmodel.cn + 127.0.0.1）。
- **教训**：复用上游引擎时，它默认开启的每一个联网副作用（遥测、扩展下载、更新检查）都要逐一盘点并关停——既是稳定性问题也是安全外呼面问题。
- **出处**：`dev_runs/experiments/pilot001/runs/*/stdout.log`（原始日志实测）；`7ce6c95` diff；`runner.py:211-212`；`test_runner_parse.py:213` 注释；dossier 07 §B3。

### E2 模型串 `openai/deepseek-chat` 被 DeepSeek API 400 拒绝（pilot002）
- **现象**：pilot002 浏览器初始化已过，3-6 秒即死；日志 `openai.BadRequestError: 400 - The supported API model names are deepseek-flash, deepseek-v4-pro, but you passed openai/deepseek-chat.`
- **定位**：pilot002 stdout 全文（`generated__F3__M0__s1949068991/stdout.log`，simple_hercules.py:1191 捕获的错误原文）。
- **根因**：spec §4.3 初稿按 litellm 惯例写 `openai/deepseek-chat` 前缀模型串；DeepSeek 端点不认前缀名，其 API 只接受 `deepseek-flash`/`deepseek-v4-pro`。spec 里该组合本就是「D2 pilot 验证点」，pilot 兑现了它的设计目的。
- **修复**：`LLM_MODEL_NAME = "deepseek-v4-pro"`（runner.py:41）；单测锁值并注明 pilot002 复盘。
- **教训**：模型串/base_url 的精确组合必须真调一次才作数——spec 把它列为 pilot 验证点而不是拍死，是对的；反例是文档未回写（E9）。
- **出处**：pilot002 原始日志；`7ce6c95`；`test_runner_parse.py:207`；spec §4.3（待回写项）。

### E3 蒸馏标题污染 JUnit 文件名：`://` 残留 `//` 被当目录分隔符（pilot002，P1-4）
- **现象**：pilot002 进程收尾抛 `FileNotFoundError: .../run_20260921_024901/Recorded_flow_on_http//127.0.0.18461_MiniShop_商城_results.xml`——即使 LLM 侧修好，JUnit 也写不出来。
- **定位**：文件名里的 `//` 来自默认标题 `Recorded flow on http://127.0.0.1:8461` 的 `://`；上游用标题生成结果文件名，`//` 被文件系统当嵌套目录。
- **根因**：蒸馏器对标题无文件名安全清洗；这是阶段 1 分析（phase1-merge-analysis P1-4）预告过的缺陷类别，在 pilot002 以硬失败形态坐实。
- **修复**：标题只保留 `[字母数字下划线、CJK、空格、连字符]`，其余连续段折叠为单个 `_`，首尾 `_`/空格剥除，清洗后为空回退默认标题（`templates.py` + distiller spec.md:61 + `test_templates.py:250` 回归，随 `7ce6c95` 落码）。
- **教训**：自由文本（页面标题）进文件名的每一处都要过同一套安全清洗；pilot 的价值正是让这类缺陷在 3 格小规模暴露，而不是 sweep 30 格全面开花。
- **出处**：pilot002 stdout；`dev_docs/distiller/spec.md:61`；`7ce6c95`；`dev_docs/reports/phase1-merge-analysis.md`（P1-4）。

### E4 pilot 两轮 8 连败：预算护栏如何吸收 + 报告口径的一处不精确（矛盾两说）
- **现象**：pilot001+002 共 8 次执行全部 `no_junit`、tokens=None（零 LLM 成本）；experiment-report §1 概括为「pilot 前两轮 8 次失败**在浏览器初始化**，未产生 LLM 成本」。
- **定位**：原始日志显示两轮根因**不同**：round1（pilot001）= uBlock ConnectTimeout，确在浏览器初始化；round2（pilot002）= uBlock 已禁用、浏览器初始化通过，死于首次 LLM 调用 400 + JUnit 文件名。
- **根因**：报告的概括性表述对 round2 不够精确（后半句「无 LLM 成本」与 results.jsonl tokens=None 相符，前半句只对 round1 成立）。
- **修复/处置**：三层修复落码（E1/E2/E3）后 pilot003 3/3；本卡即口径澄清。预算上：pilot 每轮 ≤4 次、护栏合计 39≤40，8 次失败均为零 token 成本的基础设施失败，未挤占 sweep 的 30 次配额。
- **教训**：实验报告的概括句也要经得起原始数据对账；「失败在哪个阶段」是归因问题，不能靠记忆写。
- **出处**：pilot001/002 原始日志与 results.jsonl；`dev_docs/evaluation/experiment-report.md` §1；STATUS D2。

### E5 macOS 系统代理劫持 127.0.0.1：urllib 拿 502（evaluation 实现期）
- **现象**：本机开系统代理时，sweep 对本地 demo 服务的 `/__control`、`/healthz` 调用失败——urllib 对 `127.0.0.1` 的请求经 `ProxyHandler` 被转发给系统代理，返回 502。
- **定位**：evaluation 实现阶段实测（test-report 已知 #11）；playwright chromium 走自己的网络栈不受影响——只有 Python urllib 系调用中招。
- **根因**：macOS 系统代理默认不豁免回环地址；Python urllib 默认读系统代理。本机回环 ≠ 免疫代理。
- **修复**：sweep 的 `_http_json` 统一 `build_opener(ProxyHandler({}))` **显式禁用代理**；测试同法。演示机若异常：关系统代理或用 file:// fixture（final-analysis R7）。
- **教训**：凡代码里对 localhost 发 HTTP 的测试，代理环境下要么显式禁代理、要么用 file://，并把该假设写进已知问题。
- **出处**：`dev_docs/evaluation/test-report.md` §6-11；`sweep.py:142-148` 注释；`dev_docs/reports/final-analysis.md` R7；`dev_docs/benchmark/plan.md` 风险表。

### E6 chromium 走系统代理不转发回环：任务页 chrome-error → 回环旁路门控（benchmark r2-pre，回灌 evaluation）
- **现象**：系统代理开启时 chromium 也走系统代理，而代理不转发 `127.0.0.1`——本地任务页加载为 `chrome-error://chromewebdata/`（benchmark MiniWoB 场景；exp001 当期未中招，因为 MiniShop 本地服务恰在代理开/关的间隙跑通，且 playwright 在 exp001 环境未走代理）。
- **定位**：benchmark r2 准备期定位到 chromium 的系统代理行为。
- **根因**：chromium 默认跟随系统代理且不含回环豁免；`<-loopback>` 是 chromium 的回环旁路语法。
- **修复**：上游侧新增门控 `playwright_manager._proxy_bypass_args()`——env `CHROMIUM_PROXY_BYPASS_LOOPBACK=1/true/yes` 时追加 chromium 启动参数 `--proxy-bypass-list="<-loopback>"`（只旁路回环、外网仍走系统代理；**默认关，不设 env 无行为变化**，提交 `77a76ec`）；evaluation 的 `build_child_env` 同步注入 `CHROMIUM_PROXY_BYPASS_LOOPBACK="1"`（runner.py:216）。
- **教训**：同一个「回环被代理劫持」问题在 urllib 与 chromium 两层各要解一次：urllib 层显式空 ProxyHandler，chromium 层 `--proxy-bypass-list="<-loopback>"`；上游改动要 flag 门控默认关。
- **出处**：`testzeus_hercules/core/playwright_manager.py:580-588`（源码实测）；提交 `77a76ec`；`runner.py:213-216`；dossier 07 §B2。

### E7 上游 JUnit 实际落盘路径 ≠ spec 文档口径：时间戳子目录
- **现象**：spec §4.2 按「`--output-path` 直落 `<feature>_result.xml`」设计取数；离线实测产物在 `<PROJECT_SOURCE_ROOT>/output/run_<TS>/` 时间戳子目录。
- **定位**：evaluation 实现阶段实测 `get_junit_xml_base_path()`（test-report 已知 #17）。
- **根因**：上游用 import 时求值的时间戳（`config.py:14` `TS`）做 run 隔离，`--output-path` 只写入 `JUNIT_XML_BASE_PATH` 配置项、不影响该 getter——文档口径与实现脱节。
- **修复**：`find_junit_xml` 两级定位（直连路径 → `output/*/` 取最新 mtime）；新单测锁定；`--project-base` 保持唯一有效隔离手段；若 pilot 再发现其它落点只需扩这一个函数。
- **教训**：复用上游前先跑一次「契约探测」，把实测行为写进 spec，别按文档想象设计取数路径。
- **出处**：`dev_docs/evaluation/test-report.md` §6-17；`runner.py:245-268`。

### E8 上游 config import 即初始化 Sentry：离线链路也被外发
- **现象**：跑离线测试/蒸馏时进程退出输出 `Sentry is attempting to send 2 pending events`——「全程离线」的验收被上游遥测打破。
- **定位**：distiller 测试收口期发现（telemetry.py:20 读 env、:65-67 import 时 `sentry_sdk.init`、:76 置空 argv）。
- **根因**：import 副作用 + atexit 外发；凡 import 过上游 config 的进程都会尝试联网。
- **修复**：所有执行链路标准 env 恒含 `ENABLE_TELEMETRY="0"`（runner child env、`distill_batch.py` 在 import 上游前 `os.environ.setdefault`、distiller 测试 conftest 同法）。
- **教训**：「离线」要按「零网络尝试」验收；import 副作用型依赖必须在测试入口统一关断。
- **出处**：`dev_docs/distiller/test-report.md` §2；spec §4.3 注释；`runner.py:208`；`distill_batch.py:22`；dossier 07 §B5。

### E9 spec §4.3 的模型串结论未回写：文档-代码漂移点（如实两说）
- **现象**：spec §4.3 至今写着 `LLM_MODEL_NAME="openai/deepseek-chat"`（pilot 待验证组合），而代码/单测自 `7ce6c95` 起为 `deepseek-v4-pro`；experiment-report 记录执行模型为 deepseek-v4-pro。
- **定位**：对照 spec §4.3 与 `runner.py:41`、`test_runner_parse.py:207`。
- **根因**：pilot 结论通过代码与测试注释固化，spec 正文漏了「回写本节」这一步（spec 自己规定的验收动作只完成了一半）；pilot003 3/3 后火力转向 sweep，回写被跳过。
- **修复/口径**：引用模型配置时以代码与 experiment-report 为准（deepseek-v4-pro @ https://api.deepseek.com）；spec §4.3 该行视为历史稿。此漂移不影响实验数据正确性（env 注入值有单测锁定）。
- **教训**：「pilot 结论回写 spec」这类闭环动作要列入完成的定义，否则一定被进度挤掉；好在单测把真相锁住了。
- **出处**：`dev_docs/evaluation/spec.md` §4.3/§12；`runner.py:40-43`；`7ce6c95`。

### E10 cost 全是 0.0 而不是缺键：None 防线没拦住的 cost_unavailable
- **现象**：spec 规定 cost 缺失记 None 禁记 0；实际 30 行 `cost_usd` 全为 **0.0**（manifest `cost_missing_count=0`、`runs_with_cost=30`）——但 litellm 价格表无 deepseek-v4-pro，USD 数字不可用。
- **定位**：results.jsonl/manifest 复算 + experiment-report §2 的口径声明。
- **根因**：litellm 对无价目型号回传显式 0.0（而非缺键/None），解析层的 None 防线天然不触发；「0 美元」是计价缺失不是免费。
- **修复/口径**：报告层定性 `cost_unavailable`，以 token 为主计量（4,109,386），并注明 DeepSeek 前缀缓存会显著折扣实际费用；不虚构 USD。
- **教训**：指标防线要分「解析层」与「解释层」两层——解析层管缺失，解释层管「拿到了但不可信」的值。
- **出处**：`dev_docs/evaluation/experiment-report.md` §2；exp001 results.jsonl 实测；spec §6.3。

### E11 manifest 被末阶段覆盖：`budget.hercules_used=0` 不是没跑
- **现象**：exp001/manifest.json 显示 `started 2026-09-20T20:10:44 / finished 20:11:08`（24 秒）、`budget.hercules_used: 0`——但 exp001 明明跑了 30 次 Hercules。
- **定位**：full 与 baseline 是两次 sweep 调用、共用同一 exp 目录；manifest 每阶段收尾重写，**最后写的 baseline 阶段**（本地 playwright、0 次 Hercules、23.8s）覆盖了 full 阶段的 manifest。results.jsonl 是追加式，60 行两方法俱全。
- **根因**：manifest 为阶段级产物而非 exp 级累积；STATUS D3 已明确「stage full 只含 generated 侧，基线需单独 `--stage baseline` 补入（manifest 缺数据 ≠ 执行失败）」。
- **修复/口径**：读数以 `results.jsonl` 与 `manifest.metrics`（它聚合全部行）为准，budget 计数看各阶段日志/pilot 目录；不改代码（收尾禁重构）。
- **教训**：多阶段共用目录时，产物文件的所有权（阶段级 vs 实验级）要在 spec 里写死，否则「看板数字」会骗人。
- **出处**：exp001/manifest.json 实测；`dev_docs/evaluation/test-report.md` §5；STATUS 决策日志 D3；`sweep.py:394-424`。

### E12 DeepSeek 402 余额耗尽：重试格秒崩 → 熔断 → 最终弃用（benchmark 线，影响 evaluation runner 演化）
- **现象**：MiniWoB r1 全量中 3 个格的重试行因 LLM 402（余额不足）秒崩，烧掉基础设施重试预算；其后 deepseek 余额彻底耗尽。
- **定位**：r1 失败结构分析逐行核对 results.jsonl 与 stdout，识别 402 形态。
- **根因**：预付费余额是运维状态不是代码状态；无熔断时它以「正常失败格」形态消耗重试池。
- **修复**：r2 加熔断器（402 裸子串匹配 + 连续熔断）与 1-token 预检；**deepseek 余额尽后 r2 实验整体切换 GLM**（coding plan，glm-5.3-flash）——为此给 evaluation runner 长出 `LLMProviderConfig`/`resolve_provider` 开关（默认 deepseek 逐字节不变，见 §3.6）。
- **教训**：付费 API 的余额是实验变量的一部分；先熔断、再预检、最后才烧预算；换 provider 做成开关而不是改常量。
- **出处**：`dev_docs/benchmark/round1-report.md` §4-5（原文「3 格重试行因 LLM 402（余额不足）秒崩——r2 需熔断器」）；STATUS 7.6 行（「deepseek 余额尽」）；`dev_docs/benchmark/test-report.md` §13；dossier 07 §A1。**注意归属：发生在 benchmark r1/r2（exp001 之后），非 exp001 本体。**

### E13 GLM 通用端点 403 model_access_denied → coding 专用端点（benchmark r2 接入期）
- **现象**：用智谱通用 paas 端点调 coding plan 签发的 key 被拒（403 model_access_denied；一手实测路径为 `/api/v1`）。
- **定位**：provider 接入期 dry-run/实测；`GLM-Key.txt` 固化的正确 `BASE_URL` 为 `open.bigmodel.cn/api/coding/paas/v4`。
- **根因**：智谱 coding plan 走独立服务路径，通用端点不认 coding plan key——base_url 与 key 是绑定销售的。
- **修复**：GLM provider 常量固定 `https://open.bigmodel.cn/api/coding/paas/v4`（OpenAI 兼容，`LLM_MODEL_API_TYPE` 恒 `openai`）；C1c 预检端点同步走 provider 配置。
- **教训**：换 provider 时端点常量化 + 预检探测，比「先跑一格再报错」便宜得多。**固化等级注明**：403 字面与 `/api/v1` 原始路径为总编排一手实测、dev_docs 无独立固化记录（dossier 07 §A2 有总编排核验注）；dev_docs 固化的是最终正确端点。
- **出处**：`dev_docs/benchmark/test-report.md` §13.1-13.2；`runner.py:47-49`；dossier 07 §A2。

### E14 NLTK 语料下载挂死：进程活着、日志零进展（benchmark r1；排查方法论样板）
- **现象**：3 个格子进程活着、日志零进展，耗到引擎 600s 超时判负。
- **定位**：先排除再定位，约 5 分钟——3 格共同点是连一个 agent 轮次都没产出（排除模型问题）；CPU 近零不退出（排除死循环，像在等）；**py-spy 抓调用栈**：主线程停在 NLTK 语料下载的读取里；**lsof 看文件描述符**：一条对外网络连接挂着。
- **根因**：无头环境首次运行触发 NLTK 分词语料自动下载，网络受限（CDN 不通）无限阻塞。
- **修复**：r2 起语料**预装**进环境（一行运维变更）。
- **教训**：挂死别猜——py-spy 看栈、lsof 看连接，两个命令把「卡在哪」变成事实；环境依赖的「首次自动下载」在受限网络下全是定时炸弹。**归属注明：发生在 benchmark r1，非 exp001 本体；收录因它是本实验体系共享执行栈的同源环境坑。**
- **出处**：`dev_docs/interview/pitch-and-demo.md` §4 卡 1；`dev_docs/benchmark/round2-report.md` §2-8（「NLTK 语料预装（r1 挂死根因）」）；dossier 07 §B4。

## 5. 面试深挖要点（追问 → 一句话答 → 指向哪节）

| 追问 | 一句话答 | 指向 |
|---|---|---|
| 为什么是这 5 类变异？ | 与总计划定死的变异集一致不增不减：M0 对照、M1/M2/M3 三类结构性变异对应「机制性自愈」、M4 文案对应「纯赌 LLM 语义映射」的概率边界——主指标只选机制性存活的变异 | §2 D4/D5 |
| M1/M2/M3 为什么算「结构性」进主指标、M4 不算？ | M1/M2/M3 改的是结构与标识符，考定位与感知机制；M4 概率性且两边运气成分不同（生成侧可能静默失败、基线侧天然免疫），6 格给不出点估计，单列如实呈现 | §2 D5；§3.2-10 |
| 基线是不是稻草人？ | 不是：属性 selector 是 codegen/录制回放生态最强常见形态，M1/M2 下基线照样 6/6 且报告如实并列（差距全部来自 M3）；预期结果实验前就预注册在 plan §5，防事后改口径 | §2 D6/D7；§3.11-46 |
| 为什么不用 get_by_role 语义基线 / 自愈库？ | 那会把对比变成「我们的语义 vs 另一种语义方案」，偏离「selector 腐烂」这个立论点；口径在 spec 与最终报告里显式声明 | §2 D6 |
| 为什么不用绝对 XPath？ | 固化文档未单列（推论口径）：XPath 比属性 selector 更脆、连 M1/M2 布局变异都会挂，用它反而是更弱的稻草人——选属性 selector 是对基线更有利的形态 | §2 D6 |
| 为什么是这 6 条流程？ | 覆盖输入/同名按钮/select/checkbox/hash 导航/计数断言全部形态，且全部无密码字段（绕开 test_data 注入）、断言只引用固定文案（不与 M4 改写冲突） | §2 D8 |
| 变异怎么实现的？改 DOM 吗？ | 不是事后改 DOM：变异是渲染参数 (mutation, seed)，在元素注册表结构层变换后序列化，同 seed 字节级可重复、纯函数可离线单测 | §2 D3；§3.2 |
| M3 的「动态 id」怎么定义？ | id/class/testid 全部换成 sha256(seed:key) 派生的 m3-/c- 前缀哈希，aria/label/文案不动；同 seed 幂等、同 exp 内异格必异（单测锁定），每格 sweep 分配新 seed | §3.2-9 |
| 变异后页面自己会不会坏？ | 不会：内联 JS 与 HTML 同源——绑定从变异后注册表生成 IDS map，review 必改 2 专设 A12 单测 + 真浏览器用例证明 M3 下搜索/筛选/下单行为完好 | §2 D16；§3.10 |
| 怎么保证可复现？ | seed 从格坐标 sha1 派生（review 修掉的循环引用）、manifest 记全部参数、同格重跑同页面字节；agent 非确定性不靠重跑抹平，靠可复现背书 | §2 D13/D23 |
| 60 格怎么跑的、预算怎么控？ | pilot 4 + full 30 + 重试 5 = 39 ≤ 40 红线，计划期+运行期双拦截且护栏落在单测里；重试只限 timeout/no_junit，用例失败绝不重跑 | §2 D15；§3.9-38 |
| pilot 三轮各发现了什么？ | pilot001：uBlock 下载 ConnectTimeout 全灭；pilot002：uBlock 已禁用后暴露 DeepSeek 400 模型名 + JUnit 文件名 `//`；pilot003：三修复后 3/3——pilot 机制兑现了设计目的 | §3.11-41..44；§4 E1-E3 |
| 唯一失败格 F5/M4 怎么归因的？ | JUnit 断言原文 `Missing: '去下单' (found '马上下单')`——Then 断言绑定录制时原文案，文案改写后断言红是测试的正确行为；操作步骤全成功；归因器 S10 签名 + 无 LLM 诚实降级 | §3.11-47；experiment-report §3-2/3 |
| 成本多少？ | 30 次 4,109,386 token（均值 136,980）；F3 类单次可达 42 万、F1 类约 7.3 万；USD 不可得（litellm 无该型号价目，回 0.0 而非缺键），按 cost_unavailable 处理并以 token 计量 | §3.11-48；§4 E10 |
| token 怎么防双计？ | 与上游同一规则：只取 `usage_including_cached_inference.` 前缀且 `.total_tokens` 结尾的键求和，无前缀键才回落——review 必改 3 修的，字面全取会虚高一倍 | §2 D17 |
| 为什么每 run 一个子进程 + 独立 project-base？ | 上游 StaticLTM 单例/config 单例/输出目录 import 时时间戳/PlaywrightManager 关闭竞态四个实证禁止进程内连跑；时间戳别名与证据混叠要求每 run 独立根 | §2 D9/D10 |
| key 怎么不泄漏？ | 只经 subprocess env 注入（不进 argv/日志/manifest）；stdout 先 mask_secret（含 8 字符前后缀）再落盘；收口对四个目录实值 grep 零命中（含前后缀） | §3.5-24；spec §10.6 |
| OPENAI_API_KEY 为什么也要注入？ | 那是 benchmark C5 文件路由的另一条消费链：agents_llm_config 文件加载成功即屏蔽 env provider，planner 是裸 ChatOpenAI 只认 `OPENAI_API_KEY` env，不读 `MODEL_API_KEY`——evaluation 的四键（LLM_MODEL_*）走 legacy 路径，不需要它 | §3.6-29/30 |
| 换 GLM 是怎么回事？ | exp001 用的 deepseek-v4-pro；后余额耗尽（r1 已有 3 格 402 秒崩），r2 起 provider 做成开关切 GLM coding plan（glm-5.3-flash + coding 专用端点，通用端点 403 model_access_denied），默认 deepseek 口径逐字节不变 | §4 E12/E13；§3.6 |
| 为什么 sweep/测试要显式禁代理？ | macOS 系统代理劫持 127.0.0.1：urllib 经 ProxyHandler 拿 502——回环 HTTP 调用统一空 ProxyHandler；chromium 层另有 CHROMIUM_PROXY_BYPASS_LOOPBACK → `--proxy-bypass-list="<-loopback>"` 门控解 chrome-error | §4 E5/E6 |
| JUnit 文件怎么找？ | 上游实际落 `<project-base>/output/run_<TS>/` 时间戳子目录（与文档口径不符），实现两级定位 + 最新 mtime 兜底，单测锁定 | §4 E7；§3.5-22 |
| evaluation 77 条测试测什么？ | 渲染/变异 16 + 服务 7 + 真浏览器 6 + JUnit/执行器 18 + 指标 5 + 矩阵护栏 4 + D1 管线 12 + provider 9；全部离线（浏览器组仅回环），新克隆环境 12 例 skip 自带重建命令 | §3.10 |
| 结论边界是什么？ | 单 demo 应用、6 流程、单模型（deepseek-v4-pro）；M4 的 5/6 是 6 格样本不给概率点估计；checkbox 双事件在 F3 未阻碍执行；主指标话术以 M3 为主打、主动并列 M1/M2 双免疫 | experiment-report §3；final-analysis §3 |
| 首轮通过率口径有没有水分？ | pilot 跑过 F3×M0，所以 FirstPass 以正式 sweep 首跑为准、pilot 独立 exp 目录且 feature 文件零修改——实质公允，口径在报告里写明 | §2 D19 |
| 遇到失败数据怎么处理？ | 全进分母按 0 计并列失败清单；manifest 每格 seed 可复现；绝不「重跑到好看」——M3 下基线 6 个 2 秒超时格就是原样保留的失败数据 | §2 D23；§3.11-46 |

## 6. 数字与出处对照表

| 数字 | 上下文 | 出处（已复核方式） |
|---|---|---|
| 60 格 = 6 流程 × 5 变异 × 2 方法 | 矩阵定义 | `sweep.py:88-90`；spec §7（代码复算） |
| 主指标 **1.00 vs 0.667** | generated vs baseline 的 M1/M2/M3 均值 | experiment-report §2；exp001 manifest.metrics headline（results.jsonl 复算一致） |
| M3：**6/6 vs 0/6** | 标识符随机化列 | experiment-report §2；results.jsonl 复算（baseline 6 格全 2s 超时形态） |
| M0-M2：两方法各 **6/6** | 对照与结构变异列 | results.jsonl 复算 |
| M4：**5/6 vs 6/6** | 文案改写列；唯一失败格 generated__F5__M4__s1324679310 | results.jsonl 复算；failure_message 原文（`Missing: '去下单' (found '马上下单')`） |
| FirstPass = **6/6 = 1.00** | generated×M0 首跑 | manifest.metrics.first_pass；experiment-report §2 |
| **4,109,386 token / 30 次 / 均值 136,980** | 成本口径（136,979.53 四舍五入） | experiment-report §2；final-analysis §2「逐行复算分毫不差」；本次 results.jsonl 重算一致 |
| F1 ≈ **7.3 万**；F3 ≈ **42 万** | 简单/复杂流程单次 token 量级 | experiment-report §2；STATUS 阶段 6（CLI F1 验收 62s/7.3 万）；42 万锚定 pilot003 F3×M0 = 423,880（dev_runs/experiments/pilot003/results.jsonl 实测） |
| cost_unavailable | 30 行 cost_usd=0.0、litellm 价格表无 deepseek-v4-pro | experiment-report §2；results.jsonl 实测（`set(cost)= {0.0}`） |
| 30 + 3 + ≤7 = **33 次成功执行**；pilot 前两轮 **8 次失败**零 LLM 成本 | 预算实况 | experiment-report §1；pilot001/002 results.jsonl（8 行全 no_junit、tokens=None） |
| 预算护栏 **39 ≤ 40**；单价上限 **$0.13**；合计上限 **$5.07 ≤ $10** | 预算设计 | spec §7；plan §4；`sweep.py:58-63` |
| 77 例（16/7/6/18/5/4/12/9） | evaluation 测试分布 | 本次 `pytest --collect-only` 实测 77 collected；STATUS 基线行；test-report §1（收口时 68） |
| 68 passed（8.3s）/ 156 passed（21.9s） | 实现收口时离线全绿 | test-report §1 |
| smoke-baseline **22 秒**、M3 格失败耗时 ≈**2.3s**、30 格 <30s | 基线离线预演 | test-report §4 |
| 端口 **8461**；`Cache-Control: no-store`；M3 文档含 **34 处 `m3-`** | demo 服务 | spec §3；test-report §2 验收 2（CLI 实测） |
| 快照延迟 **300ms**；步间停顿 **450ms**；快照窗口 **8 条** assert_texts | 录制节奏三数 | `record_flows.py:41-44`；test-report §6-6 |
| F1-F6 事件数 **3/3/11/4/5/8** | 六流程录制事件 | test-report §3 |
| F5 的 Then 断言 **16 条**；视图切换最多带出 **8 条**新文本 | F5 断言偏多原因 | test-report §3/§6-7 |
| M2 卡片数 **9**（2 list + 7 order）+ m2-flex **2 处** | M2 结构 | `demo_app.py:166-170`；test-report §6-9 |
| M4 改写表 **7 键** × 2 候选 | 文案改写 | `demo_app.py:173-181`；spec §2.2（spec 写 5 元素组，代码按商品按钮拆 7 键，语义一致） |
| seed 公式 `int(sha1("{exp_id}:{method}:{flow}:{mutation}")[:8],16)` | 派生规则 | `sweep.py:104-107`；spec §2.3（review 必改 1 修订后） |
| pilot003 三格：**423,880 / 349,797 / 62,903 token**；时长 285.66 / 383.49 / 59.76s | pilot 通过轮 | dev_runs/experiments/pilot003/results.jsonl（逐行实测） |
| pilot001 4 败 8-16s / pilot002 4 败 3-6s | 两轮失败形态 | pilot001/002 results.jsonl + stdout（ConnectTimeout / 400 模型名 / `//` 文件名） |
| `deepseek-v4-pro` @ `https://api.deepseek.com`；`glm-5.3-flash` @ `https://open.bigmodel.cn/api/coding/paas/v4` | 双 provider 常量 | `runner.py:41-49`；benchmark test-report §13.1-13.2 |
| mask 碎片长度 **8 字符**；REDACTED=`***REDACTED***`；timeout 默认 **900s** | runner 常量 | `runner.py:35-38` |
| key 双 grep 零命中（实值 + 8 字符前/后缀，四目录） | 脱敏验收 | test-report §1；spec §10.6 |
| 492 = 21/68/**77**/81/54/191 | 全局测试基线（2026-09-24 实测） | STATUS 基线行；final-analysis §2（292 时代分解） |
| generated 总时长 ≈3677s（≈61 分钟，派生值）；baseline 阶段 23.8s | sweep 墙钟（派生，非文档原文） | results.jsonl duration_s 求和（本次计算，标注为派生） |
| 首轮通过率成功标准 ≥70% → 实际 100% | PLAN 目标达成 | final-analysis §3-4；PLAN §3.4 |

**矛盾/限定两说清单**（引用时必须带上）：① experiment-report 概括「pilot 前两轮 8 次失败在浏览器初始化」——原始日志显示仅 round1 是浏览器初始化（uBlock），round2 是模型 400 + JUnit 文件名（§4 E4）；② spec §4.3 仍写 `openai/deepseek-chat`，代码与单测为 `deepseek-v4-pro`，以代码为准（§4 E9）；③ spec §2.2 M4 改写表写「5 个参与元素」，代码为 7 键（商品按钮逐个列出），语义一致（§3.2-10）；④ E13 的 403/`/api/v1` 细节为总编排一手实测、dev_docs 无独立固化记录；⑤ E12/E14（402/NLTK）发生在 benchmark 线、非 exp001 本体，收录因共享执行栈；⑥ 「F1 7.3 万」同时锚定 CLI 验收运行与 exp001 F1 格区间，「F3 42 万」锚定 pilot003 F3×M0 单次（非 exp001 内 F3 各格 12-39 万区间）；⑦ generated 总时长 61 分钟为派生值。
