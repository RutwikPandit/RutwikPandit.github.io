---
title: "Speed of light, one interface at a time"
date: 2026-08-20
dek: "One roofline tells you whether you are compute bound or memory bound. It will not tell you which interface ran out of room, and it will not tell you whether your answer is true. This is the method that does both."
kicker: "Performance modeling"
tags: ["gpu", "perf-modeling"]
figure: "datapath"
math: true
draft: false
---

A roofline is a good first question and a bad last one. You get one bound from peak
arithmetic throughput, one from peak memory bandwidth, and a verdict: compute bound, or
memory bound. Useful for triage on day one. Useless on day two, when the kernel is memory
bound at 40 percent of peak bandwidth and someone has to say why.

The reason is that a chip has no single memory interface. It has a chain of them in series,
each with its own ceiling, its own units, and its own reasons for falling short. Instruction
issue. Register file bandwidth. L1 and shared memory. The on-die interconnect. L2. The memory
controllers. The DRAM devices themselves. A request crosses all of them, and the one that
saturates first is the only one that matters for that workload. Aggregate peak bandwidth is the
last link in that chain, which is precisely why quoting it explains so little.

{{< figure name="datapath" caption="Figure 1. Illustrative, not measured. Per-interface utilization for one directed workload: five interfaces have headroom, one does not." >}}

## Speed of light is not the number on the spec sheet

Speed of light for an interface is the highest rate that interface can sustain, expressed in
the units of the workload you are running. It is a derating of peak, and every derating is a
thing you have to know and defend.

Take the simplest one, access granularity. Memory is moved in sectors, not in bytes, and on
current NVIDIA parts the sector is 32 bytes. A kernel striding through `float` data with a
stride of 8 elements touches four useful bytes in every 32 byte sector it pulls:

$$ \eta = \frac{4\ \text{bytes used}}{32\ \text{bytes moved}} = 12.5\% $$

The speed of light for that access pattern is therefore one eighth of peak DRAM bandwidth. A
kernel hitting 40 percent of peak looks broken; the same kernel hitting 40 percent of peak
while running a stride-8 pattern is at 320 percent of a ceiling it cannot exceed, which means
your model is wrong, not the kernel. Getting this backwards is the most common way performance
analysis goes quietly bad: the bound was never right, so every conclusion drawn against it was
noise.

The second derating is concurrency. Bandwidth is not a property you can request; it is what you
get when enough requests are in flight to cover latency. Little's law gives the amount:

$$ \text{bytes in flight} = \text{bandwidth} \times \text{latency} $$

On an H100 SXM, published HBM3 bandwidth is about 3.35 TB/s. Latency is not a published spec, so
it has to come from measurement: microbenchmark studies of GH100 report on the order of 660 cycles
for a global load with a footprint large enough to miss everywhere, which at the 1.755 GHz boost
clock is roughly 375 ns. Mind those units. Latency gets quoted in cycles far more often than in
nanoseconds, and treating one as the other is a quick way to be wrong by most of a factor of two.
Sustaining peak therefore requires roughly

$$ 3.35 \times 10^{12}\ \text{B/s} \times 375 \times 10^{-9}\ \text{s} \approx 1.3\ \text{MB} $$

in flight at all times. Spread across 132 streaming multiprocessors that is about 9.5 KB per SM,
or on the order of 300 sectors per SM outstanding, continuously. Now the occupancy question has
a quantitative answer instead of a rule of thumb: if the kernel's achievable outstanding-request
count is half of that, its speed of light is half of peak, and no amount of tuning inside the
inner loop will move it. The fix is more memory-level parallelism, not better instruction
selection.

Do this for every interface and you stop having opinions. You have a ranked list of ceilings,
in the units of your workload, and triage becomes finite.

## What makes a workload directed

A directed workload exists to make exactly one interface the limiter, by construction, so that
the measurement means something. Four properties, and skipping any one of them costs you the
result:

- **A known working set.** Sized to sit inside one level of the hierarchy, and verified to be
  there. If you claim to be measuring L2 bandwidth, the hit rate is part of the result, not an
  afterthought.
- **A known access pattern.** Stride, alignment, and coalescing behavior stated up front,
  because they set the derating above.
- **Enough parallelism to saturate.** By the Little's law argument, a test that cannot keep the
  interface busy measures your occupancy, not the interface.
- **A closed-form expected value.** Written down before the run. This is the part everyone
  skips.

A latency probe is a dependent pointer chase, one outstanding request at a time, which is the
opposite construction: it deliberately starves concurrency so that what remains is latency. A
bandwidth probe is a wide independent stream. A tensor-pipe probe keeps arithmetic intensity
high enough that no memory interface is near its ceiling. Each test answers one question, and
answers nothing else. Resist the urge to make a test that measures two things.

