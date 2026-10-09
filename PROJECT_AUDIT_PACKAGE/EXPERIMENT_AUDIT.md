# Experiment Audit（实验审计）

## 审计依据

本审计将 `d:\code\1` 当前实现与 ISRC 完整实验方案（`ISRC实验方案(1)(1).md`）逐条对照。方案定义 5 个实验组（E1、E1-R、E2、E2-T、E3）、11 个 Gate（G0-G10）、基线 B0-B8、控制器 P0-P6/P5b 及完整统计流程。

当前项目为 **合成数据流上的 toy 实现 + 第二阶段新增审计模块**。本次更新（2026-10-09）逐行重新核对了全部状态。

## 本次审计执行记录（2026-10-09）

- 一键审计脚本 `PROJECT_AUDIT_PACKAGE/reproduce/audit_all.py` 完整运行：environment / config / tests / determinism / minimal_experiment / output_check 六步全部 **PASS**，OVERALL: **PASS**。
- pytest 全量测试：**96 passed in 7.64s**（既有 30 项 + 第二阶段新增 66 项，逐项 PASSED 证据见 `results_raw/pytest_verbose_output.txt` 同规格的本次运行记录与 `logs/audit_run_log.md`）。
- E2-T toy 端到端：`scripts/run_e2t_toy.py --fast` 于 **合成代用流（synthetic surrogate）** 上运行完成（260 cell，结果在 `results_e2t/`，PROVENANCE.txt 已注明不得用于论文主张）。

## 统一阻塞说明

**凡需要真实数据集或 GPU 训练/服务器的实验项，统一阻塞原因：需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行。**

## Gates（方案 §11）

| ID | Experiment/Test | Executed | Result Location | Status | Notes |
|----|----------------|----------|----------------|--------|-------|
| G0 | Cache/data gate：5 类划分、online-vs-cache、cache hash | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| G1 | Bit gate：profile 往返、长度、CRC、边界 | Yes（toy 比特级） | tests/test_structure.py, tests/test_crc_bch.py, tests/test_mother_code.py | PASS | toy 可执行范围：13 profile 噪声精确往返、8192 bit 长度、CRC-16/BCH(127,64) 控制块比特往返与纠错全部 PASS；注：BCH 控制块尚未集成进 mother_code 主链路（见 CODEC-5 notes） |
| G2 | Topology gate：graph hash、算子序列、固定迭代 | No | - | NOT RUN | 未实现 TorchScript/ONNX 静态图导出与审计管线（不依赖数据集，属待开发项） |
| G3 | Sanity gate：B8 ≥ Ego-only、协同正增益 | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| G4 | Role gate：E1 角色认证（75% certified、≥25pp 优势） | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行；B5/B6/B8 角色基线与干预证书均为 stub |
| G5 | Role-robustness gate：E1-R 压力保持 | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| G6 | Oracle gate：P6 oracle gap ≥ 3% | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行；P6 控制器类已实现并有单元测试（tests/test_controllers.py），正式 profile 表须来自真实数据训练 |
| G7 | Formal-anchor gate：E2-T.0 公式单元测试、safe β=1 包络 | Yes（toy） | tests/test_e2t_pipeline.py | PASS | 五项强制公式单元测试（test_e2t0_1 误差免费回放 / test_e2t0_2 单擦除与全拒绝 / test_e2t0_3 协方差手算 / test_e2t0_4 边缘-联合与 U 计数 / test_e2t0_5 safe β=1 RHS 覆盖）全部 PASS |
| G8 | Empirical-tight/controller gate：E2-T tight LP/bootstrap | Partial | tests/test_e2t_pipeline.py | INCONCLUSIVE | tight β LP 约束满足测试与 Romano-Wolf/empirical-Bernstein/CP UCB 单元测试 PASS；正式规模（10000 bootstrap、VAL_BOUND 冻结 manifest）需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| G9 | Adaptation gate：E2 P5b vs P1b 风险 AUC | No（正式） | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行；控制器框架与 P5b 已实现并在合成代用流上贯通（results_e2t/controller_snr_auc.csv，不得用于论文主张） |
| G10 | Exact/frontier gate：E3 N0 契约、overload frontier | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |

