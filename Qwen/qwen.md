# Qwen
千问系列模型，包含预训练模型、代码(SFT)、数学(SFT)、视觉语言模型（SFT）对话模型（RLHF）和奖励模型
![img.png](images/qwen.png)\

- 上下文长度扩展，推理阶段用NTK-aware 插值
- 两种注意力机制：LogN-Scaling和和窗口注意力
- 观察：较低层相比高层在扩展上下文长度时更为敏感
- 观察:训练方法对模型最终性能有显著影响。
### 数据
- 公共网络文档、百科全书、书籍、代码等
- 多语言，中英文为主
- 精确匹配去重以及使用 MinHash 和 LSH 算法进行模糊去重
- 使用语言模型、文本质量评分模型以及用于识别潜在冒犯或不适当内容的模型过滤低质量数据
- 3T tokens
### Tokenizer
 - BPE,tiktoken实现
 - 词表：cl100k base + 中文及其他语言增强，数字被拆分为单个数字，最后词表大小约152k
### 模型架构
 ![img.png](images/qwenmodel.png)
基于Llama,修改点包括：
- 非绑定（untied）嵌入
- RoPE，用FP32 精度用于逆频率矩阵
- 偏置，在注意力机制的 QKV 层中添加了偏置项
- Pre-Norm & RMSNorm
- SwiGLU,8/3d hidden size

### 训练设置
- context lengths:2024
- Flash Attention
- 优化器：AdamW，s β1 = 0.9, β2 = 0.95, ϵ = 10e−8
- cosine learning rate schedule，衰减至最大值的10%
- BFloat16 mixed precision

### 预训练
 - 使用多任务指令对语言模型进行预训练可以增强其零样本和少量样本学习的性能，因此在预训练过程中纳入了高质量的指令数据。

### SFT
- ChatML 风格的格式
- 对人类风格的对话进行标注
#### 训练设置
- batch_size 128
- step 4000(warm up  1430)
- weight decay 0.1
- dropout 0.1
- 梯度clip 1
- 最大学习率：2 × 10e−6

### post阶段训练
#### reward model
 - 使用与 QWEN 相同规模的预训练语言模型作为基础。
 - 在模型中加入一个池化层用于从特定的结束标记提取句子的奖励值。
 - 设置学习率为 3 × 10e−6，批次大小为 64，序列长度为 2048。
 - 训练过程持续一个周期。
#### ppo
 - 策略模型和价值模型的学习率分别设置为 1 × 10^−6 和 5 × 10^−6
 - 采取同时为每个查询采样两个响应的策略
 - 价值损失裁剪，裁剪值为 0.15
 - 策略 top-p 设定为 0.9
 - 实现了预训练梯度以减轻对齐税

### 工具使用
- Utilizing unseen tools through ReAct prompting 
- 使用 Python 代码解释器来增强数学推理、数据分析等功能
- 作为agent访问 Hugging Face 的大量多模态模型并与人类互动
##### 训练
- 自我指导 (self-instruct) 策略来进行监督微调 (SFT)
- 样本集大约包含 2000 个高质量样本
- 将这些高质量样本与所有其他通用 SFT 样本混合在一起

## CODE-QWEN
- 从基础模型 QWEN 开始，继续在大约 900 亿个代码token数据上预训练
- 3% 的预热迭代次数且不进行学习率衰减
- 训练模型以支持长达 8192 的上下文长度
- 学习率,CODE-QWEN-14B:6.0 × 10e−5 / CODE-QWEN-7B:3.0 × 10e−5
- sft阶段，发现多阶段监督微调 (SFT) 策略相比其他方法表现最佳

## MATH-QWEN
- 在增强的数学指导数据集上进行了数学监督微调 (SFT)
- 使用了 1024 的序列长度来进行更快的训练
- 数学 SFT 数据集中的大多数用户输入都是考试题目
- 学习率峰值设置为 2 × 10e−5 
- 训练步数为 50,000 


# Qwen2
![img.png](images/qwen2.png)
### Tokenizer
 - 同qwen
 - 所有模型都使用一个共同的词汇表，其中包括151,643个常规tokens和3个控制tokens。
### 模型架构
- 双块注意力机制(Dual Chunk Attention, DCA)
- YARN,重新缩放注意力权重
#### MOE
![img.png](images/moe.png)
- 粒度：细粒度专家（Dai等人, 2024），创建了更小规模的专家，同时激活更多的专家
- 路由：MoE层中整合共享专家和特定路由的专家（Rajbhandari等人, 2022; Dai等人, 2024）
- 初始化：upcycling（Komatsuzaki等人, 2023）的方式初始化专家

