---
title: "从论文到源码：沿一次生成理解 DSpark"
date: "2026-11-03"
draft: false
description: "以 DeepSpec 的 Qwen3 实现为主线，追踪上下文特征、草稿生成、概率验证与两套 KV 缓存怎样衔接。"
categories: ["论文心得"]
tags: ["LLM 推理", "投机解码", "源码分析", "DSpark"]
---

[上一篇](/papers/dspark-reading-notes/)整理了我对 DSpark 论文的初步理解：计算较重的部分尽量并行，用轻量的顺序模块补上块内依赖，再考虑哪些候选值得送去验证。继续读源码时，我想弄清楚的是：**这些模块怎样围绕同一段回答工作，接受或拒绝之后，又怎样开始下一轮？**

这次以 DeepSeek 官方 [DeepSpec](https://github.com/deepseek-ai/DeepSpec/tree/005e03b81cec38b7da6399833d609ee89a2587f2) 的 Qwen3 分支为主，分析其中的单请求离线生成流程。vLLM、SGLang 的集成设计暂时不在这里展开。

> 本文记录的是固定版本的源码行为和我目前的理解，没有运行具体 checkpoint，也没有测量加速效果。下面的 `W=3`、字母 token 和概率表都是手工构造的解释例子，不是模型实测输出。

## 1. 先找到生成循环，弄清楚谁在推进回答

从 `eval.py` 出发，`eval.py` 根据 draft 配置中的架构名，选择并创建 Qwen3 的 DSpark 评测器 `Qwen3DSparkEvaluator`。评测器从数据集中准备一条问题，通过 `Qwen3DSparkEvaluator.generate_one_sample()` 将这条输入和 DSpark 方法交给供不同 evaluator 复用的通用生成循环函数 `generate_decoding_sample()`，由后者运行完整的生成循环。

把数据集处理、分布式启动和指标打印暂时放到旁边，生成主线是：

```text
Target prefill：处理 prompt，选出第一个 token
    ↓
_init_context：保存初始 Target hidden，创建空的 Draft KV
    ↓
_propose：Draft 根据上下文和 anchor 提出候选
    ↓
verify_draft_tokens：Target 验证候选
    ↓
生成循环：写输出、推进位置、裁剪 Target KV
    ↓
_update：保存下一轮需要的新增 Target hidden
    ↓
下一次 _propose；或遇到停止条件后返回
```

这里有一个职责划分：evaluator 提供 `_propose`、`_update` 等回调，通用循环负责在合适的时候调用它们。**验证函数返回结果，并不意味着输出、位置和缓存都已经更新。** Target 会先计算整段候选，因此缓存中可能暂时包含被拒绝的部分。验证完成后，生成循环再写入确定的输出，并裁掉未被采用的缓存后缀，为下一轮生成做好准备。

当前循环明确断言 batch size 为 1。输入来自数据集文本，经对话模板与 tokenizer 编码后成为 `input_ids`；返回的是 token IDs 和统计。这是离线评测链路，阅读时不需要先理解 HTTP 请求或多请求调度。[入口与回调绑定][entry]、[生成循环][loop]。

### Prefill 之后，第一个 token 已选出，但还没有自己的 KV

设 prompt 长度为 `N`，用 `P` 代表整个 prompt。Target 先处理 `P`，保存 prompt KV，并用最后位置的 logits 选择首个回答 token `A`。

此时的状态是：

| 对象 | 内容 |
| --- | --- |
| 已确定的回答，仅生成部分 | A |
| `output_ids` 中的有效内容 | P + A |
| Target KV | 覆盖整个 prompt P |
| 待交给 Draft 的 Target hidden | P 对应的特征 |
| Draft KV | 已创建，但为空 |
| 本轮 anchor | A，位于 `output_ids[:, start]`，其中 `start=N` |

**A 已经被选中，但还没有在自己的位置作为 Target 输入执行过，因此还没有自身的 Target KV 和 hidden。** 它将作为本轮 anchor，和候选一起进入后续验证。

这个区别会贯穿整条链路：输出 token 的确定时间，与它的缓存计算时间并不相同。[Prefill 与上下文初始化][prefill]、[_init_context][init]。

## 2. Target 交给 Draft 的，是特征，不是它的 KV

Draft 接收的 `target_hidden_states` 来自 Target 的指定层。辅助函数 `extract_context_feature` 按 `target_layer_ids` 选取 hidden，然后沿特征维度拼接。

HuggingFace 的 hidden states 元组开头还有 embedding 输出，因此零起算的 decoder 层 `i` 对应 `hidden_states[i+1]`；源码也支持用 `-1` 选择 embedding 输出。假设选取两层，每层为 `[1,S,H]`，拼接后就是 `[1,S,2H]`：token 数量不变，每个位置携带的特征变宽了。

| 元组下标 | 保存的数据 |
|---|---|
| `hidden_states[0]` | Embedding 后的初始向量 |
| `hidden_states[1]` | 第0个 decoder 层处理后的向量 |
| `hidden_states[2]` | 第1个 decoder 层处理后的向量 |
| …… | …… |

Draft 再通过 `fc` 和归一化，将这些特征投影到自己的上下文宽度。实际取哪些层、维度多大，需要读具体 checkpoint 配置，不能从示意形状推断。[特征提取][features]、[Backbone 投影][backbone]。

这里还有一个容易忽略的检查：评测代码禁止选择 Target 的最后一个 decoder 层。断言给出的理由是，HuggingFace 对应项保存最终归一化后的 hidden，而训练缓存路径保存原始 decoder 层输出。即使形状一样，特征含义也可能不同。关于作者最初为什么选这套写法，是另一个需要额外证据的问题。[层检查][layer-check]。

进入 Draft 后，各个 Draft 层用自己的投影生成上下文 K/V。**它利用的是 Target 的 hidden，没有直接把 Target 的 KV 搬过来。** 因此需要区分三件东西：

- Target hidden：交给 Draft 的上下文特征。
- Target KV：Target 自己保存的注意力缓存。
- Draft KV：Draft 根据上下文特征、用自身参数构造的缓存。

## 3. 从 anchor 和 mask，到一块候选

### 三行输入，为什么能预测三个候选

设草稿块大小 `W=3`，当前 anchor 为 `J`。`_propose` 创建三个 mask 占位，再把首项替换为 anchor：

```text
Draft 输入：J / mask / mask
```

这三个位置先经过 embedding 和并行 Backbone，得到 `block_hidden`。它仍然是特征，还不是候选 token。

在这份实现里，首行输入 J，但首行输出用于预测 J 后面的第一个候选。位置关系可以写成：

| Draft 输入行 | 用于预测 |
| --- | --- |
| 第0行：J | 候选 K |
| 第1行：mask | 候选 L |
| 第2行：mask | 候选 M |

所以三行可以产生三个候选，送给 Target 的完整验证输入则是 `J/K/L/M`，共四行。不能因为首行输入了 anchor，就直接把这行预测丢掉。[输入块构造][propose]、[候选封装][proposal]。

这也是阅读不同实现时需要单独核对的约定：草稿查询行数、候选数、验证输入数，并不是同一个量。

### 相同的 mask，怎样产生不同位置的特征

Mask 是输入占位 token，没有真实候选的身份。同一个 mask ID 查表得到相同的初始 embedding，但这些位置还接收上下文、anchor 和位置信息。

在当前 Draft attention 中：

| 张量 | 来源 |
| --- | --- |
| Q | 预测块当前层的 hidden |
| 新增上下文 K/V | 投影后的新增 Target hidden |
| 临时块 K/V | 预测块当前层的 hidden |
| 历史 K/V | 该 Draft 层已经缓存的上下文 |

预测块的 Q 读取上下文与块内 K/V；不同位置经 RoPE 后，与上下文的匹配关系可以不同，因此并不需要等到 Markov Head 才能区分位置。这不意味着任何输入下输出向量都必然不同。

这条推理路径传入 `attention_mask=None`、`is_causal=False`。块内位置可以共同计算，但此时真正的 K/L/M 还没被采样出来。Mask token 也不等于“屏蔽这个位置”的 attention mask。[Attention][attention]、[Draft forward][draft-forward]。

### 为什么 Draft KV 先增长，再 crop

假设 Draft 已缓存到 C，上一轮新确认了 E/F，对应的 Target hidden 正等待补入 Draft，当前 anchor 为 J：

```text
调用前：历史缓存 P A B C；新增 hidden E F；输入 J/mask/mask
计算时：P A B C | E F | J mask mask
crop后：P A B C E F
返回值：J/mask/mask 三行对应的 block_hidden
```

Draft 每层先追加新增上下文和临时块的 K/V，完成计算后调用 `past_key_values_draft.crop(start)`，只保留 anchor 之前的上下文。

J 虽然已经确定，但这次在 Draft 中属于临时预测块。等 Target 处理 J，下一轮才用 J 的 Target hidden 构造可复用的 Draft 上下文。裁剪缓存不会抹掉已经计算出来、即将送给 Head 的 `block_hidden`。

这里能确认的是缓存的逻辑前缀；仅凭 `crop` 调用，还不能断言物理显存立即归还或实现了零拷贝回滚。[缓存更新与 crop][draft-forward]。

### Markov Head：先改分数，再选 token

Backbone 输出经过 LM head，得到各位置的基础 logits，形状为 `[1,W,V]`。其中 V 是词表大小，logits 是词表分数，不是正确率。

下面以源码支持的 Vanilla Markov 分支为例。前一个 token 经 `markov_w1` embedding，再经 `markov_w2` 投影，形成对整个词表的偏置，加入当前基础 logits：

```text
位置1基础 logits + J 的偏置 → 采样 K
位置2基础 logits + K 的偏置 → 采样 L
位置3基础 logits + L 的偏置 → 采样 M
```

顺序依赖就在这个循环里：选出 K 后，下一个位置用的是实际选出的 K。若采用随机采样，K 未必是 argmax；不能另做一次 argmax，再拿那个结果作为后续条件。

计算较重的 Backbone 已经并行完成，顺序阶段主要做偏置与采样。这对应了第一篇里“把顺序依赖放在便宜的地方”的理解，但具体延迟开销仍需测量。

接口同时返回候选和修正后的 `draft_logits`。后者按相同温度转换成 `draft_probs`，后面验证要用它知道：**这份草稿当时究竟按照什么分布提出候选。**

源码也支持 gated、RNN，以及禁用 Markov 的配置；未指定 checkpoint 时，不能把 Vanilla 当作所有模型必经的分支。[Vanilla Markov][markov]、[采样接口][sampling-interface]。

## 4. Confidence 决定提交多长，不能代替 Target 验证

候选生成后，程序还可能计算 `confidence_logits`。根据配置，输入包含当前位置 hidden，以及前驱 token 的 Markov embedding。对于 J 起步、候选 K/L/M 的例子，前驱依次是 J/K/L。

这里有三个不同的数：

| 对象 | 表示什么 |
| --- | --- |
| draft_probs | 草稿实际使用的词表采样分布 |
| sigmoid(confidence_logits) | Head 对各位置接受可能性的预测 |
| 验证时的接受概率 | Target 与 Draft 对实际候选的概率共同决定的数 |

当前 `_confident_prefix_length` 的规则很直接：逐项检查 sigmoid 分数，找到第一个低于阈值的位置，保留它之前的前缀。

例如分数为 `[0.9,0.4,0.8]`、阈值为 `0.6`，只提交 K，验证输入变为 J/K。不能跳过 L，再把 M 接上。阈值为 0 时不截短；即使保留零个候选，仍可以处理 anchor 并产生下一 token。[Confidence 与截断][confidence]。

这段代码也明确了成本发生的先后：**Backbone 和整块采样已经完成，confidence 截断主要缩短的是 Target 验证，不会撤销前面已经付出的 Draft 计算。**

另一条诊断路径会把 sigmoid 分数累乘，评估整个前缀的存活概率。它和这里逐位置比较阈值的选择规则需要分开看。

论文中的硬件感知预算还涉及系统负载与引擎代价。本文追到的离线阈值函数，不能直接代表那套完整调度流程；STS 和服务端预算选择留待后续继续分析。[前缀概率诊断][recorder]。

## 5. Target 验证的是概率，而不是候选是否“通顺”

提交三个候选时，Target 一次前向消费 J/K/L/M。自回归模型的输出预测下一个位置，因此分布对应关系是：

| Target 输入行 | 输出分布的用途 |
| --- | --- |
| J | 验证 K |
| K | 验证 L |
| L | 验证 M |
| M | 全部接受后采样 bonus token |

代码让 `verify_input_ids[:,1:]` 与 `target_probs[:,:-1,:]` 对齐，分别取出实际候选在 Target 和 Draft 分布中的概率 p、q，计算：

$$
\alpha=\min\left(1,\frac{p}{q}\right).
$$

再与随机数比较，得到逐位置接受结果。源码对 q 做了数值下限保护。普通随机采样路径不能简写成“比较两边 argmax 是否一致”。[概率验证][verify]。

接受的是连续前缀。假设逐位置结果为 `[1,0,1]`，`cumprod` 后为 `[1,0,0]`，只能保留 K。M 的预测以包含 L 的前文为条件，L 被拒绝后，不能跨过它保留 M。

### 拒绝后，为什么不是直接按 Target 分布再抽一次

标准拒绝采样要考虑前面已经通过接受规则留下的概率质量。第一个拒绝位置从正差额归一化后的分布重采样，而全部接受时，从最后一个 Target 分布采样额外的 bonus token。

用同一前文下的小词表说明：

| token | Target p | Draft q | 正差额 max(p−q,0) | 归一化修正分布 |
| --- | ---: | ---: | ---: | ---: |
| L | 0.3 | 0.6 | 0 | 0 |
| N | 0.4 | 0.3 | 0.1 | 1/3 |
| P | 0.3 | 0.1 | 0.2 | 2/3 |

按每100次的期望数量看，Draft 提议60/30/10次，接受规则平均留下30/30/10次；剩余30次需要补0/10/20次，才能得到 Target 的30/40/30。这就是修正分布的作用，实际随机次数会有波动。

如果无视已接受的贡献，拒绝后直接再按 Target 分布抽，组合起来就不是上面这套分配了。代码里的 `sample_residual` 执行正差额、归一化与采样，并对差额总和极小的情况作数值保护。[修正采样][residual]。

这解释了算法意图，不构成对当前代码全部数值和终止组合的分布实测证明。

## 6. 验证之后，把回答和两套缓存接回下一轮

假设 anchor 为 J，候选是 K/L/M，仅接受 K，拒绝 L，并在 L 的位置修正得到 N。

回答尾部变为：

```text
原候选：J K L M
有效尾部：J K N
```

N 替代 L，L/M 不保留；N 也还没有在自身位置经过 Target 前向。

接下来，外层生成循环分别做这些事：

1. 将 anchor 和接受前缀 J/K 写回原位置，J 没有被重复追加。
2. 将新采样的 N 写在后面。
3. `start` 前进“接受候选数加一”，现在指向 N。
4. 将 Target KV 裁剪到新 start，只保留 N 之前的有效计算。
5. `_update` 保存 J/K 的 Target hidden，供下一轮 Draft 补上下文。

**这轮新增的输出是 K/N；保留的验证输入是 J/K。两者数量都为二，但内容不同。** 这也是只看一个“接受长度”容易混淆的地方。

`_update` 本身不写输出、不推进 start，也不直接追加 Draft KV。它只是更新待补特征，下一次 Draft forward 才真正使用它们。[正常推进][commit]、[更新 hidden][update]。

### 用三轮回放检查状态是否接上

下面仍是假设 W=3、预算充足，confidence 不截断。P 代表整个 prompt，字母只是位置标记。

| 时刻 | 已确定的回答，省略 P | anchor | Target KV | Draft KV | 待补 Target hidden |
| --- | --- | --- | --- | --- | --- |
| Prefill 后 | A | A | P | 空 | P |
| 第一轮后 | A B C E | E | P A B C | P | A B C |
| 第二轮后 | A B C E F G H I | I | P A B C E F G H | P A B C | E F G H |
| 第三轮结束 | A B C E F G H I J EOS | 无 | P A B C E F G H I J | P A B C E F G H | 无需下一轮更新 |

第一轮由 A 起步，提议 B/C/D，接受 B/C，拒绝 D，修正得到 E。

第二轮先把 A/B/C 的 Target hidden 补进 Draft 上下文，再由 E 起步提议 F/G/H。全部接受，额外得到 I。

第三轮先补 E/F/G/H 的特征，由 I 起步提议 J/EOS/L，接受前缀包含 J/EOS，终止到 EOS，不采用额外 next_token，也不再执行正常 update。

表里两套 KV 没有始终等长，但这不一定表示状态缺失：Draft 缓存与待补 hidden 合起来，衔接着下一轮需要的有效上下文。阅读时要同时看缓存边界和待补特征，不能只盯着一个长度。

### EOS 和长度收尾另有分支

接受前缀里出现 EOS 时，循环保留到 EOS 就退出，不再使用验证结果里的额外 next_token。若 EOS 是修正或 bonus token，则先完成正常推进，再检查退出。首个 anchor 本身就是 EOS 时，prefill 后直接返回，不进入验证循环。

长度上限也有一个实现细节：这份循环可能整块提出并验证候选，再按 `max_length` 裁剪最终输出。因此“计算过多少位置”“每轮接受统计”和“最终返回多少 token”不能直接混用。[终止分支与收尾][commit]。

## 7. 这份源码解释了什么，接下来还要验证什么

读完这条链路，论文中的模块开始对应到具体的数据：Target hidden 给 Draft 提供上下文，Backbone 产生并行特征，Markov 根据实际前驱修正分布，confidence 选择提交前缀，Target 验证后再由循环整理输出与状态。

对我后续接入 gLLM 更直接的启发，是需要把几个边界写清：谁拥有有效上下文，谁推进位置，谁保留新增 hidden，以及新 anchor 何时真正拥有缓存。公开离线循环中的这些职责，进入服务引擎后仍然需要明确的调用方。

这篇还没有给出具体 checkpoint 的运行结果，也没有验证采样分布、数值等价或论文性能。后续若补实验，需要同时记录 Target/Draft revision、配置、输入、环境和原始输出。

vLLM、SGLang 的请求调度、分页缓存和异步执行目前还没有读到同样细的程度。下一篇结合 gLLM 的集成工作，再追这些职责怎样落入现有引擎，并用实际测试说明实现到了哪一步。

## 源码入口

本文所有源码引用固定到同一 commit。需要回到代码复查时，可以从这些入口开始：

| 文件 | 主要职责 |
| --- | --- |
| [eval.py][entry-file] | 选择 evaluator、启动评测 |
| [eval/dspark/evaluator.py][evaluator-file] | DSpark 的上下文初始化、propose 与 update |
| [eval/base_evaluator.py][loop-file] | 通用生成循环、验证、终止与状态推进 |
| [eval/dspark/draft_ops.py][ops-file] | Draft forward、confidence 截断与 proposal 封装 |
| [modeling/dspark/qwen3/modeling.py][model-file] | Backbone、Attention 和 Head 接口 |
| [modeling/dspark/markov_head.py][head-file] | Markov 变体与顺序采样 |
| [utils/sampling.py][sampling-file] | 概率转换与修正采样 |

[entry]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/dspark/evaluator.py#L162
[loop]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/base_evaluator.py#L308
[prefill]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/base_evaluator.py#L333
[init]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/dspark/evaluator.py#L85
[features]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/modeling/dspark/common.py#L52
[backbone]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/modeling/dspark/qwen3/modeling.py#L361
[layer-check]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/base_evaluator.py#L100
[propose]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/dspark/evaluator.py#L99
[proposal]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/dspark/draft_ops.py#L96
[attention]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/modeling/dspark/qwen3/modeling.py#L87
[draft-forward]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/dspark/draft_ops.py#L22
[markov]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/modeling/dspark/markov_head.py#L8
[sampling-interface]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/modeling/dspark/qwen3/modeling.py#L309
[confidence]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/dspark/draft_ops.py#L57
[recorder]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/dspark/confidence_head.py#L345
[verify]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/base_evaluator.py#L186
[residual]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/utils/sampling.py#L34
[commit]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/base_evaluator.py#L407
[update]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/dspark/evaluator.py#L134
[entry-file]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/eval.py
[evaluator-file]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/dspark/evaluator.py
[loop-file]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/base_evaluator.py
[ops-file]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/eval/dspark/draft_ops.py
[model-file]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/modeling/dspark/qwen3/modeling.py
[head-file]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/modeling/dspark/markov_head.py
[sampling-file]: https://github.com/deepseek-ai/DeepSpec/blob/005e03b81cec38b7da6399833d609ee89a2587f2/deepspec/utils/sampling.py
