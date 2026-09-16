---
layout: post
title: "From encrypted resource index to a safe catalog"
date: 2026-06-22
author: 0x00b4
categories: [s4-client, resources, reverse-engineering]
tags: [v1267, encryption, seed, x7, lzo]
---

The first useful question in an S4 League v1267 reconstruction is not “how do I
render a map?” It is “how does the client find the bytes that describe the
map?” The answer is a layered resource pipeline: an encrypted index, compact
resource records, hashed on-disk blobs, optional compression, and a second
small decryption step for individual payloads.

## The index is the directory

The original index is a 4,774,840-byte encrypted stream. After decoding, it
contains 17,300 records. Each record is 272 bytes and contains the logical path
(for example, a scene or texture path), an identifier, and the decoded size.
The identifier is kept as a lowercase hexadecimal string. Treating it as a
JavaScript number would lose precision and silently select the wrong file.

The decoded catalog is deliberately boring JSON:

```json
{
  "path": "resources/mapinfo/bginfo-19days.ini.oct",
  "id": "8c213d46f2db7b3d",
  "size": 2924198,
  "ext": "oct"
}
```

The path is the client-facing name. The ID selects a blob from the private
`_resources` directory. A web client should never be allowed to turn an
arbitrary URL into a filesystem path, so the server uses this catalog as an
allowlist.

## What “encrypted” means here

The v1267 resource protection is a sequence of transformations, not one magic
function:

1. Complete 4-byte blocks are transposed in 4×4 groups.
2. A 16-byte SEED key and 16-byte IV are removed from positions inside the
   remaining index data.
3. An X7 table operation applies XOR and a one-bit rotate.
4. SEED-128 in CTR mode decrypts the stream with a big-endian counter.
5. Complete blocks are transposed again.

The index derives its table selection from the payload length and a fixed
constant. The per-index SEED key and counter are read from the index itself.
The implementation keeps this material local and does not put it into the
browser bundle.

Individual resource blobs have their own path. The loader opens the lowercase
hex ID, swaps the bounded prefix/suffix region, optionally decompresses LZO1X,
and applies the old-capped byte transformation. The LZO decoder is a native
offline dependency in the exporter, not a browser-side game DLL.

## Why the boundary matters

The safe architecture is therefore:

```text
private original files
        ↓ authorized offline exporter
catalog + selected decoded resources
        ↓ allowlisted HTTP endpoint
browser/WASM parser
```

The browser receives decoded bytes only for an explicitly catalogued path. It
never receives archive keys or the private codec table. SHA-256 values in the
local manifest provide reproducibility; they are not presented as a publisher
signature or an integrity guarantee that the original client itself did not
prove.

The complete pipeline is useful precisely because it stops making claims where
the evidence stops. A filename extension does not decrypt a file, and a valid
prefix does not make a truncated resource safe to render.