## 数据准备（方案 §3）

| ID | Task | Executed | Result Location | Status | Notes |
|----|------|----------|----------------|--------|-------|
| DATA-1 | OPV2V 数据下载与 hash | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| DATA-2 | Feature cache 提取（PointPillars） | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| DATA-3 | 数据划分（TRAIN/VAL_SELECT/VAL_ALIGN/VAL_BOUND/TEST_AUDIT） | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| DATA-4 | 协作者选择（pair_manifest.yaml） | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| DATA-5 | split_audit.csv 生成 | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| DATA-6 | 复杂度/难度分箱（TRAIN 分位数） | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| DATA-7 | SHA-256 序列交集检查 | No | - | NOT RUN | 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |

## Profile 与 Codec（方案 §6）

| ID | Task | Executed | Result Location | Status | Notes |
|----|------|----------|----------------|--------|-------|
| PROFILE-1 | 13-profile 确定性生成 | Yes | tests/test_profile_generator.py | PASS | 13 profiles、sum(n)=8064、q<=n、确定性 hash |
| PROFILE-2 | Profile mask 生成（128-bit） | Yes | src/modules/profile_generator.py | PASS | SHA-256 掩码 |
| PROFILE-3 | Profile 表 rank/可解码性检查 | No | - | NOT RUN | 未实现 |
| CODEC-1 | 母码编码（profile 头 + 角色 payload） | Yes | tests/test_structure.py | PASS | 128-bit 头 + 8064-bit payload = 8192 bits（母码-profile 链路 toy PASS） |
| CODEC-2 | 母码解码（固定 10 次迭代 bit-flipping） | Yes | tests/test_structure.py | PASS | 固定 10 次迭代、单比特错误纠正 |
| CODEC-3 | Shortening 接口 | Yes | tests/test_mother_code.py | PASS | 前 n 位置 0 |
| CODEC-4 | Puncturing 接口 | Yes | tests/test_mother_code.py | PASS | 删除末尾 n bits |
| CODEC-5 | BCH(127,64) + CRC-16 控制块编码 | Yes | src/modules/crc_bch.py, tests/test_crc_bch.py | PASS | CRC-16/CCITT-FALSE + BCH(127,64) 系统编码（t=10，生成多项式由 cyclotomic coset 构造并验证 g(1)=1、g(x) | x^127-1）；控制块比特往返、字段宽度、CRC 覆盖 source bits 均有测试。注：尚未集成进 mother_code 主链路（主链路头仍为 32-bit profile_id） |
| CODEC-6 | 比特精确往返（全部 profile、空/满/边界） | Yes（toy） | tests/test_structure.py, tests/test_crc_bch.py | PASS | 噪声less 母码往返精确（test_noiseless_round_trip_is_exact）+ 控制块比特往返（test_bch_roundtrip_bit_exact, test_control_block_roundtrip）；全 13 profile×空/满量化边界覆盖仍属正式实验范围 |
| CODEC-7 | 单比特翻转测试（纠正或检出） | Yes（toy） | tests/test_structure.py, tests/test_crc_bch.py | PASS | 母码单比特纠正（test_profile_decoder_corrects_a_single_bit_error）+ BCH ≤t 纠错（test_bch_corrects_up_to_t_errors）+ 超过 t 不静默（test_beyond_t_never_silent_wrong） |

## 信道（方案 §6.3）

