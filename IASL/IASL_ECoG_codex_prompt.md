# 交给 Codex 的执行提示词

你现在处于 IASL 项目环境中。请在当前 checkout 内执行 DS004194 Visual ECoG 的小规模 IASL 可行性验证。不要创建新的 worktree、独立仓库或大型训练环境。

## 目标

THINGS-EEG Phase 0A–0D 因低重复试次可靠性、SAE feature overlap 接近 null、以及干预选择性未超过 energy-matched null 而判定 NO-GO。不要覆盖该结论。

本任务只验证：在高信噪比但电极覆盖异质性强的 ECoG 数据上，IASL 的时间域机制是否值得继续研究。不要一开始训练完整 Full IASL，也不要直接宣称跨被试泛化。

## 约束

1. 使用 DS004194 Visual ECoG 的公开版本和官方 derivatives；记录版本、下载方式、license 和 commit/version。
2. 不使用未验证的网络地址或私有数据；若数据不可获取，停止并写报告。
3. 不下载大型预训练模型。
4. 不读取或提交与当前实验无关的大型原始文件。
5. 所有结果放在 results/ecog_iasl/。
6. 保留已有 THINGS-EEG、synthetic 和 IASL 结果不变。
7. 不把真实 ECoG、原始二进制或大型衍生文件提交到 Git。
8. 先执行数据审计，审计未通过时不得训练 SAE。

## Step 1：环境和数据审计

数据集位置：https://huggingface.co/datasets/EEGDash/ds004194
Load this dataset:
This repo is a pointer. The raw EEG data lives at its canonical source (OpenNeuro / NEMAR); EEGDash streams it on demand and returns a PyTorch / braindecode dataset.

pip install eegdash
from eegdash import EEGDashDataset

ds = EEGDashDataset(dataset="ds004194", cache_dir="./cache")
print(len(ds), "recordings")

If the dataset has been mirrored to the HF Hub in braindecode's Zarr layout, you can also pull it directly:

from braindecode.datasets import BaseConcatDataset
ds = BaseConcatDataset.pull_from_hub("EEGDash/ds004194")

How to use from the Datasets library:

from datasets import load_dataset

ds = load_dataset("EEGDash/ds004194")

首先执行：

    pwd
    git rev-parse --show-toplevel
    git status --short
    git log -1 --oneline

检查 BIDS metadata、participants、events、channels 和 derivatives。生成：

    results/ecog_iasl/phase0_audit/
    ├── dataset_manifest.json
    ├── subject_task_matrix.csv
    ├── trial_counts.csv
    ├── channel_quality.csv
    ├── stimulus_intersection.json
    └── audit_report.md

必须回答：哪些 subject 共享同一 task；哪些 subject 共享同一 stimulus/condition；每个 subject 有多少有效 trial；哪些电极为 bad 或 epileptic；哪些电极位于视觉皮层或共同 ROI；是否存在统一 event onset 和时间窗；哪些 subject/session 需要排除及理由。

不要把 14 名参与者直接当作 14 名同任务被试。只有实际共享任务和条件的 subject 才能进入共同分析。

### 数据 Gate E0

若不满足以下任一条件，停止真实实验：至少 4 个 subject 共享同一任务/条件；有足够共同 trial；事件锁定方式一致；存在可解释的视觉皮层电极或共同表征空间；数据质量和 trial 数没有严重失衡。

输出状态：ECOG_NO_GO_DATA。

## Step 2：可靠性与预处理敏感性

只在通过 E0 的共同任务上执行。比较：

    A_current_derivative
    B_common_average_reference
    C_baseline_corrected
    D_broadband_or_high_gamma
    E_visual_response_electrode_mask
    F_trial_average

若官方 derivatives 已提供 broadband/high-gamma，不要重复实现不一致的滤波流程；直接记录其定义并审计。

每个版本计算 single-trial reliability、repeated-trial split-half、Spearman-Brown correction、bootstrap CI、event/time-window reliability、electrode-wise reliability 和 subject-wise reliability。电极 mask、时间窗和阈值只能在训练 subject 内确定。

生成：

    results/ecog_iasl/phase1_reliability/
    ├── reliability_metrics.json
    ├── preprocessing_comparison.csv
    ├── subject_summary.csv
    ├── figures/
    └── report.md

## Step 3：构建 ECoG 时间域表示

建立 PCA/linear control 与 tiny temporal encoder + SAE。初始设置：hidden 128/256，latent width 2×/4×，seeds 0/1/2，max epochs 50，early stopping 开启，CPU 或单 GPU。禁止使用大型 EEG foundation model。

主分析单位必须是 electrode group / temporal group / low-rank subspace。不能把 electrode index 或 SAE latent index 直接解释为跨被试神经 feature identity。

