# 蛋白质-小分子亲和力预测

## 一些依赖
Create conda environment: mamba create --name torch-esa python=3.11 -y mamba activate torch-esa

Install PyTorch (2.5.1) mamba install pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia -y

Install PyTorch Geometric (2.6.1) and auxiliary packages mamba install pyg -c pyg -y mamba install pytorch-scatter -c pyg -y mamba install pytorch-sparse -c pyg -y mamba install pytorch-cluster -c pyg -y

Install xformers (v0.0.28.post3) pip install xformers --index-url https://download.pytorch.org/whl/cu121

Install Flash attention (v2.7.0.post2) pip install flash-attn --no-build-isolation

Install specific version of transformers from huggingface pip install transformers==4.35.0 datasets==2.14.6 accelerate==0.24.1 These are required in order to make sure that some of the node-level/3D task adaptations work as intended.

Install other requirements pip install pytorch_lightning pandas scikit-learn wandb rdkit bitsandbytes yacs admin_torch Cython ogb

## 数据集
从PDBbind数据集建图：
对整个复合物建一个图，节点为原子。
可视化复合物中原子的3D位置。
### pdbbind数据集目录：
datasets
└── pdbbind
    ├── metadata
    │   ├── affinities.json
    │   ├── identity30_split.json
    │   ├── identity60_split.json
    │   ├── index
    │   │   ├── 2019_index.lst
    │   │   ├── INDEX_core_cluster.2013
    │   │   ├── INDEX_core_data.2013
    │   │   ├── INDEX_core_name.2013
    │   │   ├── INDEX_general_NL.2019
    │   │   ├── INDEX_general_PL.2019
    │   │   ├── INDEX_general_PL_data.2019
    │   │   ├── INDEX_general_PL_name.2019
    │   │   ├── INDEX_general_PN.2019
    │   │   ├── INDEX_general_PP.2019
    │   │   ├── INDEX_refined_data.2019
    │   │   ├── INDEX_refined_name.2019
    │   │   ├── INDEX_refined_set.2019
    │   │   └── INDEX_structure.2019
    │   ├── lig_smiles.json
    │   ├── readme
    │   │   ├── PDBbind-101.txt
    │   │   └── pdbbind_2019_intro.pdf
    │   ├── scaffold_split.json
    │   └── seqs.json
    └── pdb_files
        ├── 10gs
        │   ├── 10gs.dssp
        │   ├── 10gs.obj
        │   ├── 10gs.pdb
        │   ├── 10gs_base.obj
        │   ├── 10gs_fixed.pdb
        │   ├── 10gs_ligand.mol2
        │   ├── 10gs_ligand.sdf
        │   ├── 10gs_out.csv
        │   └── 10gs_pocket.pdb
        ├── 184l
        │   ├── 184l.dssp
        │   ├── 184l.obj
        │   ├── 184l.pdb
        │   ├── 184l_base.obj
        │   ├── 184l_fixed.pdb
        │   ├── 184l_ligand.mol2
        │   ├── 184l_ligand.sdf
        │   ├── 184l_out.csv
        │   └── 184l_pocket.pdb
        ├── 185l
        │   ├── 185l.dssp
        │   ├── 185l.obj
        │   ├── 185l.pdb
        │   ├── 185l_base.obj
        │   ├── 185l_fixed.pdb
        │   ├── 185l_ligand.mol2
        │   ├── 185l_ligand.sdf
        │   ├── 185l_out.csv
        │   └── 185l_pocket.pdb
        ├── 186l
        │   ├── 186l.dssp
        │   ├── 186l.obj
        │   ├── 186l.pdb
        │   ├── 186l_base.obj
        │   ├── 186l_fixed.pdb
        │   ├── 186l_ligand.mol2
        │   ├── 186l_ligand.sdf
        │   ├── 186l_out.csv
        │   └── 186l_pocket.pdb
        ├── 187l
        │   ├── 187l.dssp
        │   ├── 187l.obj
        │   ├── 187l.pdb
        │   ├── 187l_base.obj
        │   ├── 187l_fixed.pdb
        │   ├── 187l_ligand.mol2
        │   ├── 187l_ligand.sdf
        │   ├── 187l_out.csv
        │   └── 187l_pocket.pdb
        ...

## 辅助材料：ESA模型简介：

---

### **关于论文《An end-to-end attention-based approach for learning on graphs》中ESA模型的详细报告**

#### **1. 模型概述**

