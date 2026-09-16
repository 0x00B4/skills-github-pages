---
layout: post
title: "The actor state machine: run, jump, fall, and walljump"
date: 2026-09-06
author: 0x00b4
categories: [gameplay, locomotion, reverse-engineering]
tags: [state-machine, walljump, actor]
---

Movement in the v1267 client is a state machine, not a collection of unrelated
velocity formulas. An actor state owns entry, update, input, exit, and often a
virtual dispatch slot. The recovered browser implementation mirrors selected
transitions while clearly labelling the rest as adapter behavior.

## Directional running

The native run state compares the facing direction with the movement direction.
Small angles select a forward sector, large angles select a backward sector,
and the intermediate sectors use the cross product to distinguish left from
right. The selected sector chooses a directional speed ratio. This is why
“eight direction movement” is more than eight button labels.

The browser port uses the recovered speed ratios and game-tempo values, but its
input and camera adapters are not the original input dispatcher. That is a
useful separation: constants can be evidence-based even when the surrounding
platform integration is new.

## Jump and fall

Jumping, falling, and landing are distinct states with timers and event paths.
The browser can reproduce a bounded version of their transitions using the
static collision structure. It does not claim the complete native animation,
network synchronization, stamina system, or every gameplay modifier.

## Walljump phases

The recovered BoundJump state has multiple phases rather than one impulse. A
wall contact must satisfy several conditions, including contact flags, a height
limit, a mostly horizontal wall normal, and a facing-direction dot-product
threshold. The horizontal wall normal is projected against facing, scaled, and
combined with a vertical component before the pending impulse is stored.

A simplified phase outline is:

```text
phase 1: prepare impulse and pause movement
    ↓ after the entry timer
phase 2/3: launch with directional push
    ↓ after the launch timer
phase 4/5: recovery branches
    ↓ on ground
phase 6: clear impulse and leave after recovery
```

The timers are evaluated with strict expiry rules, while the sequence clock
elsewhere uses inclusive end ticks. Mixing those two conventions produces
subtle one-frame errors.

## What the port deliberately omits

The original actor also owns animation parameters, sound/effect dispatch,
resource attachment, stamina costs, equipment modifiers, and native collision
world behavior. The browser version uses a static collision solver, a composed
rest-pose avatar, and explicit controls for Ghost, reset, and camera movement.
Those are integration choices, not evidence that the native state machine has
been fully ported.

A good reverse-engineering result is not the one with the most state names. It
is the one that can show which transition, operand, timer, and resource is
actually supported—and which one is still a research task.
