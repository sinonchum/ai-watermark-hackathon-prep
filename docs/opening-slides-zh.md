# 开幕幻灯片：日程、三道题、Jülich

Adam Dziedzic 的开幕幻灯片，78 页，2026-11-21 14:03 UTC 导出。原文件：[`EuroAIChampionshipParis2026-11-21.pdf`](EuroAIChampionshipParis2026-11-21.pdf)。三道题的细节以各自题面为准，这里只记幻灯片里多出来的、会改变做法的部分。

- **【幻灯片】**：这套 PDF 上的字。
- **【不要照抄】**：第 68 页之后标了 BACKUP，里面的项目号是 `training2614`，不是这次的。

## 日程【幻灯片】

巴黎时间。

- 周六 12:00：签到，午餐在 Lobby
- 周六 13:00：开幕、主题演讲、讲题
- 周六 14:30：开始做题。不是 Luma 上的 15:00
- 周六 17:00：晚餐在 Lobby
- 周六 18:00：通宵
- 周日 08:00：早餐在 Lobby
- 周日 12:00：做题结束，午餐
- 周日 13:00：前 6 名各讲 6 分钟
- 周日 14:00：评委讨论
- 周日 14:30：颁奖
- 周日 15:00：结束

一等奖 €10,000。幻灯片只写了一等奖。Discord 是 <https://discord.gg/urUG53tnbJ>，和 11 月 18 日邮件里的邀请链接不一样，以这页为准。

## Jülich【幻灯片】

项目号 `training2666`。40 个计算节点，每个作业占 1 个节点。节点是 4× A100 40GB、128 核、512 GB。12 个登录节点，上面有 2× Quadro RTX8000，可以在上面写代码。

登录前要做完四步：JuDoor 注册、打开 MFA、签使用协议、上传 SSH 公钥。公钥生效最多等 15 分钟。登录后要填 TOTP，幻灯片写的是 FreeOTP 或 Google Authenticator。

```bash
ssh -i ~/.ssh/id_ed25519 qin6@jureca.fz-juelich.de
```

队长只跑一次 `hackathon_setup.sh`。队友等队长跑完，再跑 `teammate.sh`。两个脚本不要在同一个账号上跑。用户名填真实的 Jülich 账号，空位留空。`TEAM_FOLDER` 队友必须和队长一致。数据下在：

```text
/p/scratch/training2666/<lead>/<team>/
```

下面有 `unlearning/`、`model_equality/`、`Hackathon_Watermark_Localisation_Task/`，各有 `.venv`。队友不要重新下载数据，也不要重建环境。脚本从 Hugging Face 的 `SprintML/hackathon` 下，按那个 README 做。

每次登录和每个 GPU 作业：

```bash
jutil env activate -p training2666
cd /p/scratch/training2666/<lead>/<team>/<task_folder>
source .venv/bin/activate
```

他们给的作业头是：

```bash
#SBATCH --account=training2666
#SBATCH --gres=gpu:1
#SBATCH --partition=dc-gpu
#SBATCH --reservation=cispahack
#SBATCH --output=output/%j.out
```

先 `mkdir -p output`，再 `sbatch main.sh`。用 `squeue -u $USER` 看队列。登录时不要重跑 setup。

预约名是 `cispahack`。不写这个，作业不进这次比赛的保留队列。BACKUP 页里的 `training2614` 和 `$PROJECT/$USER/Hackathon` 是旧例子，不要抄。

## 三道题，幻灯片多确认的几点

题面笔记仍是细版：[`task1-zh.md`](tasks/task1-zh.md)、[`task2-zh.md`](tasks/task2-zh.md)、[`task3-zh.md`](tasks/task3-zh.md)。

Task 2 这页把身份写死了。不同模型：基座、量化、LoRA。同一模型：基座加水印、加系统提示、或只改解码策略（例子是 greedy 对 beam search）。家族单独最多 0.1 分。

评分页写的是先截断再加权：`MCC_x = max(0, MCC)`，然后 `score = 0.1 × MCC_family + 0.9 × MCC_model`。PDF 题面里 MCC 和 MMC 的写法不一致，以这页为准。

Task 3 再次确认：forget 图没有标签。门槛是隐藏验证集 70%，低于则两项都是 0。最终分是 forget 乘 retain。gold 不提供，它是若干次不含 forget 图的重训的平均。
