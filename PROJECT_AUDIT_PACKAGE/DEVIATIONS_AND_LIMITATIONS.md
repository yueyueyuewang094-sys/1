# Deviations, Simplifications, and Unfinished Content

This document compares the current implementation at `d:\code\1` against the complete ISRC experiment plan in `ISRC实验方案(1)(1).md`.

## A. Fully Implemented Per Plan

| Item | Plan Reference | Implementation | Notes |
|------|---------------|----------------|-------|
| 13-profile deterministic generation | Section 6.1.1 | profile_generator.py | 1 uniform + 8 single-emphasis + 4 paired-emphasis; sum(n)=8064; q<=n; SHA-256 hashes; 128-bit masks |
| Fixed parity graph shared by encoder and decoder | Section 6.1 | mother_code.py, decoder.py | 256x1008 projection, 1008x1264 H matrix; both use seed+7001 |
| Fixed 10-iteration decoding (no early stopping) | Section 6.1, 10.1 | decoder.py | Exactly 10 iterations in _decode_profile_stream |
| QPSK modulation with unit energy | Section 6.3 | software_channel.py | (1-2b)/sqrt(2) mapping; symbol energy=1.0 |
| AWGN with SNR-derived noise std | Section 6.3 | software_channel.py | noise_std = sqrt(1/(2*10^(snr/10))) |
| Profile header in frame | Section 6.2 | mother_code.py | 128-bit header (32-bit profile_id + 96 zeros) |
| Private RNG generator (no global pollution) | Section 6.3 | software_channel.py, input_mask.py | torch.Generator() with manual_seed |
| Erasure never uses ground truth | Section 6.2 | erasure.py | Erased bits set to -1; soft-info threshold only |

## B. Simplified Implementations

### Deviation D1: Parity check matrix is random sparse, not QC-LDPC

| Field | Content |
|-------|---------|
| Deviation ID | D1 |
| Plan requirement | QC-LDPC mother graph with n_c=1536, k_c=256, normalized min-sum decoding |
| Current implementation | Random sparse projection matrix (256x1008, 3 ones per row from RandomState(seed+7001)). Bit-flipping decoder instead of normalized min-sum. n_c effectively 1264, not 1536. |
| Difference | No QC structure; no lifting; no min-sum normalization coefficient; bit-flipping instead of soft-decoding |
| Reason | Toy implementation; QC-LDPC construction requires specialized libraries |
| Impact on results | Cannot claim QC-LDPC properties; error correction capability is weaker; decoding is hard-decision only |
| Future remediation | Implement QC-LDPC with commpy or sionna; use normalized min-sum with VAL_SELECT coefficient |

### Deviation D2: Header is plain profile_id, not BCH(127,64)+CRC-16

| Field | Content |
|-------|---------|
| Deviation ID | D2 |
| Plan requirement | 128-bit header: 8-bit version + profile_id + overload + K-bit validity + CRC-16, zero-padded to 64 bits, BCH(127,64) encoded + 1 padding bit = 128 bits |
| Current implementation | 32-bit profile_id + 96 zeros. No version, no overload, no validity, no CRC, no BCH. |
| Difference | No error detection on header; header decode failure cannot reject frame |
| Reason | Toy implementation; BCH/CRC requires additional codec implementation |
| Impact on results | No header protection; no frame rejection on control block failure; no declared erasure/fallback path |
| Future remediation | Implement CRC-16 + BCH(127,64) encoder/decoder; add version/overload/validity fields |

> **阶段二更新（2026-10-09）**：CRC-16/CCITT-FALSE 与 BCH(127,64)（t=10，GF(2^7)，生成多项式由 cyclotomic coset 程序化构造）已实现于 `src/modules/crc_bch.py`，含 §6.2 控制块字段（8-bit version + 4-bit profile_id + 1-bit overload + 8-bit sender_validity + CRC-16 覆盖 source bits）与解码失败拒帧语义，单元测试 14 项全部通过（tests/test_crc_bch.py）。**剩余偏差**：该控制块尚未集成进 mother_code/receiver 主链路，主链路头仍为 32-bit profile_id。

### Deviation D3: Erasure detection uses soft threshold, not syndrome/LLR-validity

