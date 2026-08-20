---
title: "When max(compute, comm) lies"
date: 2026-08-26
dek: "Overlap is an assumption, not a guarantee. A slowdown term per side, a queueing term below 64 KiB, and why the only honest way to measure interference is to control the overlap yourself."
kicker: "Performance modeling"
tags: ["gpu", "nccl", "perf-modeling"]
figure: "overlap"
math: true
draft: true
---

Every training-step estimate I have seen starts from the same assumption: if compute and
communication are on separate streams, the step costs the larger of the two. It is a clean model.
It is also the first thing to break, and it breaks in a way a timeline view will not show you.

The assumption is not that overlap happens. Overlap does happen. The assumption is that overlap is
*free*, and that is really a claim about shared resources. The collective wants streaming
multiprocessors to run its kernel. It wants L2 to stage its buffers. It wants memory bandwidth the
GEMM already considered its own. Two kernels on two streams are not two kernels on two machines.

## What the naive model says

Write the pair makespan the way the estimate does, but with a slowdown factor per side instead of
one scalar, and the shape of the error becomes visible.

{{< eq note="rho is per-resource contention pressure. Q(b) is the small-message queueing term, material below roughly 64 KiB." >}}
$$ \hat{T}_{\text{pair}} = \max\left(T_c \cdot s_c(\rho),\; T_m \cdot s_m(\rho)\right) + \beta_q \cdot Q(b) $$
{{< /eq >}}

Set both slowdown terms to one and the queueing term to zero and you recover
`max(compute, comm)`. Every interesting question lives in what you just deleted.

## Where the time actually goes

The figure below is the mechanism, not a measurement. A collective launched partway into a GEMM
does not simply hide underneath it: it takes streaming multiprocessors the GEMM was using, it
stretches the GEMM, and the tail it leaves behind lands on the critical path anyway.

{{< figure name="overlap" caption="Figure 1. Schematic only, no measured data. The orange band is the part the naive model predicts does not exist." >}}

## Measuring it on purpose

The only honest way to separate a stretched kernel from a slowed collective is to control the
overlap yourself, rather than accepting whatever the framework happened to schedule. Launch the
collective at a fixed fraction of the isolated compute duration, then sweep that fraction.

```cpp
// isolated baselines first, one resource at a time
float t_c = time_isolated(gemm_bf16, m, n, k, s_compute);
float t_m = time_isolated(all_reduce, bytes, s_comm);

// then overlap, with the offset as an explicit knob
for (float phi : {0.00f, 0.25f, 0.50f, 0.75f}) {
  launch_gemm(s_compute, m, n, k);
  delay_on_stream(s_comm, phi * t_c);
  ncclAllReduce(sbuf, rbuf, count, ncclBfloat16,
                ncclSum, comm, s_comm);
  record_pair_makespan(phi);
}
```

Three compute classes, two collectives, five message sizes, four offsets. One hundred and twenty
configurations, and the point of every one is to make a single resource the thing that changed.
That is the whole method, borrowed intact from pre-silicon work, where you cannot profile your way
out of a bad hypothesis because the chip does not exist yet.

<!--
DRAFT NOTE, delete before publishing.

This file is a scaffold built from your own Tier-1 contract, not a finished post. It states the
model form and the method only. No measured numbers appear anywhere, deliberately: nothing here
can be published as a result until the holdout session runs.

To publish: set draft = false, fix the date, and rewrite in your own voice.
-->