| ID | Task | Executed | Result Location | Status | Notes |
|----|------|----------|----------------|--------|-------|
| CHANNEL-1 | QPSK 调制（单位能量） | Yes | tests/test_software_channel.py | PASS | 符号能量 = 1.0 |
| CHANNEL-2 | AWGN 噪声（SNR 派生 std） | Yes | tests/test_software_channel.py | PASS | 噪声功率与计算 std 一致 |
| CHANNEL-3 | 固定种子可复现 | Yes | tests/test_structure.py | PASS | 同种子 = 相同噪声观测 |
| CHANNEL-4 | 5 点 AWGN 扫频 {-2,0,2,4,6} dB | Partial（toy surrogate） | results_e2t/controller_snr_auc.csv | INCONCLUSIVE | SNR-AUC 梯形已在合成代用流上计算（scripts/run_e2t_toy.py），明确标注 synthetic surrogate；正式 5 点扫频×5 seeds 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| CHANNEL-5 | Burst-mid（Gilbert-Elliott） | No | - | NOT RUN | 未实现 |
| CHANNEL-6 | Region-erasure-mid（per-role 0.10） | No | - | NOT RUN | 未实现 |
| CHANNEL-7 | Counter-based PRNG（dataset,seq,frame,model,seed,bitpos） | Yes | src/modules/counter_rng.py, tests/test_counter_rng.py | PASS | Philox 计数器模式实现；同 key 可复现、bit_position 前缀一致（test_prefix_property）、channel_seed/bit_position 改变流。注：尚未接入 software_channel 主链路 |

## 接收与擦除（方案 §6.2）

| ID | Task | Executed | Result Location | Status | Notes |
|----|------|----------|----------------|--------|-------|
| RECV-1 | 接收端完整流程 | Yes | tests/test_end_to_end.py | PASS | 解码 + 擦除 + 错误检出 |
| RECV-2 | 解码失败检出（all-0/all-1/diff>50%） | Yes | fixed_resource_comm/test.py | PASS | 全部失败模式检出 |
| RECV-3 | 软信息擦除（阈值） | Yes | tests/test_decoder_erasure.py | PASS | |noisy I/Q| < 阈值标记擦除 |
| RECV-4 | Syndrome/LLR-validity 擦除 | No | - | NOT RUN | 未实现；现为软阈值擦除 |
| RECV-5 | CRC 拒帧 → 全角色 declared erasure | Partial | src/modules/crc_bch.py, tests/test_crc_bch.py | INCONCLUSIVE | 拒帧语义已实现并有测试（test_control_block_rejects_source_bit_error：解码失败返回 success=False，语义为整帧拒绝→全角色擦除）；尚未接入 receiver 完整链路 |
| RECV-6 | 未检出错误（U_t）统计 | Partial | src/modules/e2t_pipeline.py, tests/test_e2t_pipeline.py | INCONCLUSIVE | U 事件流计量已在 E2-T.0.4 单元测试与 surrogate 管线中实现；逐比特与发射端真值比对的正式链路未接入 |

## 数学/上界函数（方案 E2-T）

| ID | Task | Executed | Result Location | Status | Notes |
|----|------|----------|----------------|--------|-------|
| MATH-1 | R0 计算 | Yes | tests/test_math_functions.py | PASS | 手算验证 |
| MATH-2 | d_k 计算 | Yes | tests/test_math_functions.py | PASS | 截断验证 |
| MATH-3 | w_k 计算 | Yes | tests/test_math_functions.py | PASS | 均值验证 |
| MATH-4 | p_e_k 计算 | Yes | tests/test_math_functions.py | PASS | 边缘概率验证 |
| MATH-5 | p_e_ij 计算 | Yes | tests/test_math_functions.py | PASS | 联合概率（非乘积）验证 |
| MATH-6 | eta_k 计算 | Yes | tests/test_math_functions.py | PASS | |Cov(E,d)| = 0.25 |
| MATH-7 | p_u 计算 | Yes | tests/test_math_functions.py | PASS | 2/5 错误率 |
| MATH-8 | LP 接口（scipy linprog） | Yes | tests/test_lp_interface.py | PASS | 3 个 LP 问题正确求解 |

