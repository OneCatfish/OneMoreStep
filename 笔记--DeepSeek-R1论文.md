DeepSeek-R1-Zero和DeepSeek-R1，推理模型的OG，DeepSeek-R1-Zero 没有经过SFT，直接大量RL，虽然表现强大，但是有可读性差和语种混乱的情况，因此推出了DeepSeek-R1，它在RL之前有多阶训练和数据冷启动。DeepSeek-R1推理能力超越<font style="color:rgb(0,0,0);">OpenAI-o1-1217！甚至开源了6个基于DeepSeek-R1蒸馏的Qwen和Llama模型。</font>![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1738803960986-dff5574f-041c-48e0-a503-693d0941ee06.png)

<font style="color:rgb(0,0,0);"></font>

## **<font style="color:rgb(0,0,0);">Introduction</font>**
首先提出post-training有助于提升准确率、推理能力，比pre-training需要的训练资源更少。

OpenAI的o1（OpenAI，2024b）系列模型首次通过增加Chain-of-Thought过程的长度来引入推理时间缩放。这种方法对coding，mathematic，scientific reasoning有显著成效，但是，时间尺度仍然是个问题。

   	文章想探索（劈脸展示）一下如何用纯纯的RL，不加一丝有监督数据来提升大语言模型推理能力：具体是使用<font style="color:rgb(0,0,0);">DeepSeek-V3-Base模型，训练出的DeepSeek-R1-Zero，在经过几千步的RL训练后，pass@1 score on AIME 2024分数直接从15.6%飙升至71.0%！！！直接对齐OpenAI-o1-0912！</font>

<font style="color:rgb(0,0,0);">但是，纯RL训练出的DeepSeek-R1-Zero有可读性差和语言混乱的问题，于是团队推出了由少量cold-start data 和 multi-stage training pipeline组合训练的DeepSeek-R1。还是基于DeepSeek-V3-Base，先由几千个cold-start data微调，接着使用和DeepSeek-R1-Zero一样的reasoning-oriented RL，在RL接近收敛的时候，使用RL checkpoint拒绝采样来生成新的SFT数据（此步莫不是模型自我提升），结合DeepSeek-V3的写作、factual QA、self-cognition等领域的数据重新训练DeepSeek-V3模型。新数据微调后，考虑到所有的提示场景，再加一个RL过程，由此得到了DeepSeek-R1，比肩OpenAI-o1-1217。</font>

<font style="color:rgb(0,0,0);">基于DeepSeek-R1蒸馏了几个Qwen和Llama模型，效果也是相当的好。</font>

### **<font style="color:rgb(0,0,0);">Contributions</font>**
**post-training: 基于Base Model 大量的强化学习**

+ 无SFT的纯RL让模型探索了CoT（思维链）,里程碑式的探索   ------>   DeepSeek-R1-Zero
+ **两阶段的SFT**作为模型推理和非推理能力的种子，**两阶段的RL**旨在改进推理模式和对齐人类偏好   ------->DeepSeek-R1

**蒸馏：小模型也可以很强大**

+ 大模型的推理模式用于蒸馏小模型表现更好
+ 基于DeepSeek-R1生成的数据蒸馏了一众小模型

### 评估结果总结
+ **推理：**<font style="color:rgb(0,0,0);">DAIME2024上评分79.8%，轻超OpenAI-o1-1217；MATH-500得分97.3%，与OpenAI-o1-1217旗鼓相当；代码能力达到2029 Elo，超96.3%人类参与者；工程能力比DeepSeek-V3更强。</font>
+ **<font style="color:rgb(0,0,0);">知识：</font>**<font style="color:rgb(0,0,0);">90.8% on MMLU, 84.0% on MMLU-Pro, and 71.5% on GPQA Diamond，稍逊于OpenAI-o1-1217，超其他闭源模型。</font>
+ **<font style="color:rgb(0,0,0);">其他：</font>**<font style="color:rgb(0,0,0);">win-rate of 87.6% on AlpacaEval 2.0； win-rate of 92.3% on ArenaHard；</font>

## 方法
### 概述
以往的工作都是依赖大量的有监督数据来提升模型性能，本文就是告诉你纯RL就能显著提升推理性能，少量的冷启动数据就能再进一步提升。