### 预训练
工作重心主要集中在精炼数据集和研究有效处理扩展上下文长度的方法上。
#### 精炼数据集
- 数据质量提升：过滤算法通过额外的启发式和基于模型的方法进行了改进，同时用Qwen模型合成高质量的预训练数据。
- 数据扩充：收集了大量高质量的代码、数学和多语言数据，支持大约30种语言。
- 分布改进：在缩小规模的模型上进行了实验，以优化来自各种来源和领域的数据混合。
- token数：7T,(放宽质量阈值的12万亿个tokens的数据训练的模型并没有显著的性能提升)。
- 高质量的多任务指令数据仍被集成到Qwen2的预训练过程中，以增强情境学习和指令跟随的能力。

#### 扩展上下文长度
- 在预训练的最后阶段将上下文长度从4,096个tokens增加到了32,768个tokens
- YARN机制和双块注意力机制
- 将RoPE的基频从10,000调整到了1,000,000


### post阶段训练
- 增强编程、数学、逻辑推理、指令跟随和多语言理解等领域的能力。
- 与人类价值观对齐。
- 研究了如何获取高质量的演示和偏好数据，用于SFT和RLHF，以最小化对人类标注的需求，同时最大化数据的质量和可靠性。
#### 数据
1.协作数据注释
- 自动本体提取：采用InsTag细粒度标签器，用于从大规模指令数据集中提取底层本体。随后的手动细化确保了提取本体的准确性。
- 指令选择：每个带有标签的指令都会根据标签多样性、语义丰富性、复杂性和意图完整性进行评估。基于这些标准，选择一组代表性指令。
- 指令进化：使Qwen模型向现有指令添加约束或要求，从而增加其复杂性，并确保数据集内有不同难度级别的多样性。
- 人类注释：使用多种生成策略和不同规模的Qwen模型获取每个指令的多个响应。注释者根据他们的偏好对这些响应进行排名，确保最佳响应符合既定的标准，从而产生了演示数据和偏好数据。

2.自动化数据合成

在大规模上保持指令响应注释的质量面临着重大挑战，特别是那些需要专业知识、经验、细心或耐心的任务。为了解决这些问题，Qwen team设计了各种自动对齐策略来规模化合成数据。
- 拒绝采样:对于数学或其他具有明确最终答案的任务，应用拒绝采样（Yuan等人, 2023）来提高解决方案的质量。
- 执行反馈:对于编程任务，LLM被用来生成解决方案及其相关的测试案例。对于具有特定约束条件（如长度限制）的每个指令，LLM被要求生成一个Python验证函数以确保响应符合指令的要求。
- 数据再利用:在文学写作任务中创建专业的响应非常具有挑战性。为了解决这个问题，Qwen team从公共领域收集高质量的文学作品，并利用LLM开发具有不同详细程度的指令。这些指令与原始作品配对，作为演示数据。
- 宪法反馈:宪法AI指的是引导LLM根据预定义的原则集生成响应的过程（Bai等人, 2022）。为了确保遵守诸如安全和价值观等准则，Qwen team编制了一个宪法数据集。该数据集界定了需要遵循的原则以及要避免的原则。它被用来指导LLM生成要么符合要么偏离这些准则的响应，作为演示和偏好数据的参考。
#### sft
- 构造一个庞大的指令数据集，包含超过500,000个示例，覆盖了诸如指令遵循、编程、数学、逻辑推理、角色扮演、多语种以及安全性等方面的技能。
- 使用32,768个tokens的序列长度进行了两轮微调。
- 为了优化学习过程，将学习率从7×10e-6逐渐降低到7×10e-7
- 为了应对过拟合问题，设置 drop out 0.1 ,梯度clip 1.0

#### RLHF
包含两个连续的阶段：离线训练和在线训练。
- 离线训练阶段，使用预先编译的偏好数据集来最大化正负样本之间的似然差（DPO方法）。
- 在线训练阶段，模型实时地迭代改进其性能，利用奖励模型提供即时反馈。具体来说，从当前策略模型中采样多个响应，奖励模型选择最偏好和最不偏好的响应，形成偏好对，这些偏好对在每一回合中用于DPO。
- 使用在线合并优化器（Online Merging Optimizer, Lu等人, 2024a）来缓解对齐税