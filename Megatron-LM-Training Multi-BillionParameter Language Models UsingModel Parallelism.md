
# Megatron-LM-Training Multi-BillionParameter Language Models UsingModel Parallelism
# 摘要

近期在语言模型方面的研究表明，训练大型 Transformer 模型能推进自然语言处理应用的技术前沿。然而，非常大的模型由于内存限制，可能相当难以训练。在本研究中，我们展示了我们用于训练大型 Transformer 模型的技术，并实现了一种简单高效的层内模型并行方法，这使得我们可以训练含有数十亿参数的 Transformer 模型。我们的方法不需要新的编译器或库改变，与流水线模型并行是正交且互补的，并且可以通过在 PyTorch 中插入少量通信操作来完全实现。我们通过用512个 GPU 训练高达83亿参数的 Transformer 模型来展示这种方法。我们在整个应用中保持了15.1 PetaFLOPs 的性能，与单个 GPU 的基线相比，达到了76%的扩展效率，而单个 GPU 的基线维持在 39 TeraFLOPs，占峰值 FLOPs 的30%。为了证明大型语言模型可以进一步推进技术前沿（SOTA），我们训练了一个与 GPT-2 类似的 83 亿参数的 Transformer 语言模型，以及一个与 BERT 类似的 39 亿参数模型。我们表明，当模型大小增长时，注意 BERT 类型模型中层标准化（layer normalization）的位置对于提高性能至关重要。使用 GPT-2 模型，我们在 WikiText103 (10.8 与 SOTA 混淆度 15.8 相比) 和 LAMBADA (66.5% 与 SOTA 准确率 63.2% 相比) 数据集上取得了 SOTA 结果。我们的 BERT 模型在 RACE 数据集上实现了 SOTA 结果 (90.9% 与 SOTA 准确率 89.4% 相比)。

# 1. 引言
自然语言处理（NLP）的快速发展在某种程度上归功于计算能力和数据集大小的增长。大量的计算能力和数据使得通过无监督预训练训练越来越大的语言模型成为可能（Devlin et al., 2018; Radford et al., 2019）。经验证据表明，较大的语言模型在完成文章、回答问题和自然语言推理等 NLP 任务上有明显更好的效果（Lan et al., 2019; Raffel et al., 2019）。通过微调这些预训练语言模型在下游的自然语言任务上，可以实现最先进的结果，这已经在最近的研究中得到展示（Devlin et al., 2018; Peters et al., 2018; Howard & Ruder, 2018; Radford et al., 2018; 2017; Ramachandran et al., 2016; Liu et al., 2019b; Dai et al., 2019; Yang et al., 2019; Liu et al., 2019a; Lan et al., 2019）。

随着这些模型变得越来越大，它们超过了现代处理器的内存限制，并需要额外的内存管理技术，例如激活检查点（Chen et al., 2016）。广泛使用的优化算法如 ADAM 需要为每个参数提供额外的内存来存储动量和其他优化器状态，这降低了可以有效训练的模型的大小。一些模型并行方法通过划分模型以克服此限制，使得权重及其关联的优化器状态不需要同时驻留在处理器上。例如，GPipe（Huang et al., 2018）和 Mesh-Tensorflow（Shazeer et al., 2018）提供了各种类型的模型并行框架。然而，它们需要重写模型，并依赖于仍在开发中的自定义编译器和框架。

![](./img/megatron/f1.png)

在本工作中，我们使用层内模型并行法实现了一个简单而有效的模型并行方法。我们利用基于 Transformer 的语言模型的内在结构，实现了一个简单的模型并行方法，可以在 PyTorch 中高效训练，无需自定义的 C++ 代码或编译器。这种方法与 GPipe（Huang et al., 2018）等方法提倡的基于流水线的模型并行是正交的。

为了展示我们的方法的可扩展性，我们通过在单个 NVIDIA V100 32GB GPU 上训练一个有 12 亿参数的模型来建立基线，其维持 39 TeraFLOPs。这是 DGX-2H 服务器中单个 GPU 配置的理论峰值 FLOPs 的 30%，因此是一个强大的基线。将模型扩展到 512 个 GPU 上的 83 亿参数，并使用 8 路模型并行，我们在整个应用中达到了每秒 15.1 PetaFLOPs 的性能。这是与单个 GPU 案例相比的 76% 扩展效率。图 1 显示了更详细的扩展结果。

