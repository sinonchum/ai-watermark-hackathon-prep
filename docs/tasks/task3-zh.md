# Task 3：Unlearning

官方题面：[`Unlearning.pdf`](Unlearning.pdf)（5 页，Google Docs 导出）。下面是题面原文的整理，以及我们的做法。标注：

- **【题面】**：PDF 里写的。
- **【已核实】**：2026-11-21 核对过的论文摘要。
- **【比赛用法】**：我们的判断，不是主办方给的解法。

主办：Adam Dziedzic、Franziska Boenisch。出题：Maitri Shah、Jack Le、Vincent Hanke。

## 先做这个

1. 克隆 starter kit，用它的模型构建和预处理，不要自己猜归一化：<https://huggingface.co/datasets/SprintML/unlearning>
2. Jülich 环境仍走三题共用仓库：`git clone https://huggingface.co/datasets/SprintML/hackathon`
3. **不要跑赛前笔记里的 SCRUB / NegGrad+。** 那套要有标签的 forget set 和 retain set。这题只给 3500 张图，没有标签，也没有其余训练图。梯度上升把 forget 损失拉高，会被当成没忘掉。
4. 目标是贴近「从没见过这 3500 张」的重训模型，包括它在每张图上选哪个错类。损失只对齐均值不够，标准差和温度 2 的整份 softmax 都要贴近。忘过头和忘不够一样扣分。
5. 隐藏验证集准确率低于 70%，forget 和 retain 都记 0。最终分是两项相乘，任何一项是 0，总分就是 0。
6. 提交必须 `strict=True` 载入 `task_template.py` 里的结构。float32，无 NaN / Inf，不超过 450 MB。LoRA 或 adapter 必须折回原结构。

## 目标【题面】

给定一个在自定义 100 类图像数据上训练好的 ViT-B/16，以及 3500 张必须忘掉的训练图。提交的模型要从给定 checkpoint 改出来，表现得像训练时从没见过这 3500 张，同时在其余数据上仍然好用。

朴素梯度上升可以让这 3500 张的准确率很低，但行为和「从没见过」的模型仍然分得开。要对齐的不只是置信度，还有每张图选哪个错类，而且精确原图和轻微改过的副本都要对齐。

除了 forget 图本身，没有别的带标签训练数据。不能在干净数据集上重训。方法只能用原模型和 forget set。

成功的提交要同时做到：

- forget 图及其改动副本上的行为，接近训练时没见过它们的模型
- 其余数据上的准确率和输出行为保住
- 在一份通用验证集上保持合理准确率。为了把遗忘信号做高而把整个模型打坏，不算有效遗忘

## 三个模型【题面】

- `orig`：发给你们的 `model.pt`，训练数据包含 forget 图。
- `gold`：从头重训、完全不含 forget 图的参考模型。要复现的是它的行为。gold 是若干次独立重训的平均。不提供给参赛者。
- `sub`：提交的模型。

对模型 m 和图像 x：

- `L_m(x)`：用真实标签算的交叉熵。
- `p_m(x)`：logits 除以温度 2 之后的 softmax。
- `acc_m(control)`：control 集上的准确率。

gold 的 `mean(L_gold)` 和 `std(L_gold)` 是若干 gold 模型上的平均。`p_gold(x)` 是它们预测分布的平均。KL 是两个类别分布之间的 KL。mean 和 std 都是在被评分的那组图上算的。

## 评分【题面】

效用门槛：一份没有发给参赛者的验证集上，准确率至少 70%。低于 70%，forget 和 retain 都记 0，不管别的项多好。

Forget 在 forget 原图和隐藏的改动副本上各算一组，两组权重相同。每组两个分。

损失分布：

```text
d_sub  = |mean(L_sub)  − mean(L_gold)| + |std(L_sub)  − std(L_gold)|
d_orig = |mean(L_orig) − mean(L_gold)| + |std(L_orig) − std(L_gold)|
mean_std_score = 1 − clip(d_sub / d_orig, 0, 1)
```

预测分布，逐图比较，温度 2：

```text
prediction_score = 1 − clip( mean KL(p_gold || p_sub) / mean KL(p_gold || p_orig) , 0, 1 )
```

两个分各自在原图和改动副本上平均，然后相乘：

```text
forget_score = mean_std_score * prediction_score
```

0 表示没遗忘（sub = orig）。1 表示对齐 gold。只对齐平均损失不够。

Retain 在一份没提供的 control 集上：

```text
retain_accuracy_score = clip( acc_sub(control) / acc_orig(control) , 0, 1 )
retain_kl_similarity  = 1 / (1 + mean KL(p_orig || p_sub))
retain_score = retain_accuracy_score × retain_kl_similarity
```

准确率保住但输出分布偏离 orig，KL 项仍然扣分。

```text
final_score = forget_score × retain_score
```

两项都不能拿来换另一项。

## 数据【题面】