Edge-Set Attention (ESA) 是一种为图学习设计的、纯粹基于注意力机制的端到端架构。其核心思想是**将图视为边的集合（Set of Edges）而非节点的集合**，并在此基础上进行学习。该模型通过一个创新的编码器和一个注意力池化机制，旨在克服传统图神经网络（GNNs）和现有图Transformer模型的局限性，如对人工设计算子的依赖、过平滑/过挤压问题、以及复杂且计算昂贵的预处理步骤。ESA模型设计简洁，不依赖位置编码、结构编码或任何复杂的图变换，却在超过70个图/节点级别的基准测试中取得了顶尖性能。

#### **2. 设计动机**

论文作者提出ESA模型，主要为了解决现有图学习方法的几个痛点：

* **GNN的局限性**：

  * **依赖人工设计**：消息传递（Message Passing）机制虽然灵活，但设计新的、更强大的聚合与更新函数（如PNA）非常困难，且通常需要领域知识和手动调参。
  * **固定的读出（Readout）函数**：GNN通常使用简单的、非学习性的读出函数（如sum, mean, max）来获得图级别的表示，这限制了模型的表达能力。
  * **过平滑与过挤压**：深层GNN容易出现节点表示趋同（过平滑）和信息在瓶颈边丢失（过挤压）的问题。

* **现有图Transformer的复杂性**：

  * **性能问题**：许多复杂的图Transformer在严格的基准测试中，性能并不总能超越经过精细调优的简单GNN。
  * **预处理复杂**：模型如Graphormer需要大量计算密集的预处理步骤（如计算全图最短路径、中心性编码等），这极大地增加了计算开销和使用门槛。
  * **对编码的过度依赖**：许多模型严重依赖各种位置编码（Positional Encoding）和结构编码（Structural Encoding）来注入图结构信息，这使得模型本身变得复杂。

ESA的目标是提供一个**简单、可扩展且高性能**的纯注意力解决方案，摆脱上述依赖和复杂性。

#### **3. 核心架构与组件**

ESA的整体架构由两部分组成：一个用于学习有效的边表示的**编码器（Encoder）**，以及一个用于将边表示聚合为图级表示的**池化模块（Pooling Module）**。

*上图为根据原文Fig. 4对ESA架构的示意性重构。*

**3.1 输入表示：将图视为边的集合**

与传统方法不同，ESA的操作对象是**边**。对于图中的每条边 $e_{ij}$（连接节点 $i$ 和 $j$），其初始特征表示 $x_{ij}$ 由以下三部分拼接而成：
$x_{ij} = \text{concat}(n_i, n_j, e_{ij})$
其中 $n_i$ 和 $n_j$ 分别是源节点和目标节点的特征，$e_{ij}$ 是该边自身的特征（如果存在）。所有边的特征构成一个集合，作为模型的输入。

**3.2 关键组件：注意力模块**

ESA架构由三种核心的注意力模块构建而成：

* **Masked Self-Attention Block (MAB) - 掩码自注意力模块**
  这是ESA的核心创新。它是一种标准的自注意力模块，但在计算注意力分数时引入了一个**掩码矩阵（Mask Matrix, M）**。其计算公式为：
  $\text{SDPA}(Q, K, V, M) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}} + M\right)V$
  关键在于掩码 $M$ 的设计。在ESA中，$M$ 是一个**边邻接矩阵（Edge Adjacency Matrix）**。两条边被认为是“邻接”的，如果它们在原图中共享一个公共节点。这个掩码矩阵限制了注意力计算的范围，使得每条边只能关注到与其直接相连的边。这相当于将图的局部连接性作为一种强烈的归纳偏置（Inductive Bias）注入到模型中。该模块的结构如原文Fig. 4D所示，包含层归一化（LayerNorm）、多头掩码自注意力和一个MLP层，并使用残差连接。

* **Self-Attention Block (SAB) - (标准)自注意力模块**
  SAB是一个标准的Transformer编码器层，如原文Fig. 4C所示。它与MAB的唯一区别是不使用掩码（或者说，掩码矩阵 $M$ 为全零矩阵）。这允许模型中的每个边表示可以关注到输入集合中的**所有其他边**，从而能够捕捉超越原始图拓扑的、更全局或更抽象的依赖关系。