为了分析模型大小扩展对准确性的影响，我们训练了从左到右的 GPT-2（Radford et al., 2019）语言模型以及 BERT（Devlin et al., 2018）双向 Transformer，并在几个下游任务上对它们进行评估。我们发现，现有的 BERT 架构会导致模型随着大小增大而退化。我们通过重新排列 Transformer 层中的层标准化和残差连接来克服这个挑战，并展示了随着模型大小的增大，对开发集的下游任务的结果会单调改善。此外，我们还展示了我们的模型在 WikiText103，cloze-style 预测精度在 LAMBADA，以及阅读理解 RACE 数据集上取得的测试集的最先进（SOTA）结果。

总结起来，我们的贡献如下：
- 我们通过对现有 PyTorch transformer 实现进行少量的针对性修改，实现了一个简单而高效的模型并行方法。
- 我们对我们的模型和数据并行技术进行了深入的实证分析，并展示了使用 512 个 GPU 可以实现高达 76% 的扩展效率。我们表明，注意 BERT 类型模型中层标准化的位置对于随着模型增长而提高准确性至关重要。
- 我们证明了模型大小的扩展会导致 GPT-2（研究了高达 83 亿参数）和 BERT（研究了高达 39 亿参数）模型的准确性提高。
- 我们展示了我们的模型在测试集上取得了最先进的结果：在 WikiText103 上的困惑度（10.8 ppl），在 LAMBADA 上的准确度（66.5%），以及在 RACE 上的准确度（90.9%）。
- 我们将我们的代码以及训练和评估流程开源在 https://github.com/NVIDIA/Megatron-LM 上。

# 2 背景与挑战

## 2.1 神经语言模型预训练

预训练语言模型已经成为了自然语言处理(NLP)研究者工具箱中不可或缺的一部分。利用大规模语料库预训练来学习语言的强大神经表示是过去十年的一个活跃研究领域。早期预训练和传递语言神经表示的例子表明，预训练词嵌入表相比于从头开始学习的词嵌入表，能够提高下游任务的结果(Mikolov等人，2013; Pennington等人，2014; Turian等人，2010)。后来的工作通过学习和传递捕获单词上下文表示的神经模型，推动了这个领域的研究(Melamud等人，2016; McCann等人，2017; Peters等人，2018; Radford等人，2017;2019)。最近的并行工作(Ramachandran等人，2016; Howard & Ruder，2018; Radford等人，2018; Devlin等人，2018; Liu等人，2019b; Dai等人，2019; Yang等人，2019; Liu等人，2019a; Lan等人，2019)进一步在这些思想上进行建设，不仅把语言模型传递来提取上下文单词表示，还对下游任务进行端到端的语言模型微调。通过这些工作，最先进的技术已经从只传递词嵌入表到传递整个数十亿参数的语言模型。这种方法的进展，促使了对能够有效地在大规模运算并满足不断增长的计算需求的硬件、系统技术和框架的需求。我们的工作旨在为这一趋势提供必要的工具，以推动其更进一步的发展。

![](./img/megatron/f2.png)

## 2.2 Transformer语言模型与多头注意力

由于其卓越的精确性和计算效率，NLP的当前工作趋势是使用Transformer模型(Vaswani等人，2017)。原始的Transformer公式被设计为一种机器翻译架构，它使用两部分，编码器和解码器，将输入序列转换为另一个输出序列。然而，最近利用Transformer进行语言建模的工作，如BERT (Devlin等人，2018)和GPT-2 (Radford等人，2019)，根据他们的需要只使用了编码器或解码器。这项工作探索了一个解码器架构GPT-2和一个编码器架构BERT。

图2展示了我们使用的模型的示意图。我们建议读者参考之前的工作以了解模型架构的详细描述(Vaswani等人，2017; Devlin等人，2018; Radford等人，2019)。值得一提的是，GPT-2和BERT都使用GeLU (Hendrycks & Gimpel，2016)非线性性和对多头注意力和前馈层的输入进行层次归一化(Ba等人，2016)，而原始的Transformer (Vaswani等人，2017)使用ReLU非线性性并对输出应用层次归一化。