计算 EVR、reconstruction error、dead latent ratio、activation sparsity、decoder matching、group overlap、CKA、Procrustes、principal angle、bootstrap stability、row-permutation null、subject-label permutation null 和 PCA-only null。

生成：

    results/ecog_iasl/phase2_sae_subspace/
    ├── metrics.json
    ├── feature_matching.json
    ├── group_stability.json
    ├── subspace_stability.json
    ├── figures/
    └── report.md

如果单 feature overlap 接近 null，但 group/subspace 稳定，立即将主分析单位固定为 group/subspace，不再优化 feature-level matching。

## Step 4：energy-matched intervention

对候选 group/subspace 执行 group ablation、latent ablation、source-to-target activation patching、positive/negative steering、random-direction control 和 PCA-component control。

所有干预必须匹配 removed latent energy 或 removed reconstruction energy，至少使用 low、medium、high 三个 energy budget。记录：

    delta_task
    delta_reconstruction
    removed_latent_energy
    removed_eeg_energy
    target_class_damage
    non_target_class_damage
    selectivity
    subject_consistency
    seed_consistency

计算：

    C_raw = delta_task + lambda * delta_reconstruction
    C_norm = C_raw / (removed_energy + epsilon)
    selectivity = target_class_damage - non_target_class_damage

使用 subject-wise bootstrap、group-wise CI 和 energy-matched permutation null。若 intervention effect 与 null 无可靠差异，报告：No reliable semantic intervention selectivity was established. 不得将 raw damage 单独解释为表征纠缠。

生成：

    results/ecog_iasl/phase3_intervention/
    ├── intervention_metrics.json
    ├── normalized_damage.csv
    ├── null_comparison.json
    ├── figures/
    └── report.md

## Step 5：预注册决策门槛

### Gate A：可靠性

- 至少 4 个 subject 的可靠性稳定为正；
- trial-average 明显优于 single-trial，或 single-trial 已达到可分析水平；
- bootstrap CI 不全部覆盖零；
- 没有明显 task/session leakage。

### Gate B：子空间

- CKA/Procrustes 高于 PCA-only 和 permutation null；
- principal angle bootstrap 稳定；
- 结果不由单一 subject 驱动；
- 至少两个 seed 和两个 stimulus subset 方向一致。

### Gate C：干预

- energy-matched normalized effect 高于 null；
- target selectivity 高于 non-target effect；
- 结果不由单个 group 或 subject 驱动；
- 不能完全由 removed energy 解释。

只有 A、B、C 全部通过，才可进入 LOSO。

## Step 6：小规模 LOSO

只有通过 E0 和 Gates A–C 才执行。每个 fold 一个 subject 完全留出；PCA、encoder、SAE、electrode mask、group matching、threshold 和 alignment weight 只在训练 subject 内拟合；held-out subject 只能用于最终评估。

比较：No alignment、Standardization/EA、CORAL、Stability-only、Subspace-only、Full IASL。

Full IASL 使用 z_s = A_s z_shared + u_s，其中 u_s 是 subject-private residual，不得被强制消失。评价 held-out performance、subject-wise variance、negative-transfer rate、subject leakage AUC、intervention damage 和 calibration。

生成：

    results/ecog_iasl/phase4_loso/
    ├── fold_*/
    ├── aggregate_metrics.json
    ├── ablation_table.csv
    └── report.md

## Step 7：最终报告和状态

生成：

    results/ecog_iasl/final_report.md
    results/ecog_iasl/final_report.json
    results/ecog_iasl/reproducibility_manifest.json

最终状态只能是：

    ECOG_NO_GO_DATA
    ECOG_SUBSPACE_ONLY
    ECOG_INTERVENTION_SUPPORTED
    ECOG_LOSO_ELIGIBLE

含义：ECOG_NO_GO_DATA 表示共同任务或可靠性不足；ECOG_SUBSPACE_ONLY 表示子空间稳定但 feature/intervention 证据不足；ECOG_INTERVENTION_SUPPORTED 表示干预有独立信号但尚未证明泛化；ECOG_LOSO_ELIGIBLE 只表示具备运行小规模 LOSO 的资格，不表示 IASL 已成功。

报告必须区分 THINGS-EEG 历史 NO-GO、DS004194 真实结果、synthetic 结果、feature/group/subspace 结论、exploratory 与 confirmatory 分析、失败结果和停止原因。

最后执行：

    python -m compileall .
    pytest -q
    git diff --check

不要修改已有 THINGS-EEG NO-GO 结论，不要上传原始 ECoG 文件，不要在 Gates A–C 未通过时训练完整 Full IASL。
