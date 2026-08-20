---
title: "Speed of light, one interface at a time"
date: 2026-08-20
lastmod: 2026-08-20
dek: "One roofline tells you whether you are compute bound or memory bound. It will not tell you which resource ran out of room, and it will not tell you whether your answer is true. This is the method that does both, and the places where the method itself lies to you."
kicker: "Performance modeling"
tags: ["gpu", "perf-modeling"]
figure: "datapath"
math: true
draft: false
---

A DRAM roofline is a good first question and a bad last one. You get one bound from peak
arithmetic throughput, one from peak memory bandwidth, and a verdict: compute bound, or memory
bound. That is genuinely useful for triage. It stops being useful the moment the kernel is
memory bound at 40 percent of peak bandwidth and someone has to say why, because the verdict
names a category rather than a resource.

The reason is that a chip has no single memory interface. It has many resources, and a request
does not visit all of them. Shared memory traffic never reaches L2 or DRAM. An L1 hit stops at
L1. `cp.async` and the tensor memory accelerator use different machinery from an ordinary load.
Read and write paths can have different ceilings. So the picture is not a chain where the
narrowest link decides everything; it is a routed, partitioned, backpressured graph, and the
useful question is which resource runs out of capacity relative to what your workload demands
of it.

Write that as a bound rather than a story:

$$ \text{max workload rate} \;\le\; \min_{r \,\in\, \text{resources},\ p \,\in\, \text{partitions}} \frac{\text{capacity}(r, p)}{\text{demand}(r, p)\ \text{per unit work}} $$

The minimising resource is a **candidate** bottleneck. It is not the answer until a perturbation
you predicted in advance moves the number the way you said it would. Two resources can be
co-limiting. Neither may be saturated at all, because latency, synchronisation, launch overhead
or a power or clock cap can bind while every bandwidth interface still has headroom.

{{< figure name="datapath" caption="Figure 1. Illustrative, not measured. Per-interface utilization for one directed workload. Also a simplification twice over: these are aggregates across slices and partitions, and a high number is a candidate bottleneck rather than a proven cause." >}}

## Speed of light is not the number on the spec sheet

Speed of light for a resource is the highest rate it can sustain, expressed in the units of the
workload you are running. It is a derating of peak, and every derating is a thing you have to
know and defend.

### Derating one: granularity, and which bandwidth you mean

Memory moves in sectors, not bytes, and on current NVIDIA parts the sector is 32 bytes. A kernel
striding through `float` data with a stride of 8 elements, so that each access lands in a
different sector with no reuse, touches four useful bytes out of every 32 moved:

$$ \eta = \frac{4\ \text{bytes used}}{32\ \text{bytes moved}} = 12.5\% $$

Before drawing any conclusion from that, separate two quantities that both get called
bandwidth:

- **Requested bandwidth.** Useful bytes the kernel asked for, divided by time.
- **Actual bandwidth.** Bytes the hardware actually moved, sectors or DRAM traffic, divided by
  time.

Efficiency is the ratio. The 12.5 percent above is an efficiency ceiling, so it caps *requested*
bandwidth at one eighth of peak. It says nothing about actual bandwidth, which can sit at 40
percent of peak while the kernel is doing 5 percent of peak useful work. That is exactly what a
badly strided kernel looks like from the DRAM side: the bus is busy, and most of what crosses it
is waste.

So the sentence "the kernel is at 40 percent of peak" is not a measurement until you say which
of the two you mean. And the ceiling is not always 12.5 percent either, because caches can reuse
over-fetched sectors when a later access lands in one, which is why NVIDIA's own guidance
distinguishes requested from actual throughput rather than treating coalescing efficiency as a
fixed derating.

### Derating two: concurrency, and what Little's law can and cannot tell you

Bandwidth is not something you request. It is what you get when enough requests are in flight to
cover latency. Little's law gives the accounting:

$$ \text{bytes in flight} = \text{bandwidth} \times \text{latency} $$

Run it on one clearly stated configuration, because mixing SKUs here is the same sin as mixing
bandwidth definitions above. Take H100 SXM5: 3.35 TB/s of HBM3, 132 SMs, boost clock about 1.98
GHz. Latency is not a published spec, so it has to come from measurement, and published Hopper
microbenchmark dissections put a fully missing global load in the region of 600 to 700 cycles.
At 660 cycles and 1.98 GHz that is roughly 333 ns. Mind the units: latency is quoted in cycles
far more often than in nanoseconds, and treating one as the other will cost you most of a factor
of two. Mind the SKU too, since the H100 PCIe part clocks at 1.755 GHz with 114 SMs, and a
cycle count measured on one part converted at the other part's clock is not a derating you can
defend.

