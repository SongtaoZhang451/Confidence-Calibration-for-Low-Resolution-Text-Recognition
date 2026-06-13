# 低分辨率文本识别的置信度校准

[English](README.md) | [简体中文](README_zh-CN.md)

本项目评估场景文字识别模型在识别低分辨率图像时，所输出的置信度是否可靠。

本研究在 TextZoom 数据集上比较了三种预训练文字识别模型：**CRNN、ASTER 和 MORAN**。除识别准确率外，项目还分析了模型校准、高置信度错误，以及低对比度、模糊和复杂背景等视觉退化因素的影响。

随后，本项目采用一种轻量级后处理方法——**Platt Scaling（Platt 缩放）**，用于检验在不修改识别模型、也不改变文字预测结果的情况下，是否能够提升置信度的可靠性。

## 研究问题

本项目主要研究以下问题：

1. 文字识别模型是否会在低分辨率条件下，以较高置信度输出错误预测？
2. 哪些视觉条件与高置信度错误的关系最强？
3. 能否通过轻量级置信度校准，在不改变识别准确率的情况下提升置信度可靠性？

## 数据集

本项目使用 **TextZoom** 数据集。该数据集包含成对的低分辨率和高分辨率场景文字图像。

主要评估在 `test/medium` 子集上进行：

* 评估样本数：**1,411**
* 校准数据划分：`train2`
* 预测标准化：仅保留小写英文字母和数字
* 统一输出字符集：`a-z` 和 `0-9`

本仓库包含 `all_labels.csv`，其中保存图像路径、标签和数据集划分信息。TextZoom 图像文件本身未包含在本仓库中。

## 模型

本项目评估了三种预训练场景文字识别模型。

### CRNN

CRNN 结合了：

* 用于视觉特征提取的卷积神经网络
* 用于序列建模的循环神经网络
* 用于解码的连接时序分类（CTC）

CRNN 在本项目中作为基线识别模型。

### ASTER

ASTER 使用：

* 空间矫正网络
* 基于注意力机制的序列识别器

其矫正阶段旨在提升对畸变文字和不规则文字的识别能力。

### MORAN

MORAN 结合了：

* 多目标矫正
* 基于注意力机制的序列识别

该模型为比较不同网络架构如何影响识别准确率和置信度可靠性提供了另一个参考。

## 置信度定义

由于三种识别模型使用了不同的解码器和字符词表，本项目将它们的输出统一标准化为可比较的格式。

对于每个预测单词：

1. 提取每个被选中字符位置上的最大概率。
2. 删除统一字符集不支持的字符。
3. 使用保留字符概率的几何平均值作为单词级置信度分数。

## 评估指标

| 指标 | 说明 |
| --- | --- |
| 单词准确率（Word Accuracy） | 标准化预测结果与标准化真实标签完全一致的比例 |
| 期望校准误差（ECE） | 在不同置信度区间内，预测置信度与实际准确率之间的差异 |
| Brier Score | 置信度与二元正确性标签之间的均方误差 |
| 高置信度错误率（HCER） | 预测错误且置信度大于或等于 0.8 的样本比例 |

ECE、Brier Score 和 HCER 越低，说明置信度可靠性越好。

## 主要结果

### 校准前的模型表现

| 模型 | 单词准确率 | ECE | Brier Score | HCER |
| --- | ---: | ---: | ---: | ---: |
| ASTER | **0.4111** | **0.2899** | **0.2170** | **0.0914** |
| MORAN | 0.3664 | 0.3416 | 0.2334 | 0.0992 |
| CRNN | 0.2055 | 0.4021 | 0.2739 | 0.1573 |

ASTER 获得了最高的单词准确率，并且在校准前表现出最好的置信度可靠性。CRNN 的准确率最低，高置信度错误率最高。

## 视觉退化分析

本项目分析了三种视觉条件。

### 模糊

模糊程度使用拉普拉斯方差进行衡量。数值越低，表示图像越模糊。

