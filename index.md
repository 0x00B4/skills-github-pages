---
layout: default
title: eSperClient 1267 Research Notes
---

# eSperClient 1267 Research Notes

This blog is a technical, evidence-based reconstruction log for the S4 League
v1267 client. The focus is on how the client stores, decrypts, parses, loads,
and updates its resources—not on reproducing private credentials or publishing
key material.

The series covers the resource archive, `.scn`, `.seq`, and `.oct` formats,
map loading, millisecond ticks, effect activation, locomotion state machines,
and the limits of a browser/WASM reconstruction.

All posts are written under the author name **0x00b4**. The work described here
is static analysis and controlled format reconstruction; unsupported behavior is
marked as unsupported instead of being filled in with guesses.

## Series

1. [From encrypted resource index to a safe catalog](/2026/06/22/resource-index-pipeline/)
2. [SCN, SEQ, and OCT: three different binary worlds](/2026/07/12/scn-seq-oct/)
3. [How a map becomes geometry in the client](/2026/08/02/map-loading-pipeline/)
4. [Ticks, activation windows, and deterministic effect updates](/2026/08/23/ticks-and-effects/)
5. [The actor state machine: run, jump, fall, and walljump](/2026/09/06/actor-state-machine/)
6. [What the browser/WASM port can and cannot claim](/2026/09/13/reconstruction-boundaries/)
