# mqo-parity-coverage-tracker

Turns one-shot cross-backend parity results into a build-over-build coverage series and a regression tripwire.

## Why

`mqo-cross-backend-parity` answers a point-in-time question: do two backends agree on this measure right now? It emits a `ParityReport` and forgets it. Nothing stamps that result with a server build, accumulates it across builds, or divides it by a denominator — so "1 measure verified" never becomes "1 of 80 measures verified at build rc1, down from 3 at rc0."

That denominator is the whole point. A verdict without a denominator tells you what passed; it does not tell you what you have not checked. This tool supplies the missing axis: it records each verdict against a build id in an append-only log, measures coverage against a fixed corpus, and flags any measure that went from verified to mismatch between two builds.

It computes; it does not test. Comparison happens upstream in `mqo-cross-backend-parity`; this tool consumes the verdict, never re-derives it. No network, no credentials, no ports.

## Install

```bash
cargo install --path .
# installs the `parity-tracker` binary
```

Requires Rust 1.88+ (edition 2021). To build without installing:

```bash
cargo build --release   # ./target/release/parity-tracker
```

## Quickstart

Record a parity result, stamp it with a build id, then read coverage against a corpus:

```bash
# 1. Record a ParityReport from mqo-cross-backend-parity
parity-tracker record \
  --input results/parity-run.json \
  --build-id 2026.06.10-rc1 \
  --history ./history.jsonl
# Recorded build '2026.06.10-rc1': 1 pair(s), all_skipped=false, digest=7154b870

# 2. Compute coverage against the corpus denominator
parity-tracker coverage \
  --build-id 2026.06.10-rc1 \
  --corpus corpus.json \
  --history ./history.jsonl
```

```
Coverage for build '2026.06.10-rc1' [dax, denominator: 3 corpus cells (parity-corpus.v1)]: 33.3%
  verified=1 mismatch=0 never-tested=2
  ungrounded (has_grounding=false): 1

  MEASURE                                            BACKEND_A    BACKEND_B    STATUS
  ------------------------------------------------------------------------------------------
  Total Store Sales                                  dax          sql          verified
  Margin %                                           dax          sql          never-tested
  Returns Amount                                     dax          sql          never-tested [ungrounded]
```

One verdict recorded, three corpus cells in the denominator: 33.3% coverage, two cells never tested.

## Input shape

`record` ingests a `ParityReport` — the JSON `mqo-cross-backend-parity` emits. One report, one measure, one or more backend pairs:

```json
{"measure":"Total Store Sales","pairs":[["dax","sql","Equal"]],"overall":"Agree"}
```

Each pair is `[backend_a, backend_b, verdict]`. `Equal` and `WithinTolerance` normalise to **verified**; `Mismatch` to **mismatch**; a skipped backend to **skipped** (no coverage evidence). Pass `--live-harness` to ingest an array of per-case results from `mqo-backend-live-harness` instead.

The corpus is a `parity-corpus.v1` document — the denominator, produced by `mqo-parity-corpus-gen`:

```json
{
  "version": "parity-corpus.v1",
  "catalog": "tpcds",
  "cases": [
    {"case_id": "tss_total", "measures": ["Total Store Sales"], "grain": "total", "has_grounding": true},
    {"case_id": "returns_total", "measures": ["Returns Amount"], "grain": "total", "has_grounding": false}
  ]
}
```

## Commands

```
parity-tracker record   --input <path|-> --build-id <id> [--history <path>] [--live-harness]
parity-tracker coverage --build-id <id> --corpus <path>  [--history <path>] [--format json|human]
parity-tracker delta    --build <N> --against <N-1> --corpus <path> [--history <path>] [--no-fail]
parity-tracker regressions [--history <path>] [--format json|human]
```

Default history store: `~/.local/share/mqo-parity/history.jsonl` (override with `--history`).

### record

Appends one entry to the JSONL store. `--build-id` is mandatory and never inferred from the capture — the build that produced a result is exactly the fact the capture cannot carry, so the caller must supply it.

### coverage

Reports per-(measure, backend-pair) status — `verified`, `mismatch`, or `never-tested` — and an overall percentage: verified cells over corpus cells. The backend pair is read from the recorded history for that build; supply `--backend-a`/`--backend-b` to hint it when the history is empty.