## E2-T 子实验（方案 §9 E2-T）

| ID | Task | Executed | Result Location | Status | Notes |
|----|------|----------|----------------|--------|-------|
| E2-T.0 | 五项强制公式单元测试 | Yes | tests/test_e2t_pipeline.py | PASS | 五项全部 PASS：test_e2t0_1_error_free_replay（|R-R0|<1e-8）、test_e2t0_2_single_erasure_and_all_reject、test_e2t0_3_covariance_hand_samples、test_e2t0_4_marginal_vs_joint_and_u_counting、test_e2t0_5_beta_one_rhside_coverage（1000 随机 (t,A) safe β=1 RHS 覆盖） |
| E2-T.1 | 观测量（R0/d/w/p_e/p_e,ij/η/p_u/R_hat 序列簇均值） | Yes（toy surrogate） | src/modules/e2t_pipeline.py, tests/test_e2t_pipeline.py, results_e2t/bound_terms.csv | PASS | compute_cell_terms 簇均值实现 + test_cluster_mean_two_level/test_p_e_ij_not_product 单元测试；surrogate 260 cell 数值已产出（PROVENANCE.txt 注明 synthetic surrogate） |
| E2-T.2 | 交互常数 + 有界交互审计 | Yes（toy surrogate） | tests/test_e2t_pipeline.py, results_e2t/bound_profile_table_safe.csv, bound_profile_table_tight.csv | PASS | safe β=1 覆盖测试（test_safe_beta_covers_generated_data）+ tight LP 约束满足测试（test_tight_lp_satisfies_constraints）；surrogate 生成 safe/tight 各 20 格表 |
| E2-T.3 | 受控耦合压力回放 | No | - | NOT RUN | 未实现场景级/角色级故障回放 |
| E2-T.4 | Simultaneous UCB + freeze | Partial | tests/test_e2t_pipeline.py | INCONCLUSIVE | Romano-Wolf max-t、empirical-Bernstein/CP、支持门槛、cross-fit 均已实现并有单元测试（test_ucb_not_below_point_estimate 等）；正式 10000 bootstrap 与 VAL_BOUND freeze manifest 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| E2-T.5 | TEST 评估 + 控制器比较 | Partial | results_e2t/coverage_report.csv, controller_comparison.csv | INCONCLUSIVE | surrogate 上 TEST coverage 100%（CP CI [0.753,1.0]）、0 confirmed violation、六种消融可运行；全部标注 synthetic surrogate 不得用于论文主张；正式 TEST_AUDIT 需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |

## 基线（方案 §5.1）

| ID | Baseline | Executed | Result Location | Status | Notes |
|----|----------|----------|----------------|--------|-------|
| B0 | Ego-only sanity | No | - | NOT RUN | 未实现（stub）；正式运行需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| B3 | Silent truncation | No | - | NOT RUN | 未实现（stub）；同上 |
| B4 | Flat fixed vector（3 seeds） | No | - | NOT RUN | 未实现（stub）；同上 |
| B5 | Grouped fixed bottleneck（5 seeds） | No | - | NOT RUN | 角色基线训练 stub 未实现；需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| B6 | Query slots、无 role loss（5 seeds） | No | - | NOT RUN | 角色基线训练 stub 未实现；需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| B8 | Full ISRC（5 seeds） | No | - | NOT RUN | 角色基线训练 stub 未实现；需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |

## Profile 控制器（方案 §5.2、§7.4）

