---
title: "System One Models 与 Jev：不生成字符串的模型，凭什么快两个数量级"
date: 2026-09-16T14:46:00+08:00
tags: ["推理架构", "结构化输出", "延迟预算"]
author: "繁星"
---

请求路径里放一个模型做判断——这笔转账要不要放行、这条工单要不要升级。现在的做法是发 prompt、解析返回的 JSON：单次请求几秒到几分钟，输出 token 按字计费，而 SLO 是五百毫秒。这个差距调参补不回来。

这时有人递来一个数字：193.6x Faster, 444.6x Cheaper。它印在官网首页 [1]，脚注指向发布博客《Introducing System One Models & Jev》（2026-09-15，作者 Diogo Almeida）[2]。首个公开发布的模型叫 Jev，early access 阶段，输入 \$0.042/MTok，输出 token 免费 [2]。

本文只做一件事：把「快两个数量级」拆成能逐格核对的条目，标清哪格能自己复核、哪格只是厂商自述。下文关键数字带证据级别（A 官方一手可复现，B 官方一手未复现，C 厂商自述，D 推断）。

## 一、变的是接口契约，不是推理速度

官方博客的对比表把差异写在 Sampling 一行 [2]：LLM 是 Sequential，一次一个 token、每个都以前一个为条件；System One 是 Parallel，一次查询产出全部输出。Diogo Almeida 在 Hacker News 上把它讲成硬约束 [10]："strings (and all sequential data structures) are not allowed at all"——禁止字符串，输出才能整体并行算完，输出 token 成本才是零。

落到 API 上只剩三种提问类型：

| 类型 | 返回 | 带 confidence |
| --- | --- | --- |
| Choice | choice / probabilities / confidence | 是 |
| Score | score / legend / probabilities / confidence | 是 |
| Noul | 0–1 概率 | 否 |

为什么这件事要紧：并行求值若成立，延迟就不随问题数量线性增长。不过它目前只是官方口径 [2]（B），值得量一遍：固定 state，把问题从 1 个加到 20 个看耗时与成本。

三条约束决定改造量：一个请求 = 一个 state + 一个 questions map，同一请求内所有问题互相独立并行求值，"one answer does not become context for another question" [6]——后一问依赖前一答，必须在代码里发第二个请求。state 与 questions 共享约 32,000 token 预算（约 150,000 英文字符）[6]。Choice 基数上限 255 [2]。

## 二、193.6x 与 444.6x 相对谁

同一周内官方对同一产品给过 20x / 40x / 100x / 193.6x / 200x / 400x / 444.6x 七种倍数，口径分裂值得一摆：

| 出处 | 数字 | 证据级别 |
| --- | --- | --- |
| [1] 官网首页 | 193.6x faster / 444.6x cheaper，脚注只写 "based on workflows for System One tasks" | B |
| [2] 发布博客正文 | two orders of magnitude faster；40x–200x faster | B |
| [3] 官方 X 帖 | 20–200x faster；40–400x cheaper | B |
| [4] 新闻稿 | up to 100 times faster and less expensive | B |

444.6x 超出官方 X 帖写的 40–400x 上限 [3]（博客只写 40x–200x [2]）。博客的 Nuance 段落承认首页数字来自自制 workflow evals，且 "we expect that these are on the higher end of real world gains" [2]。

逐格核对边界：

| 维度 | 官方口径 | 证据级别 | 能独立复核到哪一步 |
| --- | --- | --- | --- |
| 分子 | Jev 一次请求并行产出全部带类型答案的端到端耗时 | B | 需 API key，无 key 测不了 |
| 分母 | 某个前沿 LLM 配置的端到端耗时 | B | 不能：首页未指名 baseline |
| 参考标签 | Astra 与 Fable 5.1 的平均，两者 high thinking [5] | B | 这是模型间一致性，不是真值 |
| 定价 | 输入 \$0.042/MTok，输出免费 [2] | B | 能：238x 可复算，10 ÷ 0.042 = 238.1（A），Fable 5.1 输入价见 [13] |
| 聚合方式 | 四任务等权平均 [5] | A | 能：复算与图上标注一致 |
| 基线工程补偿 | 仓库公开提供结构化包装与 n_retry_malformed_structure 纠错重试 [11] | A | 能：README 可逐条核对；「基线套官方 adapter」只是博客自述 [2] |

为什么这件事要紧：这两个数字要进评审材料，分母是谁必须先写清楚——这正是首页脚注没答的。

