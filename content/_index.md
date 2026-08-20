---
title: "Rutwik Pandit"
headline: "Wer immer strebend sich bemüht, den können wir erlösen."
headline_lang: "de"
headline_gloss: "He who strives him we can redeem"
headline_source: "Goethe, Faust II"
specs:
  - k: "LAYER"
    v: "SASS &rarr; uarch &rarr; RTL &rarr; fabric"
  - k: "METHOD"
    v: "directed workloads, model correlation"
  - k: "TOOLS"
    v: "CUDA, NCCL, Nsight, CUPTI, C++"
  - k: "AT"
    v: "CMU MS ECE, December 2026"
  - k: "STATUS"
    v: "open to December 2026 starts"
    tone: "accent"
jobs:
  - when: "2025 – 26"
    org: "NVIDIA"
    role: "Senior Architect, GPU Architecture Performance"
    body: "Full-chip pre-silicon speed of light across the SM to DDR datapath. Directed SASS microbenchmarks per subsystem, a cycle-accurate C++ performance model correlated against RTL, per-interface bounds including the on-die crossbar, and the triage that turns a discrepancy into a buffer-sizing or next-generation design decision."
  - when: "2022 – 25"
    org: "Qualcomm"
    role: "Senior Engineer, Platform Architecture"
    body: "Instruction-throughput-driven frequency scaling for on-device LLM inference, up to 80 percent better perf per watt, filed as a patent application. A system-level cache governor for XR SoCs. Atomics performance and fairness. Workload characterization across CPU and NPU."
  - when: "Education"
    org: ""
    body: "MS ECE, Carnegie Mellon, 2026 &middot; MTech Computer Science, IIT Kharagpur, 2022 &middot; BTech Electronics, SPIT Mumbai, 2020 &middot; GATE 2020 all India rank 276"
---

I find out where the performance went. Four years of full-chip pre-silicon GPU and SoC
performance, at NVIDIA and Qualcomm: directed tests at the instruction level, a cycle-accurate C++ model correlated against RTL,
speed-of-light bounds derived one interface at a time. Every layer makes promises to the one
above it, and my job has been to find out which ones break, usually before the silicon exists
to blame.

I am now aiming the same method one level up, at LLM training and inference on real GPUs, where
the shared resource is a fabric instead of a crossbar and the thing everyone assumes is free is
overlap.
