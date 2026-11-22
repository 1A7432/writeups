# 让大模型指挥小模型做 IDA 逆向：校验代理、结论台账与一次盲测

| 项 | 内容 |
|---|---|
| 作者 | [1A7432](https://github.com/1A7432) |
| 版本 | v1.0，2026-11-22 |
| 数据采集时间 | 2026-11-13 至 2026-11-22（UTC+8） |
| 分发级别 | TLP:CLEAR |
| 代码 | [1A7432/retools](https://github.com/1A7432/retools)（快照，不维护） |
| 性质 | 工程记录与小样本实测，**不是对照严格的基准研究**，局限见第 2 节 |

---

## Executive Summary (English)

This note describes a local harness in which a large orchestrating model splits an IDA reverse-engineering job into narrow tasks, hands them to cheap small-model workers, and verifies what comes back. It reports what the harness enforces mechanically, what one blind benchmark run measured, and how often worker conclusions survived review on a real sample.

- **Enforcement is done by a proxy, not by the prompt.** The worker never talks to the IDA MCP server. It talks to a guard proxy that exposes 4 of the server's tools, checks every call against a fixed read-only API (`rekit`), enforces the call budget, and logs each call. Any internal error makes the proxy exit, which leaves the worker with no tools. After each run an audit compares the worker's event stream with the proxy log; a non-IDA tool call or a result that bypassed the proxy voids the run.
- **Blind benchmark (58 questions on 6 stripped, obfuscated ELF binaries; 53 machine-graded).** Workers alone scored 47/53. Fixing one tool bug found by the benchmark gave 48. Adding four tools aimed at the remaining error classes (crypto-constant matching, whole-program emulation, a restricted calculator, exact-value back-references) gave 52 with the same worker model and the same question text. The full workflow with orchestrator verification also scored 52; its one miss was an answer spelling the grader did not accept.
- **Five of the six initial worker errors were interface problems.** One was a tool bug, three were missing tools, one was a transcription slip that a mechanical back-reference removes. Only one was an analysis error, and it stayed wrong after the tools were added.
- **On a real sample, 40 of 94 worker findings were rejected on review (43%).** Rejection is per finding: one wrong detail rejects the whole finding. In most rejected findings the listed addresses and API names were correct, and the error was in an attribution, an offset radix, or the stated purpose of a function. "Evidence address was seen in a tool result" is therefore a weak signal and does not replace review.
- **Cost.** A narrow task costs a worker on the order of 10^5 tokens; an open-ended task ("work out what this program does") costs 1.2–2.4 million.

Each number is from a single run. The benchmark is self-made and small, there is no baseline of the large model working alone, and the reviewer on the real sample was a stronger model that saw the worker's answer.

---

## 1 关键结论

| # | 结论 | 证据类型 | 置信度 |
|---|---|---|---|
| K1 | 对工人的限制由一个 MCP 代理机械执行：只暴露 4 个工具、只放行固定的只读接口、强制调用预算、全量记录；代理内部出错即退出 | 代码与测试（`guard.py`、`guardproxy.py`，离线测试 158 项检查） | 高 |
| K2 | 盲测 53 道机器评分题：工人单独 47，修一个工具 bug 后 48，再补四项工具后 52；整套流程 52 | 评分脚本输出，各一次运行 | 中（样本小，单次运行） |
| K3 | 首轮 6 道错题中 5 道可归因于接口（工具 bug 1、缺工具 3、抄写 1），1 道是分析错误 | 逐题人工归因 + 补工具后重跑 | 中 |
| K4 | 真实样本上 94 条工人结论经复核，54 条确认，40 条驳回（43%） | 台账记录 | 中（复核方不是盲评） |
| K5 | 被驳回的结论里，地址和 API 名多数属实，错在归属、进制或定性 | 驳回备注抽读 | 中 |
| K6 | 开放式任务的 token 成本是窄任务的十倍以上 | 运行记录 | 高 |
| K7 | 这套做法是否优于“大模型单独做” | — | **没有测，不作判断** |

---

## 2 范围、方法与局限

**系统是什么**：一组本地工具。主控是一个大模型的编码代理会话，工人是无头运行的小模型（实测用 deepseek-v4.1-flash）。目标架构限 x86、x86-64、ARM64，宿主是 macOS（Apple Silicon）。

**代码怎么来的**：设计、约束规则和验收标准由作者制定，代码主要由编码代理在这些约束下写成，作者负责审查和决策。文中的“主控核验”指主控模型的核验，不是人工逐条核验，除非另有说明。

**盲测怎么做的**：

- 题库 `obf1`：6 个去符号的 ELF（x86-64 和 ARM64）、58 道题。三个是纯混淆程序（控制流平坦化、不透明谓词、死代码、布尔算术混写、字节码虚拟机），两个是密码学程序（SM4、SM3、SHA-256 变体，SM2 加密与签名，AES-CTR；带诱饵密钥、掩码常量、隐藏的曲线点）。程序都是纯计算，没有恶意行为。
- 出题和答题分离：源码、答案和评分脚本放在答题会话不读取的目录里，答题的主控在评分前没有读过它们。
- 53 道题由脚本评分；5 道解释题按要点清单人工评分。
- 题目和二进制在仓库的 `bench/obf1/public/`。源码、答案和评分脚本不公开，否则题库不能再用于盲测。

**真实样本复核怎么做的**：一个 14 MB 的 32 位 Delphi PE，工人在其上产出的结论全部记入台账，之后对照 IDA 数据库逐条复核，结果写回台账。复核方是主控和另一个更强的模型，绝大多数由后者完成。

**局限**：

- 每个配置只跑了一次，没有重复，没有方差。
- 题库是自制的，规模小，题型偏向密码学和虚拟机。47 到 52 的提升来自针对已知错题补的工具，同一批题重跑，存在对题库过拟合的可能。没有在第二套题上验证这四项工具的效果。
- 没有“大模型单独作答”和“小模型不经本系统直接用 IDA”两条基线。
- 真实样本的复核方看得到工人的答案，不是盲评；复核本身没有再被抽查。
- 另两套题库（`obf2`、`jsvm1`）已建成，本文不报告成绩。
- 代码迁移到独立仓库后，离线测试全部通过，但没有用真实模型重跑一次工人任务。

---

## 3 设计

出发点是一个观察：通用模型做逆向时出的错，多数不是推理错，而是接口错。小模型猜 IDAPython 接口容易猜错，读回几十万字符的引用列表会淹没上下文，抄写长十六进制会丢字符。所以把接口固定下来，把输出压紧，把能机械检查的都交给机器。

### 3.1 固定的只读接口

`rekit` 是 IDA 里的一个只读模块，提供函数卡片、伪代码、引用、调用树、搜索、反汇编等视图。约定：

- 每次调用返回字典，地址一律是十六进制字符串。
- 每次调用有硬性字数预算。截断时必须报总数和续取方法。
- 出错返回“错误 + 提示”，不返回 traceback。

实测：带代码的函数卡片比“原始伪代码 + 平铺引用”少约 50% 字符，不带代码的分诊卡片少约 90%；一个热点函数的引用查询从 93.5 万字符压到 1.4 KB。

### 3.2 校验代理

工人看到的 MCP 服务器是一个代理，代理再去启动真正的 IDA MCP 服务器。

| 机制 | 做法 |
|---|---|
| 工具可见性 | 只暴露 4 个工具（打开库、列出库、执行 Python、查参考），不给保存和关闭 |
| 库 | 只能打开任务指定的那一个 |
| 调用内容 | 执行 Python 的内容逐次检查，严格模式下只放行 `rekit` 调用 |
| 预算 | 调用次数上限由代理执行；接近上限时在结果里提示剩余次数 |
| 记录 | 每次调用的请求和结果完整落盘 |
| 失败方向 | 代理内部任何错误都直接退出，工人随即没有工具可用 |

选代理而不选代理框架的钩子，原因是失败方向。钩子在崩溃、超时或配置读不出时会放行调用，不适合当唯一防线。代理和工人用的框架无关，换后端可以直接套用。

### 3.3 契约、落地检查与审计

- **契约**：任务六个字段。结果是 `status / summary / findings / unknowns / needs`，每条结论带证据地址和置信度。结果有字数和条数上限。
- **落地检查**：证据地址必须在任务或工人自己的工具结果里出现过，否则标记为未落地。低于 0x1000 的数不算地址，避免结构体偏移让结论“落地”。
- **审计**：每次运行后对照工人进程的事件流和代理日志。出现非 IDA 工具调用，或出现绕过代理得到的结果，整次运行判无效。
- **精确值回填**：工人在答案里写对某次调用结果的引用，由工具换成原值。答案里剩下的长十六进制若在任何工具输出里都找不到，会被列出。

### 3.4 台账

每个数据库一份只追加的台账。工人的结论先以“未核验”入账，主控或复核方标记确认或驳回并写明理由。

一条关键规则：**函数卡片只显示已核验的笔记。** 否则未核验的工人结论会被下一个工人当作证据引用回来，错误会自我强化。

### 3.5 补充工具

| 工具 | 用途 |
|---|---|
| 函数级仿真（Unicorn） | 每次请求一个全新实例；导入全部拦截；支持先跑初始化函数再调目标函数 |
| 整程序行为报告 | PE 和 ELF 在 macOS 沙箱里仿真，子进程先自检沙箱生效再碰样本；ELF 另可在断网、零挂载、能力全丢的容器里真跑 |
| 计算草稿 | 自己写的受限解释器，不用 `eval`，无文件、导入、属性访问；内置哈希、分组密码、椭圆曲线的参考实现，标准常量可覆盖 |
| 密码常量比对 | 把库里的常量表和参考表逐项比对，近似匹配时报出不同的下标和值 |

---

## 4 盲测结果

### 4.1 成绩

| 配置 | 机器评分（53 题） |
|---|---|
| 工人单独作答 | 47 |
| 同上，受工具 bug 影响的一个任务用修好的工具重跑 | 48 |
| 同上，再加四项工具（常量比对、整程序运行、计算草稿、精确值回填） | 52 |
| 整套流程（工人 + 主控核验） | 52 |

按程序拆分：

| 程序 | 工人单独 | 加四项工具 | 整套流程 |
|---|---|---|---|
| t1_easy | 8/8 | 8/8 | 8/8 |
| t2_medium | 7/8 | 8/8 | 8/8 |
| t2_medium_arm64 | 2/2 | 2/2 | 2/2 |
| t3_hard | 8/9 | 8/9 | 9/9 |
| t4 | 10/13 | 13/13 | 13/13 |
| t5 | 12/13 | 13/13 | 12/13 |

后两列都是 52，但错的不是同一题。整套流程错的一题是曲线名称的写法不在评分器的可接受列表里，答案和参考答案指的是同一条曲线。5 道人工评分的解释题，工人单独作答和整套流程都要点齐全。

### 4.2 首轮六道错题的归因

| 题 | 原因 | 归属 |
|---|---|---|
| 中等程序的通过输入 | 仿真用的内存镜像每段少导出一个字节，正确输入被判成不通过 | 工具 bug |
| 哪些指令处理函数永不执行 | 把“永不执行”理解成“分发表里够不着” | 工人分析 |
| SHA-256 哪个常量被改 | 肉眼比对 64 个常量，漏掉一位十六进制 | 缺工具 |
| 给定输入的完整输出行 | 靠函数级仿真一步步算 HMAC，预算耗尽 | 缺工具 |
| 由密文反推明文 | 需要解密计算，工人没有可用的计算手段 | 缺工具 |
| 180 字节的原始输出 | 抄写长十六进制时丢了 3 个字符 | 抄写 |

补工具后用同样的题目文字重跑四个含错题的任务。工人主动使用了新接口：计算草稿 27 次、整程序运行 7 次、常量比对 1 次、值引用 21 处。四个任务合计耗时从 2879 秒降到 1793 秒，token 基本持平（535 万对 566 万）。

仍然答错的是“永不执行”那一题：工人从“都能执行”改成列出三个，漏掉了无效操作码的处理函数。这是分析不够细，不是缺工具。

### 4.3 开放式分析

只给一句“搞清楚这个程序在干什么”时，五个程序共 27 条必备要点，工人一轮答出约 23 条。出错的三处：

- 把被改过常量的 SHA-256 当成标准算法，并把结果对不上归咎于仿真器。
- 把循环左移 4 位读成加 4。
- 漏掉一个没有任何引用的死函数。

主控核验时三处都被发现，办法分别是比对常量表、用另一种语言重写虚拟机复算、查交叉引用。

### 4.4 成本

| 阶段 | 工人数 | token | 单个耗时 |
|---|---|---|---|
| 开放式分析 | 5 | 1050 万 | 9–16 分钟 |
| 定点答题 | 13 | 1257 万 | 2–16 分钟 |

开放式任务每个 120–240 万 token，窄任务十几万。差距来自调用次数多，且每次调用都要重读越来越长的上下文。绝大部分 token 是缓存读取。

---

## 5 真实样本上的复核结果

在那个 Delphi PE 上，工人共产出 95 条结论（87 条严格模式，8 条自由模式）。其中 94 条有复核结论：

| 复核结论 | 条数 | 占比 |
|---|---|---|
| 确认 | 54 | 57% |
| 驳回 | 40 | 43% |

驳回按整条结论计：一条结论里有一处错，整条驳回。读驳回备注，多数是“这部分对，那部分错”的形式，错处集中在几类：

| 类型 | 例子 |
|---|---|
| 定性过头 | 把跨进程读取任务栏按钮数据的函数称作进程注入例程。列出的 API 都存在，但没有写内存，也没有远程线程 |
| 参数或字段归属错 | 把传给文本测量函数的栈参数归给了另一个调用 |
| 进制错 | 代码里是十进制 732（0x2DC），结论写成 0x732 |
| 库函数认错 | 把运行库的异常收尾例程、字典的冲突计数属性当成业务逻辑 |
| 清单不全却声称完整 | 声称列全了网络相关函数，漏掉 `getpeername` 的唯一调用者 |
| 时序错 | 把发生在接收调用之前的检查说成接收之后 |

这些结论的证据地址全部“落地”，也就是地址确实在工具结果里出现过。所以落地检查只能排除凭空编造的地址，排除不了看着真地址说错话。工人自报的置信度同样偏高。

这组数据决定了台账的那条规则：未核验的结论不进函数卡片。按 43% 的驳回率，如果让未核验结论回流给后续工人，接近一半的“已知事实”是带错的。

---

## 6 讨论

**限制要做成做不到，不要做成不许做。** 提示词里写“只读”“不要调用别的工具”是请求。代理只暴露 4 个工具、出错即退出，是事实。审计再从另一侧核对一遍，两者不一致就作废。

**先看错题属于谁。** 首轮 6 道错题里 5 道不需要更强的模型，需要的是修一个导出 bug、补三个工具、加一个值引用。只看总分会得出“小模型不够用”的结论，逐题归因得出的结论相反。

**小模型的事实比定性可靠。** 盲测和真实样本一致：调用链、引用计数、字符串位置这类可以从工具结果直接读出的内容基本可靠；“这个函数是干什么的”这类概括容易过头。任务应该问事实，定性留给核验方。

**核验是成本的一部分，不能省。** 43% 的驳回率意味着不核验的台账不可用。核验用的是更贵的模型，这部分成本本文没有单独计量，所以本文不能回答“这样做是否比大模型单独做更省”。

**任务要窄。** 开放式任务贵十倍以上，要点覆盖率（约 23/27）也不比窄任务的正确率高。

---

## 7 已知问题

- 工人慢：再小的问题也要几十秒到几分钟。单个查询由主控直接用只读接口更快。
- 任务措辞有歧义时工人不会反问。计数口径要写清，并要求给出调用链。
- 自由模式只按名字拦截写操作，不是沙箱，有意绕过可以绕过，所以默认用严格模式。
- IDA MCP 服务器在最后一个会话释放数据库时会自动保存。工具没有改内容，但数据库文件的修改时间会变。
- 容器真跑不是恶意样本级别的隔离边界。
- 常量比对扫不到按公式现场计算的常量（SM4 的 CK、AES 的 Rcon）和用 SIMD 从别的数据算出来的常量。
- 整程序仿真对大型 GUI 程序价值有限：文中的 Delphi 样本在运行库初始化阶段就停了。

---

## 附录 A：复现

```sh
git clone https://github.com/1A7432/retools && cd retools
./test                      # 离线测试；需要 Docker 或工人后端的测试集会跳过
ls bench/obf1/public/       # 6 个二进制和 questions.json
```

盲测的答案和评分脚本不在仓库里，所以外部无法复算第 4 节的分数，只能复现系统本身和题目。第 5 节的台账含真实样本的分析结论，同样不公开。

## 附录 B：数据出处

| 数字 | 出处 |
|---|---|
| 47、48、52、52 及按程序拆分 | 评分脚本对四份答卷的输出，各一次 |
| 六道错题归因、27 条要点约 23 条 | 盲测当天（2026-11-14）的记录 |
| 调用次数、耗时、token | 工人运行记录 |
| 95 条结论、94 条复核、54/40 | 该样本台账，按每条结论的最后一次复核计，2026-11-22 |
| 50%、90%、93.5 万字符到 1.4 KB | 仓库 `docs/DESIGN.md` 的实测数据一节 |