## 2.3 深度学习中的数据和模型并行性

扩展深度神经网络训练到多个硬件加速器有两个核心范例：数据并行性 (Valiant，1990)，在这种情况下，训练小批量被分配到多个工作人员；以及模型并行性，在这种情况下，模型的内存使用和计算分布在多个工作人员中。通过增加小批量大小与可用工作人员的比例（即弱扩展），人们可以观察到

训练数据吞吐量近乎线性的扩展。然而，大批量训练在优化过程中引入了一些复杂性，这可能导致准确性降低或收敛时间延长，抵消了训练吞吐量的增加带来的好处 (Keskar等人，2017)。进一步的研究(Goyal等人，2017; You等人，2017; 2019)已经发展出一些技术来减轻这些影响，并缩短大型神经网络的训练时间。为了进一步扩大训练规模，一些并行的工作 (Chen等人，2016)已经将数据并行性与激活检查点结合起来：在反向传递中重新计算激活，而不在正向传递中存储它们，以减少内存需求。

然而，这些技术在可以处理的问题大小上有一个根本的限制：模型必须完全适应一个工作人员。随着语言模型（如BERT和GPT-2）的大小和复杂性的增加，神经网络已经接近了现代硬件加速器的内存容量。解决这个问题的一种方法是使用参数共享来减小模型的内存占用(Lan等人，2019)，但这限制了模型的总体容量。我们的方法是利用模型并行性将模型分割到多个加速器上。这不仅可以减轻内存压力，而且还可以增加并行性，而不依赖于微批量的大小。

在模型并行性中，还有两种进一步的范例：分层流水线并行性，和更一般的分布式张量计算。在管道模型并行中，一部分操作在一个设备上执行，然后输出被传递到管道中的下一个设备，在那里执行一组不同的操作。一些方法 (Harlap等人，2018; Chen等人，2018)使用参数服务器 (Li等人，2014)与管道并行性一起使用。然而，这些方法存在一致性问题。TensorFlow的GPipe框架 (Huang等人，2018)通过使用同步梯度下降克服了这个一致性问题。这种方法需要额外的逻辑来处理这些通信和计算操作的高效流水线，而且会遭受降低效率的流水线泡沫，或者优化器本身的改变影响精度。

分布式张量计算是一种正交的、更一般的方法，它在多个设备之间划分张量操作以加速计算或增加模型大小。FlexFlow (Jia等人，2018)，一个用于编排此类并行计算的深度学习框架，提供了一种选择最佳并行策略的方法。最近，Mesh-TensorFlow (Shazeer等人，2018)引入了一种用于指定分布式张量计算一般类的语言在TensorFlow (Abadi等人，2015)中。并行维度由最终用户在语言中指定，最终生成的图形由适当的集体原语编译。我们利用了与Mesh-TensorFlow中所利用的类似的见解，利用计算Transformer的注意力头的并行性，来并行化我们的Transformer模型。然而，我们并没有实现一个用于模型并行的框架和编译器，而只是对现有的PyTorch Transformer实现做了一些针对性的修改。我们的方法很简单，不需要任何新的编译器或代码重写，只需要插入一些简单的原语，就可以完全实现，这将在下一节中详细描述。

# 3. 模型并行变换器

我们利用变换器网络的结构通过增加一些同步原语来创建一个简单的模型并行实现。一个变换器层由一个自注意力块后跟一个两层的多层感知机（MLP）组成，如图2所示。我们分别在这两个块中引入模型并行性。

我们首先详细描述MLP块。该块的第一部分是一个GEMM，后跟一个GeLU非线性：
$$
Y = \text{GeLU}(XA) \tag{1}
$$
并行化GEMM的一个选择是将权重矩阵$A$沿着其行分割，将输入$X$沿着其列分割，如下：
$$
X = [X1, X2], \quad A =
\begin{bmatrix}
A1\\
A2
\end{bmatrix}
. \tag{2}
$$
此划分将产生 $Y = \text{GeLU}(X1A1 + X2A2)$。由于GeLU是一个非线性函数，$\text{GeLU}(X1A1 + X2A2) \neq \text{GeLU}(X1A1)+\text{GeLU}(X2A2)$，因此这种方法在GeLU函数之前需要一个同步点。