评估集中的模糊子组样本非常少，因此现有结果不足以支持关于模糊与高置信度错误之间关系的强结论。

### 低对比度

对比度使用灰度图像像素强度的标准差进行衡量。数值越低，表示图像对比度越低。

三种识别模型在低对比度组中均表现出更高的 HCER，说明低对比度可能是导致高置信度识别错误的稳定风险因素。

### 复杂背景

背景复杂度通过以下指标的组合进行估计：

* Canny 边缘密度
* 标准化灰度图像熵

复杂背景提高了 CRNN 和 MORAN 的 HCER。相较之下，ASTER 对复杂背景表现出更强的抵抗能力，其 HCER 在该分组下略有下降。

## 置信度校准

Platt Scaling 在 TextZoom 的 `train2` 划分上进行训练，并应用于 `test/medium` 的模型预测结果。

校准只改变模型报告的置信度，不改变预测文字，因此单词准确率保持不变。

### 校准后的结果

| 模型 | 校准前 ECE | 校准后 ECE | 校准前 Brier | 校准后 Brier | 校准前 HCER | 校准后 HCER |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| CRNN | 0.4021 | **0.0797** | 0.2739 | **0.0955** | 0.1573 | **0.0000** |
| ASTER | 0.2899 | **0.0682** | 0.2170 | **0.1076** | 0.0914 | **0.0319** |
| MORAN | 0.3416 | **0.0692** | 0.2334 | **0.0861** | 0.0992 | **0.0255** |

Platt Scaling 显著降低了三种识别模型的 ECE、Brier Score 和 HCER。

HCER 为零并不意味着模型没有产生错误预测，而是表示在校准后，没有错误预测的置信度仍高于所设定的阈值。

## 仓库结构

```text
.
├── README.md
├── README_zh-CN.md
├── Extractor.ipynb
├── Project.ipynb
├── Report.pdf
└── all_labels.csv
```

* `Extractor.ipynb`：从原始 TextZoom LMDB 文件中提取成对的低分辨率和高分辨率图像及标签
* `Project.ipynb`：执行模型推理、置信度提取、评估、视觉条件分析和校准
* `Report.pdf`：完整项目报告和讨论
* `all_labels.csv`：TextZoom 标签、图像路径和数据集划分信息

## 预训练模型权重

```text
.
├── aster_demo.pth
├── Moran_demo.pth
└── crnn.pth
```

* `aster_demo.pth`：评估流程中使用的预训练 ASTER 权重
* `Moran_demo.pth`：评估流程中使用的预训练 MORAN 权重
* `crnn.pth`：评估流程中使用的预训练 CRNN 权重

从以下地址下载预训练权重：

