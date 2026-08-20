---
title: "Correlation Core"
date: 2026-08-19
layout: "project"
status: "IN PROGRESS"
math: true
dek: "A public, versioned H100 compute-communication interference benchmark and an interpretable per-resource slowdown model, evaluated against predictions frozen and hashed before the measurement session runs."
intro: "The premise is not that existing training simulators ignore contention. They do not. The premise is that the calibration layer underneath them is rarely public: which resource was measured, under what directed intervention, with what run-to-run distribution, and whether the prediction could have been retuned after seeing the answer. That layer is what this publishes."
specs:
  - k: "SYSTEM"
    v: "2x H100 SXM, one NVLink domain"
  - k: "COMPUTE"
    v: "BF16 GEMM, HBM triad, short GEMM"
  - k: "COMMS"
    v: "AllReduce, ReduceScatter, 5 sizes"
  - k: "MATRIX"
    v: "120 overlap configurations"
  - k: "SCORING"
    v: "separate calibration, validation, holdout"
  - k: "RESULTS"
    v: "published when they exist"
    tone: "warn"
homespecs:
  - k: "SYSTEM"
    v: "2x H100 SXM, one NVLink domain"
  - k: "MATRIX"
    v: "3 kernels, 2 collectives, 5 sizes, 4 offsets"
  - k: "SCORING"
    v: "frozen holdout, residuals attributed"
---

## The method, one level up

| Pre-silicon GPU work | This project |
| --- | --- |
| Directed SASS tests isolating one microarchitectural behavior | Directed CUDA and NCCL microbenchmarks isolating one resource |
| Cycle-accurate C++ model predictions | Resource-aware C++ timeline replay predictions |
| Correlation against RTL ground truth | Correlation against measured H100 ground truth |
| Speed-of-light bounds per interface, SM to DDR | Bounds per resource: tensor pipes, SM, HBM, L2, NVLink |
| Regression triage into a design decision | Residual ledger, falsifying tests, software conclusion |

## What is being predicted

Four predictors are scored against the same held-out overlap conditions, so the naive assumption
has to compete rather than serve as a strawman: ideal `max(compute, comm)`, full serialization,
one scalar slowdown factor, and the per-resource model. The interesting output is not which one
wins. It is which conditions each one fails on, and whether the failure was predicted in advance.

{{< eq note="Model form only. Coefficients are fitted on calibration data and frozen before the holdout session." >}}
$$ \hat{T}_{\text{pair}} = \max\left(T_c \cdot s_c(\rho),\; T_m \cdot s_m(\rho)\right) + \beta_q \cdot Q(b) $$
{{< /eq >}}

## Claim boundaries

- A two-rank NVLink study, not rack-scale simulation.
- Causal attribution is claimed only where a controlled intervention supports it. Everything else
  is reported as accounting.
- Nothing here is novel or first. Echo, Charon and ASTRA-sim 3.0 are the baselines, cited as such.
- Every result names the GPU SKU, topology, software versions, and prediction status.

## Prior work

- [Echo](https://arxiv.org/abs/2412.12487), arXiv 2412.12487. Learned interference slowdown at scale.
- Charon, MLSys 2026. Profiled ratio-based slowdown across parallelism strategies.
- ASTRA-sim 3.0, arXiv 2606.10440. Mechanistic unified compute and communication contention.
- Measured H100 overlap characterization, [arXiv 2507.03114](https://arxiv.org/abs/2507.03114).