另一个选择是沿着其列分割$A$，即 $A = [A1, A2]$。此划分允许GeLU非线性独立地应用于每个分区GEMM的输出：
$$
[Y1, Y2] = [\text{GeLU}(XA1), \text{GeLU}(XA2)] \tag{3}
$$
这样做的优点是它消除了一个同步点。因此，我们以这种列并行方式划分第一个GEMM，并沿着其行分割第二个GEMM，使其能直接采用GeLU层的输出，而不需要任何通信，如图3a所示。然后，在传递输出到dropout层之前，通过GPUs减少第二个GEMM的输出。这种方法在MLP块的GPUs中分割了两个GEMMs，并仅在正向传递中需要一个单一的全部减少操作（g操作符），在反向传递中需要一个单一的全部减少（f操作符）。这两个操作符彼此共轭，可以用PyTorch仅用几行代码实现。作为一个例子，下面提供了f操作符的实现。

![](./img/megatron/c1.png)
![](./img/megatron/f3.png)

如图3b所示，对于自注意力块，我们利用多头注意力操作中的固有并行性，以列并行方式划分与关键字（K）、查询（Q）和值（V）相关联的GEMMs，使每个注意力头对应的矩阵乘法在一个GPU上本地完成。这样我们可以在GPUs之间划分每个注意力头的参数和工作负载，而不需要立即完成自注意力的通信。自注意力后的输出线性层（后自注意力）的随后的GEMM沿着其行并行，并直接采用平行注意力层的输出，无需GPU之间的通信。对于MLP和自注意力层的这种方法，两个GEMMs的组合消除了中间的同步点，并得到更好的扩展。这使我们能够使用仅在正向路径中的两个全部减少和反向路径中的两个全部减少来执行简单变换器层中的所有GEMMs（见图4）。

![](./img/megatron/f4.png)

变换器语言模型具有隐藏大小（H）与词汇量大小（v）之间的输出嵌入。由于现代语言模型的词汇量（例如，GPT-2使用了50,257的词汇量）达到了几万个标记的数量级，所以并行输出嵌入GEMM是有益的。然而，在变换器语言模型中，输出嵌入层与输入嵌入共享权重，需要对两者进行修改。我们沿着词汇维度E并行化输入嵌入权重矩阵$EH \times v$，即 $E = [E1, E2]$（列式）。由于每个分区现在只包含嵌入表的一部分，所以在输入嵌入之后需要一个全部减少（g操作符）。

对于输出嵌入，一种方法是执行并行GEMM $[Y1, Y2] = [XE1, XE2]$以获得logits，添加一个全部收集$Y = \text{all-gather}([Y1, Y2])$，并将结果发送到交叉熵损失函数。然而，对于这种情况，全部收集将通信$b \times s \times v$个元素（b是批量大小，s是序列长度），由于词汇量很大，所以这一数量巨大。为了减少通信大小，我们将并行GEMM的输出$[Y1, Y2]$与交叉熵损失融合，从而将维度减小到$b \times s$。相对于logits进行通信的标量损失的减少大大提高了我们的模型并行方法的效率。

我们的许多模型并行方法可以描述为旨在减少通信和使GPUs计算受限的技术。与其让一个GPU计算dropout、层归一化或残差连接的一部分，并将结果广播到其他GPUs，我们选择在GPUs上复制计算。具体来说，我们在每个GPU上保持层归一化参数的重复副本，并在模型并行区域的输出上运行dropout和残差连接，然后将这些张量作为下一个模型并行区域的输入。为了优化模型，我们允许每个模型并行工作者优化其自己的一组参数。由于所有值要么是局部的，要么在GPU上重复，所以不需要在此配方中通信更新的参数值。

我们在附录B中进一步提供了关于混合模型和数据并行性以及处理随机数生成的详细信息作为参考。总的来说，我们以上所描述的方法执行简单，仅需要在正向和反向传递中添加少量额外的全部减少操作。它不需要编译器，并且与像（Huang et al., 2018）这样提倡的管道模型并行性正交和互补。

