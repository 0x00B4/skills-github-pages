---
layout: post
title: "How a map becomes geometry in the client"
date: 2026-08-02
author: 0x00b4
categories: [maps, rendering, s4-client]
tags: [map-loading, scn, oct, webassembly]
---

A map is not one file. In the v1267 client, map loading combines metadata,
collision data, scene geometry, and textures. The browser reconstruction keeps
those jobs separate so that a missing optional resource cannot quietly become a
fabricated floor or a placeholder model.

## The loading sequence

A simplified map load looks like this:

```text
map metadata / INI
        ↓ resolve logical resource paths
SCN scene files ───────→ static geometry and transforms
OCT collision file ────→ spatial collision triangles
texture references ────→ native DDS/TGA/image decoding
        ↓
world scene + collision solver + inspector
```

The catalog maps each logical path to an exact resource ID. The server checks
that mapping before returning bytes. The client then chooses a parser from the
lowercase extension and passes an `ArrayBuffer` to the WASM engine.

## SCN and OCT have different jobs

SCN files provide typed scene groups, geometry arrays, material names, local
matrices, and selected animation metadata. The loader composes local matrices
with parent world matrices. The geometry remains local until a consumer asks
for world-space positions.

The companion OCT file is not a second visual mesh. It contains an octree-like
branch structure and leaf triangle groups used for collision and spatial
queries. The browser can turn the decoded triangles into a static collision
structure, while the renderer uses SCN geometry for visual inspection.

This distinction prevents a common error: using the collision tree as if it
were the complete textured scene. Collision triangles can be sufficient for a
walkable surface while lacking materials, UV interpretation, lighting, and
render-only geometry.

## WebAssembly as a bounded parser

The AssemblyScript module performs binary reads and returns JSON-compatible
results. Every count is checked against remaining bytes and conservative limits.
Non-finite coordinates, invalid indices, missing NUL delimiters, truncated
records, and trailing data produce diagnostics. Invalid or incomplete input
clears partial geometry instead of leaving a misleading prefix visible.

JavaScript remains responsible for HTTP, resource selection, and presentation.
It does not scan arbitrary bytes for plausible coordinates. That division keeps
format parsing reproducible and makes it possible to run the same parser from
Node tests and a browser worker.

## What “loaded” means

A resource can be loaded at several different levels:

1. its catalog record was resolved;
2. its bytes were fetched and decoded;
3. its binary structure was consumed safely;
4. its geometry was transformed into the intended coordinate space;
5. its textures and materials were rendered;
6. its animation, collision, and gameplay behavior matched the original.

The v1267 reconstruction reports these levels separately. A successful SCN
parse proves level 3 and selected parts of level 4. It does not automatically
prove native material shading or animation playback. Honest status labels are
more useful than a universal `loaded: true` flag.
