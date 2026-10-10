# 07 — 环境与基建踩坑大全 + 工程纪律（dossier）

> 生成：2026-10-10，dossier 子代理 G。范围：跨模块环境坑（模型密钥/网络系统/macOS 工具/测试 git/上游行为/视频制作六域）+ 工程纪律总表 + 演示规避清单（R1-R7）。
> 出处约定：每卡末尾给出固化文档路径与节号；凡 dev_docs 无固化记录的过程性踩坑，逐条标注「未核实（未固化）」。

## 1. 这一页讲什么（3 句）

这一页把散在 30+ 份开发文档里的**环境坑、工具坑、流程纪律**按六个域收拢成坑卡与纪律总表，是被问「踩过最深的坑」「怎么排查」「怎么保证协作质量」时的底层证据库。坑卡统一按「现象 | 定位 | 根因 | 修复 | 教训 | 出处」六要素组织，纪律表统一按「纪律 | 内容 | 为什么 | 出处」四要素组织。最深的五张卡：NLTK 语料挂死（py-spy/lsof 定位法）、C5 裸 ChatOpenAI 四键注入、max_tokens no-op（配置在位≠生效）、session 级 playwright fixture 乱序误红、macOS 系统代理劫持回环。

## 2. 环境坑大全（按域分组，每坑一卡）

### 域 A：模型与密钥域

#### A1 DeepSeek 402：重试格秒崩 → 熔断器 → 最终弃用
- **现象**：r1 全量中 3 个格的重试行因 LLM 402（余额不足）秒崩，烧掉基础设施重试预算；后续 deepseek 余额彻底耗尽。
- **定位**：r1 失败结构分析逐行核对 `results.jsonl` 与 stdout，识别出 402 形态。
- **根因**：预付费模型余额耗尽是运维状态不是代码状态；无熔断时它会以「正常失败格」的形态消耗重试池。
- **修复**：r2 加 C1b 熔断器——attempt 级三条件 + run 级连续 2 格 `BenchmarkError`；熔断 marker `"402"` 为裸子串匹配（误报方向只会「少重试/多熔断」，偏保守可接受）；attempt 日志按 `attempt<N>/stdout.log` 分目录防覆盖；deepseek 余额尽后 r2 实验整体切换 GLM（coding plan，planner=glm-5.3，nav/helper=glm-5.3-flash），provider 做成 `--provider {deepseek,glm}` 开关、默认 deepseek 现状逐字节不变；r2 后又加 C1c 预检（1-token 探测，402/余额/连接异常 → exit 3 不启服务不执行）与 429 熔断配速。
- **教训**：付费 API 的余额是实验变量的一部分；先熔断、再预检、最后才烧预算。
- **出处**：`dev_docs/benchmark/round1-report.md` §4-5（「3 格重试行因 LLM 402（余额不足）秒崩——r2 需熔断器」）；`dev_docs/STATUS.md` 7.6 行（「deepseek 余额尽」）；`dev_docs/benchmark/test-report.md` §12.4（C1b/C1c、402 裸子串）、§12.6-5、§13（provider 集成）；`dev_docs/STATUS.md` 决策日志 2026-09-23（429 熔断配速）。

#### A2 GLM 端点 403：coding plan key 只认 coding 专用端点
- **现象**：用智谱通用端点调 coding plan 签发的 key 被拒（403。**总编排核验（2026-10-10）**：`GLM-Key.txt` 固化的 `BASE_URL` 为 `open.bigmodel.cn/api/coding/paas/v4`（coding 专用端点）；「/api/v1 返回 403 model_access_denied」为 r2 接入期总编排一手实测，dev_docs 未另行固化）。
- **定位**：provider 集成时直接把端点做成常量并 dry-run 实测四键 env。
- **根因**：智谱 coding plan 走独立服务路径，普通 paas 端点不认 coding plan key。
- **修复**：GLM provider 常量固定为 `https://open.bigmodel.cn/api/coding/paas/v4`（OpenAI 兼容，`LLM_MODEL_API_TYPE` 恒 `openai`）；C1c 预检端点同步走 provider 配置。
- **教训**：换 provider 时 base_url 与 key 是绑定销售的，端点常量化 + 预检探测比「先跑一格再报错」便宜得多。
- **出处**：`dev_docs/benchmark/test-report.md` §13.1-13.2（GLM 常量、dry-run 实测四键、`LLM_MODEL_BASE_URL=https://open.bigmodel.cn/api/coding/paas/v4`）。

#### A3 coding plan 条款「仅限编码场景」的灰色地带（用户知情选择）
- **现象**：GLM coding plan 的套餐条款面向编码场景，用于 agent 基准评测属灰色地带；用户知情后选择使用（STATUS 决策日志记录「用户确认 GLM coding plan 额度消耗低于预期」并据此把 r3 起配速翻倍至单段 140 格级——额度决策由用户亲自确认）。
- **定位 / 根因**：deepseek 余额尽后需在「充值 deepseek」与「启用已购 GLM coding plan」之间选择；条款边界的合规判断属用户决策，代理不做主张。
- **修复 / 处理**：决策链完整留痕于 STATUS 决策日志；实验数据全部落在 gitignored 的 `dev_runs/`，不构成对外分发。
- **教训**：条款灰色地带的决策权与知情权都归用户，代理的义务是把事实（额度消耗、失败模式）讲清并留痕。
- **出处**：`dev_docs/STATUS.md` 决策日志 2026-09-23；`dev_docs/benchmark/plan-r3.md` §5、`plan-r4.md` 头注（配速翻倍背景）。条款原文细节：**未核实（未固化）**。

#### A4 key 纪律全链：gitignore → KV 解析 → env 注入 → 日志脱敏 → 双 grep 零命中
- **现象**：全项目双 key（DeepSeek + GLM）在 `dev_runs/`、`record2gherkin/`、`tests/`、`dev_docs/` 四个目录的实值 grep 及 8 字符前/后缀 grep **零命中**（r2 安全结论原话：「key 双 grep 零命中（GLM/DeepSeek 两 key 对 dev_runs 全目录）」）。
- **定位**：每轮实验收口必跑红线命令 `KEY="$(cat LLM-Key.txt)"; grep -rl -- "$KEY" dev_runs record2gherkin tests dev_docs`；推送 GitHub 前再复跑一次（interview-readiness 行动 1）。
- **根因（防的是什么）**：key 可能从四条通道泄漏——被 git 提交、进 argv（ps 可见）、进日志/manifest、进生成的配置文件。
- **修复（四道闸）**：① `LLM-Key.txt`、`.env.local`、`PLAN.md` 全部 gitignore，命令内联读取 `$(cat LLM-Key.txt)`；② key 读取支持两格式——文件含 `KEY=` 行时按 KV 解析（忽略 `#` 注释与空行，空 `KEY=` 明确报错），否则维持单行 `strip()` 历史行为（测试全部用 tmp 假 key，从不读真实 key 文件）；③ key 只经 subprocess `env=` 注入，绝不进 argv/文件/产物；子进程 stdout/stderr 先过 `mask_secret` 再落盘 `stdout.log`，CLI 摘要里 `failure_message` 二次脱敏（实测假 key 明文 grep = 0，`LLM_MODEL_API_KEY=***REDACTED***` 命中 1 行）；④ 生成盘的 `agents_llm_config.json` 永不含 key——写盘函数对含 key 配置直接抛 `BenchmarkError`，manifest 键集单测断言「不含 key 也不含 key 前缀」。
- **教训**：脱敏不是一处 if，是一条从文件到屏幕的全链纪律；每轮实验收口用同一条 grep 命令机械复核，不靠自觉。
- **出处**：`AGENTS.md` 二开规约·密钥安全；`dev_docs/README.md` 核心原则 2；`dev_docs/evaluation/test-report.md` §1（脱敏扫描零命中、8 字符前/后缀）；`dev_docs/evaluation/spec.md` §4.3；`dev_docs/cli/test-report.md` §3-4（假 key 明文零命中实测）；`dev_docs/benchmark/test-report.md` §12.7、§14.7、§15.6、§13.2；`dev_docs/benchmark/round2-report.md` §3；`dev_docs/reports/interview-readiness-audit.md` §4 行动 1。