以下是您请求的翻译的内容部分：

# 4. 设置

预训练的语言理解模型是自然语言处理和语言理解的核心任务。语言建模有几种表述方式。在这项工作中，我们专注于GPT-2（Radford等人，2019），一种从左到右的生成变换器基础语言模型，和BERT（Devlin等人，2018），一种基于语言模型掩蔽的双向变换器模型。我们将在以下部分解释这些模型的配置，并参考原始论文以了解更多详细信息。

## 4.1. 训练数据集

为了收集具有长期依赖性的大型多样化训练集，我们汇总了几个最大的语言建模数据集。我们创建一个聚合数据集，包括Wikipedia（Devlin等人，2018），CC-Stories（Trinh和Le，2018），RealNews（Zellers等人，2019），和OpenWebtext（Radford等人，2019）。为了避免训练集泄漏到我们的下游任务中，我们从WikiText103测试集（Merity等人，2016）中删除了Wikipedia文章。我们还从CC-Stories语料库中删除了由预处理工件引入的不必要的换行符。对于BERT模型，我们在训练数据集中包括BooksCorpus（Zhu等人，2015），但由于它与LAMBADA任务重叠，因此GPT-2训练中排除了此数据集。

我们合并了所有数据集，然后从聚合数据集中过滤掉所有内容长度小于128个标记的文档。由于相似的内容可能在聚合数据集中多次出现，我们使用局部敏感哈希（LSH）去重复具有大于0.7的杰卡德相似度的内容。最终的聚合语料库包含174 GB的重复消除文本。

## 4.2. 训练优化和超参数

为了高效训练我们的模型，我们使用混合精度训练和动态损失缩放来利用V100的张量核心（Micikevicius等人，2017; NVIDIA, 2018）。我们通过一个简单的正态分布$W \sim N(0,0.02)$初始化我们的权重$W$。然后我们通过$1/\sqrt{2N}$，其中$N$是由自注意力和MLP块组成的变换器层数，来立即缩放残差层之前的权重。对于我们的优化器，我们使用具有权重衰减（Loshchilov和Hutter，2019）$\lambda = 0.01$的Adam（Kingma和Ba，2014）。另外，我们使用1.0的全局梯度范数剪裁来提高训练大型模型的稳定性。在所有情况下，dropout为0.1。最后，为了更好地管理我们的内存占用，我们在每个变换器层后使用激活检查点（Chen等人，2016）。

对于GPT-2模型，所有训练都是以批量大小为512的1024个子词单元进行300k迭代。我们的学习率为1.5e-4，使用了3k迭代的预热期，然后在剩余的297k迭代中遵循单周期余弦衰减。我们将衰减停在最小学习率1e-5处。

对于BERT模型，我们主要遵循（Lan等人，2019）中描述的培训过程。我们使用词汇量为30,522的原始BERT词典。此外，我们根据（Lan等人，2019）的建议用句子顺序预测替换下一个句子预测头，并使用（Joshi等人，2019）的全词n-gram掩蔽。在所有情况下，我们将批量大小设置为1024，使用1.0e-4的学习率预热10,000次迭代，然后在2百万次迭代中线性衰减。其他训练参数与（Devlin等人，2018）保持相同。


# 5 实验

我们的所有实验都使用多达32台DGX-2H服务器（共512个Tesla V100 SXM3 32GB GPU）。我们的基础设施针对多节点深度学习应用进行了优化，服务器内部GPU之间的带宽为300 GB/秒（通过NVSwitch），服务器之间的互联带宽为100 GB/秒，每台服务器使用8个InfiniBand适配器。

## 5.1. 扩展分析

