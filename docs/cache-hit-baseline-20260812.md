# dsh prefix-cache hit-rate baseline (2026-08-12)

English | [中文](cache-hit-baseline-20260812.zh.md)

> **Purpose**: baseline collection for the first Tianshu capability-fusion "assessment analysis" — the benchmark for before/after Wave 4 (prefix-cache hardening). Measurement script: `examples/headless-agent/baseline-probe.mts` (real model + real session shape, reproducible).

## Methodology

- **Carrier**: `examples/headless-agent` codingHarness (real LlmDeepSeek adapter + real bash/todo tools)
- **Model**: deepseek-official / deepseek-v4-flash
- **Shape**: one session, 20 turns of a fixed task sequence (small read/write/query operations, repeatedly touching the same files — a prefix-cache-friendly typical coding session)
- **Observation**: `assistant/message` event `usage` (session/types.ts:245, includes cacheReadTokens)
- **Hit-rate definition**: `ΣcacheReadTokens / Σ(cacheReadTokens + inputTokens)` — the denominator is prompt_tokens (llm-deepseek `mapUsage` already subtracts cacheRead from inputTokens, translate.ts:57, hence the sum of both)
- **Run**: `node --env-file=.env --import tsx/esm examples/headless-agent/baseline-probe.mts <run> <out.json>`, three consecutive runs

## Results summary

| run | uncached input (inputTokens) | cache read (cacheReadTokens) | overall hit rate |
|---|---|---|---|
| run1 | 5,237 | 147,584 | **0.9657** |
| run2 | 4,494 | 139,776 | **0.9689** |
| run3 | 4,282 | 139,776 | **0.9703** |

**Three-run average: 96.8%** (range 96.6%-97.0%, stable)

## Key observations

1. **Hit rate rises monotonically across turns**: turn 1 ≈ 95.2% → turn 20 ≈ 96.6-97.0% — prefix-cache benefit grows with context; long sessions are the cache's home turf
2. **Cross-run cache accumulation**: uncached input falls run over run (5237 → 4494 → 4282) — re-running the same sequence partially hits the provider cache on previous prompt prefixes (within DeepSeek cache TTL)
3. **Turn 1 is identical across all three runs** (117 input + 2304 cache = 0.9517) — first-turn prompt is deterministic; the measurement is reproducible
4. **Turn 2 is the low point** (0.90-0.93): tool turns change the prompt / break cache — tool-result injection is the main perturbation source for prefix stability
5. **run2/run3 cache totals are exactly identical** (139,776) — steady-state cache-hit determinism

## Comparison with Tianshu

- Tianshu steady state 95-99%; **the dsh baseline 96.6-97.0% already sits in that range** — the master plan's "target ≥90% only takes effect after the baseline confirms a gap": the baseline shows **no gap**, so Wave 4's main benefit is "hardening into a contract + regression assertions + compact cache anchors" (preventing future regressions) rather than hit-rate gains

## Full per-turn data

### run1 (overall 0.9657)

| turn | input | cache | cumulative hit rate |
|---|---|---|---|
| 1 | 117 | 2304 | 0.9517 |
| 2 | 251 | 2560 | 0.9297 |
| 3 | 392 | 3200 | 0.9139 |
| 4 | 362 | 5760 | 0.9249 |
| 5 | 111 | 4480 | 0.9369 |
| 6 | 132 | 4736 | 0.9441 |
| 7 | 319 | 7552 | 0.9478 |
| 8 | 198 | 5504 | 0.9504 |
| 9 | 408 | 8704 | 0.9514 |
| 10 | 166 | 6272 | 0.9541 |
| 11 | 294 | 6528 | 0.9544 |
| 12 | 303 | 10880 | 0.9573 |
| 13 | 191 | 7808 | 0.9592 |
| 14 | 250 | 8064 | 0.9602 |
| 15 | 168 | 8576 | 0.9621 |
| 16 | 396 | 13312 | 0.9632 |
| 17 | 255 | 14464 | 0.9655 |
| 18 | 274 | 10240 | 0.9662 |
| 19 | 588 | 10880 | 0.9648 |
| 20 | 62 | 5760 | 0.9657 |

### run2 (overall 0.9689)

| turn | input | cache | cumulative hit rate |
|---|---|---|---|
| 1 | 117 | 2304 | 0.9517 |
| 2 | 389 | 2432 | 0.9035 |
| 3 | 117 | 3456 | 0.9293 |
| 4 | 427 | 5760 | 0.9300 |
| 5 | 128 | 4480 | 0.9399 |
| 6 | 141 | 4736 | 0.9461 |
| 7 | 207 | 7680 | 0.9529 |
| 8 | 190 | 5504 | 0.9549 |
| 9 | 215 | 8832 | 0.9590 |
| 10 | 158 | 6272 | 0.9610 |
| 11 | 136 | 6656 | 0.9631 |
| 12 | 361 | 10496 | 0.9637 |
| 13 | 138 | 7552 | 0.9655 |
| 14 | 213 | 7808 | 0.9662 |
| 15 | 129 | 8320 | 0.9678 |
| 16 | 347 | 12928 | 0.9686 |
| 17 | 192 | 9216 | 0.9695 |
| 18 | 271 | 9600 | 0.9697 |
| 19 | 482 | 10368 | 0.9686 |
| 20 | 136 | 5376 | 0.9689 |

### run3 (overall 0.9703)

| turn | input | cache | cumulative hit rate |
|---|---|---|---|
| 1 | 117 | 2304 | 0.9517 |
| 2 | 227 | 2560 | 0.9339 |
| 3 | 120 | 3328 | 0.9464 |
| 4 | 422 | 5632 | 0.9398 |
| 5 | 184 | 4352 | 0.9444 |
| 6 | 199 | 4608 | 0.9472 |
| 7 | 238 | 7552 | 0.9527 |
| 8 | 162 | 5504 | 0.9555 |
| 9 | 213 | 8832 | 0.9596 |
| 10 | 110 | 6272 | 0.9624 |
| 11 | 206 | 6528 | 0.9632 |
| 12 | 345 | 10496 | 0.9639 |
| 13 | 189 | 7552 | 0.9651 |
| 14 | 261 | 7808 | 0.9653 |
| 15 | 186 | 8320 | 0.9665 |
| 16 | 267 | 13056 | 0.9681 |
| 17 | 169 | 9216 | 0.9692 |
| 18 | 247 | 9728 | 0.9697 |
| 19 | 350 | 10624 | 0.9696 |
| 20 | 70 | 5504 | 0.9703 |

## Measurement boundaries

- Single-session shape (no /fork /rewind /multi-task parallelism); 20 turns is a fixed sequence — real development sessions are longer with a messier tool surface; the hit rate may be slightly lower
- DeepSeek provider cache TTL affects cross-run data (run2/3 cache accumulation)
- Not covered: cache breaks after compact triggers, multi-session parallelism (shared-cache gains)
- Reproducibility: the script stays at `examples/headless-agent/baseline-probe.mts`; future Wave 4 comparisons use the same script for comparability