#### A5 C5 坑：裸 ChatOpenAI 只认 `OPENAI_API_KEY`，少注一键 headline 报废
- **现象**：r2 的 C5 角色路由改用 `agents_llm_config.json` 文件路径后，若 child env 只注 `MODEL_API_KEY`，planner 构造/首调即抛缺 key，125 格 headline 直接报废。
- **定位**：review-r2 四层源码追踪：`AgentsLLMConfigManager.initialize()`（`agents_llm_config_manager.py:119-175`）文件加载成功后**直接 return，env provider 完全不注册**（File > Env 是源级覆盖，不是按键合并）→ 文件省略 `model_api_key` → `normalize_agent_config` 产出无 key 的配置 → PlannerAgent **裸构造 `ChatOpenAI(**filtered, **safe_llm_params)`**（`high_level_planner_agent.py:37-57`），不经 `llm_helper.create_chat_model` → 本仓库 `langchain-openai>=1.1.14` 的 ChatOpenAI 只从显式参数或 `OPENAI_API_KEY` env 解析 key，**不读 `MODEL_API_KEY`**。
- **根因**：`MODEL_API_KEY` 只被 `create_chat_model` 消费（救 nav/helper）；「配置文件加载成功则 env 全部失效」是源级覆盖语义，与直觉相反。
- **修复**：C5 开启时 child env 恰注四键：`AGENTS_LLM_CONFIG_FILE` / `AGENTS_LLM_CONFIG_FILE_REF_KEY` / `MODEL_API_KEY` / `OPENAI_API_KEY`（后两者同值；T7 断言「多一个键都不行」）；key 仍只走 subprocess env 红线不破。
- **教训**：「声明在位≠生效」的第二种形态——env 注入按键生效路径各不相同，必须沿消费端源码逐层核到构造函数；r1 能跑只是因为 env 路径恰好把 key 变成了 config 内字段，切到文件路径后该通道消失。
- **出处**：`dev_docs/benchmark/review-r2.md` M1（事实链与改法）；`dev_docs/benchmark/spec-r2.md` §6.2、T7（四键断言）；`dev_docs/benchmark/plan-r2.md` §91-92；`dev_docs/benchmark/security-review-r2-pre.md:102`。

#### A6 预检与引擎不同栈：litellm 裸模型名会把可用 key 误判为失败
- **现象**：spec-r2 初稿把预检写成 `litellm.acompletion(model, ...)`；按此实现会把「可用的 key」误判为失败并 exit 3——headline 被自己的预检 gate 挡死。
- **定位**：review-r2 M4 核实引擎实际栈是 **ChatOpenAI（langchain-openai）直连 `https://api.deepseek.com`**；litellm 对自定义端点需要 provider 前缀（`openai/deepseek-v4-pro` + `api_base`），裸模型名会被 litellm 当作未知 provider 抛错。
- **根因**：预检走了一条与引擎不同的传输栈，「预检通过」证明不了「引擎能跑」，反之误伤。
- **修复**：预检改为与引擎同栈（ChatOpenAI 直连 base_url，`benchmark/preflight.py`），1-token 探测，detail 全过 `mask_secret`，零文件写。
- **教训**：健康检查必须复用生产传输路径，另起炉灶的探测结果没有证明力。
- **出处**：`dev_docs/benchmark/review-r2.md` M4 行；`dev_docs/benchmark/spec-r2.md:94`；`dev_docs/benchmark/test-report.md` §12.4（preflight）。

### 域 B：网络与系统域

#### B1 GitHub 直连大流量不稳：动态探代理 + 命令级参数，不碰全局配置
- **现象**：本机到 GitHub 直连小请求正常，克隆/拉取等大流量操作经常中断（HTTP/2 framing error、连接超时）。
- **定位**：每次动态探测本机代理端口，不硬编码——`lsof -nP -iTCP -sTCP:LISTEN | grep -E ':(7994|7890|7891|7897|1080|8888|2080)'` 或 `scutil --proxy`；常见 Clash Verge 7897（mihomo 内核，2026-10 起在用）、UniClash 7994、Clash 默认 7890。
- **根因**：直连链路质量随网络环境波动，代理软件本身也在变（端口、是否运行）。
- **修复**：命令级附加代理，不改用户 git 全局配置——`git -c http.proxy=http://127.0.0.1:<端口> -c http.version=HTTP/1.1 clone/fetch/pull`（拉取较稳；SOCKS 用 `socks5h://`）；curl 用 `curl -x socks5h://127.0.0.1:<端口>`；无可用代理时换加速镜像（先验证可用）：`https://ghfast.top/https://github.com/...`、`https://gitclone.com/github.com/...`、`https://gh-proxy.com/https://github.com/...`；失败不盲目重试同一方式，按序换通道。
- **教训**：网络基础设施类操作要「探测一次、命令级生效、不改用户环境」；盲试同一通道是对失败模式的重复下注。
- **出处**：用户级 `~/.zcode/AGENTS.md`（Git/网络代理节，全文）。

#### B2 macOS 系统代理劫持 127.0.0.1 回环：urllib 502
- **现象**：本机 macOS 开系统代理时，`urllib` 对 `127.0.0.1` 回环地址的请求经 `ProxyHandler` 被转发给代理，拿到 502——sweep 的 `/__control`、`/healthz` 调用直接失败。
- **定位**：evaluation 实现阶段实测发现（已知问题 #11）；同类坑在 CLI 侧还有一个变体——`test_record_unreachable_url_exit_2` 依赖 `127.0.0.1:1` 连接被拒，若本机代理转发该请求并返回页面，该用例会退化（断言失败但不挂起）。
- **根因**：macOS 系统代理默认不豁免回环地址；Python urllib 默认读系统代理（`ProxyHandler`），而 playwright chromium 走自己的网络栈不受影响。
- **修复**：sweep 与测试代码对回环 HTTP 调用统一**显式禁用代理**；r2 运维清单固化「chromium 回环代理旁路」一项（旁路手段**已核实（总编排 2026-10-10 代码核验）**：`testzeus_hercules/core/playwright_manager.py:580-588` 的 `_proxy_bypass_args()` 按环境变量 `CHROMIUM_PROXY_BYPASS_LOOPBACK` 返回 `--proxy-bypass-list=<-loopback>`，docstring 原文记录了「本地任务页加载为 chrome-error://chromewebdata/」现象）。演示机若行为异常：关系统代理或改用 `file://` fixture。
- **教训**：本机回环 ≠ 免疫代理；凡代码里对 localhost 发 HTTP 的测试，代理环境下要么显式禁代理、要么用 file://，并把该假设写进已知问题。
- **出处**：`dev_docs/evaluation/test-report.md` §6-11；`dev_docs/reports/final-analysis.md` R7；`dev_docs/benchmark/plan.md` 风险表（「macOS 系统代理劫持 127.0.0.1」）；`dev_docs/benchmark/round2-report.md` §2-8；`dev_docs/cli/test-report.md` §6-11。

