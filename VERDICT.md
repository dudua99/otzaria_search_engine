READY

- Head: `81edfc1eb9f6611d583b52ba5e9c0f69ebdca259` on `dudua99/otzaria_search_engine:backport/reclaim-deleted-segments-7.2`
- Base branch: `codex/search-engine-7.2` (= tag `v7.2.1`, `6a588c8`)
- Upstream source: PR #97, merge `2d773befb` (for Otzaria/otzaria#5040). PR #96 (`9b058f7c6`, memory budget) is NOT included.

## Commits
1. `79ac15a` tests only (#97's tests, adapted)
2. `81edfc1` the backport

## Tests (`cargo test` in `rust/`, full suite, Linux)
| | passed | failed | ignored |
|---|---|---|---|
| v7.2.1 (`6a588c8`) | 943 | 0 | 30 |
| branch (`81edfc1`) | 953 | 0 | 30 |

lib unit tests: 902/0/27 -> 912/0/27. +10 = 7 tests in `search_engine_tests.rs` + 3 in `index_directory/retired_segments.rs`.

Tests fail on v7.2.1:
- commit 1 alone on v7.2.1 does not compile (15 errors: `select_deleted_to_reclaim`, `BACKGROUND_RECLAIM_MAX_BYTES/_MAX_DOCS`, `size_aware_merge_policy` missing).
- Runtime check (behavioral tests only, policy built as 7.2's `SizeAwareMergePolicy { index_path }`): `the_background_policy_reclaims_a_delete_heavy_segment` FAILED, `normal_writer_reclaims_segment_files_without_another_commit` FAILED; `normal_writer_reconfiguration_does_not_start_reclaim_work` passes on both (regression guard).

`cargo fmt --check`: clean. `cargo clippy --all-targets`: only the pre-existing `note_spans.rs:294` dead-code warning (untouched file, same on base).
Dart tests (`test/*.dart`, FFI): NOT run — no Flutter/Dart SDK in this environment. No Dart API change; only the FRB-generated "ignored functions" header comment in `lib/src/rust/api/search_engine.dart` was updated by hand as #97 did (CI ignores that line). `frb_generated.*` unchanged.

## What was adapted from #97 and why
- #97 bounds document count via `crate::memory_budget::merge_doc_limit(system_memory())` from #96, which 7.2 lacks. The background reclaim passes `BACKGROUND_RECLAIM_MAX_DOCS` (1,000,000, #97's own constant) as `max_docs` instead.
- `select_smallest_level` and `select_segments_to_compact` keep 7.2's signatures (no `max_docs`); `select_segments_to_compact` only reuses `is_delete_heavy`. One test call adapted: `select_smallest_level(&metas)`.
- Unchanged from #97: >30% deleted (or >=16 MiB estimated savings), <=128 MiB, one segment at a time (`ActiveReclaim` slot), on Tantivy's merge pool, no `optimize`, no `SearchEngine` lock; Windows deferred deletion of still-mapped segment files (`index_directory/retired_segments.rs`); `set_economy_indexing` drains with `NoMergePolicy`.
- CHANGELOG/version not touched: on this line release commits (`7.2.1` = `6a588c8`) bump `pubspec.yaml`, `Cargo.toml/.lock` and CHANGELOG separately from feature backports (`ce32e6e`).

## Release step needed
Maintainer: merge into `codex/search-engine-7.2`, then the release commit `7.2.2` (CHANGELOG entry + pubspec/Cargo version) and tag `v7.2.2` via `release-tag.yml`.
App (Otzaria/otzaria): bump the pinned engine from 7.2.1 to 7.2.2 (pubspec dependency/ref of `search_engine`, update `pubspec.lock`), keep the 7.2 line (not 7.4.0), run the app tests incl. the PDF line-highlighting and cross-line regex tests, then drop/simplify any app-side compaction workaround for #5040 if desired.
