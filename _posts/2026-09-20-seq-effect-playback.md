---
layout: post
title: "SEQ effect playback: fixing the loader path, not inventing an effect"
date: 2026-09-20
author: "0x00b4"
categories: [effects, reverse-engineering, browser]
tags: [seq, scn, webgl, testing]
---

A sequence that parses successfully can still produce an empty viewer. That
was the useful distinction in this repair: the missing wall-jump effect was
not evidence that its `.seq` file was unfinished. The resource viewer was
bypassing an existing preview adapter and rejecting data that the adapter
already knew how to handle.

This post describes a tested, partial loader preview deployed in release
`20260920T081618Z-8480051e`. It does not claim complete native effect playback.
Character animation has separate causes, covered in the
[SCN and Play investigation]({{ site.baseurl }}/blog/scn-character-animation/).

## The loader integration was narrower than the runtime

The viewer already imported `engine/seq-runtime.js`. Replacing a legacy parser
would therefore have fixed the wrong problem. Its `loadSequence` function
instead maintained a second, narrower loading path:

- A loader with nonzero fade words was rejected before rendering.
- Geometry survived, but the SCN `format` and `nodes` metadata did not. That
  prevented lookup of serialized blend, depth and cull decisions.
- Catalog-backed BMP/TGA-to-DDS aliases were not applied.
- A cache keyed by visible tracks could detect activation changes, but not
  continuously changing opacity within the same visible track.

The wall-jump resource exercises these boundaries together. Its SCN refers to
`tick2_blue.bmp`, while the extracted catalog contains the corresponding DDS.
The adapter resolves that known catalog relationship; it does not synthesize
a texture or search arbitrary filesystem paths.

The repair connects the viewer to `createSequencePreview` and the existing
`createNodeLoaderPlayback` implementation. The latter records native loader
initialization at **0x115FC90** and update behavior at **0x1160160**. Reusing that
path matters: this was an integration repair, not a new fade formula presented
as recovered native behavior.

## Opacity must reach the renderer every update

The SCN models and textures are loaded once. Subsequent sequence updates change
per-mesh opacity without rebuilding geometry buffers or resetting the camera.
The renderer multiplies fragment alpha by loader opacity, skips zero-opacity
meshes, and enables blending for partially opaque meshes while retaining the
serialized material decisions.

This is an unlit browser visualization of loader opacity. It is not the full
native material pipeline. In particular, a readable texture and nonempty GPU
pixels do not establish correct additive compositing against a native scene.

Seeking exposed another distinction. Continuous playback retains loader
history, but a manual inspector seek resets initialized opacity before
sampling. Rewinding or restarting also clears the previous run. This is an
explicit inspector policy, not a reconstruction of every possible native
history leading to an arbitrary tick. The existing controller end policy is
retained rather than extending playback to manufacture a fade-out.

## Browser tests caught both the omission and the restart bug

The new regression uses the actual application with extracted SEQ, SCN and DDS
bytes. Before integration it failed with **“fade loader must reach renderer.”**
After the effect became visible, another red test caught opacity remaining
saturated on restart: **“restart must fade in, not retain full opacity.”**

The documented local checks verify:

- the actual DDS request and retention of SCN flag metadata;
- no rendered pixels at initialized tick zero;
- visible pixels at tick 250 and stronger RGB at tick 500;
- backward-seek and restart resets;
- no WebGL error, and explicit partial/unsupported playback wording.

The captured tick-500 view was inspected and showed the cyan hexagonal
wall-jump texture rather than missing-texture polygons. A black rectangular
backdrop remained visible. That is a material/compositing limitation, not a
reason to relabel the result as native visual equivalence.

![Actual viewer rendering of the source wall-jump effect]({{ site.baseurl }}/assets/images/animation-repair/seq-wall-jump.png)

*Real SEQ/SCN/DDS resources at tick 500. The remaining black backdrop is a
compositing limitation, not a native-fidelity claim.*

The implementation notes record successful focused runs of
`seq-viewer-browser.test.mjs`, `seq-preview.test.mjs`, SEQ runtime/UI coverage,
and separate application/material integration coverage. These are scoped local
results, not a claim that the entire repository test suite is green.

One existing renderer browser test still expected depth writes after replacing
texture alpha with fully opaque values. The SCN retains a serialized
no-depth-write flag. A comparison harness without the viewer renderer changes
reproduced the same failure; the material-specific tests preserve the
independence of texture alpha and depth-write state. The failed assertion is
reported rather than silently counted as a passing regression.

## Access errors and release artifacts are separate gates

Forced 401 and 403 browser tests also exposed an unwanted navigation to
`/login`. The viewer now leaves denied access as an explicit inspector error
instead of initiating that navigation. Those local tests do not establish the
policy of a deployed authentication server.

The staged test harness can serve all UI, engine and resource bytes exclusively
from a supplied release root, with no fallback to local code. That distinction
caught an actual packaging mismatch: the inspected staged runtime wrapper did
not export `createNodeLoaderPlayback`, so module loading failed before catalog
initialization. The matching wrapper must accompany the viewer change.

The matching wrapper is now included in the frozen release payload. Its compiled
SEQ dependencies were compared separately and are unchanged. A second acceptance
harness launches the actual staged server entrypoint with its compressed files,
public-access policy, CSP and persistent-resource scanning. That gate passes
required module/export checks, Play idle/run poses, SEQ fade pixels, standalone
SCN interpolation, X7 decoding/download and forced 401/403 without navigation.
The fixture-backed rendering harness and this production-policy gate remain
separate evidence; neither alone certifies public-host reachability.

## What remains unsupported

This change renders supported loader geometry and its opacity. Particle render
integration, trails, lightning, IF-specific behavior, actor/node control,
moving-target history and native billboards remain outside that coverage.
A separately available bounded emitter API does not make particles integrated
into this viewer. SEQ-embedded SCN animation is also still unsupported;
standalone SCN playback is a separate integration.

Exact alpha-test behavior, special blend states, fog, glow, haze and native
scene compositing are not reconstructed here. Unsupported tracks keep their
original class names and explicit diagnostics. No substitute sprites, random
particles or invented motion were added to make an incomplete effect look
finished.

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

The implementation and verification record is `seq-viewer-fix.md`; the native
character/effect distinction is recorded in `animation-native-analysis.md`.
Those research notes support the addresses and local test observations above.
No original asset bytes, private decoding tables or private evidence artifacts
are reproduced here. For the format-level distinction, see
[SCN, SEQ, and OCT]({{ site.baseurl }}/blog/scn-seq-oct/).
