# Task 1：Localize Image Watermark

官方题面：[`Watermark_Localization.pdf`](Watermark_Localization.pdf)（4 页，Google Docs 导出）。下面是题面原文的整理，以及我们的做法。标注：

- **【题面】**：PDF 里写的。
- **【比赛用法】**：我们的判断，不是主办方给的解法。

主办：Adam Dziedzic、Franziska Boenisch。出题：Simon Hector、Michel Meintz、Louis Kerner、Maitri Shah。

## 先做这个

1. 克隆 starter kit，不要自己重写加载器：<https://huggingface.co/datasets/SprintML/Hackathon_Watermark_Localisation_Task_Paris>
2. Jülich 环境按这个仓库的 README，三道题共用：`git clone https://huggingface.co/datasets/SprintML/hackathon`
3. 测试集**给了水印方法和嵌入的比特串**。用对应提取器，对已知消息算比特一致率。不要训一个「这是不是水印」的分类器当第一版。
4. 第一版不要用 Watermark Anything 的权重。这三种方法没有按定位训练。WAM 认不出 PixelSeal / TrustMark / StegaStamp。
5. 提交必须是 `uint8` 或 `bool` 的 0/1。`float32` 大约 67 MB，超过 20 MB 上限，会被自动拒。
6. 先在 train 上用 starter kit 的 local mIoU 跑通，再交。公开榜只占测试集 30%，不要对着它调死。

## 目标【题面】

找出图像里哪些部分带水印。每张图都用三种公开后处理水印之一，打在原图的一个矩形子集上：PixelSeal、TrustMark、StegaStamp。之后的编辑会盖掉或只留下一部分水印。

对每张测试图交一个二值 mask：1 = 有水印，0 = 干净。

矩形子集的长宽比：PixelSeal 和 StegaStamp 不超过 16:1，TrustMark 不超过 2:1。所以前两个可以是细条，TrustMark 不会是细条。

题面自己写了难点：Watermark Anything 是按定位训练的；PixelSeal、TrustMark、StegaStamp 只被训练成从整张图解一条消息。这题问的是：现成的后处理水印能不能被定位。编辑从大半张图到细物体、人脸，三种水印行为差很多，没有一种方法通吃。

动机：没有定位的话，后来加进去的内容也会算到原来的水印所有者头上，第三方可以用添上的不良内容陷害水印所有者。

## 评分【题面】

图像 512×512。评分网格是 128×128，一格 = 4×4 像素。一格的真值是 1，当且仅当这一格里至少 50% 的像素有水印。starter kit 里有这个下采样函数，交之前用它，不要自己写一个近似版。

每一类 c ∈ {0, 1}：

```text
IoU_c = |P_c ∩ G_c| / |P_c ∪ G_c|
```

并集为空时 IoU_c = 1。一张图的分数是两类 IoU 的平均。总分是所有测试图的平均。越高越好，范围 0 到 1。每张图权重相同，不因编辑面积加权。假阳性和假阴性都扩大并集，都会扣分。只预测全 0 或全 1 会在另一类上崩。

## 数据【题面】

1530 张 PNG，三种水印各占 1/3。

- Train：510 张，每种 170。有图、512×512 和 128×128 的真值 mask、水印方法、嵌入消息。
- Test：1020 张，每种 340。有图、水印方法、嵌入消息。没有 mask。

编辑类型：

- Outpainting：带水印的图被扩成更大的图。水印还在，但只占一部分。mask 接近一个矩形。
- Object removal / alteration：带水印的图上有物体被改掉或删掉。水印里出现一个洞。
- Object addition：干净内容盖到带水印的图上。同样是一个洞。
- Object transplantation：把带水印图上的物体贴到干净图上。水印只在这个物体上，形状不一定是矩形。

## Starter kit【题面】

<https://huggingface.co/datasets/SprintML/Hackathon_Watermark_Localisation_Task_Paris>

