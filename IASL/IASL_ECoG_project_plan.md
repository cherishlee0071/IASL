# IASL-ECoG：DS004194 小规模机制验证方案

## 1. 定位

THINGS-EEG 的 Phase 0A–0D 显示：重复试次可靠性接近零，单 SAE feature matching 接近随机方向 null，但 pooled subspace CKA 较高。因此不把 THINGS-EEG 的 NO-GO 当作 IASL 的最终证伪，而把它定义为低可靠性头皮 EEG 条件下的边界结果。

本项目转向 DS004194 Visual ECoG，目标不是立即证明跨被试泛化，而是在更高信噪比、但电极覆盖异质性强的皮层电信号上，低成本检验 IASL 的时间域机制：

1. 是否存在跨被试可复现的 electrode-group/temporal subspace；
2. 归一化干预损伤是否超出 energy-matched null；
3. intervention-aware 权重是否比 stability-only 提供独立信息。

NSD 不进入本轮实现；它作为第二阶段的空间域 IASL 候选，用于 voxel/ROI subspace 验证。

## 2. 数据集边界

DS004194 是临床视觉 ECoG 数据。虽然包含 14 名参与者并具有 CC0 许可，但电极覆盖、任务数量、试次数、视觉响应强度和数据质量在被试之间差异明显。因此不能把“14 名参与者”直接当作“14 名共享同一视觉任务的被试”。

第一轮只选择任务和刺激条件真正重合的被试。若图像类别任务只有少数被试可用，则不得宣称 14-subject 图像级 LOSO；应改为共同任务/共同条件的机制验证。

## 3. 核心假设

### H1：group/subspace 比单电极更稳定

不同被试的 electrode index 不具有直接可比性。IASL 的主单位必须是视觉 ROI 内的 electrode group、时间模式或低秩 subspace；单电极 identity 只作诊断。

### H2：干预安全性有独立信息

在匹配 removed latent/EEG energy 后，稳定且任务相关的 group 应比随机方向具有更高的类别选择性。

### H3：电极覆盖差异应保留为 private residual

共享表征与 subject-private residual 分离：'z_s = A_s z_shared + u_s'，其中 A_s 为小型 adapter，u_s 为稀疏或正交约束的私有残差。

## 4. 预注册实验流程

### Phase E0：数据审计与共同任务筛选

禁止先训练模型。记录 subject、session、run、task、共同刺激和事件锁定点、有效 trial 数、电极类型与坏电极标记、视觉皮层覆盖、采样率、参考方式、滤波、时间窗、license 和版本。

输出目录：results/ecog_phase0/audit/。

停止条件：没有至少 4 个 subject 共享同一任务/条件、共同 trial 太少、无法建立统一事件锁定、或视觉响应质量严重不平衡。

### Phase E1：可靠性与预处理敏感性

比较当前 derivatives、common-average reference、baseline correction、broadband/high-gamma envelope（若官方 derivatives 可用）、视觉响应 electrode mask、全 trial 与 trial-average。报告单 trial、重复平均和 bootstrap reliability；所有 electrode mask 和阈值只能在训练 subject 内确定。

输出目录：results/ecog_phase1/reliability/。

### Phase E2：时间域 SAE 与子空间稳定性

建立 PCA/linear control 与 tiny temporal encoder + SAE。初始设置：hidden 128/256，latent width 2×/4×，seed 0/1/2，最多 50 epochs，CPU 或单 GPU。主分析单位为 electrode-group/temporal subspace。测量 EVR、重构误差、dead latent、稀疏度、group overlap、CKA、Procrustes、principal angle、bootstrap stability 和 permutation null。

输出目录：results/ecog_phase2/sae_subspace/。

### Phase E3：energy-matched intervention

对稳定 group/subspace 做 group ablation、latent ablation、source-to-target activation patching、正负 steering，以及 random-direction/PCA-component controls。每次匹配 removed latent 或 reconstruction energy，并记录 delta_task、delta_reconstruction、removed_energy、target_damage、non_target_damage、selectivity。

核心指标：C_norm = (delta_task + lambda * delta_reconstruction) / (removed_energy + epsilon)，selectivity = target_damage - non_target_damage。

输出目录：results/ecog_phase3/intervention/。

### Phase E4：小规模 LOSO

仅当 E0–E3 通过门槛时执行。每个 fold 完全留出一个 subject；测试 subject 不参与 preprocessing、electrode selection、PCA、SAE、matching、threshold 或 alignment weight 拟合。比较 No alignment、Standardization/EA、CORAL、Stability-only、Subspace-only、Full IASL。

## 5. Go/No-Go 门槛

### E-Gate A：数据可靠性

- 至少 4 个 subject 的可靠性为正且稳定；
- trial-average 明显优于单 trial，或单 trial 已达到可分析水平；
- bootstrap CI 不全部覆盖零；
- 没有明显 session/task leakage。

### E-Gate B：子空间稳定性

- CKA/Procrustes 高于 PCA-only 和 permutation null；
- principal angle 在 bootstrap 中稳定；
- 结果不由单一 subject 驱动；
- 至少两个 seed 和两个 stimulus subset 方向一致。

### E-Gate C：干预信号

- energy-matched C_norm 或 selectivity 高于 null；
- 不能完全由 removed energy 解释；
- 不依赖单个 group 或 subject。

只有 A、B、C 全部通过，才允许 E4。

## 6. 结论标签

- ECOG_NO_GO_DATA：共同任务或可靠性不足；
- ECOG_SUBSPACE_ONLY：子空间稳定，但单 feature/干预证据不足；
- ECOG_INTERVENTION_SUPPORTED：干预有独立信号，但尚未证明泛化；
- ECOG_LOSO_ELIGIBLE：具备运行小规模 LOSO 的资格，不等价于 IASL 已成功。

## 7. NSD 后续路线

NSD 应作为第二阶段的空间 IASL，而不是本轮 ECoG 的替代实现。其表征单位改为 voxel/ROI pattern，干预改为 voxel-group/latent-subspace intervention，时间域 patching 不再作为主分析。NSD 需要单独处理访问协议、存储和空间配准问题。

## 8. 诚实的预期

可能结果包括：ECoG 上干预信号成立；只支持稳定子空间；或 ECoG 上仍失败并提示跨被试差异/电极覆盖是主要瓶颈。三种结果都必须保留，不得根据结果事后修改门槛。
