---
publish: true
created: 2026-11-11T01:44:08.175Z
modified: 2026-11-11T06:24:32.626Z
---

在训练 ResNet 模型时，可以采用两个阶段训练（Two-stage Training），但并不是必须的。是否有必要，主要取决于使用的是预训练 ResNet，还是从零开始训练，以及数据集的规模。

方案：

1. Stage 1：冻结 ResNet Backbone，只训练 FC（全连接分类层）。
2. Stage 2：解冻所有网络参数，进行端到端 Fine-tuning。

这是计算机视觉领域非常经典的迁移学习（Transfer Learning）策略，尤其适合使用 ImageNet 预训练权重的 ResNet。

## 1. 两阶段训练的原理

假设使用 ResNet50：
![[publish/垂类模型训练/resources/Pasted image 20261111141411.png]]

ResNet 可以分为两个部分：

- Backbone：卷积层、Residual Blocks，负责提取图像特征。
- Head：Global Average Pooling + FC，负责将特征映射到具体分类。

### Stage 1：只训练 FC

冻结 Backbone 参数：

$$
\theta_{backbone}=\text{constant}
$$

只更新分类层：

$$
\theta_{FC}^{t+1} = \theta_{FC}^{t}-\eta\nabla_{\theta_{FC}}L
$$

这样做的目的：

- 保留预训练模型已经学习到的特征。
- 让 FC 首先适应你的数据集类别。
- 避免训练初期随机初始化的 FC 层产生较大的梯度，破坏预训练特征。
- 减少训练参数，降低计算量。

### Stage 2：解冻所有参数

解除 Backbone 冻结，进行端到端训练：

$$
 \theta^{t+1}=\theta^t-\eta\nabla_{\theta}L
$$

这样做的目的：

- 让 Backbone 适应目标数据集的图像特征。
- 学习与具体分类任务更相关的特征。
- 提升模型在目标数据集上的分类能力。

需要注意：第二阶段不意味着必须使用相同的学习率。通常 Backbone 使用更小的学习率，FC 使用相对较大的学习率。

## 2. 是否应该训练所有参数？取决于数据集

|训练场景|推荐策略|
|---|---|
|ImageNet 预训练 + 小数据集|两阶段训练|
|ImageNet 预训练 + 中等数据集|两阶段或部分解冻|
|ImageNet 预训练 + 大数据集|可以直接 Fine-tuning|
|从零开始训练（Random Initialization）|通常直接训练全部参数|
|预训练领域与目标领域差异较大|建议逐步解冻更多层|

这里有一个值得特别注意的情况：

如果你的数据是金相显微图像，ImageNet 预训练的 ResNet 与金相图像存在一定的 Domain Gap。

ImageNet 主要是自然图像，而金相图像主要包含：

- 晶粒边界和轮廓。
- 铁素体、珠光体等组织纹理。
- 石墨球、夹杂物等微观结构。
- 灰度、形态和局部纹理特征。

因此，ImageNet 预训练的浅层特征仍然有价值，但高层特征未必完全适合金相分类任务。

对于金相组织分类任务，更倾向于采用两阶段训练，甚至三阶段训练。

## 3. 更推荐的训练策略

Stage 1 — Train FC
冻结 Backbone

- Backbone：Frozen
- FC：Trainable
- Learning Rate：1e-3
- Epoch：3～5

Stage 2 — Partial Fine-tuning
解冻 Layer4

- Layer1～Layer3：Frozen
- Layer4：Trainable
- FC：Trainable
- Layer4 LR：1e-4
- FC LR：1e-3
- Epoch：5～10

Stage 3 — Full Fine-tuning（可选）

全部解冻

- Layer1～Layer4：Trainable
- FC：Trainable
- Layer1～Layer4 LR：1e-5
- FC LR：1e-4
- Epoch：根据验证集表现决定

这种方式比直接从 FC-only 跳到全部参数解冻更加平滑。

相关参考：[PyTorch Transfer Learning 教程](https://wlos.github.io/AI_SS/Session12%20Building%20and%20Training%20CNN%20Models%20with%20PyTorch.html?utm_source=chatgpt.com)、[PyTorch Lightning Fine-tuning](https://media.lightning.ai/docs/pytorch/stable/advanced/finetuning.html?utm_source=chatgpt.com)。