### **<font style="color:rgb(0,0,0);">DeepSeek-R1-Zero：基于Base Model的强化学习</font>**
#### RL算法
**<font style="color:rgb(0,0,0);">Group Relative Policy Optimization （GRPO）</font>**<font style="color:rgb(0,0,0);background-color:#74B602;">节省RL开销</font><font style="color:rgb(0,0,0);">，放弃了通常于policy model大小相当的 critic model，而是从群体分数中评估基线。具体而言：对于一个问题q，DRPO从旧的策略𝜋𝜃𝑜𝑙𝑑中抽取一组输出{𝑜1，𝑜2，···，𝑜𝐺}，然后通过最大化以下目标来优化策略模型𝜋𝜃：</font>

![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739425545566-2a1849a2-6d59-4c24-8231-b48f83d2e63a.png)

来自[https://blog.csdn.net/qq_38961840/article/details/145384852](https://blog.csdn.net/qq_38961840/article/details/145384852)的解释：

关键点一：分组采样与相对奖励

关键点二：无需价值网络的高效策略优化

![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739415501784-27365b4a-7b7b-44b1-8dd8-7d61904b2110.png)

#### Reward Modeling
<font style="color:rgb(0,0,0);">DeepSeek-R1-Zero的训练采用了rule-based 奖励系统，包含两种rewards：</font>

+ <font style="color:rgb(0,0,0);">Accuracy rewards：判断response是否正确，如数学答案和代码编译结果</font>
+ <font style="color:rgb(0,0,0);">Format rewards：强制将思考结果放在‘<think>’ and ‘</think>’中</font>

没有采用基于神经网络reward model



#### Training Template
![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739436337499-c2e38271-e403-4eb3-8613-ad20374fe592.png)

限制这种格式，以便更好地观察RL进程

#### 性能、自我进化和Aha Moment
**<font style="color:rgb(0,0,0);">DeepSeek-R1-Zero</font>****性能：**

![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739436595257-072c4750-89c5-4389-aa23-4b89f9b68c6d.png)

![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739436721012-1a1978e0-58a2-4c21-ad69-34645ec85f7f.png)

**<font style="color:rgb(0,0,0);">DeepSeek-R1-Zero的自我进化过程：</font>**

<font style="color:rgb(0,0,0);">随着时间增长出现的复杂行为，例如反思之前的步骤，探索新的解决方法，这些行为是随着RL进行自然出现的，并非刻意编程为之，这大大增强了模型的推理能力。</font>

![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739438971893-872a51df-53d4-4500-a988-1d911db34a9c.png)

平均响应长度随训练增加，模型在逐渐使用更长的思考时间去回答问题

**<font style="color:rgb(0,0,0);">DeepSeek-R1-Zero的惊奇时刻：</font>**

<font style="color:rgb(0,0,0);">这个时刻出现在中间版本，模型通过重新评估初始方法来分配更多的思考时间给问题，这也让观察者为之一振，只给模型正确的激励，模型就会演化出先进的解题策略，这也提示着RL对人工智能的潜力。</font>

<font style="color:rgb(0,0,0);background-color:#74B602;">模型出现了专业的rethink过程</font>

![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739439647360-c3bf6a92-6b02-4348-bbc3-b43f1e04838d.png)

**<font style="color:rgb(0,0,0);">DeepSeek-R1-Zero的缺点：</font>**

可读性差，语言混用

### DeepSeek-R1 RL+冷启动
由R1-Zero的结果引发的两个思考：

1. 通过添加少量高质量冷启动数据，能否进一步提升推理性能或加速收敛？
2. 怎样才能训练出一个有清晰思考链、有强大通用能力的好用的模型？

由此引出DeepSeek-R1的train pipeline：

#### Clod Start
为防止RL前期冷启动的不稳定，构建了少量的长思考链数据，数据通过提示<font style="color:rgb(0,0,0);">DeepSeek-R1-Zero通过反思论证来生成答案。，再把输出格式化、最后人工后处理。通过这种方法生成了几千条cold-start数据，以微调DeepSeek-V3-Base作为强化学习的起点。与DeepSeek-R1-Zero相较，其有点在于：</font>

+ 可读性：针对R1-Zero的表现，设计了包含总结的可读范式，筛掉可读性差的数据，范式：  
<font style="color:rgb(0,0,0);background-color:#74B602;">|special_token|<reasoning_process>|special_token|<summary></font><font style="color:rgb(0,0,0);">，  
</font><font style="color:rgb(0,0,0);">reasoning process是query的思考链，summary是推理结果的总结。</font>
+ <font style="color:rgb(0,0,0);">潜力：通过cold-start的pattern，模型表现比R1-Zero好，由此相信迭代训练是更好地训练方法</font>

#### 面向推理的RL
<font style="color:rgb(0,0,0);">DeepSeek-V3-Base经由cold-start 微调后，再采用大量与训练R1-Zero相同的RL，通过良好的问题和清晰的回答，此过程旨在加强模型的推理能力，尤其是代码、数学、科学、逻辑推理。在此训练过程中存在语言混用的现象，在RL中加入了语言一致的奖励——通过计算目标语言在CoT中的比例实现，</font><font style="color:rgb(0,0,0);background-color:#74B602;">虽然此方法导致轻微的性能下降，但是却提升了可读性与人类偏好</font><font style="color:rgb(0,0,0);">。最后把推理准确性和语言一致性奖励合并一直训练到模型收敛。</font>

#### <font style="color:rgb(0,0,0);">拒绝采样和SFT</font>
面向推理的RL收敛时，使用checkpoint生成后续的SFT数据，SFT数据整合了其他领域的数据，主要提升模型的写作、角色扮演、通用目的任务。

**推理数据：**通过RL的checkpoint进行拒绝采样来管理推理prompts并生成推理轨迹，先前数据都是可通过rule-based奖励来评判，<font style="background-color:#74B602;">但是，在这一阶段，通过合并额外的数据来扩展数据集，其中一些数据使用生成式奖励模型，通过ground-truth和模型预测输入到DeepSeek-V3中进行判断</font>。此外，由于模型输出有时是混乱的，难以阅读，因此过滤了混合语言、长参数和代码块的思想链。对于每个提示，采样多个响应，并只保留正确的响应。我们总共收集了大约<font style="background-color:#74B602;">600k条</font>与推理相关的训练样本。

**非推理数据：**诸如写作、QA、自我认知、翻译的非推理数据，使用了DeepSeek-V3的pipeline和复用了一部分V3的SFT数据。<font style="background-color:#74B602;"> 对于非推理任务，使用V3在回答问题之前生成潜在的思维链，除了简单的query如“hello”这样的，不生成CoT</font>。总生成了将近<font style="background-color:#74B602;">200k条</font>非推理数据。

使用上述约800k个样本的数据集对DeepSeek-V3-Base进行了2个epoch微调。

#### 全场景RL
二次RL：提升模型帮助性和无害性同时精进推理能力。具体而言时使用reward signals结合不同的prompts分布。

**推理数据：**依然使用R1-Zero的rule-based方法。

**通用数据：**使用奖励模型去捕捉人类在复杂场景下的微妙偏好。以V3 pipeline为记住使用相似的偏好和prompts分布。

**帮助性：**只关注最终的总结，确保评估强调了对用户的响应的效用和相关性，同时最大限度地减少对底层推理过程的干扰。

**无害性：**评估模型的整个响应，包括推理过程和总结，以识别和减轻在生成过程中可能出现的任何潜在风险、偏见或有害内容。

**最终，奖励信号和不同的数据分布的整合使我们能够训练出一个擅长推理的模型，同时优先考虑帮助性和无害性。**

### 蒸馏：赋予小模型推理能力
直接微调开源模型，如Qwen和Llama 。研究结果表明，这种简单的蒸馏方法显著提高了较小模型的推理能力。这里使用的基本型号是Qwen2.5-Math-1.5B、Qwen2.5-Math-7B、qwen2.5 - 514 b、Qwen2.5-32B、Llama-3.1-8B、Llama-3.3-70B-Instruct。选择Llama-3.3是因为它的推理能力略好于Llama-3.1。

对于蒸馏模型，我们只应用SFT而不包括RL阶段，即使使用RL可以大大提高模型性能。这里的主要目标是证明蒸馏技术的有效性，将RL阶段的探索留给更广泛的研究界（狗头）。

## 牛批时刻
![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739504221293-12a083cc-417f-4dab-bdb5-02948ffbb62b.png)

蒸馏模型的表现：

![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739504303828-e3603854-e45f-46b0-b74b-3937276bfecf.png)

## 个人总结
![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739516981410-208d87ad-5156-4e6b-b799-f4c093d7f4bf.png)<font style="color:rgba(0, 0, 0, 0);">%3CmxGraphModel%3E%3Croot%3E%3CmxCell%20id%3D%220%22%2F%3</font>

<font style="color:rgba(0, 0, 0, 0);">E%3CmxCell%20id%3D%221%22%20pare</font>![](https://cdn.nlark.com/yuque/0/2025/png/34610865/1739517876946-7e16b578-3713-4a4f-aab6-508045a6be5c.png)<font style="color:rgba(0, 0, 0, 0);"></font>





