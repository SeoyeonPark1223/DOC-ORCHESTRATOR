# Shared S0-S3 Scenario Definitions

This context is injected into ALL part agents as common knowledge.

## Scenario Overview

| Scenario | Korean Title | Goal | Key Question |
|----------|-------------|------|-------------|
| **S0** | "일단 돌아가나?" | Preset-based first success | Does the default pipeline produce a working result? |
| **S1** | "왜 이런 결과가 나왔지?" | Intermediate result debugging | Why did the pipeline produce this specific output? |
| **S2** | "어떤 설정이 제일 좋은가?" | User-defined sweep experiments | Which configuration gives the best results? |
| **S3** | "이 결과를 다시 평가할 수 있나?" | Post-analysis & re-evaluation | Can we revisit and compare past results? |

## S0: First Success (Preset Pipeline)

- Minimal config, user picks model + target device only
- Default pipeline: `GO → GQ` or `AQ → GO → GQ` (depending on preset)
- No sweep, no comparison — single run
- Target: RPI5 + ExecuTorch + Xnnpack
- CLI: `np run --default`
- Success = model runs on target device, produces valid output

## S1: Result Debugging

- Layerwise SNR analysis (per-operator signal-to-noise ratio)
- Latency breakdown by optimization step
- Observer statistics (min/max/mean per tensor)
- Fallback cause analysis (why ops fell back to CPU from NPU)
- Probe visualization for graph diff before/after optimization

## S2: Sweep Experiments

- User-defined YAML with `sweep` section (multiple configs)
- Multiple AQ/GO/GQ variants compared
- `best` selection by metric (latency, accuracy, SNR, model size)
- YAML-driven: `np run --config run.yaml`
- Dashboard comparison across runs

## S3: Post-Analysis & Re-evaluation

- Re-evaluate past runs with different metrics
- Compare across experiments (cross-experiment analysis)
- Dashboard visualization for experiment comparison
- Probe graph diff between different optimization results
- Artifact export (model files, reports, configs)

## Reference Pages

| Page | Page ID |
|------|---------|
| S0 CLI Scenario | `1968439297` |
| S1 CLI Scenario | `1992327173` |
| S2 CLI Scenario | `1991770151` |
| S3 CLI Scenario | `1993244710` |
| NetsPresso V2.0 User Scenarios & Module Goals | `1998290955` |

## Cross-Cut Check Instruction

After analyzing the transcript from your part's perspective, perform the following cross-cut check:

1. **Does any decision change S0-S3 scenario definitions?**
   - e.g., changing the default pipeline order affects S0
   - e.g., adding a new metric affects S2 best-selection criteria

2. **Does any decision change the shared pipeline flow?**
   - e.g., adding a new optimization step between GO and GQ
   - e.g., changing the default target device from RPI5

3. **Does any decision affect multiple parts?**
   - e.g., a new IR node type affects both MR (data structure) and Q (QDQ pattern)
   - e.g., a new CLI command affects SWE (implementation) and the relevant domain part

If any cross-cut impact is detected, flag it in your analysis output as:
```
cross_cut_impacts:
  - scenario: S0/S1/S2/S3
    impact: "Description of how this decision affects the scenario"
  - shared_pipeline: true/false
    impact: "Description of pipeline flow change"
  - affected_parts: [list of other parts affected]
    impact: "Description of cross-part impact"
```
