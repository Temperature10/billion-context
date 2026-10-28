# Retrieval evaluation

## Reproduce

After installing dependencies and building the kernel prerequisites:

```sh
node --import tsx scripts/eval-plan-state-retrieval.ts --json plan-retrieval-results.json
node --import tsx --test tests/plan-state-retrieval.test.ts tests/plan-state-retrieval-eval.test.ts tests/search-plan-aware.test.ts
```

The runner uses real `core.search` candidate generation and `executeSearchContext` rendering. The lexical arm disables plan ranking. The old arm freezes extraction/ranking from `b0b0e66ff085b08e8e26c02f1f79ad2a8849a496`; its rendered output uses a punctuation suffix to trigger the runtime's compatibility path, with exact token-weight and ordering assertions against the frozen reference on every call. Baseline parity was checked on the unmodified source before the fix.

The 28-case fixture was authored separately before inspecting improved rankings. Four strict pairs change only task statuses. Case content SHA-256: `ed972c878998149968153a3bef790f8ab19518853f12c9d48d2bb89b3c99e7a8`. Removing three unused type exports later changed the file hash, not case content, labels or order. Current file SHA-256: `e232f1037542d3c5bca6196b84542537be38478df65daadb67a3aa1fcf9b3bc7`.

## Results

P@1 is measured using an actual limit=1 call. MRR@3 is measured using a separate limit=3 call; it is the reciprocal rank of the first accepted relevant block, or zero if absent. Every case has more than three lexical candidates, so both calls exercise the re-ranking gate.

| Arm | P@1 | MRR@3 |
|---|---:|---:|
| Lexical | 17/28 (60.71%) | 0.6726 |
| Old plan-aware | 19/28 (67.86%) | 0.8393 |
| Status-aware | 27/28 (96.43%) | 0.9821 |

Against old plan-aware: 8 first-place improvements, 20 ties, 0 regressions. Against lexical, one unknown-status case remains worse because compatibility deliberately preserves the old plan override. All eight declared legacy-order controls pass.

| Targeted category | Old P@1 | New P@1 |
|---|---:|---:|
| Status switch | 4/8 | 8/8 |
| Pending without active task | 1/2 | 2/2 |
| Completed-task noise | 1/3 | 3/3 |
| Explicit historical query | 2/3 | 3/3 |

The JSON report contains every query, plan payload, candidate, accepted label, rationale, top-1/top-3 result and rendered tool output, plus all category metrics. The intentionally failing unknown-status case remains in the denominator.

## Review-driven improvement

An initial boolean guard preferred candidates containing every query term. Independent review found that `token JWT issuer?`, `token JWT issuer behavior`, `cache CDN purge details`, and `网关 超时 截止时间？` could disable that guard and incorrectly promote the active task. These four examples are recorded in a separate supplement, not added to the original denominator or described as a held-out sample. The initial guard failed all four; distinct-term coverage passes all four. A separate test prevents repeated query words from overpowering more distinct query evidence.

## Interpretation and boundaries

This deliberately targeted synthetic corpus demonstrates the mechanism and guards against regressions; it does not estimate production query quality. It does not measure automatic tool adoption, total task completion, token savings, or arbitrary-length plans. Terminal documents remain searchable. Unknown schemas, incremental task events, oversized snapshots, unsuccessful tool executions and fuzzy/CJK query rewriting remain outside this change. Defaults, foreign-session search, small candidate pools, original tool payloads and cache breakpoints retain their prior contracts.