| ID | Controller | Executed | Result Location | Status | Notes |
|----|-----------|----------|----------------|--------|-------|
| P0 | Uniform | Yes | tests/test_controllers.py | PASS | test_p0_uniform |
| P1b | Static-best | Yes | tests/test_controllers.py | PASS | test_p1b_never_changes |
| P3 | Source-mask-only | No | - | NOT RUN | 未实现 |
| P4 | Code-mask-only | No | - | NOT RUN | 未实现 |
| P5 | Learned joint | No | - | NOT RUN | src/modules/controllers.py 中以 NotImplementedError 如实占位；训练需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行 |
| P5b-safe | Bound-selected（β=1） | Yes（toy surrogate 表） | tests/test_controllers.py, results_e2t/bound_profile_table_safe.csv | PASS | 查表、tie-break（B→R0→profile_id）、unsupported 回退 P1b 均有测试；表来自 synthetic surrogate |
| P5b-tight | Bound-selected（LP） | Yes（toy surrogate 表） | tests/test_controllers.py, results_e2t/bound_profile_table_tight.csv | PASS | 同上；tight 表由 tight β LP 生成 |
| P6 | Oracle | Yes（单元测试） | tests/test_controllers.py | PASS | test_p6_oracle_uses_labels（逐帧风险 argmin）；正式 oracle gap 评估需要真实数据（见 G6） |

## 角色干预（方案 §5.3）

| ID | Intervention | Executed | Result Location | Status | Notes |
|----|-------------|----------|----------------|--------|-------|
| INT-1 | 同步置换 | Partial | tests/test_structure.py | INCONCLUSIVE | toy 置换+逆测试通过（err<1e-5）；完整 ISRC 四分量同步置换未实现 |
| INT-2 | 接收端置换 | No | - | NOT RUN | 未实现（stub） |
| INT-3 | Matched null | No | - | NOT RUN | 未实现（stub） |
| INT-4 | 层内 shuffle | No | - | NOT RUN | 未实现（stub） |
| INT-5 | 条件替换 | No | - | NOT RUN | 未实现（stub） |
| INT-6 | Query sharing | No | - | NOT RUN | 未实现（stub） |
| INT-7 | 移除 role embedding | No | - | NOT RUN | 未实现（stub） |

注：三类 accepted 干预对应的角色证书（Stab/Sep/Nec）与匈牙利跨种子对齐均为 stub 未实现（见 DEVIATIONS_AND_LIMITATIONS.md 第 G 节）。

## 结构/时延审计（方案 §10）

| ID | Task | Executed | Result Location | Status | Notes |
|----|------|----------|----------------|--------|-------|
| STR-1 | 静态图导出（TorchScript/ONNX） | No | - | NOT RUN | 未实现（onnxruntime 已安装但无导出管线；不依赖数据集，属待开发项） |
| STR-2 | Graph hash + 算子序列 | No | - | NOT RUN | 未实现 |
| STR-3 | 张量形状审计 | No | - | NOT RUN | 未实现 |
| STR-4 | FLOPs/MACs | No | - | NOT RUN | 未实现 |
| STR-5 | 内存上包络 | No | - | NOT RUN | 未实现 |
| STR-6 | 参考时延（可选） | No | - | NOT RUN | 未实现 |

## 汇总统计

本次逐行重新核对（修正上一版 83 项的计数误差，现共 82 项）：

| Status | Count |
|--------|-------|
| PASS | 34 |
| INCONCLUSIVE | 5 |
| NOT RUN | 43 |
| FAIL | 0 |
| **Total** | **82** |

**核心结论：**
1. toy 可执行且已 PASS 的部分：比特往返（G1）、母码-profile 编解码链路（CODEC-1~7）、CRC-16/BCH(127,64) 控制块、counter-based PRNG、E2-T.0 五项强制公式单元测试（G7）、风险分层/指标/控制器（P0/P1b/P2/P5b/P6）单元测试、E2-T 管线单元测试。
2. 在合成代用流（synthetic surrogate）上贯通但不得用于论文主张：E2-T.1/.2 surrogate 表、SNR-AUC、TEST coverage、控制器比较。
3. 全部正式实验项（数据、训练、基线、正式统计、E3 契约）NOT RUN，统一阻塞原因：**需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行**。
4. 本审计不将 toy/surrogate 仿真结果等价为 ISRC 完整论文实验结论。