- `model.pt`：weights-only state dict。timm `vit_base_patch16_224`，100 类，LayerScale 初始值 1e-4。
- `forget_images.npz`：键 `images`，uint8，形状 `[3500, 224, 224, 3]`，channels last，像素 0–255，没有归一化。不含标签。
- `task_template.py`：加载模型、服务端预处理、预测辅助、`unlearn()` 空函数。直接跑会把未改动的模型存成基线 `submission.pt`。
- `submission.py`：提交脚本。API key 保留双引号，把 `/PATH/FILE.pt` 换成真实路径。

数据：<https://huggingface.co/datasets/SprintML/unlearning>

评分用的隐藏改动副本、control 集、70% 门槛的验证集都不提供，分布与训练相同。按样本标识的哈希分成公开约 30% 和赛后 70%。改动副本跟它的原图走同一个划分。

榜：<http://34.147.229.55/leaderboard_page>。只保留每队最高分。这次更低也不会覆盖，但返回里仍有这次的分数。题面没有写提交冷却。

## 提交格式【题面】

只交权重，层名和形状必须与给定模型一致。

- `.pt` 或 `.pth`，能用 `strict=True` 载入 `task_template.py` 的结构
- float32，无 NaN、无 Inf
- 最大 450 MB。float32 ViT-B 大约 345 MB
- 如果训练时加了 adapter、低秩或 PEFT，提交前必须折回原结构
- 服务端前向是 eval 模式、float16 autocast。这不改变你交的权重，但数值会和纯 float32 前向有差

## 四篇引用里哪篇对得上设定

【题面】列了四篇：

- [SCRUB](https://arxiv.org/abs/2302.09880)，NeurIPS 2023
- [Zero-Shot Machine Unlearning](https://arxiv.org/abs/2201.05629)，IEEE TIFS 2023
- [SalUn](https://arxiv.org/abs/2310.12508)，ICLR 2024
- [NPO](https://arxiv.org/abs/2404.05868)，COLM 2024

【已核实】Zero-Shot 问的就是没有训练样本时怎么遗忘。两条路：error minimizing-maximizing noise，以及 gated knowledge transfer。代码在 <https://github.com/ayu987/zero-shot-unlearning>。SalUn 用梯度显著性把更新限制在一部分权重上，标准做法要用 forget 和 retain 的标签。NPO 是语言模型上的方法，结论是梯度上升会灾难性崩掉，NPO 崩得慢一些。

【比赛用法】设定对得上的是 Zero-Shot，不是赛前笔记里的 SCRUB。SCRUB 要 retain 上的 KL 和交叉熵，这题没有 retain 图，也没有标签。NPO 不要直接搬到 ViT 上。SalUn 的「只更新一部分权重」可以当正则，但不要等标签齐了再做。

## 做法【比赛用法】

这是「忘到和重训一样」，不是「尽量忘」。忘过头时 `d_sub` 变大，`mean_std_score` 下降。预测分布如果比 orig 离 gold 更远，`prediction_score` 会被 clip 到 0。gold 在没见过的图上通常仍是一个尖的错类分布，不是均匀分布，也不是把损失拉到无穷。

1. 用 `task_template.py` 的预处理读 npz。图是 0–255、HWC，不是归一化后的张量。服务端用哪套 mean/std，以模板为准。
2. 用 orig 的 argmax 给 3500 张打伪标签，只用来知道模型记住了哪一类。不要把「远离这个类、越远越好」当目标。gold 要的是另一个仍然像样的预测，不是最大熵。
3. 第一版走 Zero-Shot 的 gated knowledge transfer，或 error min-max noise。retain 的替身只能来自模型自己：噪声、或 forget 图的强破坏版本上，让输出贴近 orig。forget 图的轻度增强上，把记住的类压下去，但不要压成均匀或 NaN。
4. 训练时加轻度增强。评分有隐藏的 slightly altered 副本，只在精确像素上遗忘不够。不要猜一套很重的增强去贴隐藏副本。
5. 每隔若干步存 checkpoint。本地能看的只有代理：forget 上 orig 的 top-1 置信度是否下降、增强后是否也下降、权重离 orig 有多远、float16 autocast 下有没有 NaN。没有 gold，本地算不出官方分。不要选遗忘最强的那个点。
6. 效用掉太多时，把该 checkpoint 和 orig 做权重插值，往 orig 拉。乘性分数下，retain 崩了和 forget 没动一样是 0。70% 是绝对准确率，不是相对 orig 的比例。
7. 提交前用模板的结构 `load_state_dict(..., strict=True)`，确认 float32、无 NaN/Inf、文件小于 450 MB。在 eval + float16 autocast 下跑一遍 3500 张，确认不会炸。
8. 公开榜约 30%，改动副本和原图在同一划分。对着公开分把某几张图记死，帮不到另外 70%，也帮不到「行为要像没见过」这件事。
