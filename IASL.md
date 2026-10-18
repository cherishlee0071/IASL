# IASL：Intervention-Aware Sparse Latent Alignment

## 面向跨被试 EEG 信号泛化的研究方案（v0.2）

## 1. 一句话定义

IASL 研究的不是“如何让所有被试的隐空间变得相同”，而是：

> 能否在不消除被试特异神经信息的前提下，找到跨被试可迁移、对干预不易崩溃的表征子空间，并用它完成 EEG 对齐。

英文核心问题：

> Can intervention-aware sparse subspaces identify transferable representations for cross-subject EEG generalization?

---

## 2. 研究背景与动机

跨被试 EEG 泛化同时受到信号统计差异、神经反应差异和采集条件差异影响。已有方法大致经历了以下路线：

1. 信号校准：重参考、归一化、Euclidean Alignment、Riemannian Alignment；
2. 分布对齐：MMD、CORAL、类条件对齐；
3. 对抗学习：DANN 及其变体；
4. 共享—私有表征：把跨被试共享因素与被试特异因素分开；
5. 对比学习、图结构、多源域适应；
6. source-free adaptation 和 test-time adaptation。

这些方法通常以整体向量、通道或域分布为对齐单位。它们没有回答：

- 哪些表征是真正跨被试稳定的？
- 哪些被试差异应该被保留？
- 对齐后表征是否仍然能够被局部干预？
- 一个看起来稳定的 latent 是否只是干预幅度较小，而不是语义更清晰？

SAE 可以把密集激活表示成稀疏的过完备表示，因此提供了更细粒度的分析入口。但 SAE feature 的身份在随机种子、训练数据和架构改变后可能并不稳定。不同 SAE 可能学习到相近的低秩激活区域，却使用不同的基底。由此，IASL 不把“逐 feature identity 稳定”作为先验，而把它作为第一个需要检验的经验问题。

---

## 3. 对已有方案的关键修正

### 3.1 分析单位从单个 feature 上移到子空间或 feature group

原始方案把 SAE latent `j` 作为主要分析单位，并为每个 `j` 建立稳定性、任务相关性和干预档案。这一做法依赖跨被试 feature matching 可靠。

IASL v0.2 改为两层结构：

1. **主分析单位：可复现的低秩子空间或 feature group**；
2. **次分析单位：只有在匹配可靠时才报告单个 feature 的结果**。

匹配可靠性、子空间相似度和 feature-level overlap 都必须单独报告。若单个 feature 的对应关系不稳定，不能把匹配算法制造的对应关系解释为神经 feature identity。

### 3.2 干预效应必须按移除信号量归一化

原始干预损伤 (C_j) 可能只是干预强度的函数。一个 feature 被干预后影响大，不一定表示它更纠缠，也可能只是移除了更多能量或方差。

因此，每次干预同时记录：

- 原始任务损伤；
- 重构误差变化；
- 被移除的 EEG 频谱能量；
- 被移除的 latent 能量；
- 归一化后的选择性损伤。

IASL 只使用归一化干预指标作为筛选依据，同时保留未归一化结果作为敏感性分析。

### 3.3 干预必须成为训练信号，而不只是事后筛选器

干预评估应在对齐前建立初始诊断，但最终对齐过程中还要持续施加 intervention-aware constraint。否则，对齐损失可能改变 feature 的使用方式，使原本安全的特征重新进入纠缠状态。

---

## 4. 核心假设

### H1：跨被试可迁移信息主要集中在可复现的稀疏子空间

相比全 latent 空间，经过跨被试复现性和任务相关性筛选的子空间具有更好的 LOSO 泛化。

### H2：干预安全性对泛化具有独立增量价值

仅按激活稳定性筛选的对齐，应弱于同时使用归一化干预安全性的 IASL。该差异必须通过专门消融验证。

### H3：被试差异应被分解为共享成分和私有残差

强行消除所有 subject information 可能造成负迁移。可用的跨被试表示应允许保留受控的被试私有残差。

### H4：单 feature 稳定性可能不足，但低秩子空间稳定性仍然可用

如果单 feature matching 不可靠，而子空间匹配稳定，IASL 应在子空间层面工作，并把 feature-level 结果降级为解释性分析。

---

## 5. 方法框架

### 5.1 固定 EEG backbone

对每个 LOSO fold，使用训练被试训练或加载同一个 EEG encoder：

\[
h_{s,i}=f_\theta(x_{s,i})
\]

建议提取早期、中间和后期三个层级的激活，分别分析低层信号、任务相关时空模式和决策表示。

测试被试不能参与 backbone、SAE、matching 或筛选阈值的拟合。

### 5.2 训练 pooled Universal SAE

