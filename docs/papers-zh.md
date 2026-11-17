# 主办方推荐论文笔记

2026-11-17 14:02（巴黎时间），CISPA 的 Adam Dziedzic 给所有参赛者发邮件：请加入 European AI Championship Discord，并建议赛前读下面三篇论文。

标注说明：

- **【论文】**：内容来自原文，数字都是原文写的。
- **【已核实】**：2026-11-17 检查过的链接和包。
- **【推断】**：我的判断，不是主办方信息。主办方没有说明题目和论文的对应关系。

后两篇不是水印论文，所以题目可能还包括模型溯源或指纹、机器遗忘。这是【推断】。

| 论文 | 一句话 | 代码 | 可能题型【推断】 |
| --- | --- | --- | --- |
| [Pixel Seal](https://arxiv.org/abs/2512.16874)（Meta FAIR） | 只用对抗损失训练的后处理图像和视频水印，256 bit | [facebookresearch/videoseal](https://github.com/facebookresearch/videoseal)，MIT | 去水印、伪造、检测 |
| [Model Equality Testing](https://arxiv.org/abs/2410.20247)（Stanford，ICLR 2025） | 用 MMD 两样本检验判断 API 背后是不是参考模型 | [model-equality-testing](https://github.com/irenasaracay/model-equality-testing)，`pip install model-equality-testing` | 端点审计、模型识别 |
| [Towards Unbounded Machine Unlearning](https://arxiv.org/abs/2302.09880)（SCRUB，NeurIPS 2023） | teacher-student 遗忘：在 forget 上远离教师，在 retain 上贴近教师 | [meghdadk/SCRUB](https://github.com/meghdadk/SCRUB) | 遗忘 forget set 并保持效用，或验证遗忘 |

## 1. Pixel Seal: Adversarial-only training for invisible image and video watermarking

Tomáš Souček、Pierre Fernandez 等，Meta FAIR，arXiv 2512.16874（2025-12）。

### 问题【论文】

后处理式多比特图像水印难以同时做到不可见和鲁棒。论文指出三个问题：

1. MSE、LPIPS 等感知损失不贴合人眼，水印仍会在平坦区域可见。
2. 鲁棒和不可见两个目标冲突：感知损失的最优解是空水印，鲁棒性的最优解是明显可见的水印，训练容易崩，需要大量调参。
3. 模型在低分辨率上训练，推理时把水印上采样到高分辨率，会产生可见伪影，提取器也面对分布偏移。

### 方法【论文】

基本框架：嵌入器 `w = Emb(x, m)`，`x_w = x + α·w`；提取器从（可能被编辑过的）图中恢复 n bit 消息。检测时固定嵌入同一消息，用二项分布给出假阳性界（式 2），不需要在干净数据上校准阈值。

1. **只用对抗损失**：去掉所有感知损失，只保留消息 BCE 损失和 patch 判别器（Rombach et al. 2022 的判别器）的 hinge 对抗损失。用 boosting 因子 β 构造 `x + β(x_w − x)` 放大伪影，β > 1 更不可见，β < 1 更鲁棒。
2. **三阶段训练**：第一阶段只用消息损失、水印强度 α0 = 1.0，训练到比特准确率饱和；第二阶段加对抗损失，按余弦把 α 从 1.0 降到 0.2；第三阶段在 α = 0.2 下微调，可换数据或攻击。
3. **高分辨率适配**：图像先缩到 256×256 算水印，上采样后乘以在原分辨率上计算的 JND 衰减图；对抗损失在原分辨率上计算；训练时模拟推理流程，消除提取器的分布偏移。
4. **视频**：推理时在 U-Net 中插入时间平均池化和反池化（temporal watermark pooling），不需要视频训练。

### 实现与评估【论文】

- 嵌入器为 U-Net，43.8M 参数；提取器为 ConvNeXt-v2 Tiny，33.4M 参数。
- 在 SA-1b 上训练 600k 步，batch 256，AdamW，学习率 5e-4；λmsg = 1.0，λadv = 0.1；消息 256 bit。
- 基线：CIN（30 bit）、InvisMark（94 bit）、MBRS（256 bit）、TrustMark（100 bit）、Video Seal 0.0（96 bit）、Video Seal 1.0（256 bit）、WAM（32 bit），视频另有 RivaGAN。
- 图像测试集：1000 张 Meta AI 生成的 1280×1280 图像，100 张 SA-1b 验证集照片。视频测试集：121 个 MovieGen 视频、96 个 SA-V 验证视频，取前 3 秒。
- 不可感知性：PSNR、SSIM、LPIPS、CVVDP，以及基于内部用户研究训练的 JND 指标（1 JND 表示 75% 的观察者能察觉差异）。
- 鲁棒性：bit accuracy 和 −log10 p（随机猜中该比特准确率的概率，便于比较不同消息长度）。
- 攻击：亮度、对比度、色调、饱和度、灰度等值域变换；JPEG；视频的 H.264、H.265；裁剪、旋转、透视、水平翻转、模糊；组合攻击（亮度 0.5 + 裁剪 50% 面积 + JPEG 40，视频中 JPEG 换成 H.264）。

### 结论【论文】

- 所有方法对简单值域变换和 JPEG 都表现良好；在裁剪等几何攻击上 Pixel Seal 明显领先。
- 组合攻击下只有 Pixel Seal 和 InvisMark 的 bit accuracy 保持在 90% 以上，而 Pixel Seal 携带 256 bit，InvisMark 为 94 bit。
- 视频压缩对所有方法都更难，Pixel Seal 仍领先。
- 时间池化插在第一个 U-Net 块之后（d = 1）时速度为基线的 2.2 倍（Quadro GP100，64 帧），性能影响很小。

### 代码与权重【已核实】

- 代码：[facebookresearch/videoseal](https://github.com/facebookresearch/videoseal)，MIT。README 中加载方式为 `videoseal.load("pixelseal")`。
- 权重：`https://dl.fbaipublicfiles.com/videoseal/pixelseal/checkpoint.pth`，约 1.24 GB。
- 论文中写的项目网站 `facebookresearch.github.io/meta-seal` 在 2026-11-17 访问返回 404。

### 可能的题型【推断】

- 去水印：给 Pixel Seal 加了水印的图，在画质约束下让检测失败。论文显示它对常规变换和 JPEG 很稳，单一常规变换大概率不够，应优先尝试 regeneration（VAE 重建、img2img）和组合攻击。
- 伪造：让干净图被检测为带水印。
- 检测：判断图是否带水印，按 TPR@低 FPR 或 p 值评分。

### 快速上手【推断】

1. 克隆 videoseal，按 README 安装，加载 `pixelseal`，对 20 张图做嵌入、检测，确认 bit accuracy 和 p 值计算。
2. 把 `wm/attacks.py` 的变换管线接上，画出「PSNR 或 LPIPS 对 bit accuracy」的曲线。
3. 算力：推理在笔记本 CPU 上可跑但较慢；diffusion 类攻击需要 GPU。

## 2. Model Equality Testing: Which Model Is This API Serving?

Irena Gao、Percy Liang、Carlos Guestrin，Stanford，ICLR 2025，arXiv 2410.20247。

### 问题【论文】

用户通过黑盒推理 API 调用大模型（包括 Llama 这类开源模型）。提供方为了降成本或加功能，可能量化、加水印、微调模型，改变输出分布而不告知用户。论文把检测这种改动形式化为 Model Equality Testing：从参考分布 P 和 API 分布 Q 各采 N 个 prompt-completion 对，做两样本检验 H0: P = Q，并把一类错误控制在 α。

### 方法【论文】

- 基于 MMD 的核两样本检验（Gretton et al. 2012）。核取 `k(z, z') = 1{x = x'}·k̃(y, y')`，即只比较同一 prompt 下的补全。
- 比较的检验：Hamming 核（逐位置相同 token 或字符的计数，短的补全右侧补特殊符号）、all-substrings 核、one-hot 核，以及多项分布的 L1、χ² 检验。
- p 值：在 P 下重复采样模拟零分布，或用置换检验（少采样，可能损失功效）。
- Hamming MMD 是伪度量：MMD 为 0 不保证 P = Q。

### 实验设置【论文】

- 仿真：prompt 为 25 条 100 字符的维基百科片段（英、德、西、法、俄），补全长度 L = 50 token，温度 1，α = 0.05，功效来自 100 次 Monte Carlo。
- 替代分布：nf4、int8、fp16 量化；Kirchenbauer 水印（bias 2.5）；在维基或 CodeAlpaca 上微调 Llama-3 8B；换成 13 个模型（8 个开源、5 个 OpenAI）中的另一个。
- 模型：Mistral 7B Instruct、Llama-3 8B/70B Instruct、Llama-3.1 8B/70B Instruct 等。
- API 审计：2024 年夏，9 家供应商（Amazon Bedrock、Anyscale、Azure、Deepinfra、Fireworks、Groq、Perplexity、Replicate、Together）的 31 个 Llama 端点；在字符空间检验，α = 0.01；另用 HumanEval 和 UltraChat 各 20 条 prompt、250 token。零假设为「等于 fp32 或 fp16 参考权重」的复合假设。

### 结论【论文】

- 每个 prompt 平均 10 个样本时，Hamming MMD 对各类改动的中位检验功效为 77.4%，在所比较的检验中样本效率最高；对 fp16 的功效明显较低。
- 补全变长时，Hamming 和 all-substrings 检验的功效保持得更好。
- 微调 1 个 epoch（42 步）后即能以超过 50% 的功效检出，即使微调数据与测试任务不同分布。
- 字符空间里，换模型几乎都能 100% 检出，Llama 家族内部较难：Llama-3.1 8B 对 70B 为 75%，Llama-3 8B 对 3.1 8B 为 76%。换模型比水印、量化更容易检出。
- MMD 可作距离：同一家族的模型更接近，规模不是主要因素。
- 31 个端点中 11 个与 Meta 发布的参考权重不同；单次审计成本低于 1 美元。部分端点与参考的距离，与换成另一个模型相当。
- HumanEval 上 MMD 与准确率差值中度相关（R² 约 0.39）。

### 代码与数据【已核实】

- 代码：[github.com/i-gao/model-equality-testing](https://github.com/i-gao/model-equality-testing)，现重定向到 [irenasaracay/model-equality-testing](https://github.com/irenasaracay/model-equality-testing)。
- 包：`pip install model-equality-testing`（PyPI 0.0.2）。核心接口 `run_two_sample_test(sample1, sample2, pvalue_type="permutation_pvalue", stat_type="mmd_hamming", b=100)`。
- 数据：论文写「100 万条补全」，README 写「1.6M 条补全、5 个模型、540 条 prompt，其中 100 条带 fp32 logprob」，数据放在 Google Drive。以仓库为准。

### 可能的题型【推断】

- 给几个黑盒端点或样本集，判断哪个是原始模型，哪个被量化、加水印、微调或替换。
- 给一批文本，判断出自哪个模型（与往届 Vienna 的 LLM attribution 题相近）。
- 按准确率或固定 FPR 下的检出率评分；多次检验要控制 FPR（论文用 Bonferroni）。

### 快速上手【推断】

1. 装包，跑通 README 示例。
2. 用一个小模型（如 Qwen2.5-0.5B 或 1.5B）生成 fp16、int8、nf4、KGW 水印四个版本的样本，每版 25 条 prompt × 10 个样本，复现一次检验。
3. 准备字符 n-gram + 逻辑回归的文本归因基线，作为只有样本、无法采样时的备选。
4. 算力：检验本身在 CPU 上即可；采样需要 GPU 或调用 API。

## 3. Towards Unbounded Machine Unlearning（SCRUB）

Meghdad Kurmanji、Peter Triantafillou、Jamie Hayes、Eleni Triantafillou，NeurIPS 2023，arXiv 2302.09880。

### 问题【论文】

从已训练的神经网络中「移除」训练集的一个子集（forget set Df），同时保持在 retain set Dr 和测试集上的效用，并且比重训便宜。论文认为「遗忘」的定义取决于应用：

- **RB 去偏**：希望 forget 误差尽可能高。
- **RC 消除混淆**：把错标样本放进 forget set，消除两个类的混淆。
- **UP 用户隐私**：forget 误差只应和重训模型相当；过高反而让被删除样本可识别，用成员推理攻击（MIA）衡量。

### 方法【论文】

- 原模型作教师，学生从教师初始化。
- 目标：在 retain 上最小化学生与教师输出的 KL 散度，并加 retain 上的交叉熵；在 forget 上最大化 KL（式 3，α、γ 为超参数）。
- 训练技巧：直接联合优化会振荡，因此交替做一个 epoch 的 max-step（forget）和一个 epoch 的 min-step（retain），最后补若干 min-step 恢复 retain 表现。
- **SCRUB+R（rewind）**：构造与 forget 同分布的验证集；SCRUB 每个 epoch 存 checkpoint；训练结束时测验证集误差作为参考，回退到 forget 误差最接近该参考的 checkpoint，用于 UP 场景防 MIA。

### 评估【论文】

- 数据集：CIFAR-10、Lacuna-10（由 VGG-Faces 构造）。模型：ResNet-18、All-CNN。原模型先在 CIFAR-100 / Lacuna-100 上预训练。
- 小规模：CIFAR-5 / Lacuna-5，每类 100 训练、25 验证、100 测试；forget 为 class 0 的 25 个样本（5%）。
- 大规模：class 遗忘（class 5 全部，10%），selective 遗忘（class 5 中的 100 个样本）。
- RC：在 class 0 和 1 之间各错标 50 个样本。
- 指标：forget、retain、test 误差；IC-ERR、FGT-ERR；Basic MIA（用遗忘后模型在 forget 与 test 上的损失训练二分类器，准确率 50% 为理想）；论文改编的 LiRA（攻击者知道遗忘算法，用影子模型和「影子遗忘」模型）。
- 基线：Retrain、Original、Finetune、NegGrad+、CF-k、EU-k、Fisher、NTK、Bad-T。

### 结论【论文】

- SCRUB 是 RB、RC、UP 三类指标中最稳定的前列方法，同时不损害 retain 和 test 表现，而且高效。
- Finetune 保留效用但忘不掉；NTK 不能扩展到小数据集以外；Fisher 有时有效但比重训还慢；NegGrad+ 是强基线；Bad-T 快但忘得差且伤效用。
- SCRUB+R 对 MIA 的防御与重训相当；回退主要在 forget set 较小时触发。
- 局限：SCRUB 没有理论保证。

### 代码【已核实】

- [github.com/Meghdad92/SCRUB](https://github.com/Meghdad92/SCRUB)，现重定向到 [meghdadk/SCRUB](https://github.com/meghdadk/SCRUB)。以 notebook 为主（small/large scale unlearning、IC test、MIA 实验），License 文件未被 GitHub 识别为标准许可。

### 可能的题型【推断】

- 给预训练分类模型、forget set、retain set，限时输出遗忘后的模型；评分综合遗忘程度、MIA 区分度和 retain/test 准确率。
- 反向题：判断某模型是否真的遗忘了某些数据（这与 CISPA SprintML 的 dataset inference 方向相近）。
- 也可能扩展到生成模型或 LLM 遗忘，论文本身只做了图像分类。

### 快速上手【推断】

1. 在 CIFAR-10 上训练 ResNet-18 原模型，并训练一个去掉 forget set 的重训模型作对照。
2. 实现 Finetune、NegGrad+、SCRUB 三个基线和 Basic MIA 评估，记录运行时间。
3. 当天先确认评分要的是「尽量忘」（类似 RB）还是「忘到和重训一样」（类似 UP）：后者要加回退步骤。
4. 算力：单卡 GPU 足够做 CIFAR-10 规模实验；笔记本 CPU 只适合小规模（CIFAR-5）设置。

## 三篇之间的联系【推断】

- Model Equality Testing 把水印（Kirchenbauer）当作一种可被检出的分布改动，连接了水印和模型溯源。
- 该论文还提到微调的影响很难限制在单一分布上，「这可能对遗忘或模型编辑有影响」，连接了模型溯源和机器遗忘。
- 一个跨方向的想法：用两样本检验比较遗忘后模型与重训模型的输出分布，作为遗忘质量的补充证据。

## 建议的赛前准备【推断】

1. 每人精读一篇，周五见面时 5 分钟讲给其他人。
2. 跑通三个仓库的最小示例：videoseal 加载 `pixelseal`；`model-equality-testing` 的 README 示例；SCRUB 的小规模 notebook。
3. 提前下载 Pixel Seal 权重（约 1.24 GB）和一个小 LLM，训好 CIFAR-10 原模型。
4. `wm-hack/` 下准备三套基线：`wm/attacks.py`（水印攻击）、`met/`（采样 + MMD）、`unlearn/`（遗忘 + MIA）。目前都只是占位，未实现。