$$ 3.35 \times 10^{12}\ \text{B/s} \times 333 \times 10^{-9}\ \text{s} \approx 1.1\ \text{MB} $$

Spread across 132 SMs that is about 8.5 KB per SM, or on the order of 260 sectors outstanding per
SM, continuously.

Now be careful about what that number is. It is a **minimum concurrency budget**: below it you
cannot reach peak. It is not a measured hardware queue depth, and it does not license the
tempting linear claim that half the concurrency gives you half the bandwidth. That inversion
holds only where latency is fixed, traffic amplification is fixed, and you are in the linear
unsaturated regime, and real parts leave that regime early. Recent H100 SXM measurements that
treat Little's law explicitly as accounting rather than as a hardware pool find bandwidth peaking
at low per-thread load, around two outstanding loads per thread, then falling by roughly 35
percent as offered load rises to eight, with DRAM traffic flat while L2 sector traffic climbs.
More requests in flight made things worse.

The honest version of the claim is therefore: the calculation gives you a floor on required
concurrency, and the attainable curve has to come from a measured concurrency-response sweep.
Compute the floor to know whether you are even in the game. Sweep to find out what the part
actually does.

Do this for every resource and you stop having opinions. You have a ranked list of candidate
ceilings in the units of your workload, and triage becomes finite.

## What makes a workload directed

A directed workload exists to make one resource the limiter by construction, so the measurement
means something. Four properties, and skipping any one costs you the result:

- **A known working set.** Sized to sit inside one level of the hierarchy, and verified to be
  there. If you claim to be measuring L2 bandwidth, the hit rate is part of the result, not an
  afterthought.
- **A known access pattern.** Stride, alignment and coalescing behaviour stated up front,
  because they set the deratings above.
- **Enough parallelism to saturate.** By the Little's law argument, a test that cannot keep the
  resource busy measures your concurrency, not the resource.
- **A closed-form expected value.** Written down before the run. This is the part everyone skips.

A latency probe is a dependent pointer chase with one outstanding request at a time, which is the
opposite construction: it starves concurrency deliberately so that what remains is latency. A
bandwidth probe is a wide independent stream. A tensor-pipe probe keeps arithmetic intensity high
enough that no memory resource is near its ceiling. Each test answers one question and answers
nothing else. Resist the urge to build a test that measures two things.

## How performance is actually validated

Here is the part that is discipline rather than cleverness. A measurement is not a result. A
measurement plus a prediction you made before you saw it is a result.

**Predict first, in writing.** Compute the expected number from the bound before the run. A
prediction made after seeing the measurement is not a prediction, it is a rationalisation, and it
will fit any data you show it. That is the whole difference between validating a model and
decorating one.

**Require two independent paths to agree.** Analytic bound against measurement. Wall-clock timing
against counter-derived timing. A cycle-accurate model against hardware. When two paths built on
different assumptions land in the same place, the agreement is evidence. A path agreeing with
itself is not.

**Read the shape of the residual, not the average.** Mean absolute error hides the interesting
part. What you want is the sign and shape across the sweep: does the error grow with size, appear
only above a working-set threshold, flip sign at a particular occupancy? A systematic residual is
a missing term, and it tells you which one. An average error of 4 percent that runs plus 20 at
one end and minus 15 at the other is a worse result than a flat, boring 10 percent, because the
flat one is at least honest about what it does not know.

**Give every discrepancy an owner.** Each gap is a model bug, a test bug, real hardware
behaviour, or a measurement artefact. Those four have very different fixes, and the failure mode
is assuming the third when it is usually one of the other three.

**Falsify, do not confirm.** Suppose you believe the loss is L2 bandwidth. Do not go looking for
more evidence that it is L2 bandwidth. Design a change that must move the number in a predicted
direction by a predicted amount if you are right: halve the working set, change the stride so
sectors per request changes, alter the number of resident blocks. If it moves as predicted,
before you ran it, the attribution holds. If it moves by the wrong amount, you have learned
something better than a confirmation.

**Distrust aggregates.** An L2 utilisation of 70 percent can conceal one saturated slice, and 97
percent does not prove causality. Report per-instance and maximum values alongside the mean, say
whether the denominator is active or elapsed cycles, and separate read from write. A single
percentage for a partitioned resource is a summary statistic pretending to be a diagnosis.

**Measure the measurement.** Lock the clocks, or record them per run. Warm up, then discard the
warm up. Report distributions with an interval, never one run against one prediction. Establish
the run-to-run noise floor first, because a 3 percent effect against a 5 percent noise floor is
not an effect. And treat the profiler as an instrument with its own perturbation: it can serialise
launches, flush or manage caches, and replay kernels, so quantify the shift once and carry the
number.