`--format json` emits a stable contract consumed by `mqo-parity-brief` and CI:

```json
{
  "build_id": "2026.06.10-rc1",
  "corpus_version": "parity-corpus.v1",
  "verified_count": 1,
  "mismatch_count": 0,
  "never_tested_count": 2,
  "coverage_pct": 33.3,
  "denominator": 3,
  "backend_a": "dax",
  "backend_b": "sql",
  "cells": [
    { "measure": "Total Store Sales", "backend_a": "dax", "backend_b": "sql",
      "status": "verified", "ungrounded": false }
  ],
  "ungrounded_cells": []
}
```

### delta

Diffs two recorded builds and reports the three transitions that matter: newly-verified, newly-broken, and newly-tested measures. Newly-broken — verified at the prior build, mismatch at the candidate — is the regression case. The command exits 2 when that list is non-empty, so it gates a release directly; pass `--no-fail` to suppress the non-zero exit.

```bash
parity-tracker delta \
  --build 2026.06.11-rc1 \
  --against 2026.06.10-rc1 \
  --corpus corpus.json \
  --history ./history.jsonl
# exits 2 if any measure regressed (verified -> mismatch)
```

If the prior build has no recorded history, the command warns and treats every current verdict as newly-tested.

### regressions

Lists every build id recorded in the history store, in the order first seen — the index you scan before picking two builds to `delta`.

## Library API

The same computation is available as a library:

```rust
use mqo_parity_coverage_tracker::{
    record_run, load_history, coverage_summary, delta_summary,
    store::InputReport,
};

// Append a result to the store
record_run(&input_report, "2026.06.10-rc1", "my-run.json", &store_path)?;

// Load all valid entries (corrupt trailing lines are skipped with a warning)
let history = load_history(&store_path)?;

// Compute coverage
let summary = coverage_summary(&history, &corpus, "2026.06.10-rc1", None);
println!("{:.1}% parity-verified", summary.coverage_pct);

// Build-over-build diff
let delta = delta_summary(&history, &corpus, "build-N", "build-N-1", None);
if !delta.newly_broken.is_empty() { /* block ship */ }
```

## Crate layout

```
src/
  lib.rs          public API re-exports
  history.rs      JSONL store schema (HistoryEntry, PairRecord, ParsedVerdict)
  corpus.rs       parity-corpus.v1 denominator types (Corpus, CorpusCase)
  store.rs        record_run / load_history (append + tolerant read)
  summary.rs      coverage_summary / delta_summary (deterministic, offline)
  main.rs         parity-tracker CLI (record / coverage / delta / regressions)
```

## Design decisions

- **Append-only JSONL.** No storage dependency; each line is one `HistoryEntry` carrying a `schema_version` so the schema can evolve additively.
- **Backend pairs are data, not constants.** `backend_a`/`backend_b` are strings from the verdict, so an MDX pair joins later without a schema break.
- **Deterministic and offline.** Identical `(history, corpus, build-id)` inputs produce byte-identical `--format json` output.
- **Tolerant reader.** A corrupt or partial trailing JSONL line is skipped with a stderr warning; valid history is always reported.
- **Idempotent-safe retry.** Each entry carries a SHA-256 of `(build_id, sorted normalised pairs)`, so a re-record of the same result de-duplicates before it is counted.
- **A skipped run is recorded but counts for nothing.** When every backend was down, the entry lands in the log but changes no verified or mismatch count — absence of evidence is not evidence.

## Where it fits

Part of the MQO parity toolchain:

- `mqo-cross-backend-parity` — produces the per-run `ParityReport` this tool ingests.
- `mqo-parity-corpus-gen` — produces the `parity-corpus.v1` denominator.
- `mqo-backend-live-harness` — alternative per-case input (`--live-harness`).
- `mqo-parity-brief` — turns the `coverage` JSON into a human-readable brief.

This tool is the longitudinal store between them: it does no comparison, no corpus generation, and no markdown hand-back.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | I/O or parse error |
| 2 | `delta` newly-broken list is non-empty (override with `--no-fail`) |
