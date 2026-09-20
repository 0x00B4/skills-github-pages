---
layout: post
title: "SCN and Play animation: clocks, interpolation, and evidence-backed clip selection"
date: 2026-09-20
author: "0x00b4"
categories: [animation, gameplay, reverse-engineering]
tags: [scn, skeletal-animation, lua, testing]
---

Fixing an effect loader cannot fix a skeleton sampled in the wrong time domain.
The character investigation found independent problems in the animation clock,
interpolation, playback bounds and state-to-clip selection. A numeric clip name
could be decoded perfectly and still be assigned to the wrong gameplay state.

The connected native deformation path uses SCN key data, transform animators,
bone matrices and skin evaluation. A separate BoundJump caller at **0x60AAD0**
creates the wall-jump sequence and assigns its effect scene a position and
orientation. SEQ can orchestrate animation, but that effect call is not the
skeletal sampler. The
[SEQ loader repair]({{ site.baseurl }}/blog/seq-effect-playback/) therefore has
its own tests and limits.

## Recover the clock before judging the pose

The native controller at **0x1153A30** advances animation at nominally **4,800
raw ticks per second**. Its float32 scale is 4.8 ticks per input millisecond,
with integer truncation on each update. The independent `timeGetTime` path at
**0x7C92A0** anchors those units; they were not inferred from how long a plausible
walk cycle ought to last.

Several details change visible playback:

- The clip starts at raw tick zero, not at its first observed channel key.
- Duration comes from serialized animation metadata, not the last key or a
  universal 32,000-tick period.
- A sample exactly equal to duration is retained. Looping modulo or nonlooping
  clamping applies on overshoot, not unconditionally at the boundary.
- Per-update truncation is not equivalent to multiplying total elapsed browser
  time by 4,800 once.

The loader at **0x1154A60** reads the duration field. Selection through
**0x7C7B00 / 0x7C7E20** stores the first successful duration in the controller.
An initial browser implementation based on measured key spans was replaced
when a red test distinguished a first key at 100 and last key at 900 from the
serialized range starting at zero and ending at 1200.

The Play adapter accumulates fractional browser milliseconds before supplying
whole milliseconds to its native-style step. It consumes the existing capped
simulation delta. This is a documented browser adaptation, not a bit-exact
reconstruction of every native outer timer and scene-update caller.

## Interpolation is recovered behavior, not cosmetic smoothing

Native constructors **0x7C8990** and **0x7C7A90** enable interpolation by default.
Sampling the predecessor key more frequently cannot reproduce that behavior.

The reconstructed sampler follows the channel boundaries at **0x116EB90**,
**0x116EC60** and **0x116ED30**: serialized defaults before the first key or for
empty channels, exact key values at their timestamps, interpolation between
keys, and the final value held after the last key. It does not blend defaults
toward the first key or wrap the last key into the first.

Translation and scale use float32-aware linear interpolation from
**0x116FC90**. Rotation follows **0x116FD50 → 0x100E7D0**, which calls imported
`D3DXQuaternionSlerp`. The port uses shortest-arc slerp for supported near-unit
quaternions, not Euler interpolation or unconditional normalized linear
interpolation. Translation, scale and rotation are sampled independently when
their timestamps differ; matrix composition remains row-vector **S × R × T**.

The imported D3DX implementation was not executed or recovered in full. The
browser's tiny-angle branch and quaternion validation envelope are declared
port policies, not native constants. Bit-exact D3DX parity is not claimed.

## A reference can be an animation alias, not another file

The loader at **0x11550E0** and registration at **0x1258300** establish a pair of
animation names. Lookup through **0x1157890 / 0x12587A0** performs one same-node
remapping before track selection.

The runtime now supports that bounded one-hop alias while preserving the
original owned key bytes and target duration. Missing targets, ambiguous
records and alias chains are rejected explicitly. It does not interpret the
second name as an external asset path or invent recursive resolution.

The actor adapter still supplies BASE for certain absent per-node tracks.
That is a preview fallback. Native missing-track behavior depends on animator
configuration; a broken alias is not permission to silently substitute BASE.

## Recover the indexed configuration, not a string association

The earlier mapping treated names such as `00000` as if they universally meant
“Run.” Native constructors already contradicted that assumption, but empty
AnimParam banks were not enough to identify the correct replacement.

The stronger evidence came from the actual actor-state Lua configuration.
The native gameplay loader at **0x8005F0**, script loader **0xAE8A30**, and
encoded-file branch **0xAE8A50** establish which resources feed initialization.
Using the existing decoder, the investigation parsed decoded Lua 5.1 chunks
and symbolically evaluated the relevant instructions. It recovered the indexed
`GetAnimParam(index):SetAnim(...)` calls, including loop, speed and reset
arguments. This was not a search for nearby string constants.

For the inspected **unarmed/default male actor**, the recovered selections
include idle `00074`, forward run `00008`, normal FastRun `00068`, and fall
`00002`. Jump has anticipation, standing/moving rise and landing phases rather
than a single universal clip. Side dodge selects `00005` on the left and
`00056` on the right. These are configuration-scoped findings, not a mapping
for every weapon, actor or body-part controller.

Native run selection at **0x635650** uses directional banks and separate part
controllers. Upper and lower entries share clip names in the inspected unarmed
bank, which supports a bounded whole-skeleton adapter, but their blend settings
remain distinct. Native movement also scales run playback by speed relative
to configured tempo. Choosing a correct name does not implement that entire
controller system.