只使用当前 LOSO fold 的训练被试激活训练 SAE：

\[
z=\operatorname{ReLU}(W_e(h-b_h)+b_z),
\qquad
\hat h=W_dz+b_h
\]

\[
\mathcal L_{SAE}=\|h-\hat h\|_2^2+\lambda_1\|z\|_1+\lambda_{dead}\mathcal L_{dead}
\]

SAE 的宽度、稀疏度和重构阈值只能在训练被试内部选择。

### 5.3 构造 feature graph，而非直接依赖 feature index

每个 latent 由以下信息描述：

- decoder direction；
- 时间激活轨迹；
- 频谱和通道拓扑；
- 类条件激活分布；
- 被试激活统计；
- 干预后的输出变化。

用这些信息构建 feature graph，并在每个被试、随机种子和 fold 中做 graph matching 或 soft optimal transport。匹配结果首先用于发现 feature group，不直接宣称 feature identity。

### 5.4 发现可复现子空间

对每个训练被试计算 feature activation matrix，并进行：

1. 被试内和被试间相关结构计算；
2. bootstrap 重采样；
3. 跨随机种子 SAE 重复训练；
4. feature graph 聚类；
5. Procrustes、CCA 或 principal-angle 子空间比较。

得到候选子空间 (U_g)。只有在以下条件下，才允许继续报告组内 feature 结果：

- group 在多个 fold 中出现；
- principal angle 较小；
- activation pattern 超过 permutation null；
- matching 结果对随机种子不敏感。

### 5.5 计算四类诊断量

对每个子空间或 feature group `g`，计算：

#### 跨被试复现性

\[
R_g=\operatorname{SubspaceStability}(U_g)
\]

可报告 group-level ICC、跨被试 CKA、principal angle 和 bootstrap 置信区间。

#### 任务相关性

\[
T_g=I(z_g;Y)
\]

也可用交叉验证 probe、类条件效应量和置换检验估计。

#### 被试泄漏

\[
D_g=\operatorname{AUC}_{subject}(z_g)
\]

高 `D_g` 不自动意味着删除。它需要与 `T_g`、`R_g` 和负迁移结果共同解释。

#### 归一化干预损伤

对 group 做 ablation、activation patching 和小幅 steering。定义：

\[
C_g^{norm}=\frac{\Delta task+\lambda_h\Delta reconstruction}
{\Delta E_{removed}+\epsilon}
\]

同时报告未归一化的 Δtask、Δreconstruction 和被移除频谱能量。

### 5.6 三类表征，而不是简单二分

| 类别 | 判据 | 处理 |
|---|---|---|
| 稳定任务子空间 | 高 `R_g`、高 `T_g`、低 `C_g^{norm}` | 强对齐 |
| 被试私有子空间 | 低 `R_g`，但可能有高 `T_g` | 保留为 private residual |
| 纠缠或危险子空间 | 高归一化干预损伤、匹配不稳定 | 降权、拆分或暂不对齐 |

阈值必须在训练被试内确定，不能根据测试被试结果事后调整。

### 5.7 学习共享—私有对齐

令 `z_{shared}` 表示稳定子空间，`u_s` 表示被试私有残差：

\[
z_s=A_s z_{shared}+u_s
\]

其中 `A_s` 是低秩或小型 adapter。对共享子空间施加类条件对齐：

\[
\mathcal L_{align}=\sum_g w_gD(P(z^s_g|Y),P(z^t_g|Y))
\]

权重可设为：

\[
w_g\propto R_gT_g(1-C_g^{norm})
\]

对私有残差施加稀疏和正交约束，但不强迫其消失：

\[
\mathcal L_{private}=\|u_s\|_1+\lambda_\perp\langle u_s,z_{shared}\rangle^2
\]

### 5.8 对齐过程中的干预约束

对共享 group 做小幅 source-to-target patching：

\[
z_g^{patch}=(1-\alpha)z_g^s+\alpha z_g^t
\]

约束：

- 输出变化应与目标类别语义一致；
- 无关类别的预测不应异常改变；
- 解码表示不能明显偏离训练流形；
- 归一化干预损伤不能随训练上升。

总损失为：

\[
\mathcal L=\mathcal L_{task}
+\lambda_{align}\mathcal L_{align}
+\lambda_{interv}\mathcal L_{interv}
+\lambda_{private}\mathcal L_{private}
+\lambda_{SAE}\mathcal L_{SAE}
\]

---

## 6. 最小可行实验路线

### Phase 0：匹配可行性审计

目标：不训练最终分类器，先回答 SAE 是否能提供可复现单位。

