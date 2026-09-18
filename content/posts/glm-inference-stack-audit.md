---
title: "GLM 自建推理栈：把「3× 吞吐、10 万国产卡、两周上线」拆开核一遍"
date: 2026-09-18T12:45:00+08:00
tags: ["推理基础设施", "GLM", "国产算力", "性能工程"]
author: "繁星"
---

智谱 2026-09-17 在 z.ai 官方博客发了《Toward Recursive Self-Improvement: How GLM Built Its Own Inference Infrastructure》，讲 GLM-5.3-Flash 怎么在十万张国产加速器上自建推理栈。中文圈当天就转开了，但对不齐的不是媒体、是口径：官方中文博客稿写「端到端服务性能约 3 倍」「部分场景的性能差距超过 20%」，唐杰 X 写「3.2× 端到端吞吐」「over 30% to under 1%」。两家转载又都同时用了两套数——量子位导语跟唐杰用 3.2×、正文转官方中文博客稿用 3×；爱范儿同一篇里 3.2× 与约 3 倍并存。同一件事，两套数。

这篇不重述那篇博客，只做一件事：把每个头条数字的口径拆开，标出基线、分母、时间窗，再标出它属于哪一级证据。

## 一、口径解耦表

| 数字 | 一手出处 | 量的是什么 | 基线 / 统计范围 | 时间窗 | 证据级别 |
|---|---|---|---|---|---|
| 3×（唐杰 X 版 3.2×） | 官方博客、中文官方文档；唐杰 X | 端到端服务吞吐 | 「同一硬件上的初始基线」，无数值、无指标定义、无聚合方式 | 未说明 | B，且两版冲突 |
| >100,000 张国产 AI 加速器 | 官方博客 | 部署规模（不是可支持规模） | 自身 | — | B（卡型未点名） |
| 不到两周 | 官方博客、官方 X、唐杰 X | 首次跑通 → 承载全部生产流量 | 两端都在厂商内部 | 无起止日，只有相对区间 | B |
| >62 万亿 token / 6 天 | 官方博客 | token 处理量 | OpenCode + OpenRouter 两平台合计 | 6 天 | B |
| >20 万亿 token / 6 天 | OpenRouter 官方 X | token 处理量 | 仅 OpenRouter 单平台 | 6 天 | B |
| per-token cost comparable to mainstream NVIDIA GPUs | 官方博客 | 单 token 成本 | 「主流 NVIDIA GPU」，卡型未点名；无单位、无折旧、无电费、无利用率口径 | 未说明 | C |

62T 与 20T 是否含 prefill + decode、是否含缓存命中、是否含 reasoning token，官方均未说明。

## 二、62T 和 20T 拼不成一个自洽账本

62T 的统计范围是两个平台，20T 的统计范围是一个平台：统计范围不同，不构成倍数关系。20T 落在 62T 的口径之内，但两边的计数方法未必一致、窗口未必对齐，所以不能相减得出「OpenCode 42T」，也不能相除。

更麻烦的是第三个数字：OpenCode 自己的数据页给 ox-alpha 记的是 56T token，窗口约八周（含发布后），56 + 20 = 76 ≠ 62。三个数字来自三个口径，拼不成自洽账本。为什么重要——任何「OpenCode 到底贡献了多少」的推论，都建立在一个并不存在的账本上。

## 三、dense feedback：方法论是真的，产物只核到一处

智谱把方法论叫「dense feedback」，并且明说不是「喂给 agent 越多日志越好」，而是三条特征：反馈要足够局部、要低成本及时获得、要支持客观验证。「足够局部」落在算子层和「并行策略 → kernel 实现」的映射层上——官方说建了这张映射表，但表本身没公开。

能外部核到的一手产物在 KDA 的 CP 路径上：FLA 仓库 PR #1180 新增了 `tests/context_parallel/test_cp_kda.py` 的三个 CP 用例（sequence cut、single long sequence、state_v_first）。可证伪的判据也清楚：同一并行配置下，切分与非切分路径的数值误差是否落在容差内。