* [预训练模型权重](https://drive.google.com/drive/folders/1LLJgzzbFV2312nnzjy_QIcQbAfn9onhb?usp=drive_link)

## 环境要求

主要 Python 依赖包括：

```bash
pip install torch torchvision pandas numpy scipy scikit-learn \
    pillow opencv-python matplotlib jupyter
```

还需要分别下载 CRNN、ASTER 和 MORAN 的原始实现及对应的预训练模型权重。

由于当前 Notebook 在 CUDA 上加载识别模型，因此推荐使用支持 CUDA 的 GPU。

## 项目流程

```text
TextZoom LMDB 文件
        |
        v
Extractor.ipynb
        |
        +--> 提取低分辨率和高分辨率 PNG 图像
        |
        +--> 生成各数据划分的 labels.csv
        |
        +--> 生成 all_labels.csv
        |
        v
Project.ipynb
        |
        +--> 运行 CRNN、ASTER 和 MORAN 推理
        |
        +--> 标准化预测结果和置信度分数
        |
        +--> 计算准确率和校准指标
        |
        +--> 分析不同视觉退化分组
        |
        +--> 应用 Platt Scaling
        |
        v
图表、数据表和最终报告
```

## 外部仓库与下载地址

### TextZoom 数据集

从以下仓库下载 TextZoom 数据集：

* [WenjiaWang0312/TextZoom](https://github.com/WenjiaWang0312/TextZoom)

### 文字识别模型实现

下载以下模型仓库：

* [ASTER — ayumiymk/aster.pytorch](https://github.com/ayumiymk/aster.pytorch)
* [MORAN — Canjie-Luo/MORAN_v2](https://github.com/Canjie-Luo/MORAN_v2)
* [CRNN — meijieru/crnn.pytorch](https://github.com/meijieru/crnn.pytorch)

## 使用的预训练权重

| 模型 | 权重文件 | 大约大小 |
| --- | --- | ---: |
| ASTER | `aster_demo.pth` | 81 MB |
| MORAN | `Moran_demo.pth` | 78 MB |
| CRNN | `crnn.pth` | 32 MB |

推荐的本地目录结构：

```text
checkpoints/
├── aster_demo.pth
├── Moran_demo.pth
└── crnn.pth
```

在 `Project.ipynb` 中更新路径：

```python
ASTER_CHECKPOINT = r"path/to/checkpoints/aster_demo.pth"
MORAN_CHECKPOINT = r"path/to/checkpoints/Moran_demo.pth"
CRNN_CHECKPOINT = r"path/to/checkpoints/crnn.pth"
```

## 运行项目

1. 下载并解压 TextZoom 数据集。
2. 下载 CRNN、ASTER 和 MORAN 的原始实现。
3. 获取对应的预训练模型权重。
4. 在 Notebook 开头附近更新数据集、模型仓库和权重文件路径。
5. 安装所需的 Python 软件包。
6. 打开 Notebook：

```bash
jupyter notebook Project.ipynb
```

7. 按顺序运行 Notebook 中的单元格。

当前 Notebook 中包含本地 Windows 路径。使用者必须将这些路径替换为与其本地环境相匹配的路径。

## 我的工作

我在本项目中的主要工作包括：

* 为三种不同识别模型构建统一评估流程
* 对不同模型词表中的预测结果进行标准化
* 定义可比较的单词级置信度分数
* 计算单词准确率、ECE、Brier Score 和 HCER
* 识别高置信度错误预测
* 测量模糊程度、对比度和背景复杂度
* 比较模型在不同视觉条件分组下的可靠性
* 训练并评估 Platt Scaling 校准模型
* 生成可靠性图、对比图和最终报告

## 局限性

* 主要评估仅使用 TextZoom 的 `test/medium` 子集。
* 模糊子组的观测数量很少，限制了模糊分析结论的可靠性。
* 视觉条件分组基于人工设计的图像统计指标，而不是人工验证的退化标签。
* 单词置信度的构造方式与原始识别模型自身的置信度定义不同。
* HCER 所使用的 0.8 阈值便于比较，但仍然属于人为设计选择。
* 校准效果仍需在其他数据集和实际部署条件下进一步验证。

## 归属说明

本仓库不主张拥有 TextZoom 数据集、CRNN、ASTER 和 MORAN 模型架构及预训练权重的原创权利。

应保留并遵守原始模型实现、论文、许可证和版权声明。本仓库主要关注围绕这些模型构建的评估、比较、视觉条件分析和置信度校准流程。

## 参考文献

1. Wang 等，*Scene Text Image Super-Resolution in the Wild*，ECCV，2020。
2. Shi、Bai 和 Yao，*An End-to-End Trainable Neural Network for Image-Based Sequence Recognition and Its Application to Scene Text Recognition*，IEEE TPAMI，2017。
3. Shi 等，*ASTER: An Attentional Scene Text Recognizer with Flexible Rectification*，IEEE TPAMI，2019。
4. Luo、Jin 和 Sun，*MORAN: A Multi-Object Rectified Attention Network for Scene Text Recognition*，Pattern Recognition，2019。

## 作者

**Songtao Zhang**