* **Pooling by Multi-Head Attention (PMA) - 多头注意力池化模块**
  PMA模块用于取代传统的 `sum/mean/max` 读出函数，实现一个可学习的、从边表示集合到单个图表示的聚合过程。其灵感来源于Set Transformer。它的工作方式如原文Fig. 4B所示：

  1. 引入一个可学习的、包含 $k$ 个“种子向量”（Seed Vectors）的张量 $S_k$。
  2. 将 $S_k$ 作为Query，将编码器输出的边表示 $Z$ 作为Key和Value，进行一次跨注意力（Cross-Attention）计算。
  3. 输出的 $k$ 个向量再经过一个或多个SAB层进行内部信息交互，最终聚合成一个或多个图级别的表示向量。
     PMA提供了一种更强大和灵活的聚合机制，能够自适应地从边表示中提取对预测任务最重要的信息。

**3.3 编码器：交错式层叠 (Interleaving)**

ESA编码器的独特之处在于它\*\*垂直交错（Vertically Interleaves）\*\*地堆叠MAB和SAB模块。例如，一个典型的ESA编码器结构可能是 `M-S-M-S-P`（M代表MAB, S代表SAB, P代表PMA）。

这种交错设计具有重要意义：

* **MAB层**利用图的显式连接信息（边邻接关系）作为强先验，有效地在局部邻域内传播信息。
* **SAB层**则允许模型“跳出”这个局部结构的限制，建立任意两条边之间的联系，从而学习图中可能存在的、更全局或隐含的关系，并有能力纠正输入图中可能存在的噪声或错误连接（misspecification）。

论文中的消融实验（Table 5）证明，这种交错结构比单纯使用MAB或SAB的性能更优，显示了结合两种注意力的必要性。

#### **4. 与其他模型的区别**

* **与GAT的区别**：GAT的注意力也局限于邻居节点，可以看作一种掩码注意力的特例。但ESA的掩码机制更通用，且其在“边-边”邻接关系上操作，而非“节点-节点”。此外，ESA采用了标准且更具表达力的缩放点积注意力。
* **与Graphormer/TokenGT的区别**：ESA完全不使用任何形式的预计算编码（如最短路径、中心性、拉普拉斯特征向量等），极大地简化了模型并降低了计算成本。它将图结构信息完全通过MAB中的掩码来注入。
* **与传统GNN的区别**：ESA是纯粹的注意力模型，完全摒弃了消息传递范式。它的信息聚合是通过全局（SAB）或局部受限（MAB）的注意力完成的，并且拥有一个强大的可学习池化层（PMA）。

#### **5. 性能与可扩展性**

* **性能**：论文在分子性质预测（QM9, DOCKSTRING, PCQM4MV2）、视觉图、社交网络、异质图/同质图节点分类等70个任务上进行了全面评估。结果显示，ESA不仅全面优于调优后的GNN基线，也超过了Graphormer、TokenGT等复杂的图Transformer模型。在加入简单的结构编码后，ESA+模型在多个基准上达到了当前最优（State-of-the-art）水平。
* **可扩展性**：得益于现代高效注意力实现（如FlashAttention），ESA在理论上具有线性的内存复杂度。尽管当前深度学习库的实现导致其在存储密集掩码时存在瓶颈，但其训练时间和内存占用仍然优于PNA、GATv2等强GNN基线，并且远比Graphormer高效。这使得ESA能够处理拥有数十万条边的大图。

#### **6. 总结**

ESA模型是一个设计思想优雅且功能强大的图学习框架。它通过将学习的中心从“节点”转移到“边”，并巧妙地结合了利用图结构的**掩码自注意力（MAB）**和捕捉全局依赖的**标准自注意力（SAB）**，成功地构建了一个既简单又高效的纯注意力模型。它摆脱了对复杂预处理和人工设计组件的依赖，为图表示学习提供了一个极具潜力的、可作为强大基线的全新范式。


## 模型设计

模型细节：物理信息增强的3D-ESA (Physics-Informed 3D-ESA)
1. 原子（节点）特征定义 (Atom/Node Features)
对于蛋白质-配体复合物中的每一个原子 i，我们可以提取一个初始特征向量 n_i。这些特征可以使用 RDKit 等化学信息学库轻松获取。

原子类型 (Atom Type)：使用原子序数进行独热编码（One-hot encoding）。例如，可以覆盖常见的生物元素 C, N, O, S, P, F, Cl, Br, I 等。

维度示例：约10-15维

形式电荷 (Formal Charge)：原子的形式电荷，通常是整数（如-1, 0, +1）。

维度：1维

杂化类型 (Hybridization)：原子的杂化状态（SP, SP2, SP3, ...）。独热编码。

维度示例：约5维

芳香性 (Aromaticity)：该原子是否处于芳香环中。布尔值（0或1）。

维度：1维

重原子邻居数 (Number of Heavy Neighbors)：与该原子成键的非氢原子数量。

