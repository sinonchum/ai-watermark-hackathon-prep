# European AI Championship Paris 赛前准备：AI 水印

给 Simon、Chuan Li、Xuanhao Hu、Wenhan Ding 的赛前资料：技术速览、可能题型、项目方向和准备清单。正式题目要等 11/21 上午 Meta 开场才公布。

## 目录

- [已知信息](#已知信息)
- [往届题目参考](#往届题目参考)
- [技术速览](#技术速览)
- [Meta 开源工具](#meta-开源工具)
- [可能的题型与准备](#可能的题型与准备)
- [项目方向](#项目方向)
- [赛前环境准备](#赛前环境准备)
- [4 人分工与时间线](#4-人分工与时间线)
- [来源](#来源)

> **这是准备资料，不是官方题目。** 本页根据公开信息和往届题目推测，具体题目、评分方式和规则以主办方当天公布为准。所有链接于 2026-11-16 检查过可以打开。

## 已知信息

| | |
| --- | --- |
| 时间 | 11/20（周五）19:00 开场见面；11/21 9:00 签到，Meta 讲题后约中午开始；11/22 中午停止开发，评委演示 |
| 地点 | Epitech Paris，24 Rue Pasteur, Le Kremlin-Bicêtre |
| 主题 | AI 水印：标记、检测、溯源 AI 生成内容；背景为 EU AI Act Art. 50 |
| 主办 | PoC Innovation、CISPA（Adam Dziedzic）、Fraunhofer AISEC（Franziska Boenisch）；Meta 赞助 |
| 评分形式 | 往届为 API 提交 + 隐藏测试集 + 实时排行榜；今年是否同样尚未公布 |

Art. 50(2) 要求生成合成音频、图像、视频或文本的系统提供方，确保输出以机器可读格式标记、可被检测为人工生成，并要求技术方案有效、可互操作、鲁棒、可靠。按 Art. 113，该条自 2026 年 8 月 2 日起适用。

## 往届题目参考

| 站点 | 题目 | 评分方式 |
| --- | --- | --- |
| Paris（Loki 队记录） | 黑盒对抗样本：最小 L2 扰动骗过分类器 | 平均归一化 L2，未成功的样本记 1.0 |
| Paris（Loki 队记录） | 图像溯源：判断图像来自 4 个已知生成模型中的哪个（open-set） | TPR / (1 + 5·FPR)，四类宏平均 |
| Paris（Loki 队记录） | 图像水印检测：Tree-Ring、BitMark、Stable Signature | TPR@1%FPR，每张图输出 0–1 分数 |
| Vienna 2025 | Dataset inference | 实时排行榜 |
| Vienna 2025 | LLM attribution（文本归因） | 实时排行榜 |
| Vienna 2025 | 去除图像水印，同时保持画质 | 残留水印可检测性 + 画质（参赛队提到 MSE） |

CISPA 报道里 Boenisch 提到，熟悉 SprintML Lab 近期论文的队伍上手更快。Vienna 一支队伍用自适应预处理加 VAE 重建去水印，也提到单一画质指标（MSE）不能完全反映感知质量。

## 技术速览

### 文本水印

| 方法 | 核心思路 | 代码 / License |
| --- | --- | --- |
| KGW green/red list（Kirchenbauer 2023） | 按前一个 token 做 hash 把词表分成 green/red，生成时给 green 加 bias δ，检测用 z-test 统计 green 比例 | `jwkirchenbauer/lm-watermarking`，Apache-2.0 |
| Unigram watermark（Zhao 2023） | 固定一个全局 green list，对编辑和 paraphrase 更稳，有可证明鲁棒性 | `XuandongZhao/Unigram-Watermark`，MIT |
| Aaronson / Gumbel 采样 | 用带密钥的伪随机数做 Gumbel-max 采样，分布不变，检测时累计伪随机分数 | 见 Aaronson 博客与 Kuditipudi 2023（distortion-free） |
| SynthID-Text（DeepMind，Nature 2024） | Tournament sampling；已集成进 transformers，配 Bayesian detector | `google-deepmind/synthid-text`，Apache-2.0 |
| Three Bricks（Meta，Fernandez 2023） | 统一统计检验，给出可靠的低 FPR p-value | `facebookresearch/three_bricks`，CC BY-NC 4.0 |

工具箱：MarkLLM（Apache-2.0，`pip install markllm`）集成了多种文本水印算法和评估；MarkMyWords（Apache-2.0）是文本水印 benchmark。

### 图像与视频水印

| 方法 | 核心思路 | 代码 / License |
| --- | --- | --- |
| Stable Signature（Meta） | 微调 LDM 的 VAE decoder，使所有生成图都带固定消息，由预训练提取器解码 | `facebookresearch/stable_signature`，CC BY-NC 4.0 |
| Watermark Anything / WAM（Meta，ICLR 2025） | 局部水印：可在一张图的不同区域嵌入多个消息，并输出水印位置 mask | `facebookresearch/watermark-anything`，MIT（仓库已归档） |
| Video Seal / Pixel Seal / ChunkySeal（Meta） | 后处理式图像和视频水印；Video Seal 256-bit，ChunkySeal 1024-bit | `facebookresearch/videoseal`，MIT，`pip install videoseal` |
| Tree-Ring（Wen 2023） | 在初始噪声的傅里叶空间嵌入环形图案，检测时 DDIM inversion 回噪声 | `YuxinWenRick/tree-ring-watermark`，MIT |
| StegaStamp（Tancik 2019） | 编码器-解码器，经打印拍照仍可解码 | `tancik/StegaStamp`，MIT |
| SynthID Image（DeepMind） | Google 产品内部使用，未开源模型，只有官方页面 | 无公开代码 |
| BitMark（SprintML 2025） | 面向 bitwise 自回归图像模型的水印；往届检测题出现过 | 见 arXiv 2506.21209 |

### 音频与 Provenance

| 方法 | 核心思路 | 代码 / License |
| --- | --- | --- |
| AudioSeal（Meta，ICML 2024） | 生成器嵌入 + 检测器逐样本定位，16-bit 消息，有 streaming 版本 | `facebookresearch/audioseal`，MIT（含权重），`pip install audioseal` |
| C2PA / Content Credentials | 把来源和编辑历史写成加密签名的 manifest 附在文件上；元数据可被剥离，与水印互补 | `contentauth/c2pa-python`、`c2patool`，Apache-2.0 |

### 攻击

1. Paraphrase 攻击：Krishna 2023 显示改写能绕过文本检测器，并提出检索式防御。
2. Regeneration / diffusion purification：Zhao 2023 证明像素级不可见水印可被「加噪再用生成模型重建」去除。
3. Watermark stealing / forging：Jovanović 2024 通过查询学到 KGW 类规则；Meta 的 WmForger（NeurIPS 2025）做黑盒单样本伪造。
4. WAVES benchmark（Huang 组）：系统评估图像水印在失真、regeneration、对抗攻击下的鲁棒性。
5. 常规变换：JPEG、裁剪、缩放、旋转、模糊、噪声、调色，是所有鲁棒性评估的基线。

### 无水印检测与模型归因

1. DetectGPT：用扰动后的 log-probability 曲率做零样本检测，需要多次前向，较慢。
2. Binoculars：用两个相关 LLM 的 perplexity 与 cross-perplexity 之比做零样本检测，BSD-3-Clause。
3. 图像归因：Yu 2019 的 GAN fingerprint，DE-FAKE（2022）对文生图模型做检测与归因；往届 Loki 队用频域 FFT + SRM 高通残差双分支网络。
4. Dataset inference（Maini 2024）：判断某数据集是否被用于训练 LLM，Vienna 站出过此题。

### 评估指标

| 指标 | 含义 | 注意 |
| --- | --- | --- |
| TPR@低 FPR | 在 FPR 固定为 1% 或 0.1% 时的检出率 | 排行榜常用；阈值要在干净样本上校准 |
| Bit accuracy | 解码消息与原消息的逐位一致率 | 随机猜测为 50% |
| p-value / z-score | 文本水印的统计显著性 | 短文本统计量不足 |
| PSNR / SSIM / LPIPS | 画质与不可感知性 | MSE 好看不代表感知质量好 |
| Robustness 曲线 | 各类变换强度下检测率的变化 | 同时报告画质，才能看出权衡 |

## Meta 开源工具

| 仓库 | 内容 | License | 状态 |
| --- | --- | --- | --- |
| `facebookresearch/content-seal` | Meta 水印研究总入口，覆盖音频、图像、视频、文本 | MIT | 活跃（2025-12 创建） |
| `facebookresearch/videoseal` | Video Seal、Pixel Seal、ChunkySeal、WmForger | MIT | 活跃 |
| `facebookresearch/audioseal` | AudioSeal 0.2，含 streaming | MIT | 活跃 |
| `facebookresearch/watermark-anything` | WAM 局部图像水印 | MIT | 已归档 |
| `facebookresearch/stable_signature` | 生成模型内嵌水印 | CC BY-NC 4.0 | 可用 |
| `facebookresearch/three_bricks` | LLM 水印统计检验 | CC BY-NC 4.0 | 可用 |

Meta 研究员可能参与出题，优先把 videoseal 和 audioseal 跑通。CC BY-NC 仓库在黑客松里研究使用没问题，但如果要做产品化演示，最好写明许可。

## 可能的题型与准备

### 攻击 / 鲁棒性题

1. **形式推测**：给带水印的图（或文本、音频），要求去水印或伪造水印，同时画质受约束；按残留可检测性和画质打分。
2. **提前准备**：写好可组合的变换管线（JPEG、resize、blur、noise、VAE 重建、img2img），并提前下载 `stabilityai/sd-vae-ft-mse` 等权重。
3. **当天策略**：先用最弱攻击提交拿到基线分，再逐步加强；本地用开源检测器做代理评估，控制提交次数。

### 检测 / 归因排行榜题

1. **形式推测**：判断图像或文本是否带水印、来自哪个模型，提交每个样本的分数；指标多为 TPR@1%FPR。
2. **提前准备**：准备 timm 分类骨干（如 EfficientNet、ConvNeXt）的微调脚本、FFT 与 SRM 高通特征提取、阈值校准和提交脚本模板。
3. **当天策略**：先手工特征 + 逻辑回归出基线，再上深度模型；留出验证集，按官方指标本地复现分数。

### 开放构建 / 演示题

1. **形式推测**：围绕 Meta 工具或 Art. 50 做一个能演示的系统，评委现场打分。
2. **提前准备**：从下方项目方向选 1 个主选、1 个备选，提前搭好 FastAPI + 前端骨架和 3 分钟演示脚本。
3. **当天策略**：保证端到端可跑优先于功能数量；准备离线演示视频以防网络或 GPU 出问题。

## 项目方向

建议：A 与 B 组合最贴合本次主题，B 的变换管线也能直接用于攻击类排行榜题。如果当天全部是排行榜题，演示方向只作为备选。

### A. Provenance Agent

| | |
| --- | --- |
| 问题 | 拿到一张图、一段音频或一段文本，普通人无法判断它是否由 AI 生成、出自谁。单一检测器只能回答自己那一种水印。 |
| 方案 | Agent 依次调用工具：读取 C2PA manifest（c2pa-python）→ 跑 VideoSeal/PixelSeal、WAM、AudioSeal 检测 → 可选无水印检测（Binoculars）→ LLM 汇总证据，输出带依据和不确定性说明的报告，不下超出证据的结论。 |
| 技术栈 | Python、FastAPI、c2pa-python、videoseal、audioseal、LLM tool calling、简单 Web 前端。 |
| 24h 可演示 | 上传 3 类样本（带 C2PA 签名、带 Meta 水印、无标记），实时生成报告；展示每条证据来自哪个工具。 |
| 风险 | 检测器只认自己的方案，「未检出」不等于「非 AI」；LLM 解释可能夸大，需要固定模板和阈值。 |
| 评委关注点 | 直接对应 Art. 50 的「可检测」要求，并把 Meta 开源工具放进一个可用流程。Simon 的 agent 经验能直接用上。 |

### B. Robustness Red-team Harness

| | |
| --- | --- |
| 问题 | 水印论文的鲁棒性实验各自为政，部署方很难回答「我的水印在真实传播链路里还剩多少」。 |
| 方案 | 变换库（JPEG、裁剪、缩放、模糊、噪声、调色、截屏模拟、VAE 重建、diffusion img2img；文本侧用 LLM paraphrase）× 水印方案（VideoSeal、PixelSeal、WAM、Tree-Ring、AudioSeal、KGW/SynthID-Text），统一输出 bit accuracy、TPR@1%FPR、PSNR/SSIM，生成对比报告。 |
| 技术栈 | PyTorch、diffusers、scikit-image、lpips、videoseal、audioseal、MarkLLM、pandas，可选 Streamlit 看板。 |
| 24h 可演示 | 2 到 3 种图像水印 × 8 到 10 种变换的完整矩阵，加一张质量与存活率的权衡图。 |
| 风险 | GPU 时间不够跑满矩阵；diffusion 攻击慢，需提前下载权重并控制样本量。 |
| 评委关注点 | 与 CISPA SprintML 的研究方向一致，同时可直接复用到攻击类排行榜题。 |

### C. 检测 API + 浏览器扩展 / Android

| | |
| --- | --- |
| 问题 | 检测能力停留在 notebook 里，用户在社交媒体看到内容时没有入口。 |
| 方案 | 统一检测 API（图像、音频、文本），前端为 Chrome 扩展右键检测，或 Android share sheet 入口（Simon 有 Kotlin 经验）。结果显示「检出水印 / 有 C2PA / 未发现标记」三态，不给「真/假」二元结论。 |
| 技术栈 | FastAPI + GPU 推理服务；Chrome Manifest V3 扩展或 Kotlin Android 应用；c2pa-python。 |
| 24h 可演示 | 在一个网页上右键图片得到结果，或从相册分享到 App 得到结果。 |
| 风险 | 前端和部署工作量大，技术深度可能不如前两个方向；平台转码常会剥离 C2PA 元数据。 |
| 评委关注点 | 最终用户视角，演示直观。 |

### D. Art. 50 合规工具包

| | |
| --- | --- |
| 问题 | Art. 50 要求生成式系统的输出可被机器识别，中小开发者不清楚该组合哪些技术。 |
| 方案 | 生成端中间件：输出图像嵌入 VideoSeal/PixelSeal 水印，音频嵌入 AudioSeal，开源 LLM 文本用 transformers 的 SynthID-Text，再附 C2PA manifest 和披露标签，写入审计日志；配套一个 verifier。 |
| 技术栈 | Python SDK、videoseal、audioseal、transformers、c2pa-python（开发用测试证书）、SQLite 审计日志。 |
| 24h 可演示 | 一个小型生成应用接入中间件前后对比，verifier 读出水印和 manifest。 |
| 风险 | 法律解读不能出错，页面和演示应写明不构成法律意见；C2PA 正式签名需要证书，黑客松里只能用测试证书。 |
| 评委关注点 | 契合主题背景，展示多层标记（水印 + 元数据）互补。 |

### E. 文本水印实验台

| | |
| --- | --- |
| 问题 | 文本水印对 paraphrase 和翻译敏感，部署时缺少可复现的对比。 |
| 方案 | 在小型开源 LLM 上对比 KGW、Unigram、SynthID-Text 的检测强度和文本质量；用 LLM agent 做 paraphrase、翻译往返等攻击；再与无水印检测（Binoculars）对照。 |
| 技术栈 | transformers、MarkLLM、synthid-text、Binoculars、一个 1.5B 到 2B 级别的开源模型。 |
| 24h 可演示 | 交互页面：输入 prompt，看带水印生成、z-score 或检测分数，再一键 paraphrase 看分数变化。 |
| 风险 | GPU 显存与生成速度；需要提前确认所选模型许可和下载权限。 |
| 评委关注点 | 如果当天出现文本水印检测类题目，可直接转化为排行榜方案。 |

## 赛前环境准备

- [ ] GPU：Colab、Kaggle Notebooks、Modal、HF Spaces ZeroGPU 各准备一个账号，额度以官方页面为准
- [ ] 确认现场是否提供 GPU（往届 CISPA 用 Jülich 超算集群给参赛者提供 GPU）
- [ ] 本地和云端都装好依赖并跑通 videoseal、audioseal 的 quick start
- [ ] 提前下载模型权重，现场网络可能慢
- [ ] 建好团队 GitHub 仓库和共享笔记，统一 Python 3.10/3.11
- [ ] 四人确认各自技能和是否全部被录取

以上截至 2026-11-16 均未完成。

```bash
python -m venv .venv && source .venv/bin/activate
pip install torch torchvision torchaudio
pip install videoseal audioseal synthid-text markllm c2pa-python \
  transformers accelerate diffusers timm scikit-image lpips \
  opencv-python pandas fastapi uvicorn streamlit

# 需要从源码安装的
git clone https://github.com/facebookresearch/watermark-anything
git clone https://github.com/YuxinWenRick/tree-ring-watermark
git clone https://github.com/ahans30/Binoculars
git clone https://github.com/XuandongZhao/WatermarkAttacker

# 预下载权重
python -c "import videoseal; videoseal.load('videoseal'); videoseal.load('pixelseal')"
python -c "from audioseal import AudioSeal; AudioSeal.load_generator('audioseal_wm_16bits'); AudioSeal.load_detector('audioseal_detector_16bits')"
huggingface-cli download stabilityai/sd-vae-ft-mse
huggingface-cli download Qwen/Qwen2.5-1.5B-Instruct
```

注意：PyPI 上的 `binoculars` 包是一个无关的二项分布库，Binoculars 检测器要从 GitHub 安装。

建议的仓库结构：

```
wm-hack/
  README.md
  pyproject.toml
  data/            # 官方数据，不入库
  weights/         # 预下载权重，不入库
  wm/
    embed.py       # 统一接口: embed(x, scheme) / detect(x, scheme)
    attacks.py     # JPEG、crop、blur、noise、VAE、img2img、paraphrase
    metrics.py     # TPR@FPR、bit acc、PSNR/SSIM/LPIPS
    features.py    # FFT、SRM 残差
  tasks/
    submit.py      # 排行榜提交模板（读 token 用环境变量）
    calibrate.py   # 在干净样本上定阈值
  app/
    api.py         # FastAPI
    agent.py       # LLM tool calling 汇总证据
    web/           # 前端或扩展
  notebooks/
```


## 4 人分工与时间线

| 角色 | 负责 | 建议人选 |
| --- | --- | --- |
| Agent 与集成 | LLM/agent、API、端到端串联、演示 | Simon |
| ML 主力 | 模型训练和推理、GPU 资源、排行榜主方案 | 待定（看技能） |
| 评估与攻击 | 变换管线、指标、本地复现官方评分 | 待定 |
| 产品与演示 | 前端或扩展、数据整理、pitch 与演示视频 | 待定 |

队友技能目前不清楚，周五见面时按各自强项调整。排行榜题建议像往届队伍一样：一起拆题，再每人负责一题，定时同步。

| 时间 | 安排 |
| --- | --- |
| 周五 19:00 | 开场见面，确认队伍、分工、主选和备选方向 |
| 周六 9:00–12:00 | 签到、Meta 讲题；读规则和评分，决定每题负责人 |
| 12:00–14:00 | 每题先提交一个基线；演示题跑通最小端到端 |
| 14:00–20:00 | 主方案迭代，每 2 小时同步一次分数和阻塞 |
| 20:00–02:00 | 冲刺主要改进；开始写演示脚本 |
| 02:00–08:00 | 两人一组轮流睡 3 小时，长时间任务挂在 GPU 上跑 |
| 周日 8:00–11:00 | 最终提交、录备用演示视频、排练 |
| 11:30–12:00 | 代码冻结，检查提交 |
| 12:00 之后 | 评委演示 |

## 来源

- [活动页面](https://networkinparis.com/event/evt-t7ZboosIQPBRAB2) — networkinparis.com
- [EU AI Act Article 50](https://artificialintelligenceact.eu/article/50/) — artificialintelligenceact.eu
- [CISPA Hackathon Championship Vienna 报道](https://cispa.de/en/cispa-hackathon-championship-vienna) — cispa.de
- [Loki 队往届 Paris 站记录](https://github.com/Loki-Hackathons/cispa) — github.com
- [SprintML Lab](https://sprintml.com/) — sprintml.com
- [Content Seal](https://github.com/facebookresearch/content-seal) — github.com
- [VideoSeal / PixelSeal / ChunkySeal](https://github.com/facebookresearch/videoseal) — github.com
- [AudioSeal](https://github.com/facebookresearch/audioseal) — github.com
- [Watermark Anything](https://github.com/facebookresearch/watermark-anything) — github.com
- [Stable Signature](https://github.com/facebookresearch/stable_signature) — github.com
- [Three Bricks](https://github.com/facebookresearch/three_bricks) — github.com
- [Video Seal 在线演示](https://aidemos.meta.com/videoseal) — aidemos.meta.com
- [A Watermark for Large Language Models](https://arxiv.org/abs/2301.10226) — arXiv 2301.10226
- [lm-watermarking](https://github.com/jwkirchenbauer/lm-watermarking) — github.com
- [Provable Robust Watermarking for AI-Generated Text（Unigram）](https://arxiv.org/abs/2306.17439) — arXiv 2306.17439
- [Aaronson：My AI Safety Lecture（含 Gumbel 水印）](https://scottaaronson.blog/?p=6823) — scottaaronson.blog
- [Robust Distortion-free Watermarks for Language Models](https://arxiv.org/abs/2307.15593) — arXiv 2307.15593
- [SynthID-Text（Nature 2024）](https://www.nature.com/articles/s41586-024-08025-4) — nature.com
- [synthid-text](https://github.com/google-deepmind/synthid-text) — github.com
- [SynthID-Text in transformers](https://huggingface.co/blog/synthid-text) — huggingface.co
- [SynthID](https://deepmind.google/models/synthid/) — deepmind.google
- [Three Bricks to Consolidate Watermarks for LLMs](https://arxiv.org/abs/2308.00113) — arXiv 2308.00113
- [MarkLLM](https://github.com/THU-BPM/MarkLLM) — github.com
- [MarkMyWords](https://github.com/wagner-group/MarkMyWords) — github.com
- [The Stable Signature](https://arxiv.org/abs/2303.15435) — arXiv 2303.15435
- [Watermark Anything with Localized Messages](https://arxiv.org/abs/2411.07231) — arXiv 2411.07231
- [Video Seal](https://arxiv.org/abs/2412.09492) — arXiv 2412.09492
- [Tree-Ring Watermarks](https://arxiv.org/abs/2305.20030) — arXiv 2305.20030
- [StegaStamp](https://arxiv.org/abs/1904.05343) — arXiv 1904.05343
- [BitMark](https://arxiv.org/abs/2506.21209) — arXiv 2506.21209
- [AudioSeal: Proactive Detection of Voice Cloning](https://arxiv.org/abs/2401.17264) — arXiv 2401.17264
- [C2PA](https://c2pa.org/) — c2pa.org
- [c2pa-python](https://github.com/contentauth/c2pa-python) — github.com
- [Paraphrasing evades detectors of AI-generated text](https://arxiv.org/abs/2303.13408) — arXiv 2303.13408
- [Invisible Image Watermarks Are Provably Removable](https://arxiv.org/abs/2306.01953) — arXiv 2306.01953
- [WatermarkAttacker](https://github.com/XuandongZhao/WatermarkAttacker) — github.com
- [Watermark Stealing in LLMs](https://arxiv.org/abs/2402.19361) — arXiv 2402.19361
- [WmForger](https://arxiv.org/abs/2510.20468) — arXiv 2510.20468
- [WAVES benchmark](https://arxiv.org/abs/2401.08573) — arXiv 2401.08573
- [DetectGPT](https://arxiv.org/abs/2301.11305) — arXiv 2301.11305
- [Binoculars](https://arxiv.org/abs/2401.12070) — arXiv 2401.12070
- [Attributing Fake Images to GANs](https://arxiv.org/abs/1811.08180) — arXiv 1811.08180
- [DE-FAKE](https://arxiv.org/abs/2210.06998) — arXiv 2210.06998
- [LLM Dataset Inference](https://arxiv.org/abs/2406.06443) — arXiv 2406.06443
- [HF Spaces ZeroGPU](https://huggingface.co/docs/hub/spaces-zerogpu) — huggingface.co
- [Modal pricing](https://modal.com/pricing) — modal.com
- [Kaggle GPU usage](https://www.kaggle.com/docs/efficient-gpu-usage) — kaggle.com