里面有：

- 解压并载入 PyTorch dataset 的代码
- PixelSeal、TrustMark、StegaStamp 的统一封装，接口相同：`decode(images) -> bits`，并能拿到中间特征图
- 比特一致率、mask 下采样到 128×128、在 train 上算 local mIoU
- `submission_template.py`

Jülich 的数据下载、Python 环境和三道题共用目录，看 `SprintML/hackathon` 的 README。计算节点没有外网，数据和权重在登录节点下到 `$PROJECT`。

## 提交【题面】

一个 `.pt` 文件：

```python
{"image_names": names, "masks": masks}
```

`names[i]` 对应 `masks[i]`。`masks` 形状 `[num_images, 128, 128]`，值只能是 0 或 1。每个图像名恰好一次。没有重复，没有非法值。文件不超过 20 MB。不合格会自动拒绝。

【比赛用法】1020 × 128 × 128 的 `uint8` 约 16.7 MB，加上文件名和 pickle 头，贴着 20 MB。`float32` 约 66.8 MB，必拒。用 `torch.uint8` 或 `torch.bool`，确认 `masks.unique()` 只有 0 和 1，再交。

评分：自动验证，在隐藏测试集上算。公开榜是测试图的 30%，最终排名是剩下的 70%。榜上只保留每队最好的一次；这次更低就不会更新，但请求的返回里仍有这次的分数。

榜：<http://34.147.229.55/leaderboard_page>

## 做法【比赛用法】

测试集给了方法和消息。定位变成：哪些像素是「解出这条已知消息」所依赖的。

1. 按题面给的方法选提取器。整图 `decode`，和已知消息算比特一致率，当作这张图的基线。一致率接近 50% 说明水印几乎没了，mask 应接近全 0，不要硬画出一块。
2. 优先用 starter kit 暴露的中间特征图，一次前向出空间响应。在 train 上对每种方法单独看：哪一层、哪个通道和 128×128 真值 mask 的 mIoU 最高。三种方法分开定阈值。题面写了它们行为差很多。
3. 特征图不够时，再做遮挡。把图分成网格，逐块换成原图模糊或邻域填充，看比特一致率掉多少。掉得多的块是水印。不要用「裁一块还能解出消息」当正样本：Pixel Seal 对 50% 裁剪仍然很稳，一块能解码只说明水印还够，不能说明整块都有水印。
4. 128×128 逐格遮挡是 1020 × 16384 次前向，做不到。先 32×32 的探针网格，再只在边界上加密。评的是 4×4 像素一格、50% 阈值，边界偏一格会扣分，中间大面不需要像素级。
5. 四种编辑可以分治，方法名字段能用来选支路，但不要相信我们还没在 train 上验证的规则：
   - Outpainting：响应应是一个实心矩形。可以在特征图上找最大矩形。
   - Removal / addition：矩形里有洞。洞里遮挡不应掉比特一致率。
   - Transplantation：只在物体上。形状不规则，矩形拟合会多出一圈假阳性，两类 IoU 都会掉。
6. TrustMark 的子集长宽比 ≤ 2:1，不要预测成细条。PixelSeal 和 StegaStamp 允许到 16:1，细条要保留，不能用「太瘦就删掉」的后处理。
7. 交之前用 starter kit 的下采样和 local mIoU，在 510 张 train 上分方法报三个数，再报总 mIoU。只报一个平均数会掩盖 TrustMark 或细条失败。
8. 公开榜 30%。一次提交换一个改动，记下返回的分数。更低的分数不会刷掉榜上的最好成绩，所以可以拿来试，但不要把公开榜当最终分布。

和今早讲座的关系：讲座里的 WAM 推理是「检测图过滤，再 DBSCAN」。这题的提取器没有检测图。要自己从 `decode` 的比特一致率或中间特征图造出那张检测图。DBSCAN 只在 transplantation 的多连通区域有用，不是主路径。