为了测试我们的实现的可扩展性，我们考虑了表1中详述的具有四组参数的GPT-2模型。为了在自注意层中保持一致的GEMM尺寸，每个注意力头的隐藏尺寸保持为96，而头数和层数的变化是从10亿到80亿的参数。1.2亿参数的配置适合于单个GPU，而80亿参数的模型需要8路模型并行（8个GPU）。原始词汇量为50,257，然而，为了使logit层的GEMM有效，每个GPU的词汇量应该是128的倍数。因为我们研究多达8路模型并行，所以我们对词汇进行填充，使其可以被$128 \times 8 = 1024$整除，从而产生填充后的词汇量为51,200。我们研究了模型并行和模型+数据并行扩展。对于模型并行扩展，所有配置都使用固定批大小8。数据并行扩展对训练许多先进模型是必需的，通常使用更大的全局批处理大小。为此，对于模型+数据并行情况，我们将所有实验的全局批量大小固定为512，相当于64路数据并行。

### 5.1.1. 模型和数据并行

在本节中，我们将展示与模型参数有关的弱扩展，包括模型并行和模型+数据并行情况。弱扩展通常通过扩展批量大小来实现，但是，这种方法并不解决无法适应单个GPU的大型模型的训练问题，并且大批量大小会导致训练收敛性降低。相反，在这里，我们使用弱扩展来训练以前不可能的较大模型。所有扩展数字的基线是表1中在单个GPU上运行的第一个配置（12亿参数）。这是一个强基线，因为在整个训练过程中实现了39 TeraFLOPS，这是DGX-2H服务器中单个GPU的理论峰值FLOPS的30%。

图5显示了模型和模型+数据并行的扩展值。我们观察到两种设置中的扩展数字都非常出色。例如，具有8路（8 GPU）模型并行的83亿参数情况实现了77%的线性扩展。模型+数据并行需要进一步的梯度通信，因此扩展数字略有下降。然而，即使对于在512个GPU上运行的最大配置（83亿参数），我们也实现了相对于强单GPU基线配置（12亿参数）的线性扩展的74%的扩展。附录D提供了进一步的扩展分析。

## 5.2. 使用GPT-2的语言建模结果

为了证明大型语言模型可以进一步推动技术的最新水平，我们考虑了表2中列出的尺寸和配置的GPT-2模型。355M模型在尺寸和配置上与BERT-Large模型（Devlin等人，2018）相当。2.5B模型比以前最大的GPT-2模型大，8.3B模型比我们所知的任何从左到右的Transformer语言模型都大。为了训练和评估我们的语言模型，我们使用第4节中描述的过程。表2还列出了推进一个时期所需的时间，相当于68,507次迭代。例如，对于512个GPU上的8.3B模型，每个时代需要大约两天。与表1中用于我们的扩展研究的配置相比，2.5B模型相同，8.3B模型具有24个注意力头而不是32个，而355M模型比以前看到的任何模型都小，同时仍使用64个GPU进行训练，从而导致每个时代的时间大大减少。

图6显示了迭代次数的函数的验证perpelixity。随着模型大小的增加，验证perpelixity减小，并达到8.3B模型的验证困惑度为9.27。我们在表3中报告了LAMBADA和WikiText103数据集上训练模型的零射击评估。有关评估方法的更多详细信息，请参阅附录E。我们观察到增加模型大小还导致WikiText103上的困惑度降低以及LAMBADA上的接近准

确度提高。我们的8.3B模型在WikiText103测试集上实现了适当调整的困惑度为10.81的艺术困惑度。在66.51%的准确度下，8.3B模型同样超过了LAMBADA任务上先前的接近准确度结果。我们在附录C中包括了从83亿参数模型生成的样本。最近，来自Microsoft的研究人员与NVIDIA合作，使用Megatron训练了一个称为Turing-NLG（Microsoft，2020）的170亿参数GPT-2模型，并表明随着模型的扩展，准确度进一步提高，突出了更大模型的价值。为确保我们不在测试集中训练任何数据，我们计算了在先前工作中完成的8-grams测试集中也出现在我们训练集中的百分比（Radford等人，2019）。WikiText103测试集最多重叠10.8%，LAMBADA测试集（Paperno等人，2016）最多重叠1.4%。我们应该注意，WikiText103测试集与WikiText103训练集（Radford等人，2019）已经重叠了9.09%。由于这些与先前的工作一致，我们确信我们的测试数据中没有文档无意中包括在我们的训练数据中。

### 5.3 使用BERT的双向Transformer结果