#### B3 uBlock 扩展启动下载卡死：child env 一键禁用
- **现象**：exp001 pilot 首跑复盘定位出的三个基础设施问题之一（另两个：模型串改 `deepseek-v4-pro`、标题文件名清洗），修复后 pilot003 3/3 通过。
- **定位**：pilot 复盘（STATUS D2 记为「uBlock 禁用」）；r2 安全审查核实 child env 终态含 `ENABLE_UBLOCK_EXTENSION=false`（`runner.py:140-161`），并把「uBlock 下载关」列入外呼面收敛（harness 侧外联只剩 bigmodel.cn + 127.0.0.1）。
- **根因**：上游引擎默认启用 uBlock 扩展，浏览器启动时要联网下载扩展；受限网络/离线环境下启动路径被下载卡住（**已核实（总编排 2026-10-10 代码核验）**：`record2gherkin/evaluation/runner.py` 注释原文——「本机直连 GitHub releases 不稳，uBlock 扩展下载超时会让浏览器初始化直接失败（playwright_manager.py:349,369 httpx.ConnectTimeout）；demo 站无广告，禁用无副作用」）。
- **修复**：child env 固定注入 `ENABLE_UBLOCK_EXTENSION="false"`，随 `ENABLE_TELEMETRY=0`、`HEADLESS=true` 一起成为标准执行环境。
- **教训**：复用上游引擎时，它默认开启的每一个联网副作用（遥测、扩展、更新检查）都要逐一盘点并关停——既是稳定性问题也是安全外呼面问题。
- **出处**：`dev_docs/STATUS.md` D2 与阶段 4 行（「pilot003 3/3（复盘三修复）」）；`dev_docs/benchmark/review.md:104`、`spec.md:248`、`security-review-r2-pre.md:104`。

#### B4 NLTK 语料 CDN 挂死：py-spy 看栈 + lsof 看连接，5 分钟定位
- **现象**：r1 全量中 3 个格子进程活着、日志零进展，一直耗到引擎 600 秒超时判负。
- **定位**：先排除再定位——3 格共同点是连一个 agent 轮次都没产出（排除模型规划问题）；进程 CPU 近零但不退出（排除死循环，像在等什么）；**py-spy 对挂住进程抓调用栈**：主线程停在 NLTK 语料下载的读取里；**lsof 看文件描述符**：一条对外网络连接挂着。全程约 5 分钟。
- **根因**：无头环境第一次跑触发了 NLTK 分词语料的自动下载，网络受限（CDN 不通）就无限阻塞等。
- **修复**：一行——r2 起语料**预装**进环境（r2 行为变更第 8 条：「NLTK 语料预装（r1 挂死根因）」）。
- **教训**：挂死别猜——py-spy 看栈、lsof 看连接，两个命令把「卡在哪」变成事实；环境依赖的「首次自动下载」在受限网络下全是定时炸弹，能预装就预装。
- **出处**：`dev_docs/interview/pitch-and-demo.md` §4 卡 1（现象/定位/根因/修复全文）；`dev_docs/benchmark/round2-report.md` §2-8；`dev_docs/reports/interview-readiness-audit.md` G8。

#### B5 上游 config import 即初始化 Sentry：离线测试也要压遥测
- **现象**：跑 pytest 时进程退出输出出现 `Sentry is attempting to send 2 pending events`——「全程离线」的验收被上游遥测打破。
- **定位**：distiller 测试收口时发现；追到上游 `testzeus_hercules/config.py` 在 **import 时**就初始化 Sentry 客户端并在进程退出时尝试外发。
- **根因**：import 副作用 + atexit 外发，测试进程只要 import 过上游就会有网络尝试。
- **修复**：`tests/record2gherkin/distiller/conftest.py` 的 `pytest_configure` 里 `os.environ.setdefault("ENABLE_TELEMETRY", "0")`（用 `setdefault`，显式设 1 时仍以环境为准）；该约定推广为所有执行链路的标准 env（child env 恒含 `ENABLE_TELEMETRY=0`）。
- **教训**：「离线」要按「零网络尝试」验收，而不是「我没调网络」；import 副作用型依赖必须在测试入口统一关断。
- **出处**：`dev_docs/distiller/test-report.md` §2（额外收口段）；`dev_docs/evaluation/spec.md` §4.3（`ENABLE_TELEMETRY="0"  # config import 即初始化 Sentry，必须压掉`）。

### 域 C：macOS 工具域

#### C1 screencapture 全屏模式被前台应用抢焦点
- **现象**：第一次录终端实录拍成了 ZCode 窗口，作废。
- **定位**：视频制作复盘（demo-video-report §2.1 踩坑记录原文）。
- **根因**：`screencapture` 全屏/主屏模式跟随焦点，前台应用切换即拍错对象。
- **修复**：改**按窗口 ID 录制** `screencapture -v -l<WID>`，与焦点无关。
- **教训**：录屏自动化要钉死对象标识（窗口 ID），不要依赖「当时谁在前台」。
- **出处**：`dev_docs/interview/demo-video-report.md` §2-1。

#### C2 Quartz 窗口属主在中文系统叫「终端」不叫 "Terminal"
- **现象**：第二次录屏得到空结果（按英文名 "Terminal" 找窗口属主找不到目标窗口）。
- **定位**：视频制作复盘明确记载：「macOS 中文环境下 Quartz 看到的窗口属主名是『终端』不是 "Terminal"（这是第二次录出空结果的根因）」。
- **根因**：macOS 中文系统对系统应用做了本地化显示名；Quartz 窗口属主返回的是本地化名。
- **修复**：窗口匹配改用本地化名「终端」（或按窗口 ID）。
- **教训**：系统 API 返回的显示名是 locale 的函数，自动化匹配要么用 ID 要么枚举比对。
- **出处**：`dev_docs/interview/demo-video-report.md` §2-1。（取窗口列表所用的 `pyobjc-framework-Quartz` 包名坑——「Cocoa 不含 Quartz、须单独装 pyobjc-framework-Quartz」——**总编排一手复现（2026-10-10）**：本会话实测 `--with pyobjc-framework-Cocoa` import Quartz 报 ModuleNotFoundError、换 `pyobjc-framework-Quartz` 即成功，与 C6 卡互证。）

#### C3 ffmpeg drawtext：字体路径带空格必须过滤器内单引号；中文走 textfile
- **现象**：字幕烧录时中文文本与带空格的字体路径两种输入都会让 drawtext 滤镜解析失败。
- **定位**：视频制作复盘（demo-video-report §2-4 原文：「中文文本走 textfile 传入以避开转义问题；字体路径里带空格时，必须用过滤器内的单引号包住」）。
- **根因**：drawtext 滤镜参数解析对空格/特殊字符敏感，命令行转义与过滤器内转义是两层。
- **修复**：中文用 `textfile=` 传文件；字体路径（Hiragino Sans GB，路径含空格）用过滤器内单引号包裹。
- **教训**：ffmpeg 滤镜参数的两层转义（shell→filter）要分开处理；能用文件传的内容就不要塞进命令行。
- **出处**：`dev_docs/interview/demo-video-report.md` §2-4。

