Llama3   105B个参数  最多可处理128K的上下文窗口信息

---

## 引言
现代基础模型开发包括两个阶段：

1. 训练前阶段，利用下一个单词预测或caption（图像描述）等简单任务对模型进行大规模训练
2. 训练后阶段，对模型进行调整，使其遵循指令、符合人类偏好并提高特定能力（如编码和推理）

---

开发高质量基础模型三个关键杠杆：**数据、规模、复杂性管理**

1. 数据：与之前的Llama相比，提升了数量和质量、包含15.6T多语言词库预料
2. 规模：405B个参数模型， 预训练时使⽤了3.8 ×10^25FLOPs ，是最⼤版本Llama2的50倍  
3. 管理复杂性(managing complexity)： 选择 了标准的密集变形模型架构并调整，而不是使用专家混合模型，我们采⽤了基于监督微调 （SFT）、拒绝采样（RS）和直接偏好优化（DPO）的相对简单的后训 练程序，而不是更复杂的强化学习算法，后者 往往稳定性较差且难以扩展。  

---

Llama3语言模型的开发分为两个阶段：

1. pre-training：语料库转换为离散的tokens，执行next token prediction，预训练阶段using a context window of 8K tokens，在此基础上 **continued pre-training，窗口扩大到128K**
2. plot-training：<font style="color:rgb(42, 43, 46);">将模型在人类反馈进行了几轮调整，每轮包都含了在 instruction tuning data 上的SFT和DPO（Direct Preference Optimization）。</font>**<font style="color:rgb(42, 43, 46);">在训练后阶段整合了工具使用等新功能</font>**

---

## **总体概述**
实验添加了图像、视频、语音功能

+ Multi-modalencoderpre-training  ：为图形和语音分别训练编码器，通过图像-文字对训练， 让模型了解视觉内容与⾃然语⾔描述之间的关系 。语音编码器使用子监督方式训练，尝试用过离散标记表示方法，mask部分语音输入内容，并重建。
+  Vision adapter training ： 训练⼀个适配器，将预先训练好的图像编码器集成到预先训练好的语⾔模型中。  适配器根据⽂本-图像 对进⾏训练，使图像表征与语⾔表征保持⼀致。在适配器训练过程中，也会更新图像编码器的参数，但有意不更新语⾔模型参数。   还在成对的视频-⽂本数据上训练视频适配器 。
+  Speech adapter training ：语音adapter将语音编码转成token representation， 在语⾳适配器训练过程中，不更新语⾔模型 ，集成了text-to-speech系统

---

## **预训练**
+ 大规模训练语料整理和筛选
+ 模型结构开发和确定scaling laws
+ dfficinent pre-training techniques
+ pre-training recipe

### 预训练数据
#### Web Data Curation 
+ **PII and safety filtering** 
+  **Text extraction andcleaning**：  
对⾮截断⽹⻚⽂档的原始HTML内容进⾏处理，以提取⾼质量的多样化⽂本； 构建了⼀个⾃定义解析器，⽤于提取HTML内容 ，对包含数学和代码内容的HTML⻚⾯进⾏了仔细处理，以保留这些内容的结构 ； 保留了图⽚的alt属性⽂本，因为数学内容通常以预渲染图⽚的形式呈现 ；We find markdown is harmful to the performance of a model，so we remove all markdown markers； 
+ **去重**： 
    - URL-level de-duplication   保留每个URL对应⻚⾯的最新版本。  
    - Document-levelde-duplication  使用 global MinHash  删除近乎重复文档
    - Line-level de-duplication   删除了在每个3000万⽂档桶中出现6次 以上的⾏  
+  **Heuristic filtering**：
    -  duplicated n-gram coverage ratio  删除诸如报错log等重复信息的很长的行
    -  “dirty word” counting  过滤成人网站
    -  token-distribution Kullback-Leibler divergence  过滤含过多利群标记的文档
