# AUDIT REPORT

## 阶段一成果汇总（Stage 1 Summary，2026-10-09 更新）

> 本节为本审计报告的最新头部汇总。**以下"Project Information / Audit Results / Overall Status"等正文为第一阶段初版审计记录（当时 25 tests、PyTorch 2.12.1+cpu），其部分数字已被本节取代，保留作为历史口径。** 两阶段口径差异不再另行建 SUMMARY 文档，一律以本节为准。

### 1. 一键自检结果（reproduce/audit_all.py，2026-10-09 实际运行）

| 自检步骤 | 状态 |
|----------|------|
| STEP 1 环境检查（Python 3.12 / torch 2.14.1+cu126 / numpy / scipy / yaml / pytest） | PASS |
| STEP 2 配置检查（configs/toy_config.yaml） | PASS |
| STEP 3 全量测试（96 tests） | PASS |
| STEP 4 确定性测试（3 次运行同 SHA-256） | PASS |
| STEP 5 最小完整实验（demo pipeline） | PASS |
| STEP 6 必需产物文件检查 | PASS |
| **OVERALL** | **PASS** |

### 2. 测试统计

| 指标 | 数值 |
|------|------|
| pytest 总数 | **96 passed, 0 failed, 0 skipped（7.64s）** |
| 既有（阶段一初版）测试 | 30 项 |
| 第二阶段新增测试 | 66 项，分布于 6 个文件：test_crc_bch.py(14)、test_counter_rng.py(8)、test_risk_strata.py(9)、test_metrics_risk.py(9)、test_controllers.py(10)、test_e2t_pipeline.py(16，含 E2-T.0 五项强制公式测试) |
| 确定性 | 3 次独立运行 SHA-256 一致 |
| E2-T toy 端到端 | `scripts/run_e2t_toy.py --fast` 完成 260 cell（**synthetic surrogate，不得用于论文主张**） |

### 3. 实验审计状态（对照 G0-G10，详见 EXPERIMENT_AUDIT.md）

| Status | Count（本次逐行核对，共 82 项） |
|--------|------|
| PASS | 34 |
| INCONCLUSIVE | 5 |
| NOT RUN | 43 |
| FAIL | 0 |

- **toy 可执行并 PASS**：比特往返（G1）、母码-profile 链路（CODEC-1~7）、CRC-16/BCH(127,64) 控制块、counter-based PRNG、E2-T.0 五项公式单元测试（G7）、风险分层/风险指标/P0/P1b/P2/P5b/P6 控制器单元测试、E2-T.1/.2（surrogate）。
- **合成代用流贯通但不得用于论文主张**：E2-T.1/.2 surrogate 表、SNR-AUC、TEST coverage、控制器比较（results_e2t/，PROVENANCE.txt 已标注）。
- **INCONCLUSIVE**：G8/E2-T.4/.5（实现+单元测试 PASS，正式规模统计需真实数据）、RECV-5/RECV-6（模块级实现，未接入主链路）。

### 4. 阻塞项（统一原因）

**需要 Linux GPU 服务器以及 OPV2V 数据集，Windows 本机环境无法执行。** 受阻项包括：G0（cache/data gate）、G3-G6、G9-G10 正式实验、DATA-1~7 全部、B4-B8 训练、P5 学习控制器（NotImplementedError 如实占位）、E2-T 正式规模 10000 bootstrap 与 freeze manifest、E3 N0 契约。另有与数据无关的待开发项：G2/STR-1~6 静态图审计、PROFILE-3、RECV-4、CHANNEL-5/6、E2-T.3、P3/P4。

### 5. 仿真边界约束（不将 toy 等价为论文结论）

1. 本阶段全部 E2-T 数值结果产自**合成代用流（synthetic surrogate）**，非 OPV2V 真实数据。
2. toy 仿真仅证明：代码正确性（96 tests）、确定性（SHA-256 一致）、管线闭环可用性（260 cell 端到端）。
3. **不构成** ISRC 完整论文实验结论：无 H1（B8 vs B5/B6 角色形成）、H2（P5b vs P1b 自适应）、H3（E3 精确契约）任何正式证据。
4. 新增模块均为模块级实现：crc_bch/counter_rng 尚未接入 mother_code/software_channel 主链路；风险分层与指标未接入真实感知输出。
5. 详见 DEVIATIONS_AND_LIMITATIONS.md 第 G 节。

---

## Project Information（以下为第一阶段初版记录，历史口径）

| Field | Value |
|-------|-------|
| Project | Fixed Resource Communication (ISRC Toy Implementation) |
| Project Path | `d:\code\1` |
| Audit Date | 2026-10-09 |
| Auditor | Automated audit agent |
| Experiment Plan Reference | `ISRC实验方案(1)(1).md` |

## Audit Results