#### C4 无 brew ffmpeg：用 imageio-ffmpeg 携带的二进制
- **现象**：本机没有可用的 brew ffmpeg；视频分段/字幕/拼接用的是 Python 包 imageio-ffmpeg 自带的 ffmpeg 二进制。
- **定位**：dev_docs 仅留旁证——demo-video-report 写「这个 ffmpeg build 上 select+metadata 没有输出」（scene 滤镜无输出与特定 build 有关）；二进制来源**总编排现场核验（2026-10-10）**：`~/.cache/uv/archive-v0/y8-WRJ0Zy4HkmpBS/.../imageio_ffmpeg/binaries/ffmpeg-macos-aarch64-v7.1` 存在且为全片制作实际调用者。
- **根因**：macOS 上 brew 安装 ffmpeg 体积大/未安装；Python 生态的 imageio-ffmpeg 自带静态编译二进制可直接调用。
- **修复**：调用 imageio-ffmpeg 提供的二进制路径完成全部分段、drawtext、concat。
- **教训**：工具缺失时优先找 Python 生态的自带二进制（零系统安装、版本随 uv 锁定）；代价是要接受该 build 的功能差异（如本例 scene 滤镜无输出，见 F2 卡）。
- **出处**：旁证 `dev_docs/interview/demo-video-report.md` §2-2；其余**未核实（未固化）**。

#### C5 macOS 无 `timeout` 命令
- **现象**：GNU coreutils 的 `timeout` 在 macOS 默认不存在，shell 层做「限时执行并杀进程」没有现成命令。
- **定位**：过程性踩坑，**未核实（未固化）**。
- **根因**：macOS 不随附 GNU coreutils。
- **修复**：项目里所有「限时杀进程」都落在 Python 层——`run_feature(..., timeout_s=600/900)` 杀进程组（evaluation spec §4.1 / benchmark §7 `timeout_s=600`/cell；CLI Ctrl+C exit 130 + 残留子进程 warning），不依赖 shell timeout。
- **教训**：跨平台脚本不要假设 GNU 工具在；超时控制下沉到 Python subprocess 层反而更可控（进程组、退出码、日志都能拿）。
- **出处**：`dev_docs/evaluation/test-report.md` §2 验收 4（timeout 杀进程组）、`dev_docs/benchmark/test-report.md` §7（timeout_s=600）；命令缺失**总编排现场核验（2026-10-10）**：本机 `which timeout` 无结果。

#### C6 窗口 API 的 Quartz 绑定在独立包
- **现象**：做窗口枚举（配合 C1/C2 录屏定位）时需要 Quartz 的 CGWindowList API；`pyobjc` 的 Cocoa 包不含 Quartz 绑定，需单独安装 `pyobjc-framework-Quartz`。
- **定位**：**总编排一手复现（2026-10-10）**——本会话实测：`uv run --no-project --with pyobjc-framework-Cocoa` 下 `import Quartz` 报 ModuleNotFoundError，换 `pyobjc-framework-Quartz` 即成功；C2 卡为旁证。
- **根因**：pyobjc 按框架拆包，Cocoa 与 Quartz 是两个发行包。
- **修复**：安装 `pyobjc-framework-Quartz`（已实测验证包名）。
- **教训**：pyobjc 系包按 macOS 框架一一对应，装包名 = 框架名。
- **出处**：**未核实（未固化）**；关联旁证 `dev_docs/interview/demo-video-report.md` §2-1。

#### C7 中文字体路径：STHeiti 不存在，落点 Hiragino
- **现象**：字幕/卡片选字体时，常见教程里的 STHeiti（黑体-简）字体文件路径在新版 macOS 不存在；最终字幕字体定为 Hiragino Sans GB，卡片字体栈 PingFang/Hiragino。
- **定位**：固化的只有选型结果（demo-video-report §2-3/4）；「STHeiti 路径不存在」的尝试过程**未核实（未固化）**。
- **根因**：macOS 各版本系统字体清单变动（冬青黑体 Hiragino 长期在、华文黑体 STHeiti 已退位）。
- **修复**：字幕用 Hiragino Sans GB；卡片 HTML 用字体栈 `PingFang/Hiragino` 逐级回退。
- **教训**：macOS 字体引用走「字体栈回退」而不是单一路径；固化文档时记结论也要留选型原因。
- **出处**：`dev_docs/interview/demo-video-report.md` §2-3/§2-4；STHeiti 尝试过程**未核实（未固化）**。

### 域 D：测试与 git 纪律域

#### D1 session 级 playwright fixture 与 CLI sync 实例冲突：子集乱序误红 6 条
- **现象**：`pytest tests/record2gherkin/recorder tests/record2gherkin/cli`（recorder 在前）会挂 cli 的 6 条 record 用例——6 failed；反序 75 例全过；官方入口 `uv run pytest tests/record2gherkin -q`（默认字母序 cli 在前）292（当时）全绿。终审实测复现。
- **定位**：final-analysis §4 R4——recorder conftest 的 **session 级 `browser` fixture** 全程持有 `sync_playwright()`，与 `cli.py:160` 的第二个 sync 实例冲突（playwright sync API 同线程不允许两个并发上下文，benchmark 侧 known issue 1 同理，故 benchmark 浏览器 fixture 设为 function scope）。
- **根因**：session 级资源（playwright 事件循环）跨模块存活，与子进程/新 sync 实例互斥。
- **修复**：演示跑测试只用一条命令（官方入口）；真修复属一行 fixture scope 调整（session→function），终审与演示前刻意不改代码（收尾禁重构纪律）。
- **教训**：fixture 的 scope 是全局契约；能讲清「这是夹具资源隔离问题、不是产品缺陷，官方入口全量是绿的」本身就是测试素养的考点。
- **出处**：`dev_docs/reports/final-analysis.md` R4（6 failed/75 例反序全过、cli.py:160、一行 fixture scope 修法）；`record2gherkin/README.md` 演示与测试注意；`dev_docs/benchmark/test-report.md` §8-1。

#### D2 新克隆环境 ≠ 本机基线：12 例 skip 与三次口径修正
- **现象**：`test_pipeline_d1.py` 12 例依赖 gitignored 录制产物（`dev_runs/experiments/recordings/`），新克隆环境会 skip 并给出重建命令；292 时代口径为「280 passed + 12 skipped」。
- **定位**：终审 R5 指出口径必须注明「本机」；红队复审 M2 进一步抓出 pitch 材料里「本机 492 passed；新克隆 280+12」的算术错位——280+12=292 是旧数，12 例 skip 在 492 基线下从未复验，应表述为「新克隆环境会有 12 例 skip，其余应全过——被追问以现场实跑为准」。
- **根因**：刻意不把大产物提交进 git（设计决策），代价是基线数字带环境前提；材料改数字时没同步改前提数字。
- **修复**：仓库内附重建命令（`uv run python -m record2gherkin.evaluation.record_flows` + `distill_batch`，产物缺失时测试自带 skip 提示）；对外口径统一为红队修正后的表述。
- **教训**：依赖 gitignored 产物的测试必须「skip 而非 fail + 自带重建命令」；基线数字换代时，所有关联前提数字要一起重算。
- **出处**：`dev_docs/reports/final-analysis.md` R5；`dev_docs/evaluation/test-report.md` §6-16；`dev_docs/interview/red-team-report.md` M2；`dev_docs/interview/pitch-and-demo.md` §5 测试基线一句话。