Forward/back dodge remains an explicit asset gap: configured clips `00038` and
`00041`, and delay clips `00039` and `00042`, have no tracks in the inspected
male skeleton resource. No arbitrary side dodge should stand in for them.
Phase transition timers and direction dispatch also cannot be inferred merely
from clip completion.

## Two browser integrations, with different policies

The standalone SCN inspector now exposes serialized clip names and samples
`evaluateSkinPose` with interpolation enabled. It updates evaluated positions
and world matrices without recreating geometry, textures, material flags or
the camera. Unsupported visibility/morph tracks and incomplete bindings remain
visible as diagnostics and source-track descriptors.

Its range is the maximum serialized duration among same-name tracks. That is
an inspector policy, distinct from the native controller's first-successful
selection. The nonlooping inspector holds the terminal pose and restarts from
zero on the next Play. It labels the timeline in raw ticks rather than
milliseconds. Standalone playback does not add SEQ-embedded SCN animation.

The first Play controller repair independently added sampling on every rendered
frame, clock reset on state changes/reset, and synchronized position, normal
and bounds updates. Unsupported sampling restores assembled rest geometry
instead of leaving an unrelated movement pose frozen. Normals are recomputed
from deformed triangles; native normal-skinning parity is not claimed.

The final integration explicitly enables interpolation and uses the recovered
idle, eight-direction run, sprint, jump/fall, wall-jump and side-dodge selections.
The original BASE/step-mode checkpoint is no longer the active gameplay path.
Looping clips wrap on overshoot; nonlooping clips clamp. Direction changes within
Run preserve phase, while the adapter applies its documented entry/reset policy.
Unavailable forward/back dodge clips render an explicitly diagnosed rest fallback,
not an unrelated animation under the requested name.

## What the local evidence actually verifies

The interpolation tests cover float32 arithmetic, independent channel times,
defaults and boundaries, quaternion midpoint/quarter-time behavior, antipodes,
near-unit input and alias rejection. Real assembled source geometry also
changes between predecessor and interpolated samples while retaining finite
positions, normals and matrices.

The SCN browser regression uses `CARD_ROTATE2` from `card_rotate.scn`. It orbits
the real viewer because the card is initially edge-on, then verifies visible
pixels, different pixel hashes at ticks 0 and 400, further change at inter-key
tick 410, restoration on seeking to zero, pause/end/restart behavior, and no
WebGL error. A resource requiring unsupported visibility evaluation disables
playback instead of pretending to animate.

The integrated Play browser regression verifies changing idle/run vertices,
stationary and moving jumps, both side dodges, and the explicit missing forward
dodge fallback. It checks clip durations, interpolation metadata, sample/render
counts, refreshed normals and absence of JavaScript/WebGL errors. The rendered
idle and run screenshots were inspected from a sensible third-person camera:
the assembled character has a relaxed non-T idle and a distinct running pose.
The existing real-walljump browser scenario also passes after correcting its
pointer-lock test input: Chromium could deliver a late warp delta after the
initial heading assertion, so the test now steers through the real mouse handler
before issuing the wall-jump input.

The frozen production-layout browser gate also passes: actual server entrypoint,
public policy and CSP, required module exports, idle/run poses, SEQ opacity,
SCN inter-key pixel changes, X7 decoding/download, and forced 401/403 without
navigation. These are scoped checks, not a green repository-wide suite claim.
Remaining limits include
part-controller blending, exact native fallback and transition policies,
forward/back dodge assets, fixed initial camera bounds in the inspector,
visibility/morph sampling and native material/compositing parity. Particles,
trails and lightning are not completed by skeletal animation work either.

## Rendered evidence

![Recovered idle in Play mode]({{ site.baseurl }}/assets/images/animation-repair/play-idle.png)

*Unarmed idle from the recovered `00074` selection, not BASE/T pose.*

![Recovered forward run in Play mode]({{ site.baseurl }}/assets/images/animation-repair/play-run.png)

*Forward run uses `00008`. These captures demonstrate the selected source poses,
not complete map-material or native transition fidelity.*

![Standalone SCN clip in the resource viewer]({{ site.baseurl }}/assets/images/animation-repair/scn-card.png)

*The viewer samples the real `CARD_ROTATE2` track between stored keys.*

## Release verification

The scoped animation update was deployed as **`20260920T081618Z-8480051e`**.
All nine changed files were read back exactly, and the live startup verifier
passed its module hashes, 341 packaged resources, 57 actor/map dependencies
and 318 X7 documents. The running server then reported all 17,300 catalogued
resources available through its persistent resource store. Existing public
access, crawler exclusion and private-file protections were preserved.

The actual frozen-server browser acceptance passed before deployment. Direct
requests to the public hostname and IP timed out from the build container;
external reachability is therefore not claimed as verified by those requests.
This distinction does not change the successful deployment/readback result.

### Evidence boundary

The evidence records are `animation-native-analysis.md`,
`actor-animation-mapping.md`, `skin-interpolation-fix.md`,
`scn-viewer-animation.md` and `play-animation-fix.md`. The first is static native
analysis; the mapping record adds decoded indexed configuration and asset
checks; the implementation records report focused local tests. Their scopes
must not be conflated. No private bytecode, decoding tables or original asset
files are reproduced here. The earlier
[actor state-machine notes]({{ site.baseurl }}/blog/actor-state-machine/) remain
background for movement logic, not proof of complete animation fidelity.