- 多随机种子训练 SAE；
- 计算 feature matching overlap；
- 比较 feature-level matching 与 group/subspace matching；
- 计算 permutation null；
- 比较不同 SAE 宽度和层级。

决策规则：

- 若 feature matching 稳定且显著，保留 feature-level 分析；
- 若 feature matching 弱但 subspace stability 强，正式方法转为 group/subspace IASL；
- 若两者都接近 null，暂停训练型 IASL，先做 SAE 可复现性研究。

### Phase 1：干预归一化审计

目标：检验原始干预崩溃是否只是移除能量的结果。

对每个 group 进行多种干预强度和频谱能量匹配，比较：

1. 原始干预损伤；
2. 按移除能量归一化后的损伤；
3. 类选择性；
4. 不同被试和随机种子的一致性。

### Phase 2：IASL 对齐

只在训练被试上训练共享—私有 adapter 和对齐损失，采用严格 LOSO 评估。

### Phase 3：真实泛化验证

主要终点：

- held-out subject macro-F1 / balanced accuracy；
- 被试间性能方差；
- negative-transfer rate；
- subject leakage AUC；
- feature/group matching stability；
- intervention damage；
- calibration error。

---

## 7. 必须做的消融

| 实验 | 保留内容 | 去除内容 | 目的 |
|---|---|---|---|
| Full IASL | 子空间、任务、干预、私有残差 | 无 | 完整方法 |
| Stability-only | `R_g` | `C_g^{norm}` | 检验干预的独立价值 |
| Raw-intervention | 原始 (C_g) | 能量归一化 | 检验强度混杂 |
| Feature-only | 单 latent matching | group/subspace | 检验身份不稳定风险 |
| Subspace-only | group/subspace matching | 单 feature 约束 | 检验主要修正是否有效 |
| No-private | 共享对齐 | `u_s` | 检验私有残差必要性 |
| No-intervention-loss | 筛选可用 | 训练约束去除 | 检验事前筛选与训练信号差异 |
| Random-groups | 随机 group | 稳定性构造 | 排除 group size 和维度效应 |

其中 **Stability-only** 是最关键的新对照。若它与 Full IASL 没有差异，则不能声称干预感知带来独立贡献。

---

## 8. 失败判据与停止规则

IASL 不应预设一定有效。以下结果都应作为有效结论：

1. feature matching 不稳定，只有 subspace matching 可复现；
2. 干预选择性经能量归一化后消失；
3. Full IASL 不优于 Stability-only；
4. 强对齐提升平均准确率但提高 negative-transfer rate；
5. SAE 表征可解释，但不能改善 held-out subject 泛化。

特别是，若 Phase 0 显示 SAE feature identity 不可靠，论文应转为：

> Subspace-level reproducibility and intervention limits of SAE representations for cross-subject EEG alignment.

这仍然是一个有价值的结果，也避免把不稳定的 feature matching 包装成算法贡献。

---

## 9. 与 NeuroCurate 和 Time2Depth 的关系

NeuroCurate 已显示脑—模型分歧可以复现，但整体排序一致性不能保证小预算选择稳定，也没有证明训练收益。Time2Depth 的时间—层深趋势受到 EEG reliability、低 SNR 和静态层对齐混杂，最终证据为 weak/inconclusive。

IASL 因此应避免把全局 ranking 或单一模型层作为主要证据。它把问题改写为：

1. 哪些局部表征单元在被试间可复现？
2. 这些单元在控制干预强度后是否仍然具有选择性？
3. 对齐它们是否减少 held-out subject 的负迁移？

这使 IASL 成为从“可重复的全局关系”转向“可操作的局部表征”的新阶段。

---

## 10. 预期贡献

若实验支持主要假设，贡献可以表述为：

1. 提出一个以低秩稀疏子空间为基本单位的跨被试 EEG 对齐框架；
2. 将 SAE feature identity 不稳定性作为方法前提进行显式审计；
3. 提出按移除信号量归一化的干预安全性指标；
4. 将干预结构纳入对齐训练，而非仅用于事后解释；
5. 证明共享表征和被试私有残差需要同时建模；
6. 用 LOSO 泛化、负迁移率和干预损伤共同评价可迁移性。

---

## 11. 当前最稳妥的论文主张

在出现实证结果前，不应写成“SAE 找到了跨被试神经概念”。更稳妥的主张是：

> IASL tests whether intervention-aware, reproducible sparse subspaces can serve as transferable units for cross-subject EEG alignment.

中文：

> IASL 检验干预感知且可复现的稀疏子空间能否成为跨被试 EEG 对齐的可迁移单位。

这个表述允许结果支持 feature-level、group-level 或只支持 subspace-level 结论，也不会把尚未验证的 SAE feature identity 当作前提。