#### D3 测试计数三代漂移：67/68 一格之差与 292→492
- **现象**：① STATUS/README 曾写 distiller 67 例，实际 68——终审 `--collect-only` 实测抓出差 1（+1 来自 `7ce6c95` pilot-infra 对 `test_templates.py` 的增补，晚于看板计数）；② 292 写进 README 后 benchmark 线继续开发，实测已 492，旧数字残留在 interview 材料（audit G7）。
- **定位**：终审 §2 逐目录 `--collect-only`（21/68/81/54 + distiller 68）；audit §更正注 ② 复核 492 = 21/68/77/81/54/191。
- **根因**：看板数字靠人工同步，模块后置增补与跨线开发必然漂移。
- **修复**：分解式统一为实测数（292 时代：21+68+68+81+54；最终：492 = 21+68+77+81+54+191，2026-09-24 `--collect-only` 逐模块实测）；档案纪律写明「数字两处不一致时以一手报告为准」。
- **教训**：计数类数字必须能被一条命令复现（`--collect-only`）；引用前先跑命令，不要抄看板。
- **出处**：`dev_docs/reports/final-analysis.md` §2（67 vs 68）；`dev_docs/STATUS.md` 全局测试基线行；`dev_docs/reports/interview-readiness-audit.md` 更正注 ②、G7；`record2gherkin/README.md` 模块表。

#### D4 上游 black 基线漂移：`make fmt` 会顺手重排上游文件
- **现象**：仓库既有 7 个文件存在 black 基线漂移（历史以默认 88 列格式化），跑全仓 `make fmt` 会一并重排 `base_nav_agent.py`、`multimodal_base_nav_agent.py`、`config_env_loader.py`、`state_handler.py`、`litellm_helper.py`、`simple_hercules.py`/`runner.py` 大部分行、`tests/test_simple_hercules_langgraph.py`——纯格式行合并、无语义变化；另有 9 个上游文件本就不是 black-clean（`make lint` 依赖 `make fmt` 故必然触发）。r4 时 `record2gherkin/` 不在 Makefile fmt 目标内，需按同参数手工校验。
- **定位**：benchmark test-report §12.6-7（r2 披露）、§15.4-5（r4 复核，已全部还原、保持工作区改动严格限于声明落点，并提醒总编排勿把纯格式 diff 混入 r4 功能提交）、known issue 9。
- **根因**：二开规约要求 line-length 200，上游从未按此归一化；全仓格式化目标与「不改上游」红线天然冲突。
- **修复**：二开代码只对自己目录跑 black/isort；触碰文件单独校验；提醒总编排提交时剔除上游纯格式 diff。
- **教训**：「格式化」也是一种对上游的写操作；改动范围纪律要延伸到格式化工具的作用域。
- **出处**：`dev_docs/benchmark/test-report.md` §12.6-7、§15.4-5、§1 补充说明（9 个上游文件非 black-clean）、§15.1（record2gherkin 不在 Makefile fmt 目标内）。

#### D5 单测全绿 ≠ 集成正确：冒烟揪出快照归因缺陷
- **现象**：录制器 20 条单测全绿，但「录制器→蒸馏器」集成冒烟发现 `Then` 断言出现在状态变化之前——重放必然假失败。
- **定位**：STATUS D1 + recorder test-report 补记：input 事件在下一次交互的 focusout 才定稿入队（seq 先于引发 DOM 变化的 click），其 300ms 延迟快照在 click 揭示文本之后拍摄，差集文本被归给先入队的 input 事件。
- **根因**：原 20 条用例只测了「点击后包含」，没测「点击前排除」——断言维度缺失，单测结构性盲区。
- **修复**：`scheduleSnapshot` 改为把差集文本归给**触发时刻最新已入队事件**；spec §3 同步修订；新增回归用例 `test_assert_texts_attributed_to_latest_event`；修复后全量 85 passed、集成冒烟 9/9（真录→蒸馏→上游 `split_feature_file` 可解析恰 1 场景）。
- **教训**：模块边界上的时序归因问题，单测按模块内视角写不出来——集成冒烟是必修工序，不是仪式。
- **出处**：`dev_docs/recorder/test-report.md` 补记；`dev_docs/STATUS.md` D1（「集成冒烟揪出归因缺陷并修复」阶段 1 行）。

### 域 E：上游行为与契约域

#### E1 上游 JUnit 实际落盘路径 ≠ 文档口径：时间戳子目录 + 两级定位
- **现象**：spec 按「`--output-path` 直落 `<feature>_result.xml」` 设计取数，离线实测上游 `get_junit_xml_base_path()` 返回 `<PROJECT_SOURCE_ROOT>/output/run_<TS>/`——`TS` 是子进程 import 时求值的时间戳子目录，`--output-path` 只落到配置项、不影响该 getter。
- **定位**：evaluation 实现阶段实测（`get_junit_xml_base_path()`，已知问题 #17）。
- **根因**：上游用 import 时时间戳做 run 隔离，文档口径与实现脱节。
- **修复**：JUnit 定位改两级查找——先查 `<output-path>/<feature>_result.xml`，再查 `<output-path>/*/<feature>_result.xml`（同 run 目录隔离，取最新 mtime 兜底），新单测锁定；`--project-base` 保持唯一有效 run 隔离手段；若 pilot 发现其它落点只需扩 `find_junit_xml`。
- **教训**：复用上游前先跑一次「契约探测」，把实测行为写进 spec，别按文档想象设计取数路径。
- **出处**：`dev_docs/evaluation/test-report.md` §6-17。

### 域 F：视频制作域

#### F1 慢节奏录制：demo_slow.sh 每步停顿 2-7s
- **现象/定位**：视频终端实录若按真人快速敲命令，画面切换过快观众跟不上；且录制器侧机器回放动作间须 ≥450ms 停顿（`STEP_WAIT_MS=450`），否则多个动作被压缩进最后一个事件的快照（8 条 `assert_texts` 上限）。
- **根因**：录制快照是「300ms 延迟 + 差集」机制，节奏快了断言归属就乱。
- **修复**：用 `dev_runs/demo_slow.sh` 放慢节奏版（每步之间停顿 2-7s）；真人手速天然满足 450ms。
- **教训**：演示脚本的节奏是被下游机制约束的工程参数，不是审美选择。
- **出处**：`dev_docs/interview/demo-video-report.md` §2-1；`dev_docs/evaluation/test-report.md` §6-6；`dev_docs/reports/final-analysis.md` R6。