切成两栏看：**厂商自述（B）**——193.6x / 444.6x、0% type error、成组校准；**可独立复核（A）**——238x 输入价算术、四任务等权聚合方式、基线那一侧的 wrapper 与纠错重试。

官方博客把基线延迟写成 "3 to 329 seconds" 并引到榜单页 [2]，该页 TTFT 列实为 1.9–198 s（A），两个值都对不上 [12]。193.6x 与 444.6x 的推导、baseline 名称、prefill/decode 拆分与逐例数据都未公开：闭源托管 API 加无 key，到此为止。

## 三、失败模式迁移清单

「不会出错」只到形状这一层。官方把 0% type error 写进图，同页标注 "Our number is not empirical. Schema matching is guaranteed" [2]。HN 上最短的反驳是 "it can still emit a completely wrong valid value"，以及 "An approve for an unauthorized action still meets the schema guarantee" [10]。

为什么这件事要紧：为格式错误建的修复与重试代码拦的是一个不再发生的失败，真正的错答会从中间穿过去。

| 症状 | 新接口下 | 一行验证 | 处置 |
| --- | --- | --- | --- |
| 输出非法 JSON / 字段缺失 | 不再发生，schema 匹配被保证 | 同批 state 跑 100 次统计解析失败数 | 删掉 JSON 修复与类型兜底函数 |
| 选了合法选项但选错 | 仍发生，且变成主要失败模式 | 人工标 50 例，比 schema 通过率与语义正确率的差 | 重试无效，只能靠评测集与阈值 |
| 高 confidence 却答错 | 仍发生，官方只承诺成组校准 | confidence 按 0.1 分箱统计各档实际正确率 | 低置信路由；高置信错例单独建集 |
| 单题校准但组合决策错 | 仍发生，官方不保证组合后的校准 | 同一 composite score 下的错例是否成簇 | 组合逻辑搬回代码，加领域断言 |
| 阈值定错静默放行 | 仍发生，阈值标定责任在开发者 | 自有数据扫阈值画 precision/recall 折衷 | 阈值常量集中一处，可评审 |
| 后一问依赖前一答 | 不会自动发生，同请求内问题互相独立 | 检查有无此类写法 | 拆成两个请求，延迟与成本相乘 |
| 长 state 挤掉问题 | 仍发生，共享约 32k token 预算 | 记录 usage.input_tokens，逼近预算报警 | 裁 state 或拆请求 |
| 422 请求体不合法 | 仍发生，且不在默认重试集内 [19] | 发一个畸形 question，确认没被静默重试 | 当配置错误处理，不进重试队列 |

Python SDK 默认 RetryPolicy 是 max_retries=2、退避 0.5–5 s、jitter 0.25，可重试码为 408/429/5xx；timeout=30.0 是整条重试链的总预算，不是单次尝试 [19]。

## 四、confidence 校准：目前只能写「待核实」

官方文档说 confidence 是「从概率分布算出来的一个统计量」，完整 probabilities 随响应返回，具体公式未公布，并鼓励自己定义度量 [7]。Noul 答案不带 confidence [6]。

官方声称的目标是成组的：概率 0.2 的结果应约 20% 发生，并明确 "These rates describe groups of predictions, not a guarantee about any single answer" [14]；官方文档另一处也写着「不保证单条答案正确」[15]。

官方发布的不是校准实验，是自一致性实验：14 个 Noul 问同一份保险理赔、重复 15 次，报告平均每题概率标准差 0.0102（B），并自我限定 "This does not make the model deterministic or prove automatic decisions are correct." [9]

**ECE、可靠性图、分箱校准曲线、样本量与抽样方式、分布外覆盖，这些都没有。** 在发布博客 [2]、evals 站点 [5]、confidence 文档 [7]、ML primer [14]、API 文档 [8]、两份 cookbook [9][17]、置信路由 pattern [16] 中逐页找过，未见任何一项。这是抽样所见为无，不是绝对断言，但足以说明：**「成组校准」目前只有厂商自述，无第三方独立验证。**

要自己拿到结论，最小验证六步：对 N ≥ 200 个不同 state 各跑一次，记下 choice 与 confidence；按 0.1 分 10 档统计每档实际正确率（标注成本要算）；画可靠性图看是否贴对角线；构造分布外 state 看 confidence 是否整体下移；扫阈值画 precision/recall 折衷；把未公开公式与抽样方式记为永久边界。

## 五、类型化 workflow 对比 CoT 的适用边界

