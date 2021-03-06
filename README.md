# model-compression

"目前在深度学习领域分类两个派别，一派为学院派，研究强大、复杂的模型网络和实验方法，为了追求更高的性能；另一派为工程派，旨在将算法更稳定、高效的落地在硬件平台上，效率是其追求的目标。复杂的模型固然具有更好的性能，但是高额的存储空间、计算资源消耗是使其难以有效的应用在各硬件平台上的重要原因。所以，卷积神经网络日益增长的深度和尺寸为深度学习在移动端的部署带来了巨大的挑战，深度学习模型压缩与加速成为了学术界和工业界都重点关注的研究领域之一"


## 项目简介 

基于pytorch实现模型压缩（1、量化：8/4/2 bits(dorefa)、三值/二值(twn/bnn/xnor-net)；2、剪枝：正常、规整、针对分组卷积结构的通道剪枝；3、分组卷积结构；4、针对特征A二值的BN融合）


## 目前提供

- 1、普通卷积和分组卷积结构
- 2、权重W和特征A的训练中量化, W(32/8/4/2bits, 三/二值) 和 A(32/8/4/2bits, 三/二值)任意组合
- 3、针对三/二值的一些tricks：W二值/三值缩放因子，W/grad（ste、saturate_ste、soft_ste）截断，W三值_gap(防止参数更新抖动)，W/A二值时BN_momentum(<0.9)，A二值时采用B-A-C-P可比C-B-A-P获得更高acc
- 4、多种剪枝方式：正常剪枝、规整剪枝（比如model可剪枝为每层剩余filter个数为N(8,16等)的倍数）、针对分组卷积结构的剪枝（剪枝后仍保证分组卷积结构）
- 5、batch normalization的融合及融合前后model对比测试：普通融合（BN层参数 —> conv的权重w和偏置b）、针对特征A二值的融合（BN层参数 —> conv的偏置b)


## 代码结构

