# Task 2：Model Equality

官方题面：[`Model_Equality_Testing.pdf`](Model_Equality_Testing.pdf)（5 页，Google Docs 导出）。下面是题面原文的整理，以及我们的做法。标注：

- **【题面】**：PDF 里写的。
- **【已核实】**：2026-11-21 核对过的论文摘要。
- **【比赛用法】**：我们的判断，不是主办方给的解法。

主办：Adam Dziedzic、Franziska Boenisch。出题：Xun Wang、Bihe Zhao、Maitri Shah、Ivo Hoese、Aditya Kumar。

## 先做这个

1. 克隆 starter kit，不要自己写客户端：<https://huggingface.co/datasets/SprintML/model_equality>
2. Jülich 环境仍走三题共用仓库：`git clone https://huggingface.co/datasets/SprintML/hackathon`
3. **不要把赛前笔记里的 Hamming MMD 直接拿来交。** 那篇论文把水印、量化、微调都当成「模型变了」。这题把水印、系统提示、解码策略定义成同一个模型。照论文做，会把同一模型拆开，90% 的分数会掉。
4. 正式端点 `ep-01` 到 `ep-33`，每个全程最多 100 条回复。先在 `ref-01` 到 `ref-09` 上把 prompt 定下来，用 `score_practice.py` 在本地打分。题面没有写参考端点是否计入这 100 条，先看 `query_reference.py`，不要先烧正式预算。
5. 每 30 分钟只能交一次，失败也算冷却。公开榜和最终排名可以差很多。题面把 “DO NOT RELY ON THE PUBLIC SCORE” 写了三遍。
6. 同一 `model` 标签必须共享 `family` 标签。标签里不能有逗号。

## 什么叫同一个模型【题面】

33 个部署，`ep-01` 到 `ep-33`，各有一个 API。有的是同一模型的不同部署，有的是不同模型。

部署之间可以差在：解码水印、提供方加的系统提示、解码策略、量化配置、LoRA adapter。

两个端点是同一个模型，当且仅当 checkpoint、量化配置、LoRA adapter 都相同。

- 同一基座上，水印、系统提示、解码策略不同，身份不变。
- 基座、量化、LoRA 任一不同，就是不同模型。
- 只是同一家族，不算相等。

例子：同一个 OLMo 基座，量化相同、adapter 相同，一个有解码水印、系统提示也不同，仍是同一个模型。量化版本不同，或 LoRA 不同，就是另一个模型。

要交两件事：每个端点属于哪个家族（同一基座），以及每一对端点是不是同一个模型。

## 评分【题面】

对无序对打分。33 个端点有 528 对，每对只算一次。

```text
MCC = (TP·TN − FP·FN) / sqrt( (TP+FP)(TP+FN)(TN+FP)(TN+FN) )
MMC = max(0, MCC)
```

- TP：正确判成 same
- TN：正确判成 different
- FP：不同却判成 same
- FN：相同却判成 different

两个问题：

- Family，权重 10%：这一对是不是同一家族。所有 528 对都算。
- Model，权重 90%：这一对是不是同一个模型。只在同一家族内部的对上算。不同家族的对永远不是同一个模型，不进这项。

```text
score = 0.1 * MCC_family + 0.9 * MCC_model
```

题面同时写了 MMC = max(0, MCC)，组合公式里用的是两个 MCC。开幕幻灯片第 39 页写的是先截断再加权：`MCC_x = max(0, MCC)`，然后 `score = 0.1 × MCC_family + 0.9 × MCC_model`。以幻灯片为准。负的 MCC 先变成 0，再进加权。

只认出家族，最多 0.1。剩下的 0.9 是家族内部把量化版本和 LoRA 版本分开，同时不要把水印或系统提示拆开。漏掉的 same 和错报的 same 都扣分。满分 1，越高越好。

## API 和预算【题面】

每次查询向一个端点发一个 prompt，只返回生成文本。

- 每队每个端点全程最多 100 条回复。一次要多条，按条数扣。
- 每条最多 256 个生成 token，每个 prompt 最多 1024 个输入 token。
- 每队同时最多 4 个请求。
- `temperature` 和 `seed` 被忽略。system message 会被拒绝。
- 非法请求会拒，不扣预算。

33 × 100 = 3300 条。种子控不了，论文里「温度 1、固定种子」的协议在这里做不到。重复采样仍然有用，但分布是他们自己的解码策略，不是我们设的。