| Field | Content |
|-------|---------|
| Deviation ID | D3 |
| Plan requirement | Decoder-validity from syndrome + LLR-confidence threshold (frozen on VAL_SELECT); per-role erasure; full-role erasure on CRC reject |
| Current implementation | Soft-information threshold: |noisy I/Q| < 0.5 -> erased. No syndrome check, no LLR-confidence, no per-role validity. |
| Difference | No principled decoder-validity; threshold is arbitrary, not frozen on VAL_SELECT |
| Reason | Toy implementation; requires decoder syndrome extraction |
| Impact on results | Cannot compute E_k (decoder erasure indicator); cannot compute p_e_k, p_e_ij; cannot compute U_t (undetected error) |
| Future remediation | Implement syndrome-based validity; freeze LLR threshold on VAL_SELECT; implement CRC reject -> full erasure |

### Deviation D4: No counter-based PRNG for noise

| Field | Content |
|-------|---------|
| Deviation ID | D4 |
| Plan requirement | Counter-based PRNG with key (dataset, sequence_id, frame_id, channel_model, channel_seed, bit_position); different-length methods share same bit position prefix |
| Current implementation | torch.Generator with manual_seed(42); noise generated sequentially |
| Difference | No per-(frame,channel_seed,bit_position) key; methods of different lengths do not share noise prefix |
| Reason | Toy implementation; counter-based PRNG requires custom implementation |
| Impact on results | Cannot guarantee different methods see identical noise at same bit positions; fairness contract partially violated |
| Future remediation | Implement counter-based PRNG (e.g., Philox); key by (dataset,seq,frame,model,seed,bitpos) |

> **阶段二更新（2026-10-09）**：counter-based PRNG 已实现于 `src/modules/counter_rng.py`（NumPy Philox 计数器模式，key=(dataset, sequence_id, frame_id, channel_model, channel_seed, bit_position) 经 BLAKE2b 序列化，1024-bit 块对齐保证不同长度方法共享 bit position 前缀），单元测试 8 项通过（tests/test_counter_rng.py，含 test_prefix_property）。**剩余偏差**：尚未接入 software_channel 主链路（主链路仍用 torch.Generator 顺序生成噪声）。

### Deviation D5: No 5-point AWGN sweep, only single SNR

| Field | Content |
|-------|---------|
| Deviation ID | D5 |
| Plan requirement | Es/N0 in {-2, 0, 2, 4, 6} dB; 5 channel seeds per point; 3 seeds for stress |
| Current implementation | Single SNR=10dB in config; no sweep; single seed |
| Difference | No SNR sweep; no multiple channel seeds; no stress conditions |
| Reason | Toy implementation; no formal experiment pipeline |
| Impact on results | Cannot compute SNR-AUC; cannot compare profiles across channel conditions |
| Future remediation | Implement SNR sweep with 5 points; add burst-mid and region-erasure-mid stress conditions |

## C. Alternative Implementations

### Deviation D6: LP solver uses scipy.optimize.linprog, not specialized LP

| Field | Content |
|-------|---------|
| Deviation ID | D6 |
| Plan requirement | Non-negative LP for tight interaction constants; 10000 bootstrap replicates; sequence-cluster resampling; active-constraint generation; full-matrix violation scan |
| Current implementation | scipy.optimize.linprog(method='highs') wrapper; single solve; no bootstrap; no resampling |
| Difference | No bootstrap; no sequence clustering; no active-constraint generation; no full-matrix scan |
| Reason | Toy implementation; full E2-T.2 pipeline is extremely compute-intensive |
| Impact on results | Cannot produce tight UCB; cannot claim empirical envelope; cannot compute P5b-tight controller |
| Future remediation | Implement full E2-T.2 with LP matrix prebuild, receiver batching, active-constraint generation, 10000 bootstrap |

> **阶段二更新（2026-10-09）**：E2-T.2 完整交互常数管线已实现于 `src/modules/e2t_pipeline.py`（safe β=1 包络、tight β 非负 LP min Σβ s.t. Δ_t(A)≤Σd+Σβ，scipy linprog highs），并有单元测试（test_safe_beta_covers_generated_data、test_tight_lp_satisfies_constraints）。**剩余偏差**：10000-bootstrap 正式规模、VAL_BOUND 真实数据冻结 manifest 仍未运行（需 Linux GPU 服务器与 OPV2V 数据集）。

