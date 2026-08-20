---
title: "Speed of light, one interface at a time"
date: 2026-09-02
dek: "How pre-silicon architects bound performance before there is anything to measure, why per-interface bounds beat one aggregate roofline, and what survives contact with a real GPU."
kicker: "Performance modeling"
tags: ["perf-modeling", "gpu"]
draft: true
---

An outline, not a post yet.

## The problem with one roofline

A single roofline gives you one bound against one aggregate bandwidth number, which tells you
whether you are compute bound or memory bound and nothing about where the loss happened.

## Bounds per interface

Each interface along the datapath has its own ceiling and its own units. Deriving them separately
turns a single "we are at 60 percent" into a specific claim about a specific resource.

## What a directed test is for

A directed test exists to make exactly one resource the limiter, so the measured number can be
compared against a bound that means something.

## What transfers to a shipped GPU

The bound derivation transfers. The counters do not, and neither does the assumption that you can
hold everything else still.

<!--
DRAFT NOTE, delete before publishing.
Structure only, written to hold the slot. Nothing here is a claim yet.
No confidential architecture detail belongs in this post: keep it to public
Hopper and Blackwell documentation plus your own measurements.
-->
