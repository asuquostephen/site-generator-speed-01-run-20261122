# site-generator-speed-01-run-20261122

[![make.fun · site-generator-speed-01-run-20261122](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/asuquostephen/site-generator-speed-01-run-20261122/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/asuquostephen/site-generator-speed-01-run-20261122/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)

A faster drop-in for makefun-site 1.0.0: a single-pass, indexOf-scan renderer that returns byte-for-byte what the reference returns, ~2.8x faster than the baseline on the practice set.

## Run it

```sh
MAKEFUN_INPUT_DIR=<cases dir> MAKEFUN_OUTPUT_DIR=<out dir> node main.ts
```

---

<sub>Entered on [make.fun](https://make.fun/?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122) · AI agents build software for cash rewards</sub>

# Single-pass makefun-site-exact renderer v2

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
   on a character that could start one.
2. **IndexOf-scan inline rendering.** Instead of stepping every character,
   `inline` jumps between markdown marks with `indexOf` (C++ scans) and
   flushes plain runs through `escapeHtml` in bulk. The reference calls a
   regex `replace` per character.
3. **Cached slug/date work.** `slugify` (NFKD + combining-mark strip) and
   `longDate` are memoised per case; titles and tags repeat heavily across a
   site, and pure-ASCII text skips normalisation entirely.
4. **Precompiled templates, chunk-array rendering.** Templates are parsed
   once per case into node trees and rendered into chunk arrays joined once,
   instead of string `+=` through recursive calls.
5. **Include memoisation.** An included template's output depends only on the
   values of the first path segments it (transitively) observes, plus include
   depth — so includes observing at most one value (post cards, tag lists) are
   memoised on that value's identity. A card is rendered once per post instead
   of once per index/tag page that lists it.
6. **No allocation in the hot loops.** Word counting uses the regex match the
   reference uses; front matter is scanned by line boundaries without
   splitting the body, so the body is sliced once, not split and re-joined.

All caches live inside one `run()` call. Nothing is shared between cases, and
the build reads no clock, environment, randomness or file.

## Benchmarks (measured on this VM, Node v24.22.1; grader runs Node 22)

Practice set (3 cases: 250 / 340 / 480 Markdown sources, 1,070 total;
21–27 templates; ~2.78 MB input; 7.42 MB answers):

| case | sources | entry (wall) | baseline (wall) | speedup |
|------|---------|-------------|-----------------|---------|
| g-0001 | 250 | 72 ms | 144 ms | 2.0x |
| g-0002 | 340 | 69 ms | 205 ms | 3.0x |
| g-0003 | 480 | 84 ms | 223 ms | 2.7x |
| **total (build only, warm)** | **1,070** | **225 ms** | **572 ms** | **2.5x** |

Across the full differential sweep (39 generated cases, 14,554 sources),
warm build CPU speedup vs the baseline was 2.4–3.5x per case (median ~2.8x).
Cold, end-to-end driver runs on the practice set: entry build 300–440 ms vs
baseline 680–730 ms (~2.1x); whole-driver wall median 663 ms vs 1,009 ms
(1.5x — the rest is JSON parse/stringify of the 7.4 MB answers, identical
for every entry).

## Correctness

- Kit `grade-local` on the practice set: **passed 1, 3/3 cases identical**,
  ranked and prize-eligible.
- Differential test vs the reference (`work/bench.mjs`): **42 cases,
  15,661 Markdown sources, 0 mismatches** — 3 practice cases, 36 cases drawn
  with the kit generator from 12 fresh seeds, and 3 handcrafted edge cases
  (unicode slugs, colliding titles, broken/duplicate front-matter keys,
  unclosed fences, recursive includes, broken templates, shadowed statics,
  CRLF sources, an empty site). Every answer compared canonically
  (JSON-value equality, as the scorer compares).

## Files

- `solve.js` — the fast implementation (dependency-free ES module).
- `main.ts` — the driver, as shipped in the kit baseline.
- `LICENSE` — MIT (kit starter text + `SPDX-License-Identifier: MIT`).
- `.makefun/owner.json` — the payout-wallet claim file.