## D. Partial Implementations

### Deviation D7: Math functions exist but no pipeline integration

| Field | Content |
|-------|---------|
| Deviation ID | D7 |
| Plan requirement | E2-T.1: per-(train_seed, a, gamma, m) computation of R0, d_k, w_k, p_e_k, p_e_ij, eta_k, p_u, B(m|a,gamma) |
| Current implementation | Individual NumPy functions for each quantity; no pipeline connecting them to actual decoder/channel outputs |
| Difference | Functions are tested in isolation with synthetic data; no real per-frame computation from actual decoder runs |
| Reason | Toy implementation; requires full decoder + channel pipeline with per-role erasure |
| Impact on results | Cannot compute actual bound values; cannot freeze bound tables; cannot evaluate controllers |
| Future remediation | Integrate math functions into full pipeline; compute per-(seed,a,gamma,m) from actual decoder/channel runs |

### Deviation D8: Synchronized permutation test is simplified

| Field | Content |
|-------|---------|
| Deviation ID | D8 |
| Plan requirement | Synchronized permutation of query, packet region, profile mask, and receiver embedding; output logits max abs error < 1e-5 |
| Current implementation | Tests permutation of group features and quantizer embeddings + profile mask; verifies permutation+inverse produces identical output |
| Difference | Does not test full synchronized permutation of all 4 components; does not test output logits |
| Reason | Toy implementation; no selector/receiver with role embeddings |
| Impact on results | Cannot verify full gauge equivalence; partial structural verification only |
| Future remediation | Implement full synchronized permutation with selector, profile mask, receiver embedding |

## E. NOT Implemented

The following major components from the ISRC plan have NO implementation:

### E1: Data and Backbone (Sections 3-4)
- OPV2V dataset: NOT IMPLEMENTED
- V2V4Real dataset: NOT IMPLEMENTED
- OpenCOOD framework: NOT IMPLEMENTED
- PointPillars backbone: NOT IMPLEMENTED
- Feature cache (Zarr/NPZ, FP16/FP32): NOT IMPLEMENTED
- Data splits (TRAIN/VAL_SELECT/VAL_ALIGN/VAL_BOUND/TEST_AUDIT): NOT IMPLEMENTED
- Collaborator selection: NOT IMPLEMENTED
- Difficulty binning: NOT IMPLEMENTED
- Cache equivalence verification: NOT IMPLEMENTED
- Admission (confidence-based token selection): NOT IMPLEMENTED

### E2: Network Components (Section 4.3)
- Role extractor (masked cross-attention, K learned queries): NOT IMPLEMENTED
- Selector (profile logits MLP): NOT IMPLEMENTED
- Receiver fusion (cross-attention with role embeddings): NOT IMPLEMENTED
- Cold-start/erasure symbol: NOT IMPLEMENTED

### E3: Training (Section 7)
- Cached-interface training: NOT IMPLEMENTED
- Cross-view role stability training: NOT IMPLEMENTED
- Role loss (L_role): NOT IMPLEMENTED
- Decode loss (L_decode): NOT IMPLEMENTED
- Calibration loss (L_cal): NOT IMPLEMENTED
- Gumbel-softmax selector annealing: NOT IMPLEMENTED
- Optimizer configuration (6-grid search): NOT IMPLEMENTED
- Early stopping (weighted risk loss): NOT IMPLEMENTED
- 5 training seeds {1103, 2207, 3301, 4409, 5513}: NOT IMPLEMENTED

### E4: Baselines (Section 5.1)
- B0 (Ego-only): NOT IMPLEMENTED
- B3 (Silent truncation): NOT IMPLEMENTED
- B4 (Flat fixed vector): NOT IMPLEMENTED
- B5 (Grouped fixed bottleneck): NOT IMPLEMENTED
- B6 (Query slots, no role loss): NOT IMPLEMENTED
- B8 (Full ISRC): NOT IMPLEMENTED

### E5: Profile Controllers (Section 5.2)
- P0 (Uniform): NOT IMPLEMENTED
- P1b (Static-best): NOT IMPLEMENTED
- P3 (Source-mask-only): NOT IMPLEMENTED
- P4 (Code-mask-only): NOT IMPLEMENTED
- P5 (Learned joint): NOT IMPLEMENTED
- P5b-safe (Bound-selected, beta=1): NOT IMPLEMENTED
- P5b-tight (Bound-selected, LP): NOT IMPLEMENTED
- P6 (Oracle): NOT IMPLEMENTED