「低成本及时」有实例：KDA Decode kernel 的除法优化让 v1 缩短 9.6%，得到 v2。但「optimization skeletons（优化骨架库）」只有方向性描述——无仓库、无 schema、无条数。它只能算方法论声明，不能写成「已建成可复用的优化知识库」。

容易被揉在一起的还有一组：官方那三条是「特征」（local / cheap / verifiable），同一篇博客另给了三类「反馈」（correctness / system behavior / performance），不是同一组东西，别合并成「三原则」。

## 四、KDA 的 TF32：框架默认，与上游实际的合并形态是两回事

这条是全文落差最大的地方，得拆成两句来写。

智谱说，KDA kernel 的 Context Parallelism（CP）路径上误差会随序列变长而累积，修复方式是「为两个操作显式设置 `input_precision="tf32x3"`」。这不是他们配错了——Triton 官方文档写明 `tl.dot` 的 `input_precision` 在有 tensor core 的设备上默认就是 tf32，哪怕输入是 FP32。默认值来自框架，不是本栈配置。

而唯一合入上游的那份修复（FLA PR #1180，2026-08-27 合并，merge commit `81c0155`）实际形态是：新增环境变量 `FLA_INTRACARD_TF32X3`，默认 `0`，描述里直接写着 NVIDIA only；非 NVIDIA 平台回退 `ieee`；PR 自己的 Benchmark 段写的是 "Neutral — this is an opt-in accuracy path, not a performance change"。而智谱跑的是非 NVIDIA 国产卡。国产卡上究竟怎么触发 TF32，目前没有一手证据。这两句不能合并——合并成一句，读者会以为国产卡默认跑了 tf32x3。

顺带一个术语陷阱：KDA 全称在智谱全部一手材料里都没出现过。给了定义的是上游 FLA 的 `fla/ops/cp/README.md`——KDA (Kimi Delta Attention)。中文圈已经有人把它补成 "Kernel Density Attention"，无任何来源支持。另外智谱把这条路径叫 Context Parallelism (CP)，FLA 叫 KCP，是同一路径的两个名字。

## 五、GIL：源码逐条对得上，但「上游已修复」是错的

智谱的说法是：PyTorch 侧线程进 C++ 不会自动释放 GIL，而 DeepEP 的 intranode 路径没释放，KV Transfer 就被卡住。这条我拿 v1.2.1 源码核过：

- 全文 `pybind11::gil_scoped_release` 只出现 1 次，位于 `internode_dispatch` 内（L666）；
- 它上方 L663–L665 的注释逐字说明，释放就是为了让 KV transfer 这类 Python 代码不被 GIL 卡住；
- `intranode_dispatch`（L306 起）与 `intranode_combine`（L543 起）内没有任何 GIL 释放，而 `intranode_dispatch` 的 L449–L469 确实在 CPU 侧 busy-wait 轮询 GPU 计数器。

同一个库里，跨节点路径释放了，节点内路径没有。这是可证伪的代码事实。

百分比要谨慎读。官方给的分母是「同一 workload 下单独 Prefill 基准」，指标是 Prefill + KV Transfer 相对它的性能差距：验收线 ≤5%，实测某些场景超过 20%，修复后 <1%。原文限定 "in some scenarios"，不是全局平均。唐杰 X 把同一件事写成 "transfer overhead fell from over 30% to under 1%"，措辞换了，数字也换了。证据级别：C（单一来源自述，无第三方复现）。

不能写错的一处：intranode 的 GIL 修复没有合进 DeepEP 上游。PR #555「release gil in intranode::dispatch」，作者 reyoung，2026-01-04 提出，至今 state=open、merged=false。上一轮同类修复 PR #142（2025-05-08 合并）只覆盖 inter-node。所以「GIL 问题已上游修复」是事实性错误——上游视角下，这是一个今年一月就有人提、至今没合的开放问题。

## 六、1.71× 是 kernel 微基准，端到端收益官方没给

Infra Agent 发现 KDA Decode kernel 的原始实现沿 V 维度分块，同一份 FP32 normalization 和 gating 计算被重复了四次。修法是把这些分块合并进单个 thread block，中间结果留在寄存器里，用一次 warp-level reduction 替代重复计算。