+  **Model-based quality filtering：fasttext、Roberta等分类**器
+ ** Code and reasoning data**：代码和推理数据，特定提取和过滤
+  **Multilingual data  **
    - 使用 fasttext-based  模型将文本归类为<font style="background-color:#FBDE28;">176种语言</font>
    - 每种语言中执行<font style="background-color:#FBDE28;">文档级</font>和<font style="background-color:#FBDE28;">行级</font>重复删除
    -  language-specific heuristics and model-based filters <font style="background-color:#FBDE28;"> 过滤低质文档</font>
+ 使用了Llama 2的多语言分类器对文档<font style="background-color:#FBDE28;">质量排序</font>，确保高质量的文档得到优先处理

#### Determining the Data Mix  
    要获得⾼质量的语⾔模型，确定不同数据源在预训练数据混合中的⽐例⾄关重要。

+ **Knowledge classification：**知识分类，以更有效地确定知识混合
+ **Scaling laws for data mix**：在小模型上训练混合数据，用来预测大模型上的混合性能
+ **Data mix summary**：

<font style="background-color:#FBDE28;">50% general knowledge</font>

<font style="background-color:#FBDE28;">25% mathematical and reasoning</font>

<font style="background-color:#FBDE28;">17% code</font>

<font style="background-color:#FBDE28;">8% multilingual</font>

#### 退火数据 Annealing Data
     在⼩量的⾼质量代码和数学数据上进⾏退⽕（⻅第3.4.3节）可以提⾼预训练模型 在关键基准上的性能。  

### 模型架构
Llama3结构与Llama，Llama2无显著差异，性能提升主要是由于数据质量和多样性的提升以及训练规模的增加。

![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1727249916374-aede2bfd-7a37-4db1-9f29-73e614fdaa99.png)

+ 使用了 <font style="background-color:#FBDE28;">grouped query attention  QGA</font>和<font style="background-color:#FBDE28;">8 key-value heads</font> 提高推理速度，并在解码过程中减少键值缓存的大小
+ 使用注意力掩码防止在同一序列中的两篇不同的文档做自注意力（这个改变在标准的预训练中影响有限，但在很长的序列的continued pre-training中很重要）
+ 词汇表128K，其中100K来自tiktoken，增加了28K用以支持非英语语言，英文token压缩率从llama2的3.17提高到3.94（也就是说平均每个token包含3.94个字母characters），也使得每个token包含更多的信息，增加的非英文tokens在不影响英文token的条件下提高了压缩比和下游性能
+ <font style="background-color:#FBDE28;">增加了RoPE的基础频率超参数</font>到500000

#### Scaling Laws
1. ⾸先建⽴计算最优模型在下游任务上的负对数似然与训练FLOPs之间的相关性。
2. 将下游任务上的负对数似然与任务准确性相关联，利⽤规模定律模型和⽤更⾼计算 FLOPs训练的旧模型。![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1727339913046-d6a1fb4b-1c4c-4780-ac42-a309ef954a53.png)

随着计算预算的增加，IsoFLOPs曲线在最小值周围的曲率变平，这意味着旗舰模

型的性能对于模型⼤⼩和训练标记之间权衡的微⼩变化相对鲁棒。

### Infrastructure, Scaling, and Efficiency
llama3 405B

#### Training Infrastructure
+ **Compute：**16K个H100 GPUs，每块GPU700W的TDP，80GB HBM3，每个服务器配备8块卡，服务器内部卡之间用NVLink连接，训练作业使用MAST调度。
+ **Storage**：Meta 通用分布式文件系统：7500个SSD服务器总240PB存储空间，2TB/s的持续吞吐量，7TB/s的峰值吞吐量
+ **Network：**<font style="color:rgb(31,35,41);">Llama 3 405B使⽤了基于Arista 7800和Minipack2开放计算项⽬OCP机架交换机的RDMA over Converged Ethernet (RoCE)⽹络。Llama 3系列中较⼩的模型使⽤Nvidia Quantum2 Infiniband⽹络进⾏训练。RoCE和Infiniband集群都在GPU之间利⽤400Gbps的互连。</font>