### E6: Role Interventions (Section 5.3)
- All 7 intervention operators: NOT IMPLEMENTED
- Role certification (Stab/Sep/Nec): NOT IMPLEMENTED
- Hungarian cross-seed alignment: NOT IMPLEMENTED
- Bootstrap alignment uncertainty: NOT IMPLEMENTED

### E7: E2-T Bound Audit (Section E2-T)
- E2-T.0 full 5-test suite: NOT IMPLEMENTED
- E2-T.1 per-(seed,a,gamma,m) pipeline: NOT IMPLEMENTED
- E2-T.2 bounded-interaction audit (2^K-1 subsets, tight LP): NOT IMPLEMENTED
- E2-T.3 coupling stress replay: NOT IMPLEMENTED
- E2-T.4 simultaneous UCB (10000 bootstrap, Romano-Wolf): NOT IMPLEMENTED
- E2-T.5 TEST evaluation + controller comparison: NOT IMPLEMENTED
- Cross-fit diagnostics: NOT IMPLEMENTED
- Freeze manifest: NOT IMPLEMENTED

### E8: E3 Exact Contract (Section E3)
- Three N0 contracts: NOT IMPLEMENTED
- Bit-exact N0/C0 audit: NOT IMPLEMENTED
- Graph/operator/shape/iteration audit: NOT IMPLEMENTED
- FLOPs/memory upper envelope: NOT IMPLEMENTED
- Overload/fallback frontier: NOT IMPLEMENTED
- Silent truncation comparison: NOT IMPLEMENTED

### E9: Statistics (Section 8.5)
- Paired bootstrap (10000): NOT IMPLEMENTED
- Romano-Wolf simultaneous correction: NOT IMPLEMENTED
- Holm correction: NOT IMPLEMENTED
- Sequence-level statistical unit: NOT IMPLEMENTED
- Channel seed nesting: NOT IMPLEMENTED

### E10: Structure Audit (Section 10)
- Static graph export: NOT IMPLEMENTED
- Graph hash: NOT IMPLEMENTED
- Operator sequence audit: NOT IMPLEMENTED
- Tensor shape audit: NOT IMPLEMENTED
- FLOPs/MACs: NOT IMPLEMENTED
- Memory upper envelope: NOT IMPLEMENTED
- Reference latency: NOT IMPLEMENTED

## F. Cannot Be Verified in Current Environment

| Item | Reason |
|------|--------|
| OPV2V data experiments | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| GPU training (正式规模) | 需要 Linux GPU 服务器（本机为 RTX 3050 Ti 4GB + PyTorch 2.14.1+cu126，可运行 toy/surrogate 规模，不满足正式训练条件） |
| OpenCOOD/PointPillars | Not installed（需随 OpenCOOD 框架源码安装，且 mmdet3d 在 Windows 编译困难） |
| CUDA-specific behavior (正式实验) | 正式实验需 Linux GPU 服务器环境 |
| Container/Conda lockfile | Not provided |
| Git commit hash | 本地仓库存在（d:\code\1\.git），审计时尚未打审计 tag |

## G. 阶段二新增偏差与边界声明（2026-10-09）

本节记录第二阶段（实现 ISRC 缺失模块 + 本审计更新）新增/需明确的偏差与边界。

### G1. 当前仅为合成数据流 toy 仿真

- 所有 E2-T 管线数值结果（`results_e2t/` 下 bound_terms.csv、bound_ucb_table.csv、bound_profile_table_safe/tight.csv、coverage_report.csv、controller_comparison.csv、controller_snr_auc.csv）均产自 **synthetic surrogate（合成代用流）**，由 `scripts/run_e2t_toy.py` 内置的确定性合成流生成器产生，**不是 OPV2V 真实数据**。
- `results_e2t/PROVENANCE.txt` 已注明该数据不得用于论文主张。**toy 仿真结果不等价为 ISRC 完整论文实验结论。**
- surrogate 中 P1b 选择 profile 0、TEST coverage 100% 等数值仅证明管线数值正确性与闭环可用性，不代表真实感知场景下的性能。

