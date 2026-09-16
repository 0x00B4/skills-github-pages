---
layout: post
title: "What the browser/WASM port can and cannot claim"
date: 2026-09-13
author: 0x00b4
categories: [wasm, methodology, s4-client]
tags: [verification, limitations, reconstruction]
---

A browser reconstruction is valuable when it makes its boundary visible. The
v1267 project uses WebAssembly for bounded binary parsing and selected runtime
math, while JavaScript handles HTTP, resource routing, and presentation. It is
not the original executable running inside a browser.

## What is reproduced with confidence

The current evidence supports a complete v1267 catalog index, allowlisted
resource resolution, the investigated encryption/decompression pipeline,
selected SCN geometry and transform parsing, OCT collision triangles, SEQ
structure, millisecond tick behavior, and a narrow deterministic particle
profile. Real decoded files exercise these paths, and malformed/truncated
inputs are tested against explicit limits.

The actor preview additionally demonstrates a composed rest-pose character,
static map collision, bounded locomotion, and selected walljump phases. These
features are useful visual and behavioral probes, not a claim that every native
subsystem has been replaced.

## What remains outside the claim

The following are intentionally not presented as complete:

- the original network/session protocol and online services;
- arbitrary avatar assembly and complete equipment rules;
- native animation blending, morph playback, and all effect types;
- stochastic particle playback with recovered CRT random state;
- full material, shader, fog, lighting, and post-processing fidelity;
- the complete collision world, dynamic objects, and native physics details;
- integrity authentication beyond the checks actually observed;
- private keys, passwords, credentials, or deploy-only source material.

A parser returning `supported: true` means that the implemented contract was
satisfied. It does not mean that the original game would render the resource
identically. Unsupported input returns a diagnostic rather than a fallback
cube, guessed animation, or fabricated texture.

## Why this is a useful method

The workflow is repeatable:

1. inspect the read-only client and Ghidra evidence;
2. record the exact byte order, field boundaries, and call-site behavior;
3. implement a bounded parser or runtime in a separate module;
4. exercise real decoded resources and adversarial truncations;
5. compare results at the correct layer—bytes, structures, ticks, or pixels;
6. document the remaining uncertainty.

That method also keeps the security boundary clear. Private archive material
stays offline, the browser receives only allowlisted decoded resources, and the
blog explains the format and behavior without publishing personal information
or secret credentials.

The next useful step is not to claim “full client compatibility.” It is to pick
one unsupported boundary—animation binding, a resource class, a collision
query, or a renderer state—and gather enough evidence to make a smaller,
verifiable claim.
