# AI Watermark Hackathon Prep

Preparation notes for the [European AI Championship Paris](https://networkinparis.com/event/evt-t7ZboosIQPBRAB2) (20–22 November 2026). The announced theme is AI watermarking: marking, detecting, and attributing AI-generated content, against the background of [EU AI Act Article 50](https://artificialintelligenceact.eu/article/50/).

On 17 November 2026 the organizers (CISPA) recommended three papers to all participants. Only one of them is about watermarking, so the challenges may also cover model identification and machine unlearning. That is our inference; the organizers have not said how the papers relate to the challenges.

| Paper | Topic | Code |
| --- | --- | --- |
| [Pixel Seal: Adversarial-only training for invisible image and video watermarking](https://arxiv.org/abs/2512.16874) (Meta FAIR) | Post-hoc image and video watermarking, 256-bit | [facebookresearch/videoseal](https://github.com/facebookresearch/videoseal) |
| [Model Equality Testing: Which Model Is This API Serving?](https://arxiv.org/abs/2410.20247) (ICLR 2025) | Two-sample (MMD) tests on black-box LLM API outputs | [model-equality-testing](https://github.com/irenasaracay/model-equality-testing) |
| [Towards Unbounded Machine Unlearning](https://arxiv.org/abs/2302.09880) (SCRUB, NeurIPS 2023) | Teacher-student unlearning, evaluated with membership inference | [meghdadk/SCRUB](https://github.com/meghdadk/SCRUB) |

This repository is preparation based on public information, past tasks, and these papers. It is not the official brief. The problem statements, scoring, and rules will be announced after Meta's opening talk on the morning of 21 November 2026. Links were checked on 16 November 2026; the paper code links were checked again on 17 November 2026.

The notes are in Chinese, written for Simon Qin, Chuan Li, Xuanhao Hu, and Wenhan Ding. Each paper note separates what the paper states from our inference.

## Contents

- [Paper notes (Chinese)](docs/papers-zh.md): problem, method, evaluation, code, likely challenge format, and quick-start plan for each paper
- [Full analysis (Chinese)](docs/prep-zh.md)
  - [主办方邮件与推荐论文](docs/prep-zh.md#主办方邮件与推荐论文)
  - [已知信息](docs/prep-zh.md#已知信息)
  - [往届题目参考](docs/prep-zh.md#往届题目参考)
  - [技术速览](docs/prep-zh.md#技术速览)
  - [Meta 开源工具](docs/prep-zh.md#meta-开源工具)
  - [可能的题型与准备](docs/prep-zh.md#可能的题型与准备)
  - [项目方向](docs/prep-zh.md#项目方向)
  - [赛前环境准备](docs/prep-zh.md#赛前环境准备)
  - [4 人分工与时间线](docs/prep-zh.md#4-人分工与时间线)
  - [来源](docs/prep-zh.md#来源)

## Starter skeleton

The analysis includes a suggested project layout and the packages listed for pre-event setup. They are in [`wm-hack/`](wm-hack/) and [`wm-hack/requirements.txt`](wm-hack/requirements.txt):

- `wm-hack/wm/`: watermarking
- `wm-hack/met/`: model equality testing
- `wm-hack/unlearn/`: machine unlearning

All Python files are placeholders only (comments, no implementation).

## License

MIT