在本节中，我们将我们的方法应用于BERT风格的Transformer模型，并研究模型扩展对几个下游任务的影响。先前的工作（Lan等人，2019）发现，超过BERT-Large的336M参数的模型大小会导致意外的模型降级。为解决这个降级问题，该工作的作者（Lan等人，2019）引入了参数共享，并表明与原始BERT模型相比，他们的模型扩展得更好。

我们进一步研究了这种行为，并通过图7中显示的方式实证证明调整层规范化和残差连接的顺序对超越BERT-Large的BERT风格模型的扩展至关重要。图7中的架构（b）消除了使用原始BERT架构（a）观察到的不稳定性，并且还具有较低的训练损失。据我们所知，我们是首次报告这样的变化能够训练更大的BERT模型的人。

使用图7（b）中的架构更改，我们考虑表4中详细描述的三种不同情况。336M模型与BERT-large的大小相同。1.3B与先前显示获得比336M BERT-large模型差结果的BERT-xlarge配置相同（Lan等人，2019）。我们进一步通过增加隐藏大小和更多层来扩展BERT模型，以达到3.9B参数的情况。在所有情况下，每个注意力头的隐藏大小保持为64。336M和1.3B的模型分别训练了200万次迭代，而3.9B的模型训练了150万次迭代，并且仍在训练。

在3%的保留集上，336M、1.3B和3.9B的模型分别实现了验证集困惑度为1.58、1.30和1.16的验证集困惑度，模型大小呈递减趋势。我们在GLUE基准测试（Wang等人，2019）、Stanford问题回答数据集（Rajpurkar等人，2016; 2018）的MNLI和QQP、SQuAD 1.1和SQuAD 2.0以及阅读理解RACE数据集（Lai等人，2017）上微调训练过的模型。对于微调，我们遵循与（Liu等人，2019b）相同的程序。我们首先对批量大小和学习率进行超参数调整。一旦我们获得最佳值，我们就会报告5个不同随机种子初始化的中值开发集结果。每个模型和任务使用的超参数在附录A中提供。

表5显示了MNLI、QQP、SQuAD 1.1和SQuAD 2.0的开发集结果以及RACE的测试集结果。对于RACE的测试集结果，我们首先使用开发集找到在5个随机种子上给我们中值分数的检查点，然后报告该检查点在测试集上的结果。我们还报告了SQuAD的开发集和RACE测试集的5路集合结果。从表5中，我们观察到（a）随着模型大小的增加，所有情况下的下游任务性能均有所改善，（b）我们的3.9B模型与其他基于BERT的模型相比在开发集上建立了最先进的结果，（c）我们的3.9B模型在RACE测试集上实现了单个模型以及集成的最先进结果。

### 6. 结论和未来工作

在这项工作中，我们通过只对现有的PyTorch transformer实现进行少量修改，成功地超越了传统的单GPU模型训练所带来的限制。我们在512个NVIDIA V100 GPU上使用8路模型并行高效地训练了多达83亿参数的基于transformer的模型，并在整个应用程序中实现了高达15.1 PetaFLOPs的持续性能。我们还表明，对于BERT模型，小心注意BERT-like模型中层规范化的位置对于随着模型大小的增加而提高准确性至关重要。我们研究了模型大小对下游任务准确性的影响，并在下游任务上取得了远优于先前的结果，并为WikiText103、LAMBADA和RACE数据集建立了新的最先进技术。最后，我们开源了我们的代码，以促进未来利用模型并行transformer的工作。

有几个未来工作的方向。继续增加预训练的规模是一条有前景的研究方向，将进一步测试现有的深度学习硬件和软件。为实现这一目标，将需要优化器的效率和内存占用的改进。此外，训练超过160亿参数的模型将需要超过DGX-2H盒子内16个GPU的内存。对于这样的模型，混合的内层和层间模型并行以及节点间模型并行将更为合适。另外三个调查方向包括（a）预训练不同的模型家族（例如XLNet、T5），（b）评估大型模型在更困难和多样化的下游任务（例如生成问题回答、摘要和对话）上的性能，以及（c）使用知识蒸馏从这些大型预训练教师模型训练小型学生模型。