**Know what your counters and clocks actually measure.** Counters are not only event counters:
elapsed, active and stall cycle counters exist, and profilers report kernel duration directly. The
hazard is subtler than "counters count events". It is that attributing *duration* to a unit
depends on which cycle domain you are dividing by, on collection semantics such as replay and
cache handling, and on whether a metric was gathered in hardware or patched into the code, which
changes its overhead by orders of magnitude. The same care applies to timing: host timers, CUDA
events, `%globaltimer` and profiler timestamps bound different intervals. Name the instrument and
say what interval it covers, every time.

## Where this sits relative to existing work

None of the resource decomposition here is new. Hierarchical Roofline already extends roofline
analysis to L1, L2 and HBM traffic on NVIDIA GPUs with automated machine and application
characterisation, and it is the right starting point if you want per-level ceilings rather than
one DRAM bound.

What directed tests add on top is not the decomposition. It is the epistemics: a closed-form
expectation written before the run, a prediction frozen so it cannot be retuned, and a
falsification step that can take the attribution away from you. Hierarchical Roofline tells you
which level to look at. This tells you whether to believe what you find there.

## What a worked case has to contain

This post is method, not measurement. Figure 1 is illustrative, and a methodology post with no
falsifiable number in it should be read as an argument rather than as evidence. A worked case is
a different post, and it needs hardware time rather than prose. When it comes, it has to contain
all of this:

```text
directed kernel, one resource by construction
  -> closed-form predicted ceiling, written first
  -> achieved rate, with per-instance counters and the noise floor
  -> residual across a sweep, with sign and shape
  -> one perturbation predicted in advance
  -> attribution confirmed or rejected, and published either way
```

If a performance claim you read does not have those parts behind it somewhere, it is a story
about a machine rather than a measurement of one. Including, until I publish that case, this one.

## The short version

1. Enumerate resources and partitions, not a single chain. Requests do not visit everything.
2. Bound each one by capacity over demand, in your workload's units, on one stated SKU.
3. Say which bandwidth you mean, requested or actual, every time you quote a percentage.
4. Treat Little's law as a floor on concurrency, then measure the response curve.
5. Predict, measure, and compare distributions rather than averages.
6. Call the worst resource a candidate, and try to break your own attribution.
7. Publish the noise floor and the instrument, or the number does not mean much.

Nothing here is clever. It is the difference between knowing where the performance went and
having a plausible story about it.

## Further reading

The empirical counterpart to all of this, and worth your time: Fergus Finn traced a single
`LDG.E` through an
[RTX 4090's memory path](https://blog.doubleword.ai/what-happens-when-a-gpu-reads-memory),
reverse-engineering the L1 set index and the L2 slice hash by timing rather than documentation,
then validating the slice function by showing throughput scale nearly linearly from one predicted
slice to thirty six. That is what a directed test looks like when the prediction is sharp enough
to have been wrong.

- [Hierarchical Roofline Performance Analysis for Deep Learning Applications](https://arxiv.org/abs/2009.05257), Yang, Wang, Farrell, Kurth and Williams, 2020.
- [Concurrency Response of Plain Global Loads on the NVIDIA H100](https://arxiv.org/abs/2608.15764), 2026, on Little's law as accounting and bandwidth falling as offered load rises.
- [Understanding Latency Hiding on GPUs](https://escholarship.org/uc/item/1wb7f3h4), Volkov, 2016, on where the standard concurrency models break.
- Hopper and Blackwell microbenchmark dissections: [arXiv 2402.13499](https://arxiv.org/abs/2402.13499) and [arXiv 2507.10789](https://arxiv.org/abs/2507.10789).
- [CUDA C++ Best Practices Guide](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#coalesced-access-to-global-memory) on requested versus actual throughput.

## Revisions

Published 20 August 2026. Revised the same day, after review, in ways worth naming rather than
hiding:

- The original quoted DRAM latency as 600 to 700 **nanoseconds**. The published figure is 600 to
  700 **cycles**. The original also converted at 1.755 GHz, which is the PCIe part's clock, while
  quoting SXM bandwidth and SM count. Corrected to one configuration throughout: bytes in flight
  2.2 MB becomes 1.1 MB, per SM 16 KB becomes 8.5 KB, 500 sectors becomes 260.
- The serial-chain framing, and the claim that the first saturated interface is the only one that
  matters, is replaced by a demand-and-capacity bound over resources and partitions, with
  candidate bottlenecks rather than proven ones.
- Requested and actual bandwidth are now distinguished, because the stride example was ambiguous
  between them.
- The claim that halving concurrency halves attainable bandwidth is withdrawn, and replaced with
  a floor plus a measured response curve.
- "Counters count events, not time" was simply wrong, and is replaced with the real hazard.
- Hierarchical Roofline is credited as prior art for the per-level decomposition.