| Audit Item | Status | Notes |
|------------|--------|-------|
| Source/package | PASS | Complete source code copied to `source/`. All 10 modules in `src/modules/`, 9 test files, config, scripts present. |
| Project structure | PASS | `PROJECT_STRUCTURE.md` and `project_tree.txt` generated. All core files documented with function, input, output, core status. |
| Algorithm mapping | PASS | `ALGORITHM_MAPPING.md` generated. 7 implemented algorithms documented with pseudocode, source, config. Formula-to-variable-to-code mapping table complete. NOT IMPLEMENTED algorithms listed. |
| Environment/config | INCONCLUSIVE | Python 3.12.0, PyTorch 2.12.1+cpu, NumPy 2.5.0, SciPy 1.18.0. Config at `configs/toy_config.yaml`. Seed=42 used but ISRC plan seeds {1103,...} not present. No environment.yaml/lockfile. Hardcoded profile parameters. |
| Tests | PASS | 25/25 pytest tests passed in 8.70s. 6 failure/edge case tests passed. All tests cover unit, integration, and failure cases. |
| Determinism | PASS | 3 independent runs with seed=42 produced identical SHA-256 hashes for all tensors (input, encoded, received, decoded). |
| Manifest/log | PASS | 4 run manifests created (`manifests/run-001` through `run-004`). Audit run log in `logs/audit_run_log.md`. Raw outputs saved in `results_raw/`. |
| Result chain | PASS | Raw results saved (`results_raw/`), processed results generated (`results_processed/test_summary.csv`), tables generated (`tables/test_results_table.csv`, `tables/experiment_audit_table.csv`). No figures (no formal experiments). |
| Experiment audit | INCONCLUSIVE | 19 items PASS, 7 INCONCLUSIVE, 57 NOT RUN, 0 FAIL. Project implements codec/channel core only; vast majority of ISRC plan not implemented. |
| Deviation audit | PASS | `DEVIATIONS_AND_LIMITATIONS.md` generated. 8 fully implemented, 5 simplified, 1 alternative, 2 partial, 50+ not implemented. All deviations documented with reason and impact. |

## Overall Status

**OVERALL STATUS: PARTIAL**

### Justification

The project is a **toy/skeleton implementation** of the ISRC experiment plan. It successfully implements and validates:
1. 13-profile deterministic generation (Section 6.1.1)
2. LDPC profile encoding with 128-bit header -> 8192 bits (Section 6.1)
3. Fixed 10-iteration bit-flipping decoding (Section 6.1)
4. QPSK+AWGN software channel (Section 6.3)
5. Soft-information erasure detection (Section 6.2, simplified)
6. Basic E2-T bound math functions (R0, d_k, w_k, p_e, eta, p_u) (Section E2-T.1)
7. scipy linprog LP interface (Section E2-T.2, simplified)

However, the project does NOT implement:
- Any data pipeline (OPV2V, OpenCOOD, PointPillars, feature cache)
- Any baseline (B0, B3, B4, B5, B6, B8)
- Any profile controller (P0-P6, P5b-safe, P5b-tight)
- Any role intervention or certification
- Any training pipeline
- Any statistical testing (bootstrap, UCB, Romano-Wolf)
- Any structure audit (graph hash, FLOPs, memory)
- Any E2-T sub-experiment beyond basic math functions
- Any E3 exact contract
- BCH(127,64)/CRC-16 header protection
- Burst/region-erasure channel models
- Counter-based PRNG

The implemented core is **correct and deterministic** — all 25 tests pass, determinism is verified, and the toy acceptance flow succeeds. But the ISRC plan's core research contributions (role formation, profile adaptation, risk bound audit, exact contract verification) are entirely absent.

The status is PARTIAL (not FAIL) because:
- All implemented code works correctly and is well-tested
- The implemented subset is deterministic and reproducible
- The deviation documentation is complete and honest

The status is PARTIAL (not PASS) because:
- 57 out of 83 audited experiment items are NOT RUN
- No formal experiments have been executed
- The ISRC plan's core contributions are not implemented
- No data, no baselines, no training, no statistics

## PASS/FAIL/INCONCLUSIVE/NOT RUN Summary

| Status | Count (from experiment audit table) |
|--------|-------------------------------------|
| PASS | 19 |
| INCONCLUSIVE | 7 |
| NOT RUN | 57 |
| FAIL | 0 |
| **Total** | **83** |

## Most Important Unfinished/Failed Items

1. **No data pipeline** — OPV2V, OpenCOOD, PointPillars, feature cache all absent. This blocks ALL downstream experiments.
2. **No baselines (B5/B6/B8)** — The core H1 comparison (role formation) cannot be run.
3. **No profile controllers (P0-P6)** — The core H2 comparison (joint adaptation) cannot be run.
4. **No E2-T bound audit pipeline** — Only basic math functions exist; no full bootstrap, UCB, LP, freeze.
5. **No BCH/CRC header protection** — Frame rejection and declared erasure/fallback not possible.
6. **No counter-based PRNG** — Fairness contract for noise sharing across methods not satisfiable.
7. **No 5-point SNR sweep** — Cannot compute SNR-AUC, the primary H2 endpoint.
8. **No training seeds {1103, 2207, 3301, 4409, 5513}** — Cannot produce multi-seed results.

## One-Click Reproduction

```bash
python PROJECT_AUDIT_PACKAGE/reproduce/audit_all.py
```

This script:
1. Checks environment (Python, packages)
2. Checks configuration
3. Runs all 25 tests
4. Runs determinism test (3 runs)
5. Runs minimal experiment (demo pipeline)
6. Checks required output files
7. Prints audit summary