维度：1维

氢原子总数 (Total Number of Hydrogens)：与该原子成键的氢原子总数。

维度：1维

最终原子特征向量 n_i 是将上述所有特征拼接（concatenate）起来的结果。

总维度示例：约20-30维

2. 边（相互作用）特征定义 (Edge Features)
这是模型创新的核心。对于任意两个距离在截断半径（e.g., 5Å）内的原子 i 和 j，我们定义它们之间的边特征 e_ij。

距离特征 (Distance Feature)：

高斯基函数扩展 (Gaussian Basis Functions, GBF)：这是将标量距离 d_ij 转化为向量的标准方法，能更好地被神经网络学习。

公式：GBF(d_ij)_k = exp(-(d_ij - μ_k)^2 / β^2)，其中 μ_k 是预设的一系列距离中心点（例如，从0Å到5Å，每0.5Å一个点），β 是高斯函数的宽度。

这会将一个距离标量变成一个向量，捕捉不同距离区间的信号。

维度示例：16维（如果中心点设16个）

方向特征 (Directional Feature)：

归一化的相对位置向量：dir_ij = (pos_j - pos_i) / d_ij。

这是一个3D单位向量，精确地描述了相互作用的方向。这是模型3D能力的关键。

维度：3维

电荷相互作用特征 (Charge Interaction Feature)：

部分电荷乘积：首先使用RDKit计算每个原子的Gasteiger部分电荷 q_i 和 q_j。

特征为它们的乘积：charge_prod_ij = q_i * q_j。

这个标量直观地反映了库仑静电作用的强度和性质（正值为排斥，负值为吸引）。

维度：1维

最终边特征向量 e_ij 是将上述特征拼接的结果：e_ij = concat(GBF(d_ij), dir_ij, charge_prod_ij)

总维度示例：16 + 3 + 1 = 20维

3. 模型架构 (Model Architecture)
输入准备 (Input Preparation)：

给定一个蛋白质-配体复合物3D结构。

构建相互作用图：节点为所有原子，边为所有距离在截断半径内的原子对。

计算所有节点的特征 n_i 和所有边的特征 e_ij。

初始边表示 (Initial Edge Representation)：

遵循ESA的范式，将每条边的信息与其两端节点的信息结合起来。

x_ij = concat(n_i, n_j, e_ij)

维度示例：(2 * 30) + 20 = 80维

所有 x_ij 构成一个集合，作为编码器的输入。

编码器 (Encoder)：

这是一个由 L 层交错的MAB和SAB模块组成的堆栈。

推荐结构: M-S-M-S-...，例如4层或6层。

MAB (Masked Self-Attention Block)：掩码基于共享原子。它学习局部相互作用的耦合关系，例如，A-B作用如何影响B-C作用。

SAB (Self-Attention Block)：无掩码。它学习全局的、长程的相互作用协同效应。

每个块内部结构：LayerNorm -> Multi-Head Attention -> Residual Add -> LayerNorm -> MLP -> Residual Add。

池化 (Pooling by Multi-Head Attention, PMA)：

编码器输出了一系列优化后的“相互作用”表示。

PMA层将这些表示聚合成一个或多个全局向量，代表整个结合事件的能量画像。

它通过可学习的“种子向量”作为Query，对所有相互作用表示进行Cross-Attention，实现加权聚合。

预测头 (Prediction Head)：

将PMA输出的全局向量输入一个简单的多层感知机（MLP），例如 Linear -> ReLU -> Linear。

最终输出一个标量值，即预测的结合亲和力（pKd或ΔG）。

## 实验设计
阶段一：基础框架验证 (Difficulty: Easy, Importance: High)
目标：搭建一个能跑通的最小化可行产品（MVP），验证ESA框架在亲和力预测任务上的基本可行性。

特征简化：

原子特征：原子类型（独热编码，10维）

边特征：高斯基函数扩展的距离（16维）。这是最关键的3D信息，且实现相对简单。暂时不加入方向和电荷。

界面过滤：只保留蛋白质中距离配体8Å内的原子，大幅减少图的规模，专注于结合界面的相互作用。

模型简化：

编码器：只使用 MAB 层，例如2层MAB。先不引入SAB，以降低复杂性。

PMA和预测头：按标准实现。

训练与评估：

目标不是SOTA，而是观察损失是否能正常下降，以及预测值是否显著优于随机猜测。

预期成果：一个可以工作的代码库。验证界面过滤+位置编码+ESA的基础框架，为后续扩展奠定基础。

阶段二：完整的3D-ESA模型