![img1](https://github.com/666DZY666/model-compression/blob/master/readme_imgs/code_structure.jpg)

## 环境要求

- python >= 3.5
- torch == 1.1.0
- torchvison >= 0.3.0
- numpy

## 使用

### 量化

#### W（FP32/三/二值）、A（FP32/二值）

--W --A, 权重W和特征A量化取值

```
cd WbWtAb
```

- WbAb

```
python main.py --W 2 --A 2
```

- WbA32

```
python main.py --W 2 --A 32
```

- WtAb

```
python main.py --W 3 --A 2
```

- WtA32

```
python main.py --W 3 --A 32
```

#### W（FP32/8/4/2 bits）、A（FP32/8/4/2 bits）

--Wbits --Abits, 权重W和特征A量化位数

```
cd WqAq
```

- W8A8

```
python main.py --Wbits 8 --Abits 8
```

- W4A8

```
python main.py --Wbits 4 --Abits 8
```

- W4A4

```
python main.py --Wbits 4 --Abits 4
```

- 其他bits情况类比

### 剪枝

稀疏训练 ——> 剪枝 ——> 微调

```
cd prune
```

#### 正常训练

```
python main.py
```

#### 稀疏训练

-sr 稀疏标志, --s 稀疏率(需根据dataset、model情况具体调整)

- nin(正常卷积结构)

```
python main.py -sr --s 0.0001
```

- nin_gc(含分组卷积结构)

```
python main.py -sr --s 0.001
```

#### 剪枝

--percent 剪枝率, --normal_regular 正常、规整剪枝标志及规整剪枝基数(如设置为N,则剪枝后模型每层filter个数即为N的倍数), --model 稀疏训练后的model路径, --save 剪枝后保存的model路径（路径默认已给出, 可据实际情况更改）

- 正常剪枝

```
python normal_regular_prune.py --percent 0.5 --model models_save/nin_preprune.pth --save models_save/nin_prune.pth
```

- 规整剪枝

```
python normal_regular_prune.py --percent 0.5 --normal_regular 8 --model models_save/nin_preprune.pth --save models_save/nin_prune.pth
```

或

```
python normal_regular_prune.py --percent 0.5 --normal_regular 16 --model models_save/nin_preprune.pth --save models_save/nin_prune.pth
```

- 分组卷积结构剪枝

```
python gc_prune.py --percent 0.4 --model models_save/nin_gc_preprune.pth
```

#### 微调

--refine 剪枝后的model路径（在其基础上做微调）

```
python main.py --refine models_save/nin_prune.pth
```

### 剪枝 —> 量化（注意剪枝率和量化率平衡）

剪枝完成后,加载保存的模型参数在其基础上再做量化

#### 剪枝 —> 量化（8/4/2 bits）（剪枝率偏大、量化率偏小）

```
cd WqAq
```

- W8A8
- nin(正常卷积结构)

```
python main.py --Wbits 8 --Abits 8 --refine ../prune/models_save/nin_refine.pth
```

- nin_gc(含分组卷积结构)

```
python main.py --Wbits 8 --Abits 8 --refine ../prune/models_save/nin_gc_refine.pth
```

- 其他bits情况类比

#### 剪枝 —> 量化（三/二值）（剪枝率偏小、量化率偏大）

```
cd WbWtAb
```

- WbAb
- nin(正常卷积结构)

```
python main.py --W 2 --A 2 --refine ../prune/models_save/nin_refine.pth
```

- nin_gc(含分组卷积结构)

```
python main.py --W 2 --A 2 --refine ../prune/models_save/nin_gc_refine.pth
```

- 其他取值情况类比

### BN融合

```
cd WbWtAb/bn_merge
```

--W 权重W量化取值(据训练时W量化(FP/三值/二值)情况而定)

#### 融合并保存融合前后model

```
python bn_merge.py --W 2
```

或

```
python bn_merge.py --W 3
```

#### 融合前后model对比测试

```
python bn_merge_test_model.py
```

### 设备选取

现支持cpu、gpu(多卡、单卡)

--cpu 使用cpu，--gpu_id 选择gpu

- cpu

```
python main.py --cpu
```

- gpu单卡

```
python main.py --gpu_id 0
```

或

```
python main.py --gpu_id 1
```

- gpu多卡

```
python main.py --gpu_id 0,1
```

或

```
python main.py --gpu_id 0,1,2
```

默认：使用服务器全卡

## 模型压缩数据对比（示例）

注：以下为cifar10测试，可在更冗余模型、更大数据集上尝试其他组合压缩方式

|            类型             |  Acc   | GFLOPs | Para(M) | Size(MB) | 压缩率 | 损失  |                        备注                        |
| :-------------------------: | :----: | :----: | :-----: | :------: | :----: | :---: | :------------------------------------------------: |
|        原模型（nin）        | 91.01% |  0.15  |  0.67   |   2.68   |  ***   |  ***  |                       全精度                       |
|  采用分组卷积结构(nin_gc)   | 90.88% |  0.15  |  0.58   |   2.32   | 13.43% | 0.13% |                       全精度                       |
|            剪枝             | 90.26% |  0.09  |  0.32   |   1.28   | 44.83% | 0.62% |                       全精度                       |
|        量化(W/A二值)        | 90.02% |  ***   |   ***   |   0.18   | 92.21% | 0.86% |                  W二值含缩放因子                   |
|      量化(W三值/A二值)      | 87.68% |  ***   |   ***   |   0.26   | 88.79% | 3.20% | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |
|   剪枝+量化(W三值/A二值)    | 86.13% |  ***   |   ***   |   0.19   | 91.81% | 4.75% | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |
| 分组+剪枝+量化(W三值/A二值) | 86.13% |  ***   |   ***   |   0.19   | 92.91% | 4.88% | W三值不含缩放因子,适配一些无法做缩放因子运算的硬件 |

## 相关论文

### 量化

#### 二值

- [BinarizedNeuralNetworks: TrainingNeuralNetworkswithWeightsand ActivationsConstrainedto +1 or−1](https://arxiv.org/abs/1602.02830)

- [XNOR-Net:ImageNetClassiﬁcationUsingBinary ConvolutionalNeuralNetworks](https://arxiv.org/abs/1603.05279)

- [AN EMPIRICAL STUDY OF BINARY NEURAL NETWORKS’ OPTIMISATION](https://openreview.net/forum?id=rJfUCoR5KX)

- [A Review of Binarized Neural Networks](https://www.semanticscholar.org/paper/A-Review-of-Binarized-Neural-Networks-Simons-Lee/0332fdf00d7ff988c5b66c47afd49431eafa6cd1)

#### 三值

- [Ternary weight networks](https://arxiv.org/abs/1605.04711)

#### 任意位数

- [DOREFA-NET: TRAINING LOW BITWIDTH CONVOLUTIONAL NEURAL NETWORKS WITH LOW BITWIDTH
  GRADIENTS](https://arxiv.org/abs/1606.06160)

### 剪枝

- [Learning Efficient Convolutional Networks through Network Slimming](https://arxiv.org/abs/1708.06519)
- [RETHINKING THE VALUE OF NETWORK PRUNING](https://arxiv.org/abs/1810.05270)

### 针对专用芯片的模型压缩

- [Convolutional Networks for Fast, Energy-Efficient Neuromorphic Computing](https://arxiv.org/abs/1603.08270)

## 后续补充

- 1、使用示例部分细节说明
- 3、imagenet测试

## 后续扩充

- 1、Nvidia、Google的INT8量化方案
- 2、对常用检测模型做压缩
- 3、部署（1、针对4bits/三值/二值等的量化卷积；2、终端DL框架（如MNN，NCNN，TensorRT等））