## How performance is actually validated

Here is the part that is mostly discipline rather than cleverness. A measurement is not a
result. A measurement plus a prediction you made before you saw it is a result.

**Predict first, in writing.** Compute the expected number from the bound before the run. A
prediction made after seeing the measurement is not a prediction, it is a rationalization, and
it will fit any data you show it. This is the entire difference between validating a model and
decorating one.

**Require two independent paths to agree.** Analytic bound against measurement. Wall-clock
timing against counter-derived timing. A cycle-accurate model against hardware. When two paths
built on different assumptions land in the same place, the agreement is evidence. When one path
agrees with itself, that is not evidence of anything.

**Look at the distribution of error, not the average.** Mean absolute error hides the
interesting part. What you want is the sign and shape of the residual across the sweep: does
the error grow with message size, appear only above a working-set threshold, flip sign at a
particular occupancy? A systematic residual is a missing term in your model and it is telling
you which one. An average error of 4 percent that is plus 20 percent at one end and minus 15
percent at the other is a worse result than a flat, boring 10 percent, because the flat one is
at least honest about what it does not know.

**Give every discrepancy an owner.** Each gap is one of: a model bug, a test bug, real hardware
behavior, or a measurement artifact. Those four have very different fixes, and the failure mode
is assuming the third when it is usually one of the other three. Label each one by how well it
is supported, and keep the unresolved ones visible instead of averaging them away.

**Falsify, do not confirm.** Suppose you believe the loss is L2 bandwidth. Do not go looking
for more evidence that it is L2 bandwidth. Design a change that must move the number in a
predicted direction by a predicted amount if you are right: halve the working set, change the
stride so sectors per request changes, alter the number of resident blocks. If the number moves
the way you said it would, before you ran it, the attribution holds. If it moves by the wrong
amount, you have learned something better than a confirmation.

**Measure the measurement.** Locked clocks, or clocks recorded per run. Warm up, then discard
the warm up. Report distributions with an interval, never one run against one prediction.
Establish the run-to-run noise floor first, because a 3 percent effect against a 5 percent noise
floor is not an effect. And treat the profiler as an instrument with its own perturbation: it
serializes work, changes occupancy, and can move the very thing you are measuring, so quantify
that shift once and carry the number.

**Remember what counters actually count.** Hardware counters count events, not time. Derived
metrics carry assumptions from whoever derived them, and those assumptions are usually about
which unit was busy, which is the question you are trying to answer. Sampled totals are not
totals. Counters are excellent for attribution and dangerous as ground truth for duration; the
clock is ground truth for duration.

## Why the loop survives the chip existing

None of this depends on the hardware being unbuilt. The pre-silicon version has a model on one
side and RTL on the other; the post-silicon version has a model on one side and a real GPU on
the other. The loop is identical, and the second version is easier, because the ground truth
runs in seconds and does not require anyone to explain a waveform.

It also scales up a level. Replace interfaces with resources shared across a node, replace RTL
with a measured cluster, and the same discipline applies to a training step: bound each
resource, predict before you measure, freeze the prediction so it cannot be retuned, then
account for the residual per resource instead of reporting one aggregate error. That is the
project I am working on now, and the reason I trust the method is that the method is older
than my involvement in it.

## The short version

1. Enumerate the interfaces in the datapath. There is more than one.
2. Derate peak into a real ceiling per interface, in your workload's units.
3. Build one directed test per interface, with a closed-form expectation written first.
4. Predict, then measure, then compare distributions rather than averages.
5. Attribute every residual, and try to break your own attribution with an orthogonal test.
6. Publish the noise floor alongside the result, or the result does not mean much.

Nothing here is clever. It is just the difference between knowing where the performance went
and having a plausible story about it.

## Further reading

The empirical counterpart to all of this, and worth your time: Fergus Finn traced a single
`LDG.E` through an
[RTX 4090's memory path](https://blog.doubleword.ai/what-happens-when-a-gpu-reads-memory),
reverse-engineering the L1 set index and the L2 slice hash by timing rather than by
documentation, then validating the slice function by showing throughput scale linearly from one
predicted slice to thirty six. That is what a directed test looks like when the prediction is
sharp enough to be wrong.

For measured latency and throughput per level, the Hopper and Blackwell microbenchmark
dissections are the standard references:
[arXiv 2402.13499](https://arxiv.org/abs/2402.13499) and
[arXiv 2507.10789](https://arxiv.org/abs/2507.10789).