![](https://cdn.nlark.com/yuque/0/2024/webp/34610865/1728458474203-76813163-3bfa-4ae8-97e3-e5be687dcd0d.webp)

Clos三层架构

    - **<font style="color:rgb(0,0,0);">Network topology：</font>**<font style="color:rgb(0,0,0);">RoCE-based AI cluster由24K GPUs组成，通过三层Clos网络连接：在底层，每个底层机架托管16个GPU（这16个GPU存在2个服务器中，每个服务器包含8块GPU，通过Minipack2交换机连接）。</font><font style="color:rgb(31,35,41);">在中层，192个这样的机架通过集群交换机连接，形成⼀个拥有3,072个GPU的 pod，具有全双工带宽，确保机架间的数据传输不受限制 。在顶层，同一building中的8个这样的pod通过聚合交换机连接。在聚合连接层没有全双工带宽，而是有1:7的oversubscription比率。在3.3.2节模型并行方法和训练作业调度器中有针对优化，旨在最小化跨pod的网络通信。</font>
    - **<font style="color:rgb(31,35,41);">Load balancing</font>**<font style="color:rgb(31,35,41);">：1.库创建了16个⽹络流进⾏负载均衡。2.我们</font><font style="color:rgb(0,0,0);">Enhanced</font><font style="color:rgb(31,35,41);">-ECMP（E-ECMP）协议通过在RoCE数据包头部的额外字段上进行哈希处理，有效地在不同网络路径上平衡了 这16个流。</font>
    - **<font style="color:rgb(0,0,0);">Congestion control：</font>**<font style="color:rgb(0,0,0);">在主干网上使用了深度缓冲交换机</font>

#### **<font style="color:rgb(0,0,0);">Parallelism for Model Scaling</font>**
采用4D并行技术，结合4中不同的并行方法来分割模型：<font style="background-color:#FBDE28;">tensor parallelism(TP)张量并行</font>, <font style="background-color:#FBDE28;">pipeline parallelism(PP)流水线并行</font>, <font style="background-color:#FBDE28;">context parallelism(CP)上下文并行</font>, <font style="background-color:#FBDE28;">data parallelism(DP)数据并行</font>。

![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1733190232833-1a6e5f19-9a1f-4d12-a737-9a0fc700cf3f.png)

+ tensor parallelism：将单独权重分割成不同设备上的多个块。 
+ pipeline parallelism：通过层将将模型追至划分为不同的阶段。
+ context parallelism：将输入上下文分割成段，减少了长序列输入的内存瓶颈。
+ data parallelism：使用的是FSDP(<font style="color:rgb(0,0,0);">fully sharded data parallelism</font>)，它在实现数据并行的同事，对模型、优化器和梯度进行分片，在多个GPU上并行处理，<font style="background-color:#FBDE28;">并在每个training step后进行同步</font>。

Llama3中使用FSDP对optimizer states和梯度分片，但是对于，模型分片，在前向计算后不重新分片，以避免在反向转播期间产生额外的全收集通信。

**流水线并行改进：**

+ batch size限制：每个GPU的batch size要求被pipeline阶段的数量整除，  
例，在下图中若深度优先调度(DFS)需要N=PP=4, 而广度优先调度(NFS)需要N=M

M: <font style="color:rgb(0,0,0);">total number of micro-batches </font>

<font style="color:rgb(0,0,0);">N：number of contiguous micro-batches for the same stage’s forward or backward.</font>

<font style="color:rgb(0,0,0);">但是，预训练通常需要灵活调整batch size</font>

![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1733211704981-25532c47-2e21-4a94-ba34-2516f66baca8.png)

+ 内存不平衡：现有pipeline parallelism导致资源消耗不平衡，由于embedding和warm-up micro-batches第一阶段会消耗更多的内存
+ 计算不平衡：模型的最后一层之后，计算输出和损失，是延迟的瓶颈

解决方法：

1. 如上图，修改pipeline schedule允许灵活设置N，可在每个batch中设置任意micro-batch。使得：

（1）当batch size有限制时，减小micro-batch而不是stages

或者（2）增大micro-batch去隐藏点对点通信，寻找DFS和BFS的甜蜜点。

2. 为了平衡pipeline，first stage和 last stage各减少一个transformer层，这意味着第一阶段的第一个model chunk仅有嵌入层，最后阶段的最后一个model chunk仅有输出和损失计算。
3. 使用交错调度<font style="color:rgb(0,0,0);">with </font>_<font style="color:rgb(0,0,0);">V </font>_<font style="color:rgb(0,0,0);">pipeline stages on one pipeline rank 以减少pipeline bubbles。</font>
4. PP中采用异步点对点通信，<font style="color:rgb(31,35,41);">这在⽂档掩码引⼊额外计算不平衡的情况下显著加快了训练速度。</font>
5. <font style="color:rgb(31,35,41);">主动释放未来不会使用的张量，包括每个流⽔线阶段的输⼊和输出张量。</font>

**<font style="color:rgb(0,0,0);">Context parallelism for long sequences：</font>**

<font style="color:rgb(0,0,0);">使用CP(Context Parallelism)拓展上下文长度时提升内存效率，并使训练长度达到128k。</font><font style="color:rgb(31,35,41);">将输⼊序列划分为2 × CP块，以便每个CP等级接收两个块以实现更好的负载平衡。第i个CP等级接收了第i个和第(2×CP−1−i)个块。 </font>

**<font style="color:rgb(0,0,0);">Network-aware parallelism configuration：</font>**

<font style="color:rgb(0,0,0);">最内层并行需要最高的网络带宽和最低的延迟，因此通常限制在同一个服务器内。最外层的并行可以跨越multi-hop network，并能容忍更高的网络延迟。并行顺序 由内到外：TP-->CP-->PP-->DP。</font><font style="color:rgb(31,35,41);">DP（即FSDP）是最外层的并行，因为它可以通过异步预取分片模型权重和减少梯度来 容忍更长的网络延迟。</font>

**数值稳定性：**FP32

### <font style="color:rgb(0,0,0);">Training Recipe</font>
Llama3的预训练策略主要有三个阶段：

1. initial pre-training 
2. long-context pre-training
3. annealing（退火）

#### initial pre-training
使用余弦学习率预训练，峰值学习率为8×10^-5， 线性预热8000步，然后在1,200,000个训练步中衰减到8×10^-7。初期使用较小的batch size以提高稳定性，后续增加以提高效率。

![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1733743806489-394cd349-f855-4ccd-a94f-2de36ef97cb4.png)

数据混合：在预训练期间：

调整了几次数据混合  --->  提升特定下游任务性能

增加了非英语数据  --->  提高多语言性能

上采样数学数据  --->  提高模型数学推理性能

后期追加新数据，对识别出低质量数据下采样

#### Long Context Pre-training
在预训练的最后阶段，使用长序列训练以支持128K token的context窗口。

从8Kcontext window   历经6个阶段    达到128K context window（期间训练了8000B个token）

#### Annealing
最后的40M token训练中，学习率线性退火至0， 同时保持128K tokens context，同时调整数据混合。

## Post-Training
通过check point的几轮后训练 or 基于check point的human feedback对齐模型。每轮后训练包括SFT,DPO等。



### Modeling
post-training的backbone是reward model和language model。

首先，使用人工标注的偏好数据，在pre-training的check point上训练reward model。

然后，使用SFT对check point微调，并进一步做DPO

![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1733918187219-ee70e4c3-f800-4394-ae95-6664df70cb12.png)

#### Chat Dialog Format
为实现大模型与人类交互，需要定义一个dialog protocol以理解人类的指令并执行对话任务。llama3的<font style="background-color:#FBDE28;">工具使用</font>能力，需要在单个对话中生成多个消息并发送到不同位置(如ipython)。为支持这点，设计了<font style="color:rgb(0,0,0);">multi-message chat protocol，它使用了各种特殊的 header and termination tokens,</font>

**<font style="color:rgb(0,0,0);">header tokens：</font>**<font style="color:rgb(0,0,0);"> 用于指示对话中每条消息的来源和目的地</font>

**<font style="color:rgb(0,0,0);">termination tokens： </font>**<font style="color:rgb(0,0,0);">用于标记何时去交换人类和AI的发言。</font>

#### <font style="color:rgb(0,0,0);">Reward Modeling</font>


<font style="color:rgb(0,0,0);"></font>















