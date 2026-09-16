---
layout: post
title: "Ticks, activation windows, and deterministic effect updates"
date: 2026-08-23
author: 0x00b4
categories: [runtime, effects, reverse-engineering]
tags: [ticks, seq, particles, timing]
---

Timing is where a binary reconstruction can look correct while behaving
wrongly. The v1267 client does not treat a sequence tick as an arbitrary frame
number. It obtains elapsed time from a millisecond clock, applies speed with
specific narrowing and truncation rules, and then evaluates units against
absolute tick windows.

## The clock path

The client obtains a shared time value from a high-resolution path or a fallback
millisecond timer. Timer code stores the elapsed delta and can apply a speed
factor. The sequence controller then narrows the unsigned delta and speed to
float32 operands, multiplies them, truncates the result, and advances the
current tick.

The important edge cases are observable:

- a track is active when `startTick <= tick <= endTick`;
- the end tick is inclusive;
- termination or looping happens only after advancing past the end;
- a pure seek is not the same operation as a frame update;
- looping wraps the range without treating equality at the end as overflow.

A port that increments by one per rendered frame will drift as soon as frame
rate, speed, or a nonzero start tick changes.

## Activation is not visibility

A loader can be eligible inside its tick window while a final fade, target
binding, resource lookup, or renderer decision is still unresolved. Nonzero
fades are history-dependent. Node controls may expose serialized transforms but
not final visibility. The runtime therefore keeps `active`, `visible`, and
particle state separate instead of inferring one from another.

## Stateful particles

Particle updates are not stateless samples. A bounded emitter maintains live
particles, ages them, reuses dead slots, carries fractional emission, and
updates velocity and position in a defined order:

1. age existing particles;
2. remove particles whose age reaches lifetime;
3. advance position using the previous velocity;
4. apply gravity to velocity;
5. update size, color, and rotation along the active lifeline;
6. emit new particles at age zero.

The reconstructed supported profile is intentionally narrow: one plain
`CActParticle`, one emitter key, no variation, one explicit texture, and
identity-emitter-local space. It is deterministic for the same ordered sequence
of frame deltas and emission toggles. It is not full native effect playback.

## Determinism has a boundary

The original particle path calls the imported CRT `rand` implementation. A
serialized per-emitter seed was not established by the recovered evidence.
Therefore the port does not invent a hash-based PRNG and claim seeded native
equivalence. Zero-variation profiles can be deterministic without pretending
to reproduce every random effect.

A delta above 250 ms is rejected rather than silently split into smaller steps:
the native path has a distinct clearing behavior for large deltas. This is a
small detail, but these details are exactly where “visually plausible” and
“behaviorally reconstructed” diverge.