#### F2 scene 滤镜在此 ffmpeg build 无输出 → 1fps 抽帧按 JPEG 大小定位分段
- **现象**：需要把 50 秒终端实录按 5 个命令步骤切开；`select`+`metadata` 的 scene 检测滤镜在这个 ffmpeg build 上没有输出。
- **定位**：视频制作复盘（demo-video-report §2-2）。
- **根因**：非标准分发 build 的滤镜/元数据支持不全（关联 C4）。
- **修复**：按 1fps（每秒取一帧）抽帧，靠 **JPEG 文件大小的跳变**定位 5 步边界（t≈12.6/23.6/31.6/40.6/49.6s）——画面内容突变（命令输出刷屏）必然带来帧体积突变，比 scene 滤镜可靠。
- **教训**：检测手段失效时，找与目标事件强相关的廉价代理信号（文件大小）往往比换工具更快。
- **出处**：`dev_docs/interview/demo-video-report.md` §2-2。

#### F3 卡片不用剪辑软件：HTML+Playwright 截图
- **现象**：数据表定格卡（M3 6/6 vs 0/6 高亮 + 限定词）需要精确排版与逐字核对。
- **定位/修复**：用 HTML（`/tmp/r2g_cards.html`）+ Playwright 截图生成；字体栈 PingFang/Hiragino；卡上数字与 experiment-report §2 逐字核对。
- **教训**：程序员做视频，排版环节回到最熟的 DOM 渲染管线，可复现、可核对、可重渲。
- **出处**：`dev_docs/interview/demo-video-report.md` §2-3。

#### F4 九段素材统一规格后 concat + faststart；裁掉标题栏
- **现象**：终端窗原始 3040x1786，直接拼接会分辨率/像素格式不一致。
- **修复**：9 段素材统一转 **1920x1080@30 yuv420p**；终端画面先 crop `3040:1710:0:76`（裁掉标题栏）再缩放；concat demuxer 拼接并加 **faststart**（moov 前置，浏览器秒开）。成片 89.7s ≤ 120s 达标，1.7MB。
- **教训**：concat 前统一编码参数是铁律；faststart 是网络播放的必要项。
- **出处**：`dev_docs/interview/demo-video-report.md` §2-5、§1（89.7s/1.7MB）。

#### F5 导出后红线复查：抽帧视觉审查查 key 泄漏
- **现象/定位**：视频含 `run --dry-run` 画面，env 列表是泄密高危区。
- **修复**：导出后按帧抽查——最密文字帧 t=35s（dry-run 段）视觉审查：仅 `LLM_MODEL_API_KEY=***REDACTED***`，无任何 key/token/长十六进制串，全片仅此段展示 env；另抽 t=17.5/78/86s 共 4 帧（字幕完整性、表格 9 个数字逐字一致、放大可读性）。红队 S5 意见（镜 8 字幕须注明「另一条已验收链路 F1」）落地后**重新合成**并再抽查 t=87s。
- **教训**：脱敏验证要作用于**最终产物**（渲染后的帧），不是源素材；「改一点重合成」之后必须重抽查。
- **出处**：`dev_docs/interview/demo-video-report.md` §3 红线检查、§4 抽查记录、§6 红队联动。

#### F6 画质边界如实披露：800x450 放大 2.4x 软化
- **现象**：镜 8 源是 800x450 的 Playwright 固化录屏，放大到 1080p 有轻微软化。
- **处理**：字幕注明「另一条已验收链路 F1：录制 MiniShop 搜索 → 蒸馏 → 真实执行 62s 通过（JUnit tests=1 errors=0）」；备答口径备好（可随时 1080p 重录同链路）；不掩饰、不重录凑完美。
- **出处**：`dev_docs/interview/demo-video-report.md` §5 已知边界；`dev_docs/interview/pitch-and-demo.md` §2 备答。

## 3. 工程纪律总表（纪律 | 内容 | 为什么 | 出处）