1.71× 的基准是自家中间版本 v2，不是 v0、不是社区通用 kernel、不是端到端。演进链条是：引入 ReplaySSM 让 v0→v1 变慢，除法优化让 v1 缩短 9.6% 得到 v2，消除重复 norm/gating 才拿到 1.71× over v2。Figure 4 的标题本身写的就是 "a representative KDA Decode kernel"，这是算子级微基准。证据级别：C（单一来源自述，无第三方复现）。

端到端被什么掩盖，官方自己说了：「给一个计算 kernel 更多资源可能缩短它自己的执行时间，却让 KV Transfer kernel 拿到更少资源，最终拖慢整个流水线」，「在微基准里成立的优化未必转化为端到端收益」。但 1.71× 对应到端到端多少，没有披露。不能自行相乘，不能外推。

## 七、核不了的部分

拆完之后，原文之外仍无法核验的至少还有：3× 的数值与统计窗口、10 万卡的实际卡型与互联、「两周」的精确起止日、62T 与 20T 的 token 计数方法、「成本与 NVIDIA 相当」的成本定义与对标卡型、tf32x3 的实际代价（上游 PR 自述性能中性）、长上下文误差累积的量化曲线、「优化骨架库」本体、Infra Agent 的实际自主程度，以及 Figure 1/3/4 图内数值——图是图片，只读到图注。

证据级别约定：A = 一手材料（含上游仓库/PR）且已源码级核验；B = 官方一手但外部不可复现；C = 单一来源自述；D = 推断。本文的 A 级事实集中在两处产物上——FLA PR #1180 的合并形态，与 DeepEP v1.2.1 的 GIL 释放位置；这两条我都自己打开源码和官方文档复核过。其余按 B、C 标注，不预判结论。

## 参考链接（均实测可打开）

- 官方博客原文：https://z.ai/blog/glm-built-its-inference-infrastructure
- 官方中文文档「跑在国产芯片上」：https://docs.bigmodel.cn/cn/guide/models/vlm/glm-5.3-flash
- 智谱官网研究页：https://www.zhipuai.cn/zh/research/163
- 20T 的唯一一手出处（OpenRouter 官方 X）：https://x.com/OpenRouter/status/2092616758612172994
- 唐杰 X 长文（3.2× / 30%）：https://x.com/jietang/status/2100482019088060470
- FLA PR #1180（唯一「已合并上游」的硬产物）：https://github.com/fla-org/flash-linear-attention/pull/1180
- FLA `ENVs.md`（`FLA_INTRACARD_TF32X3` 默认 0 / NVIDIA only）：https://github.com/fla-org/flash-linear-attention/blob/main/ENVs.md
- FLA `fla/ops/cp/README.md`（KCP 与 KDA 原文定义）：https://github.com/fla-org/flash-linear-attention/blob/main/fla/ops/cp/README.md
- DeepEP v1.2.1 `csrc/deep_ep.cpp`（GIL 释放位置）：https://github.com/deepseek-ai/DeepEP/blob/v1.2.1/csrc/deep_ep.cpp
- DeepEP PR #555（intranode GIL，仍 open）：https://github.com/deepseek-ai/DeepEP/pull/555
- DeepEP PR #142（internode GIL，2025-05-08 已合并）：https://github.com/deepseek-ai/DeepEP/pull/142
- Triton `triton.language.dot`（`input_precision` 默认 tf32）：https://triton-lang.org/main/python-api/generated/triton.language.dot.html
- OpenCode Data 页（56T / 93% cache ratio / $475K）：https://opencode.ai/data/unknown/ox-alpha
- 量子位转载（导语跟唐杰用 3.2×、正文转官方中文博客稿用 3×）：https://www.qbitai.com/2026/09/491357.html
- 爱范儿报道（同一篇里 3.2× 与约 3 倍并存）：https://www.ifanr.com/1680560
- 「Kernel Density Attention」这一无来源全称的出处（繁体稿）：https://www.aiposthub.com/z-ai-glm-built-own-inference-infra-agent-rsi-deep-dive/