| 判断条件 | 偏向类型化 | 偏向 CoT |
| --- | --- | --- |
| 输出空间 | 可枚举（分类 / 排序 / 打分 / 是非） | 开放（代码、长文、创意） |
| 单步映射 | 单步原子判断 | 多步依赖，需要中间搜索 |
| 错误成本 | 错一次代价高，需要知道模型有多不确定 | 错误可被自动验证 |
| 延迟预算 | 数百毫秒，要进请求路径 | 秒到分钟可接受 |
| 可观测性 | 结构化日志、可解释分支 | 思维链本身作为审计材料 |

官方自己划了线 [6]："Ask for a judgment a knowledgeable person makes in a second given the right context." 需要延展推理的，就该拆成原子问题再用代码组合。明确不适用：生成代码或长文、单请求内存在真实数据依赖、先探索再决定、多模态输入（官方称目前不含图像 [2]）、开源权重或本地部署。官方也自定位为非 agent："not agents. It does not generate code or choose its own next action." [18]

## 六、落地检查项

| 项 | 变化 | 一行验证 |
| --- | --- | --- |
| 校验层 | 从「拦格式」降为「防回归」，主防线移到语义断言 | 观察 schema 校验器还响不响 |
| 重试 | 解析失败不再可重试，只剩 408/429/5xx 与网络/超时错误 | CI 注入 429 看退避是否生效 |
| 超时 | 重试链总预算，非单次 | 打印一条重试链耗时核对 SLA |
| 可观测性 | 没有 token 流，输出 token 免费 | 看 usage 是否还带 output_tokens |
| 控制流 | 权重与阈值回到代码 | 阈值抽到一个文件做 code review |
| 降级 | 低置信才升级到 CoT 或人工 | 造低置信样例看降级是否真被触发 |
| 合规 | 判断回传单一厂商的美国托管 API，无本地路径 | 先列「哪些字段不能出网」再定接入范围 |

## 证据级别与来源

证据级别定义见开头。以下 [n] 对应来源，链接均已实际打开。中文圈两处可举证的转述失真（把 "gives up string generation" 译成「不擅长」[20]，把 System One 写成「慢思考」[21]）只作对照、不单独引用。

- [1] 官网首页 https://typesafe.ai
- [2] 官方发布博客（2026-09-15）https://typesafe.ai/blog/introducing-system-one-models-and-jev
- [3] 官方 X 公告帖 https://x.com/CompleteSkeptic/status/2099925682726002904
- [4] 官方新闻稿（Business Wire）https://www.businesswire.com/news/home/20260915525333/en/TypeSafe-AI-Emerges-From-Stealth-With-%2440M-in-Funding-With-New-Model-for-Composable-AI
- [5] 官方 workflow evals https://evals.typesafe.ai
- [6] 官方文档 primitives https://docs.typesafe.ai/primitives.md
- [7] 官方文档 confidence https://docs.typesafe.ai/confidence.md
- [8] 官方文档 API（错误模型）https://docs.typesafe.ai/api.md
- [9] 官方 cookbook 自一致性 https://docs.typesafe.ai/cookbooks/consistency_noul_cookbook.md
- [10] HN 讨论帖 https://news.ycombinator.com/item?id=49717558
- [11] 官方 LLM adapter 仓库 https://github.com/typesafe-ai/system-one-adapter-python
- [12] 榜单页（官方博客所引）https://llm-benchmarks.diegoromero.es
- [13] Anthropic Claude Fable 5.1 定价 https://www.anthropic.com/claude-fable-and-mythos-5-1
- [14] 官方文档 ML primer（RLCD 与校准）https://docs.typesafe.ai/introduction/machine-learning-primer.md
- [15] 官方文档 System One（定位与差异）https://docs.typesafe.ai/concepts/system-one.md
- [16] 官方文档置信路由 pattern https://docs.typesafe.ai/patterns/confidence-routing.md
- [17] 官方 cookbook 护栏 https://docs.typesafe.ai/cookbooks/llm_guardrails.md
- [18] 官方文档「如何用 System One 构建」https://docs.typesafe.ai/concepts/how-to-build-with-system-one.md
- [19] 官方文档 Python SDK Retries https://docs.typesafe.ai/sdk/python/api/retries.md
- [20] CSDN 转述（译作「不擅长」）https://blog.csdn.net/techforward/article/details/165572264
- [21] 掘金转述（写作「慢思考」）https://juejin.cn/post/7685648064544342062