| # | 纪律 | 内容 | 为什么 | 出处 |
|---|---|---|---|---|
| 1 | 文档固化四件套 | 每模块 plan/spec/review/test-report 缺一视为未完成；spec 要求「实现者读完不再需要做设计决策」；review 只出 PASS/REVISE | 没有固化文档的工作视为未完成；可回顾、可审计 | `dev_docs/README.md` 生命周期表；`AGENTS.md` 二开规约 |
| 2 | 核心目标优先（最高原则） | 绝不在低收益边界问题上打转；边界写进 spec 的 Out of Scope 或 test-report 已知问题清单，核心功能完成后再补 | 追求一次性周全只会导致目标无法收敛 | `dev_docs/README.md` 核心原则 1；`AGENTS.md` |
| 3 | 审查只拦致命问题 | 审查子代理只拦「会导致核心目标失败的问题」，不吹毛求疵 | 审查的产出是收敛不是完美 | `AGENTS.md` 二开规约；`dev_docs/README.md` 生命周期表 |
| 4 | 密钥三文件 gitignore + 内联读取 | `LLM-Key.txt`/`.env.local`/`PLAN.md` 已 gitignore；任何文档、代码、日志、git 提交不得出现密钥明文；命令用 `$(cat LLM-Key.txt)` 内联读取 | 泄漏不可逆；内联读取让 key 不落第二份持久化 | `AGENTS.md`；`dev_docs/README.md` 核心原则 2 |
| 5 | key 只走 env + 收口双 grep | key 仅经 subprocess `env=` 注入（不进 argv/文件/manifest/日志），stdout 先 `mask_secret`；每轮收口与推送前跑 `KEY="$(cat LLM-Key.txt)"; grep -rl -- "$KEY" …` 红线，双 key 及 8 字符前后缀零命中 | 机械复核替代自觉；生成配置的写盘函数对含 key 配置直接抛错 | A4 卡各出处；`interview-readiness-audit.md` 行动 1 |
| 6 | Git 纪律 | 子代理不做任何 git commit/branch/merge，总编排统一操作；功能在新分支开发→测试通过→合 main→合并后由总编排启动分析子代理复核 | 权责单一，合并质量有人兜底 | `AGENTS.md` 二开规约·Git 纪律 |
| 7 | 约定式提交 | `feat(...)`/`fix(...)` 必须（emoji 允许）；STATUS 决策日志与 git 历史逐条可对上（终审核验过 D0-D3 全部提交号） | 决策可追溯到提交 | `AGENTS.md`；`final-analysis.md` §1.2 |
| 8 | 代码规约 | 二开新代码只进 `record2gherkin/` + `tests/record2gherkin/`，不改 `testzeus_hercules/`（PLAN.md 明确接线点除外）；black line-length 200 + isort；Python 用 logger 不用 print（录制器 JS 例外）；实验产物进 `dev_runs/`（gitignore），结论固化进 dev_docs | 包边界即贡献边界；dev_runs 防大产物污染仓库 | `AGENTS.md` 二开规约·代码规约；`dev_docs/README.md` 代码位置约定 |
| 9 | 上游引擎改动纪律 | 不改主包（接线点逐个申报）；引擎改动全部 flag 化**默认 off** 且默认行为=r1 现状；off 档 **golden 逐字节锁定**（r4 T3a/T9a：flatten 迁移前先对固定合成树抓 golden 留档 `dev_runs/r4_impl/capture_golden.py`，迁移后 `json.dumps` 输出逐字节相等；T8a：补丁层 off golden）；引擎回归守卫 `tests/test_simple_hercules_langgraph.py` 12 passed（stash 前后对照各跑一次）；判分链 diff 审计 0 行 | 「我的每一点提升都能和上游基线干净对比」；可单独 revert | `AGENTS.md`；`dev_docs/benchmark/test-report.md` §12.3、§14.1、§15.2、§15.3 |
| 10 | 演示规避清单 R1-R7 | **R1** record 跨文档导航即中止（exit 2 不落盘）→ 绝不演示录多页站点，用本地单页 demo 或 file://；**R2** `--polish`/`--llm` 真机未验收 → 演示不开这两个 flag；**R3** run 真实执行要 key、约 60s+、约 7.3 万 token/简单流程 → 现场用 `--dry-run`（秒级）+ 固化产物/录屏兜底；**R4** recorder 在前的子集乱序会误红 6 条 → 只用官方单命令跑测试；**R5** 新克隆 ≠ 本机 292/492（12 例 skip）→ 口径固定「本机」；**R6** 脚本驱动录制动作间 ≥450ms → 用默认节奏；**R7** macOS 系统代理劫持 127.0.0.1（urllib 502）→ 关系统代理或 file:// | 现场演示翻车成本最高，规避清单按翻车概率排序 | `dev_docs/reports/final-analysis.md` §4；`pitch-and-demo.md` §0 演示三禁 |
| 11 | 数字纪律 | 数字逐字一致（视频表格帧 9 个数字与 experiment-report §2 视觉核对）；报告内禁止未注明出处的数字；口径主动给限定词（单 demo 单模型、6 格样本不给点估计）；历史文档不改写，更正以新文档固化 | 数字是本项目第一说服力，也是第一风险面 | `demo-video-report.md` §3；`benchmark/test-report.md` §10；`final-analysis.md` §3；`qa-bank.md` Q（历史文档不改写） |
| 12 | 归因敢往低了改 | 发现 max_tokens no-op 后把自己报告里的改进修正为无效、把功劳改记给有逐格证据的改动（超时 41→21 归功 planner 双层超时）；引用数字（81.6%）追到一手出处后废弃整条叙事并固化更正 | 诚实修正比好看数字值钱；「归因要敢往低了改」 | `pitch-and-demo.md` §4 卡 2/卡 3；`round3-report.md` §20/31；`baselines.md` 更正行 |
| 13 | 防作弊纪律 | 页面原生奖励唯一权威（无 LLM 裁判）；五类作弊扫描每轮做；实验前后双安全审查；r2 一次绕过扫描事件按**预登记契约** 10 格无效化压分（58/125=46.4% → 干净口径 55/115=47.8%）——宁可掉分不洗分 | 分数的可信性=实验的全部价值 | `round2-report.md` §3；`interview-readiness-audit.md` §5-14；`benchmark/test-report.md` §11 |
| 14 | 预算与配速纪律 | 预算护栏计划期+运行期双拦截（exp001：pilot 4 + full 30 + 重试 5 = 39 ≤ 40 红线；benchmark 累计 cap 144、红线 142/$20）；基础设施重试仅限 timeout/no_junit，用例失败绝不重跑；分段执行 + 429 熔断 + `--max-cells` 配速；断点续跑=重跑同命令；禁拆 exp-id 分窗（护栏是纪律条款，跨目录靠人工盘点） | 钱与墙钟是实验的硬约束；护栏强度如实分级披露 | `evaluation/test-report.md` §5；`benchmark/test-report.md` §7、§12.6-6；`security-review-r4-pre.md` V8；`STATUS.md` 决策日志 2026-09-23 |
| 15 | 变异测试防空转 | 对审查必改项做变异验证：临时改坏实现→对应用例必须红→复原并校验 sha1 不变（两处必改项均验证） | 防「测试空转」（断言无效的假绿） | `cli/test-report.md` §2.1 |
| 16 | 收尾禁重构 | 终审结论「不建议在演示前做任何重构」；D1 的 fixture scope 一行修法也留而不改 | 收尾阶段稳定性 > 完美性 | `final-analysis.md` §5、R4 |
| 17 | 引用追一手出处 | 81.6% 锚点事件的产出纪律：引用数字前克隆仓库/核对论文原文；数字可比前先对齐观察方式、判分方式、任务集、样本量四要素 | 二手归属错误是引用链的结构性风险 | `qa-bank.md` Q（「引用数字前克隆仓库/核对原文」纪律）；`agent-e-anatomy.md` |
| 18 | 收口口径三件套 | 每模块收口必含：测试命令与结果、已知问题清单（recorder 12 条/distiller 4 条/evaluation 18 条/attributor 23 条/cli 11 条/benchmark 10+ 条）、格式化与脱敏复核 | 已知问题清单是「review→fix 闭环真实发生过」的旁证 | `final-analysis.md` §1.1；各模块 test-report.md |

## 4. 面试深挖要点（追问 → 一句话答 → 指向哪节）

| 追问 | 一句话答 | 指向 |
|---|---|---|
| 踩过最深的坑？ | 首推 max_tokens no-op：注入链验证在位、端点实测 768 配置产出 19,349 token——「声明在位≠生效」，我在自己报告里修正为无效改进并摘除 flag。 | §2 域 A（关联 A1）；纪律 §3-12 |
| 挂死怎么定位？ | 不猜：py-spy 抓栈（停在 NLTK 语料下载读取）+ lsof 看连接（一条对外连接挂起），5 分钟定位，修复是预装语料一行。 | §2 B4 |
| key 怎么管理才不泄漏？ | 四道闸：gitignore+内联读取、KV 解析假 key 测试、只走 subprocess env+落盘前 mask_secret、收口双 key grep 零命中（含 8 字符前后缀）。 | §2 A4 |
| 为什么 planner 会缺 key？ | C5 切文件路由后 File>Env 是源级覆盖、env provider 不再注册，而 planner 是裸 ChatOpenAI 只认 `OPENAI_API_KEY`——所以 child env 恰注四键，多一个键都不行。 | §2 A5 |
| 换过模型吗？为什么？ | deepseek 402 余额尽（r1 就有 3 格重试行 402 秒崩），r2 起换 GLM coding plan（glm-5.3/glm-5.3-flash），provider 做成开关、默认 deepseek 现状逐字节不变，并披露多变量混杂。 | §2 A1/A2/A3 |
| 测试乱序为什么会红？ | recorder 的 session 级 playwright fixture 全程持有 sync 实例，与 CLI 的第二个 sync 实例互斥——特定子集顺序误红 6 条，官方单命令入口全绿，修法是一行 fixture scope（收尾阶段刻意不改）。 | §2 D1 |
| 别人克隆你仓库能跑吗？ | 492 全绿是本机口径；12 例吃 gitignored 录制产物的管线回归会 skip——skip 而非 fail、仓库附重建命令，被追问以现场实跑为准。 | §2 D2 |
| 改了执行内核怎么保证不改坏？ | 全部 flag 化默认 off + off 档 golden 逐字节锁定（T3a/T9a）+ 引擎回归 12 passed 守卫 + 判分链 diff 审计 0 行；每一点提升都能和上游基线干净对比。 | §3-9 |
| macOS 上踩过什么坑？ | 三连：screencapture 全屏被抢焦点（改按窗口 ID -l<WID>）、Quartz 窗口属主中文系统叫「终端」（两次录空的根因）、drawtext 字体路径空格要过滤器内单引号+中文走 textfile。 | §2 C1/C2/C3 |
| 系统代理会干扰吗？ | 会：macOS 系统代理劫持 127.0.0.1，urllib 拿 502——回环 HTTP 调用显式禁代理，演示机异常就关系统代理或用 file://。 | §2 B2 |
| 视频里会泄 key 吗？ | dry-run 画面 key 显示 `***REDACTED***`；导出后抽帧视觉审查（最密文字帧 t=35s）确认全片零 key/token/长十六进制串；改一帧重合成后再抽查。 | §2 F5 |
| 演示现场怕什么？ | 三禁：不现场真跑 LLM（key、60 秒起、约 7.3 万 token 一次）、不录多页站点（注入 JS 随导航销毁，exit 2 干净退出）、不开 --polish/--llm（真机未验收）；规避清单 R1-R7 按翻车概率排序。 | §3-10；§2 F/域 D |
| 单测全绿为什么还有 bug？ | 快照归因缺陷：20 条单测只测「点击后包含」没测「点击前排除」，集成冒烟一跑就暴露 Then 在状态变化之前——单测盲区要靠集成冒烟兜。 | §2 D5 |
| 协作流程怎么保证质量？ | 四件套缺一视为未完成 + 核心目标优先不打转 + 子代理零 git/总编排统一 + 审查只拦致命问题 + 合并后独立分析复核——STATUS 决策日志与 git 历史逐条可对上。 | §3-1/2/3/6 |