## 参考端点【题面】

`ref-01` 到 `ref-09` 是另一个小模型，伪装和陷阱与计分端点同类。答案在 `practice_answers.csv`。`score_practice.py` 可以本地反复打分。

查询用 `query_reference.py`：填 API key，prompt 写在 `reference_prompts.txt`，输出 `reference_answers.jsonl`。

起点文件是 `task_template.py` 和 `submission.py`。API key 保留双引号。把 `/PATH/FILE.csv` 换成真实结果路径。

## 提交【题面】

用 `submission.py`。CSV 每个端点一行，三列：`endpoint,family,model`。表头可选，行序无关。标签是任意文本，不能含逗号。

```text
endpoint,family,model
ep-01,A,A1
ep-02,B,B1
ep-03,A,A2
```

- 相同 `family`：认为是同一基座。
- 相同 `model`：认为是完全同一个模型，并且必须同一 `family`。
- 模板里的示例是每个端点各自一家、各自一个模型，也就是「全不同」。

为防穷举，每 30 分钟只能交一次，大约 24 小时 48 次。格式错误也进入冷却。

榜：<http://34.147.229.55/leaderboard_page>。只保留每队最高分。公开分只覆盖一部分 pair，赛后用没见过的部分排名。对着公开榜调，可能榜上很高、最终接近 0。

## 三篇引用里哪篇不能照做

【题面】列了三篇：

- [Model Equality Testing](https://arxiv.org/abs/2410.20247)（MET，ICLR 2025）
- [Rank-Based Uniformity Test](https://arxiv.org/abs/2506.06975)（RUT，ICLR 2026）
- [AgentProv](https://arxiv.org/abs/2609.00052)（出题人 Xun Wang、Bihe Zhao，以及 Boenisch、Dziedzic，EMNLP 2026）

【已核实】AgentProv 的摘要写：系统提示会把文本分布扭到让文本检验误伤诚实提供方。MET 在系统提示注入下的假阳性约 67%，RUT 约 53%，他们的工具调用分布检验约 7%。RUT 需要一个本地的真实模型做对照。这题没有给 `ep-01` 到 `ep-33` 的参考权重。

【比赛用法】这题的 API 只返回文本，而且拒绝 system message。AgentProv 的工具调用信道不一定开着，不要假设能跑他们的协议。能搬走的是否定结果：原始文本上的 MMD 会被水印和解码策略拆开同一模型。RUT 没有本地权重就做不了。

## 做法【比赛用法】

家族只值 0.1，模型身份值 0.9。预算花在「同一基座内部，量化 / LoRA 敏感，水印 / 系统提示 / 解码不敏感」的特征上。

1. 在 `ref-01` 到 `ref-09` 上先跑一小套 prompt，用 `score_practice.py` 打分。一个特征如果把「只有水印或系统提示不同」的参考端点拆开，就丢掉。这一步没过，不要查询 `ep-*`。
2. Prompt 用短的、答案几乎唯一的事实题和计算题，把生成压在几十个 token 里。水印和采样噪声主要堆在长生成上。再加少量能力探针（代码、数学、一个窄领域），用来抓 LoRA。不要要散文，不要要角色扮演。
3. 所有端点用同一组 prompt。建议 20 到 25 条 prompt，每条 4 个样本，每个端点 80 到 100 条，留几条预算补洞。同时只开 4 个请求，按端点并行，不要 33 个串行。
4. 家族（0.1）用粗特征就够：答案里稳定出现的专有 token、语言习惯、知识边界。分完家族再在家族内部分模型。不要用一个距离同时做两件事。
5. 家族内部不要用 green-token 比例、长度、礼貌程度、拒绝话术。这些跟着水印、系统提示和解码走。优先用封闭题的对错、关键事实 token 是否出现。量化通常改的是细分布，LoRA 改的是某一类题的行为；两边都要有探针，但不能让风格差淹没它们。
6. 聚类结果写成 CSV 前检查：相同 `model` 的行，`family` 必须相同；标签无逗号；33 个端点各一行。
7. 第一次正式提交要在参考端点上已经高于「全不同」。全不同在模型项上几乎是 0，因为漏掉的 same 全是 FN。30 分钟一次，不要用第一次去试格式。格式用 `submission.py`，先对参考答案跑通再换 `ep-*`。
8. 公开榜只当粗检查。方法改不改，看 `score_practice.py`，不看榜。