### G2. stub 未实现项（如实清单）

| Stub 项 | 现状 | 阻塞条件 |
|---------|------|---------|
| B5/B6/B8 角色基线训练 | 未实现（无网络组件、无训练循环） | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| 匈牙利跨种子对齐 | 未实现 | 依赖 B5/B6/B8 多种子训练产物，同上 |
| 三类 accepted 干预角色证书（Stab/Sep/Nec） | 未实现（INT-2~INT-7 全部 stub） | 依赖真实训练模型与数据，同上 |
| LP-bootstrap 全套风险上界计算（正式实验级） | 管线已实现于 `src/modules/e2t_pipeline.py` 并在合成代用流上验证，但 **正式规模（10000 bootstrap × 真实 VAL_BOUND 数据 × 全部 cell）未运行**，无 freeze manifest | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| OPV2V 流水线 | 未实现（无数据下载、无 feature cache、无划分、无 cache 等价验证） | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| P5 学习控制器 | `controllers.py` 中以 NotImplementedError 如实占位（不伪造结果） | 需要 OPV2V 训练数据 |

### G3. 模块级实现但未集成主链路

| 模块 | 状态 | 集成缺口 |
|------|------|---------|
| `crc_bch.py`（CRC-16/BCH(127,64) 控制块） | 实现并测试通过（14 tests） | 未接入 mother_code/receiver 主链路；主链路头仍为 32-bit profile_id |
| `counter_rng.py`（counter-based PRNG） | 实现并测试通过（8 tests） | 未接入 software_channel 主链路；P2 控制器已使用 |
| `risk_strata.py`/`metrics_risk.py` | 实现并测试通过 | 未接入真实感知输出（无检测框/TTC 数据源） |
| `e2t_pipeline.py` | 实现并测试通过，surrogate 端到端贯通 | 未接入真实信道/解码器逐帧输出流 |

### G4. toy 仿真可验证范围 vs 正式实验所需条件

| 维度 | toy/surrogate 可验证（本机已完成） | 正式实验所需（未满足） |
|------|-----------------------------------|------------------------|
| 比特层 | 母码/profile 往返、CRC/BCH 控制块、单比特纠错、counter PRNG 前缀一致性 | 与真实 feature 量化的端到端比特流 |
| 公式层 | E2-T.0 五项强制单元测试、LP/UCB/coverage 公式 | 真实数据上的经验包络与 freeze manifest |
| 管线层 | surrogate 260 cell 端到端、控制器查表/回退、SNR-AUC 梯形 | OPV2V 五划分、5 训练种子 {1103,2207,3301,4409,5513}、正式 TEST_AUDIT |
| 硬件 | RTX 3050 Ti 4GB（toy 规模） | Linux GPU 服务器（训练 + 大规模 bootstrap） |
| 数据 | 合成代用流（确定性生成） | OPV2V 数据集（数百 GB）+ OpenCOOD/PointPillars feature cache |

## Summary

| Category | Count |
|----------|-------|
| A. Fully implemented per plan | 8 |
| B. Simplified implementations | 5 |
| C. Alternative implementations | 1 |
| D. Partial implementations | 2 |
| E. Not implemented | 50+ items across 10 categories |
| F. Cannot be verified | 6 |
| G. 阶段二新增偏差与边界声明 | 4 小节（合成数据流边界、stub 清单、集成缺口、toy vs 正式条件） |

The current project is a **toy/skeleton implementation** of the codec/channel core. It validates the basic mechanics of profile generation, LDPC encoding/decoding, QPSK+AWGN channel simulation, and basic E2-T math functions. It does NOT implement the ISRC experiment plan's core research contributions (role formation, profile adaptation, risk bound audit, exact contract verification).

**阶段二补充（2026-10-09）**：第二阶段新增 6 个审计模块（counter_rng、crc_bch、risk_strata、metrics_risk、controllers、e2t_pipeline）与 66 项单元测试（全套 96 passed），E2-T 管线在合成代用流上端到端贯通。上述新增实现均为**模块级/合成数据级**，尚未集成主链路，也未接触真实数据。当前项目整体仍为**合成数据流上的 toy 仿真**，正式实验结论必须等待 Linux GPU 服务器与 OPV2V 数据集条件满足后产生。
