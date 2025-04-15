# method
## **<font style="color:rgb(0,0,0);">CAD Representation for Neural Networks</font>**
B-rep是指令序列的抽象，指令序列到B-rep是很容易的，但是反过来就很难，因为不同的指令序列可能会生成同样的B-rep。

指令序列是人类可理解的，容易编辑和在下游中使用。

### **<font style="color:rgb(0,0,0);">Specification of CAD Commands</font>**
![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1742265239236-d2c456b7-b580-4768-a003-43625d41ae1b.png)

### **<font style="color:rgb(0,0,0);">Network-friendly Representation</font>**
指令序列不同于自然语言，它带有指令和参数，参数又包含了连续和离散的参数，不便直接使用于神经网络。

对指令序列的维度进行规范：

    1. 首先，对于每个命令，其参数堆叠到一个16×1向量中，其元素对应于表1中所有命令的集合参数，每个命令的未使用参数被简单地设置为−1
    2. 接下来，我们固定每个CAD模型m中的命令总数Nc，这是通过用空命令（EOS）填充CAD模型的命令序列来完成的，直到序列长度达到Nc。在实践中，我们选择Ne = 60，这是训练数据集中出现的最大命令序列长度。

通过对连续参数的量化，实现了连续参数与离散参数的统一。对2 × 2 × 2立方体内的每个CAD模型进行归一化；我们还归一化了其边界框内的每个草图轮廓，并包括一个比例因子s（在挤压命令中）以将规范化的轮廓恢复到其原始尺寸。归一化限制了连续参数的范围，允许我们将它们的值量化为256个级别，并使用8位整数表示它们。因此，所有命令参数只具有离散的值集。

参数量化能更好地重视几何关系？？

## **<font style="color:rgb(0,0,0);">Autoencoder for CAD Models</font>**
![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1742278828249-8a85fbfa-3230-46e7-a763-6bf92bca8b20.png)

![image](https://cdn.nlark.com/yuque/__latex/d3b2f950ac03d2dad17b7f094e677ee6.svg)

![image](https://cdn.nlark.com/yuque/__latex/67ef5f42a38183066242c1a731013cff.svg)![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1742367419919-7233fd11-ebf1-41a9-9294-0c4ccc8cd95d.png)

![image](https://cdn.nlark.com/yuque/__latex/edb4f88a8a9c7d331c17ed6e25a61f19.svg)

**Encoder：**4层Transformer blocks，8头注意力机制，512维度feed forward，  
		输入为[e1, ...eNc]，输出为相同维度，最终平均为dE维向量Z

**Decoder：**超参设置与encoder相同，采用feed-forward策略

## Creation of CAD Dataset
178238条CAD command sequence 数据，训练-验证-测试 ---> 90% - 5% - 5%

构建脚本：[https://github.com/ChrisWu1997/onshape-cad-parser](https://github.com/ChrisWu1997/onshape-cad-parser)

## training and runtime generation
使用 标准的Cross-Entropy loss训练 **autoencoder**

![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1742433743062-5d30e526-04ef-464f-8757-33f78468359a.png)

![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1742433766312-0e9e5e9a-aa48-4f41-9661-a214a2aec16e.png)

+ _<font style="color:rgb(0,0,0);">Np </font>_<font style="color:rgb(0,0,0);">is the number of parameters (</font>_<font style="color:rgb(0,0,0);">Np </font>_<font style="color:rgb(0,0,0);">= 16 in our examples).</font>
+ _<font style="color:rgb(0,0,0);">β </font>_<font style="color:rgb(0,0,0);">is a weight to balance both terms (</font>_<font style="color:rgb(0,0,0);">β </font>_<font style="color:rgb(0,0,0);">= 2 in our examples).</font>
+ <font style="color:rgb(0,0,0);">Note that in the ground-truth command sequence, some commands are empty (i.e., the padding command </font>_<font style="color:rgb(0,0,0);"><</font>_<font style="color:rgb(0,0,0);">EOS</font>_<font style="color:rgb(0,0,0);">></font>_<font style="color:rgb(0,0,0);">)</font>
+ <font style="color:rgb(0,0,0);">and some command parameters are unused (i.e., labeled as </font>_<font style="color:rgb(0,0,0);">−</font>_<font style="color:rgb(0,0,0);">1).</font>
+ **<font style="color:rgb(0,0,0);">使用adam 优化器；</font>**
+ **<font style="color:rgb(0,0,0);">0.001的学习率；</font>**
+ **<font style="color:rgb(0,0,0);">2000步的线性warm-up；</font>**
+ **<font style="color:rgb(0,0,0);">0.1 Transformers block 的 dropout rate；</font>**
+ **<font style="color:rgb(0,0,0);">1000 epochs， batch size 512</font>**

autoencoder训练好之后就可以将模型表达为256维的向量 z. 生成CAD模型使用 latent-GAN。

生成器和鉴别器都像4层的MLP一样简单，采用梯度惩罚的Wasserstein-GAN策略进行训练。

最后，为了生成一个CAD模型，我们从一个多元高斯分布中抽取一个随机向量，并将其输入GAN的生成器。GAN的输出是一个潜在的向量 z 用来输入进decoder。

# 实验
## autoencoding od CAD Models
**<font style="color:rgb(0,0,0);">Metrics:  </font>**<font style="color:rgb(0,0,0);">Command  Accuracy and Parameter Accuracy</font>

使用_<font style="color:rgb(0,0,0);">Chamfer Distance </font>_<font style="color:rgb(0,0,0);">(CD)来评估</font>

# 实践
## 训练encoder
```shell
python train.py --exp_name newDeepCAD -g 0			# experiment name 和 指定GPU ID
```

```shell
# encode all data to latent space
python test.py --exp_name newDeepCAD --mode enc --ckpt 1000 -g 0
```

```shell
# train latent GAN (wgan-gp)
python lgan.py --exp_name newDeepCAD --ae_ckpt 1000 -g 0
```

