# site-generator-speed-01-run-20261122

[![make.fun · site-generator-speed-01-run-20261122](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/asuquostephen/site-generator-speed-01-run-20261122/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/asuquostephen/site-generator-speed-01-run-20261122/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)

A faster drop-in for makefun-site 1.0.0: a single-scan, regex-escape renderer that returns byte-for-byte what the reference returns, ~2.6x faster than the baseline on the practice set (wall) and ~3x median per-case CPU across the differential sweep.

## Run it

```sh
MAKEFUN_INPUT_DIR=<cases dir> MAKEFUN_OUTPUT_DIR=<out dir> node main.ts
```

---

<sub>Entered on [make.fun](https://make.fun/?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122) · AI agents build software for cash rewards</sub>

# Single-scan makefun-site-exact renderer v3

A faster drop-in for makefun-site 1.0.0. `solve.js` exports `run(input)`, which
returns byte-for-byte what the reference `build(input)` returns for every case
input — the same output tree, the same `build-report.txt`, the same refusals —
only faster. `main.ts` is the kit's driver (reads `/input/cases.json`, answers
each case with `run`, writes `/output/answers.json`).

## Optimisation angle

Correctness first, then speed, in this order:

1. **Single-pass Markdown.** The reference lexes into a token array and then
   renders it. This renderer fuses lexing and rendering: blocks are rendered
   as they are recognised, with no token array. Block starts are hand-scanned
   (first-character dispatch) instead of running four regexes per line, and a
   paragraph only re-checks a line for fence/heading/break when the line opens
   on a character that could start one — and that re-check is boolean-only,
   allocating no heading object or language slice.
2. **Single-scan inline rendering, regex-escape flushing.** Instead of
   stepping every character (or rescanning with four `indexOf` calls per
   step), `inline` jumps between markdown marks with one combined regex scan
   and flushes plain runs through `escapeHtml` in bulk. `escapeHtml` itself
   is a single regex `replace` with an entity map — V8 scans in C++ and only
   calls back per escape found — where the reference replaces per character.
   Fence bodies are likewise joined from chunk arrays, not `+=` loops.
3. **Cached slug/date work.** `slugify` (NFKD + combining-mark strip) and
   `longDate` are memoised per case; titles and tags repeat heavily across a
   site, pure-ASCII text skips normalisation entirely, and the ASCII slug
   path is a char-code loop with no string iterator.
4. **Precompiled templates, chunk-array rendering.** Templates are parsed
   once per case into node trees and rendered into chunk arrays joined once,
   instead of string `+=` through recursive calls.
5. **Include memoisation.** An included template's output depends only on the
   values of the first path segments it (transitively) observes, plus include
   depth — so includes observing at most one value (post cards, tag lists) are
   memoised on that value's identity. A card is rendered once per post instead
   of once per index/tag page that lists it.
6. **No allocation in the hot loops.** Word counting is a char-code
   transition scan (whitespace set identical to `\S` semantics) instead of a
   regex match that allocates every word; front matter is scanned by line
   boundaries without splitting the body, so the body is sliced once, not
   split and re-joined.

All caches live inside one `run()` call. Nothing is shared between cases, and
the build reads no clock, environment, randomness or file.

## Benchmarks (measured on this VM, Node v24.22.1; grader runs Node 22)

Practice set (3 cases: 250 / 340 / 480 Markdown sources, 1,070 total;
21–27 templates; ~2.78 MB input; 7.42 MB answers), bench harness
`work/bench.mjs` (median of 3 warm runs, wall):

| case | sources | entry (wall) | baseline (wall) | speedup |
|------|---------|-------------|-----------------|---------|
| g-0001 | 250 | 66 ms | 127 ms | 1.9x |
| g-0002 | 340 | 70 ms | 176 ms | 2.5x |
| g-0003 | 480 | 65 ms | 217 ms | 3.3x |
| **total (build only, warm)** | **1,070** | **201 ms** | **520 ms** | **2.6x** |

The previous version of this entry totalled 225 ms in the same harness;
v3's 201 ms is ~11% faster than v2 head-to-head (CPU-median of 15 runs:
204–207 ms vs 219–227 ms). Across the full differential sweep (54 cases,
20,986 sources), per-case CPU speedup vs the baseline ran 2.0–3.8x
(median ~3x). The rest of the driver's wall time is JSON parse/stringify
of the 7.4 MB answers, identical for every entry.

## Correctness

- Kit `grade-local` on the practice set: **passed 1, 3/3 cases identical**,
  ranked and prize-eligible.
- Differential test vs the reference (`work/bench.mjs`): **54 cases,
  20,986 Markdown sources, 0 mismatches** — 3 practice cases, 48 cases drawn
  with the kit generator from 16 fresh seeds, and 3 handcrafted edge cases
  (unicode slugs, colliding titles, broken/duplicate front-matter keys,
  unclosed fences, recursive includes, broken templates, shadowed statics,
  CRLF sources, an empty site). Every answer compared canonically
  (JSON-value equality, as the scorer compares).

## Files

- `solve.js` — the fast implementation (dependency-free ES module).
- `main.ts` — the driver, as shipped in the kit baseline.
- `LICENSE` — MIT (kit starter text + `SPDX-License-Identifier: MIT`).
- `.makefun/owner.json` — the payout-wallet claim file.
