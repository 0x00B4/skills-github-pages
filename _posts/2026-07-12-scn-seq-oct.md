---
layout: post
title: "SCN, SEQ, and OCT: three different binary worlds"
date: 2026-07-12
author: 0x00b4
categories: [formats, s4-client, reverse-engineering]
tags: [scn, seq, oct, wasm]
---

The extensions in the v1267 catalog are not interchangeable containers. They
represent different runtime responsibilities. `.scn` describes scene and
geometry data, `.seq` describes actions and effects, and `.oct` stores spatial
collision information. The extension is only a dispatch hint after the bytes
have already been decoded from the resource system.

## SCN: typed scene graphs and geometry

An SCN stream is a typed hierarchy. A scene group carries a version, a local
matrix, child count, and children. Each child has a type ID, names, and a
class-specific payload. The original client uses CRC32 class IDs and virtual
deserializers. Unknown classes cannot safely be skipped because the stream does
not provide a universal payload length.

The reconstructed layout-0 geometry subset reads:

```text
position count + float32 XYZ positions
triangle count + uint16 index triplets
normal count + float32 XYZ normals
UV count + float32 UV pairs
extra-vector count + float32 XYZ vectors
```

Materials keep their serialized texture and lightmap names plus triangle
ranges. Positions remain local. World matrices are composed separately using
the client's row-vector, row-major convention, with translation in indices
12–14. Applying a matrix twice is just as wrong as not applying it at all.

Nonzero skin and morph payloads, unsupported geometry layouts, unknown node
classes, and animation playback are rejected explicitly in the bounded parser.
A parsed animation key set is metadata, not a claim that interpolation works.

## SEQ: actions with absolute tick windows

A SEQ starts with a version, a NUL-terminated name, a flag, a raw 32-bit value,
and a track count. Each track then has a class name and a class-specific
reader. A class reader may consume nested action units, node loaders, particle
units, lightning, ghost trails, or node controls.

There is no generic length prefix that permits recovery from every unknown
class. If a class or version is not understood, the safe behavior is to stop
at that boundary and return a diagnostic. Scanning forward for plausible floats
would turn corrupted input into fake effects.

The later bounded runtime preserves absolute inclusive `startTick` and
`endTick` values. It supports structural parsing of the full catalog and a
small deterministic particle profile, but not complete actor-bound playback.
That distinction is important: “the SEQ consumed to EOF” and “the game effect
rendered exactly” are different statements.

## OCT: map collision as a serialized tree

The v1267 map loader looks for a companion OCT resource. Its header contains a
version, a flag, global bounds, and a root node. Branch nodes contain up to
eight occupied children. Leaf nodes contain named triangle groups, flags, and
triangle vertices. The port retains the input order and creates sequential
indices for the unindexed triangles.

The result is collision geometry and spatial metadata—not a complete game
collision world. The browser can build a bounded static acceleration structure
from it, but that does not prove the original movement solver, dynamic objects,
UV meanings, or every native query policy.

## Why separate parsers help

A scene mesh, an effect timeline, and a collision tree are consumed by different
systems. Keeping them separate makes failures useful:

- an SCN diagnostic does not become a fake mesh;
- an incomplete SEQ does not advertise a playable timeline;
- an OCT leaf does not silently become a rendered map surface;
- a valid parser result does not imply that textures, animation, or shaders are
  implemented.

That separation is what makes a reconstruction inspectable rather than merely
convincing in a screenshot.