## 5. 数字与出处对照表

| 数字 | 上下文 | 出处（文件 §节） |
|---|---|---|
| 492 = 21/68/77/81/54/191 | 全局测试基线，2026-09-24 `--collect-only` 逐模块实测 | `STATUS.md` 基线行；`record2gherkin/README.md` 模块表 |
| 292（终审实测 24.69s）→ 492（audit 实测 94.45s） | 基线两代；292 时代分解 21+68+68+81+54 | `final-analysis.md` §2；`interview-readiness-audit.md` §6 |
| distiller 67→68 | 看板少记 1 例（`7ce6c95` 后置增补） | `final-analysis.md` §2/§5-1 |
| 6 failed / 反序 75 例全过 | R4 乱序误红实测 | `final-analysis.md` R4 |
| 12 例 skip（292 时代 280+12） | 新克隆口径；红队修正表述 | `final-analysis.md` R5；`red-team-report.md` M2 |
| 3 格 402 秒崩 | r1 运维问题 | `round1-report.md` §4-5 |
| 3 格 NLTK 挂死 / 600s 引擎超时 / 5 分钟定位 | r1 挂死排查 | `pitch-and-demo.md` §4 卡 1；`round1-report.md` §1（引擎侧 600s） |
| 19,349 token（768 配置）/ 215 轮 | max_tokens no-op 实测 | `round3-report.md` §20 |
| 超时 41→21（r3）→9（r4）；r1 超时 29 | 各轮超时格数 | `pitch-and-demo.md` §1.3；`STATUS.md` 7.5 行；`round1-report.md` §4-1 |
| 77/125 = 61.6%（官方=clean 零无效格）；四轮 43.2→46.4（干净 47.8）→47.2→61.6 | r4 冻结口径 | `STATUS.md` 7.5 行；`round1/round2/round4-report.md` |
| 55/115 = 47.8%；58/125 = 46.4%；10 格无效化（W2） | r2 双口径与压分 | `round2-report.md` §1/§3 |
| 多导航格 61→16 | r2 C3 单次开局效果 | `round2-report.md` §2-4 |
| 12.9M / 9.08M / 12.9M / 12.2M token | r1-r4 每轮成本 | `round1/round2-report.md`；dossier 00 §2.2；`interview-readiness-audit.md` §5-6 |
| 6 流程 × 5 变异 × 2 方法 = 60 格；30 次执行；4,109,386 token；主指标 1.00 vs 0.667；M3 6/6 vs 0/6；M4 5/6 | exp001 全套口径 | `experiment-report.md`；`final-analysis.md` §2 逐行复算 |
| 62s / 7.3 万 token（F1 真实链路验收） | CLI 真实执行 | `STATUS.md` 阶段 6 行 |
| 39 ≤ 40；142 红线/$20；R2_BUDGET_CAP=144 | 预算护栏三代 | `evaluation/test-report.md` §5；`benchmark/test-report.md` §7、§12.4 |
| 恰四键 env（AGENTS_LLM_CONFIG_FILE/REF_KEY/MODEL_API_KEY/OPENAI_API_KEY，多一个键都不行） | C5 注入断言 T7 | `spec-r2.md` T7；`review-r2.md` M1 |
| `https://open.bigmodel.cn/api/coding/paas/v4` | GLM coding 端点常量 | `benchmark/test-report.md` §13.1-13.2 |
| 双 key grep 零命中（含 8 字符前/后缀） | 每轮收口红线 | `evaluation/test-report.md` §1；`round2-report.md` §3；`benchmark/test-report.md` §12.7 |
| `Sentry is attempting to send 2 pending events` | 遥测外发现象原文 | `distiller/test-report.md` §2 |
| 89.7s / 1.7MB / 1920x1080@30 yuv420p / crop 3040:1710:0:76（原 3040x1786） | 成片规格 | `demo-video-report.md` §1/§2-5 |
| 1fps 抽帧；边界 t≈12.6/23.6/31.6/40.6/49.6s | 分段定位 | `demo-video-report.md` §2-2 |
| 2-7s（demo_slow.sh 步间停顿）；450ms（STEP_WAIT_MS）；300ms（快照延迟）；8 条/120 字符（assert_texts 上限） | 节奏三数 | `demo-video-report.md` §2-1；`evaluation/test-report.md` §6-6；`recorder/test-report.md` §5 |
| 4 帧抽查 t=17.5/35/78/86s（红队修复后再查 t=87s）；表格 9 个数字 | 导出后红线复查 | `demo-video-report.md` §3/§4/§6 |
| 800x450 放大 2.4x | 镜 8 画质边界 | `demo-video-report.md` §5 |
| 74 commits 未推送；`ls-tree origin/main` 零命中 record2gherkin | 推送前可见性差距（G1） | `interview-readiness-audit.md` §6 |
| 代理端口族 7994/7890/7891/7897/1080/8888/2080；Clash Verge 7897；UniClash 7994 | 代理探测顺序 | 用户级 `~/.zcode/AGENTS.md` |
| 20/20×3 次（recorder 定稿连跑）、9/9 集成冒烟、85 passed（快照修复后） | recorder 线验证 | `recorder/test-report.md` §2/补记 |
| 360 文件 / 4,988,677 字节 / 360 行 sha256；125 任务 | MiniWoB vendored 与任务表 | `benchmark/test-report.md` §2 |
| 引擎回归 26 passed（焦点组 4 文件）；12 passed（langgraph 守卫，stash 对照） | 上游改动纪律的回归证据 | `benchmark/test-report.md` §15.1、§12.1、§14.1 |

---

**未核实（未固化）清单汇总**：A2 的 403 字面与「/api/v1」原始路径；A3 的 coding plan 条款原文；B2 的 `--proxy-bypass-list="<-loopback>"` 与 chrome-error 页面细节；B3 的 uBlock 卡死详细现象；C4 的 imageio-ffmpeg 二进制来源；C5 的 macOS 无 timeout 命令本身；C6 的 pyobjc-framework-Quartz 包名事件；C7 的 STHeiti 路径尝试过程。以上均为过程性踩坑，dev_docs 无固化记录，引用时须注明。
