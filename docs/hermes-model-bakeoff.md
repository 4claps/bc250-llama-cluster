# Hermes model bake-off

> **2026-09-09 update:** the winning model, `Qwen3.6-35B-A3B`, was promoted in
> place from the Q4_K_M quant to Q4_K_L (Q8_0 embedding and output weights) and
> the context size was raised to 100,000 (later 115,000). A post-swap bake-off showed
> generation, time to first token, and error-task pass rate all at parity with
> Q4_K_M, so the model choice below is unchanged; only the quant and context
> differ. See [the Q4_K_L promotion record](qwen36-q4kl-promotion.md). The
> figures in this document are the original Q4_K_M bake-off.

> **2026-09-18 update:** a standalone candidate test of
> `Qwen3.8-27B-UD-Q3_K_XL` with MTP speculative decoding (two full 27-trial
> passes) is **not recommended** for production. Production is unchanged. See
> [the UD-Q3_K_XL candidate](#qwen38-27b-ud-q3_k_xl-candidate-2026-09-18).

> **2026-09-19 update:** a standalone bake-off of two Gemma 4 models
> (`gemma-4-26B-A4B` MoE and `gemma-4-31B` dense, both Q4_K_M), with a
> same-conditions re-run of `gpt-oss-20b` as a control, found neither Gemma
> model competitive with production. Both are **not recommended**. Production
> is unchanged. See [the Gemma 4 bake-off](#gemma-4-bake-off-2026-09-19).

## Final production bake-off update

The later production bake-off and subsequent UD-Q4_K_XL follow-up supersede
the preliminary ranking below. They used the same pinned llama.cpp revision,
65,536-token target, Q8_0 K/V cache, automatic split, and qualified cooling
baseline.

1. **Qwen3.6-35B-A3B Q4_K_M** — best overall Hermes backend, with 23/27
   practical successes (85.2%) and 51.72 tok/s standardized generation.
2. **gpt-oss-20b MXFP4** — useful fast/light alternative, with 21/27 practical
   successes and 66.30 tok/s generation, but more tool errors and turns.
3. **Qwen3-Coder-30B-A3B-Instruct Q4_K_M** — retained as the historical
   control, with 19/27 practical successes under the final timeout policy.
4. **Qwen3.6-35B-A3B UD-Q4_K_XL** — not recommended after two of its first
   nine Hermes trials reached the 180-second timeout, triggering the standard
   early-stop rule.
5. **Qwen3.8-27B Q4_K_M** — not recommended because of excessive and highly
   variable agent latency.
6. **Qwen3.8-27B UD-Q3_K_XL** — not recommended after a standalone candidate
   test on 2026-09-18 (115,000 context, MTP speculative decoding): about
   19 tok/s generation, `err_big_file_read` timed out on all six trials across
   both profiles, and Bowie ran past its 80°C limits. See
   [the candidate section](#qwen38-27b-ud-q3_k_xl-candidate-2026-09-18).
7. **Gemma 4 26B-A4B Q4_K_M** — not recommended after a standalone test on
   2026-09-19 (115,000 context, no speculative decoding): 22/27 with no
   timeouts and about 39 tok/s generation, but slower than production and than
   `gpt-oss-20b` at equal or lower accuracy. See
   [the Gemma 4 bake-off](#gemma-4-bake-off-2026-09-19).
8. **Gemma 4 31B Q4_K_M** — not recommended: it ran out of memory at 115,000
   context (tested at 65,536), generated about 12 tok/s, averaged 306 s per
   trial, and pushed Bowie past its 80°C limits.
9. **GLM-4.7-Flash Q4_K** — not recommended after both opening tasks reached
   the 180-second timeout.

The production selection is **Qwen3.6-35B-A3B** with Q8_0 K/V, one slot, all
layers offloaded, and automatic layer split. It was qualified here on the
Q4_K_M quant at 65,536 context; since 2026-09-09 it runs as the Q4_K_L quant,
at 100,000 context initially and 115,000 now (see
[the Q4_K_L promotion record](qwen36-q4kl-promotion.md)).
The cross-request host-RAM prompt cache is disabled with
`--cache-ram 0 --no-cache-idle-slots`; these flags do not disable the normal KV
cache. During the bake-off, retained unrelated large prompts caused the
coordinator to be OOM-killed until that cross-request cache was disabled.

The subsequent [performance-profile characterization](qwen36-performance-profiles.md)
confirmed Moderate/1750 as the preferred 24/7 Qwen3.6 profile. Strong/1850 and
Aggressive/2000 were stable but did not provide enough real-agent benefit to
replace it.

### Qwen3.6 UD-Q4_K_XL follow-up

The follow-up tested `unsloth/Qwen3.6-35B-A3B-GGUF` file
`Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf` (22,360,456,160 bytes, SHA-256
`707a55a8a4397ecde44de0c499d3e68c1ad1d240d1da65826b4949d1043f4450`).
No llama.cpp rebuild or compatibility override was needed. The model loaded at
65,536 context, offloaded all 41 layers, and used the production-equivalent
flags, Moderate/1750 profile, cooling, and automatic layer split used by
Q4_K_M.

| Metric | Q4_K_M control | UD-Q4_K_XL | XL change |
| --- | ---: | ---: | ---: |
| GGUF size | 20.42 GB | 22.36 GB | +9.51% |
| Standard prompt processing | 588.87 tok/s | 592.14 tok/s | +0.56% |
| Standard generation | 51.72 tok/s | 50.45 tok/s | -2.46% |
| Standard TTFT | 32.27 s | 32.09 s | -0.55% |
| Standard request | 52.05 s | 52.37 s | +0.62% |
| 44,927-token generation | 41.63 tok/s | 40.84 tok/s | -1.90% |
| Minimum free Vulkan, Bowie | 2.24 GB | 1.11 GB | -50.3% |
| Minimum free Vulkan, Crockett | 1.95 GB | 1.13 GB | -41.9% |

The official Hermes policy stopped the XL run after its first repetition of
all nine tasks: 5/9 practical successes, 143.6-second mean, 152.8-second
median, and two `MODEL_TIMEOUT_180S` results. For the same first repetitions,
Q4_K_M achieved 8/9 practical successes with a 100.3-second mean; XL was about
43% slower. The five successful XL tasks averaged 3.6 LLM turns and 3.2 tool
calls, with no tool execution errors, but latency and missed outcomes dominate
that clean tool-error count.

A supplemental 2,048-token stream at 44,927-token input measured successive
generation quarters at 40.78, 40.87, 40.77, and 40.64 tok/s. The 0.36% decline
from the first to final quarter is negligible: XL did not suffer a progressive
generation collapse, but it also did not improve throughput. Both boards held
1750 MHz; peak GPU temperatures were 66/68°C in the long-context run and
66/67°C in the Hermes run. No fan safety event, thermal throttle, GPU reset,
page fault, Vulkan error, or RPC failure occurred.

**Verdict:** UD-Q4_K_XL is technically stable but is not a replacement for
Q4_K_M, `gpt-oss-20b`, or the retained Qwen3-Coder control. It consumes more
memory, generates more slowly, and was materially less useful in the Hermes
tasks. Q4_K_M remains the production model.

### Qwen3.8-27B UD-Q3_K_XL candidate (2026-09-18)

This was a standalone candidate test, not a production swap. Production
`llama-server` was stopped on Bowie, the candidate was loaded on a separate port
against Crockett's existing RPC worker, and production
`Qwen3.6-35B-A3B-Q4_K_L` and the Moderate profile were restored and verified
afterwards. No Ansible, inventory, or persistent configuration was changed.

The artifact was `Qwen3.8-27B-UD-Q3_K_XL.gguf` (13,146,393,504 bytes, 27.3 B
parameters; SHA-256 was not recorded). It ran on the same pinned llama.cpp
revision `d775b8967a46d8beb110d444aa3b8938179e0dd8` with 115,000 context (slot
`n_ctx` 115,200), Q8_0 K/V cache, one slot, all 66 layers offloaded (64
repeating layers, the output layer, and the MTP block), and automatic layer
split:

```text
--rpc 10.250.0.2:50052 --gpu-layers all --split-mode layer
--ctx-size 115000 --parallel 1 --cache-type-k q8_0 --cache-type-v q8_0 --jinja
--cache-ram 0 --no-cache-idle-slots
--spec-type draft-mtp --spec-draft-n-max 3 --no-spec-draft-backend-sampling
```

These are the flags the live production unit used at the time of the test. The
startup log listed unused `blk.64.nextn.eh_proj`, `enorm`, and `hnorm` tensors,
the same signal that revealed MTP support in Qwen3.6, so MTP speculative
decoding was enabled. Mean draft acceptance was 0.75 (Moderate) and 0.72
(Strong). The earlier Qwen3.8-27B Q4_K_M GGUF offloaded 65/65 layers and had no
MTP block.

`--cache-ram 0 --no-cache-idle-slots` were required, not optional. The first
attempt, launched without them, was OOM-killed by the kernel partway through the
first pass (host RAM, the same failure mode described above for the original
bake-off). Its results were discarded and the pass was rerun from scratch with
the flags; no OOM occurred afterwards.

#### Protocol

Each profile ran the full 27-trial error-task battery (nine tasks, three
repetitions) with the harness early-stop rule disabled and a 600-second
per-trial timeout, so a timeout was recorded as a failed trial and the battery
continued. An earlier attempt with early-stop enabled ended at 18/27 after two
`err_big_file_read` timeouts and was superseded by the full rerun. The profiles
are Moderate/1750 and Strong/1850 as defined in
[the performance-profile characterization](qwen36-performance-profiles.md).

| Metric | Moderate | Strong |
| --- | ---: | ---: |
| Pass rate | 81% (22/27) | 89% (24/27) |
| Trials that hit the 600 s timeout | 5 | 3 |
| Mean generation, tok/s | 19.08 | 19.47 |
| Mean TTFT proxy | 43.5 s (median 6.9 s) | 41.4 s (median 6.7 s) |
| MTP draft acceptance | 0.75 | 0.72 |
| Battery mean wall / total | 295 s / 133 min | 260 s / 117 min |
| Bowie CPU °C avg/peak | 76.5 / 82.1 | 80.5 / 86.2 |
| Bowie GPU °C avg/peak | 71.6 / 83.0 | 75.1 / 88.0 |
| Bowie PPT W avg/peak | 101.7 / 149.3 | 107.0 / 150.9 |
| Crockett CPU °C avg/peak | 68.0 / 72.9 | 69.4 / 73.6 |
| Crockett GPU °C avg/peak | 63.9 / 74.0 | 65.0 / 75.0 |
| Crockett PPT W avg/peak | 96.9 / 141.3 | 102.8 / 146.1 |

Generation is the mean of per-request server-side rates over requests that
generated at least 20 tokens (token-weighted: 18.94 and 18.51 tok/s). The TTFT
proxy is server-side prompt-evaluation time per request; the mean is pulled up
by a few cold ~19K-token prefills at roughly 140 tok/s, which is why the median
is about 7 s. Neither is measured the same way as the standardized 18,982-token
probe used for the Q4_K_L figures below.

#### Per-task results

| Task | Moderate ok% | Strong ok% | Moderate mean wall | Strong mean wall |
| --- | ---: | ---: | ---: | ---: |
| `err_python_env` | 100% | 100% | 212 s | 211 s |
| `err_replay_patch` | 100% | 100% | 199 s | 187 s |
| `err_ambiguous_edit` | 100% | 100% | 213 s | 215 s |
| `err_case_search` | 100% | 100% | 240 s | 277 s |
| `err_hidden_search` | 100% | 100% | 246 s | 207 s |
| `err_big_output` | 100% | 100% | 228 s | 187 s |
| `err_multi_dir` | 100% | 100% | 204 s | 202 s |
| `err_inline_script` | 33% | 100% | 516 s | 250 s |
| `err_big_file_read` | 0% | 0% | 600 s | 600 s |

Seven of nine tasks passed 3/3 on both profiles. `err_inline_script` scored 33%
on Moderate and 100% on Strong, but the two Moderate misses were 600-second
timeouts and Strong had none; at three repetitions that is timeout variance, not
a profile difference. `err_big_file_read` scored 0/3 on both profiles, and all
six trials hit the 600-second timeout rather than returning a wrong answer. The
same task also timed out (at 300 s) on an earlier Qwen3.8-27B UD-Q4_K_XL run, and
it passes on Qwen3.6 Q4_K_L, whose one failure in its 26/27 battery is
`err_hidden_search`. It is a reproducible model/task limitation, not a hardware
one.

The 3-trial no-tool arithmetic eval scored 0/3 on both profiles under the
exact-match grader. In all six trials the final line was `RESULT=323`, but
reasoning text preceded it in the captured output, so the strict match failed.
That is output-format leakage, not an arithmetic error; Q4_K_L scored 3/3.

#### Thermals and the Bowie/Crockett asymmetry

Bowie ran much hotter than it did with Q4_K_L. On Strong its peaks (86.2°C CPU,
88.0°C GPU) exceeded the 80°C limits configured for both, against 74.4°C and
75.0°C for Q4_K_L on the same profile in the 2026-09-16 bake-off; on Moderate
they were 82.1°C and 83.0°C against 72.0°C and 73.0°C. Crockett stayed at
72.9–75°C. The telemetry recorded temperatures and PPT but not GPU clocks, so
whether the overshoot reduced clocks, and therefore throughput, was not
observed.

The cause is structural asymmetry in the llama.cpp RPC split, not a
load-balancing failure:

- Automatic split and an explicit `--tensor-split 1,1` produced identical
  startup logs: Crockett (RPC0) 5,266.00 MiB and Bowie (Vulkan0) 6,739.89 MiB of
  weights (43.9% / 56.1%), a 521 MiB host buffer, identical projected use
  (7,725 / 9,183 MiB), and identical measured post-load allocation
  (7.60 / 10.12 GiB). Automatic split divides by free memory, and the two
  devices reported nearly equal free memory (13,573 / 13,632 MiB), so `1,1`
  reproduces it. KV buffers were equal at 1,912.5 MiB per device. As in the
  split experiment above, llama.cpp lists the RPC device first for
  `--tensor-split`. The per-device lines print only at `-lv 4`; the default
  verbosity omits them.
- The remaining 1.44 GiB (1,474 MiB) of weight difference sits on Bowie. The log
  shows the MTP draft context (450 MiB KV, 184.5 MiB compute) on Vulkan0 only;
  attributing the rest to the output layer is an inference from llama.cpp
  placing it on the last device, because the log does not print per-tensor
  placement.
- Crockett was not idle. Its PPT averaged 96.9 W (Moderate) and 102.8 W (Strong)
  against roughly 60 to 65 W idle, comparable to Bowie's 101.7 and 107.0 W.
- The asymmetry also exists in production Q4_K_L, smaller: live allocation was
  12.30 GiB on Bowie and 11.03 GiB on Crockett (52.7% / 47.3%), against 57.1% /
  42.9% for this candidate. No `--tensor-split` is configured anywhere
  (`llama_tensor_split` is empty and none is in the launch command), so a stale
  split value is not a factor.

#### Comparison with production Q4_K_L

| Metric | UD-Q3_K_XL (Moderate / Strong) | Q4_K_L (Moderate / Strong) |
| --- | ---: | ---: |
| GGUF size | 13.15 GB | 22.66 GB |
| Battery pass rate | 22/27 / 24/27 (600 s, no early stop) | 26/27 at promotion |
| Generation, tok/s | 19.1 / 19.5 (agent-trial mean, MTP) | 56.2 / 57.7 (standardized probe) |
| Battery mean wall | 295 s / 260 s | 97.4 s / 105.3 s |
| Bowie CPU / GPU peak °C | 82.1 / 83.0 and 86.2 / 88.0 | 72.0 / 73.0 and 74.4 / 75.0 |
| Crockett CPU / GPU peak °C | 72.9 / 74.0 and 73.6 / 75.0 | 68.0 / 69.0 and 70.0 / 72.0 |
| Combined free Vulkan after load | about 8.9 GiB | about 3.3 GiB |

Q4_K_L figures come from [the performance-profile bake-off](qwen36-performance-profiles.md)
and [the Q4_K_L promotion record](qwen36-q4kl-promotion.md); the free-memory
figures were measured the same way, idle after load, on 2026-09-18.

The Qwen3.8-27B results so far, all on the same hardware:

| Quant | Date | Generation | Hermes battery |
| --- | --- | --- | --- |
| Q4_K_M | 2026-08-23 | 14.0 tok/s, TTFT 107 s (standardized probe, 65,536 context, no MTP) | 2/21 (9.5%), early stop for excessive latency |
| UD-Q4_K_XL | 2026-09-07 | Not measured | Early stop at 13 trials: 11/13 passed, two 300 s timeouts (`err_big_file_read`, `err_case_search`), 222 s mean |
| UD-Q3_K_XL, MTP | 2026-09-18 | about 19 tok/s (agent-trial mean) | 22/27 and 24/27 at 600 s, no early stop |

The Q4_K_XL row is from a private run that had no tracked write-up; it is not
the Qwen3.6-35B-A3B UD-Q4_K_XL follow-up above, which is a different model.

**Verdict:** UD-Q3_K_XL loads cleanly at full context and works with MTP, but it
is **not recommended** for production. The smaller quant does provide far more
Vulkan headroom (about 8.9 GiB combined free against about 3.3 GiB for Q4_K_L),
so memory pressure is not what rules it out. Generation is about a third of
production Q4_K_L's, the battery ran roughly 2.5 to 3 times slower,
`err_big_file_read` failed on both profiles, and Bowie exceeded its 80°C
limits. The Qwen3.8-27B family has now shown poor throughput or latency at
Q4_K_M, UD-Q4_K_XL, and UD-Q3_K_XL, so quant size was not the bottleneck. The
model is a 27B-parameter model with no active-parameter suffix, unlike the
A3B-designated production model, which likely explains the gap, but that was not
isolated experimentally. This is a third data point for the family, not a new
avenue to explore: further Qwen3.8-27B quants are not worth testing for Hermes
on this hardware. Production stays on Qwen3.6-35B-A3B Q4_K_L with the Moderate
profile.

Comparability caveats: this run used a 600-second timeout with early-stop
disabled while the other bake-offs used 180 or 300 seconds with early stop, so
pass rates are not directly comparable; generation and TTFT were measured per
request during the agent battery rather than with the standardized probe; and
each task has only three repetitions. Raw logs (including the `-lv 4` startup
logs), telemetry CSVs, and driver scripts are retained under the gitignored
`benchmarks/private/qwen38-q3kxl-candidate-2026-09-18/`.

#### One-shot creative generation: driving game (2026-09-18)

A separate, qualitative test asked both models for the same single-file 2D
driving game. Production produced one in 102.8 s (7,895 tokens, 77.9 tok/s). The
candidate needed 48,930 tokens and 53 minutes (15.4 tok/s), and at a 24,000-token
cap it returned an empty file because it spent the whole budget on reasoning.
Its game was richer, at roughly 31 times the wall time; that is one sample per
model, judged from screenshots and scripted play. For latency-sensitive serving
the answer stays no, while offline or batch use could differ. The full write-up,
the exact prompt, and both generated games are in
[One-shot creative generation: driving game](creative-generation-driving-game.md).

### Gemma 4 bake-off (2026-09-19)

This was a standalone candidate test, not a production swap. It ran unattended
overnight: for each model, production `llama-server` was stopped on Bowie, the
candidate was loaded on a separate port against Crockett's existing RPC worker,
the full battery ran, and the candidate and its GGUF were removed before the
next one. Production `Qwen3.6-35B-A3B-Q4_K_L` and the Moderate profile were
restored and verified afterwards. No Ansible, inventory, or persistent
configuration was changed.

The plan was five models. Two Gemma 4 models completed and are the subject of
this section. `gpt-oss-20b` was already covered by the original bake-off, so its
overnight run is included only as a same-conditions control. Granite 4.2 30B did
not complete (see [Not tested](#not-tested-and-why)) and Gemma 3 27B was not
run; neither was pursued afterwards.

| Model | Hugging Face repository and file | Size (bytes) | Type |
| --- | --- | ---: | --- |
| Gemma 4 26B-A4B | `bartowski/google_gemma-4-26B-A4B-it-GGUF`, `google_gemma-4-26B-A4B-it-Q4_K_M.gguf` | 17,035,039,872 | MoE, 26B total / 4B active |
| Gemma 4 31B | `bartowski/google_gemma-4-31B-it-GGUF`, `google_gemma-4-31B-it-Q4_K_M.gguf` | 19,598,489,952 | Dense |
| `gpt-oss-20b` (control) | `ggml-org/gpt-oss-20b-GGUF`, `gpt-oss-20b-MXFP4.gguf` | 12,109,566,624 | MoE, native MXFP4 |

Each download's size matched the Hugging Face file listing to the byte and its
GGUF header was checked; SHA-256 was not recorded. Only standard Q4_K_M (or
native MXFP4) quants from established quantizers were used, with no custom
KV-cache schemes.

#### Protocol

All models ran on the same pinned llama.cpp revision as the Qwen3.8 test, with
the production-equivalent flags and the Moderate/1750 profile confirmed on both
nodes before each load. No speculative decoding was enabled for any of them, so
the comparison is like for like:

```text
--rpc 10.250.0.2:50052 --gpu-layers all --split-mode layer
--ctx-size 115000 --parallel 1 --cache-type-k q8_0 --cache-type-v q8_0 --jinja
--cache-ram 0 --no-cache-idle-slots
```

`--cache-ram 0 --no-cache-idle-slots` were included from the start, and no
coordinator OOM kills occurred during a battery. Neither Gemma startup log
listed unused `nextn` tensors, so neither model has an MTP block to enable.

Each ran the full 27-trial error-task battery (nine tasks, three repetitions)
with a 600-second per-trial timeout and the harness early-stop rule disabled, as
in the Qwen3.8 test, so a timeout was recorded as a failed trial and the battery
continued.

**Gemma 4 31B did not fit at 115,000 context.** The load was killed by the
kernel OOM killer on Bowie while amdgpu logged "Not enough memory for command
submission". The single permitted retry at 65,536 context loaded and ran the
whole battery, so every 31B figure below is at 65,536 context, not the 115,000
that production uses. The 26B-A4B and the control loaded at the full 115,000
(slot `n_ctx` 115,200).

#### Results

| Metric | Production Q4_K_L (reference) | Gemma 4 26B-A4B | Gemma 4 31B | `gpt-oss-20b` (control) |
| --- | ---: | ---: | ---: | ---: |
| GGUF size | 22.66 GB | 17.04 GB | 19.60 GB | 12.11 GB |
| Context | 115,000 | 115,000 | 65,536 | 115,000 |
| Battery pass rate | 26/27 at promotion | 22/27 (81%) | 22/27 (81%) | 23/27 (85%) |
| Trials that hit the 600 s timeout | n/a | 0 | 1 | 0 |
| Generation, tok/s | 56.2 (standardized probe) | 39.1 | 12.2 | 65.4 |
| Prompt processing, tok/s | 458.2 (standardized probe) | 476.6 | 118.2 | 414.7 |
| TTFT proxy, mean / median | 41.5 s (standardized probe) | 17.9 / 3.9 s | 58.6 / 13.0 s | 15.8 / 1.2 s |
| Battery mean trial wall | 97.4 s | 137 s | 306 s | 92 s |
| Battery total wall | n/a | 65 min | 146 min | 43 min |
| Bowie CPU °C avg/peak | 67.9 / 72.0 | 69.9 / 77.6 | 76.8 / 85.0 | 71.5 / 77.0 |
| Bowie GPU °C avg/peak | 63.9 / 73.0 | 65.4 / 78.0 | 72.9 / 86.0 | 67.4 / 78.0 |
| Bowie PPT W avg/peak | 90.6 / 144.4 | 89.9 / 139.6 | 102.5 / 144.8 | 93.5 / 129.7 |
| Crockett CPU °C avg/peak | 64.1 / 68.0 | 64.0 / 69.1 | 68.4 / 74.1 | 66.8 / 71.0 |
| Crockett GPU °C avg/peak | 60.2 / 69.0 | 60.3 / 70.0 | 65.1 / 75.0 | 63.4 / 72.0 |
| Crockett PPT W avg/peak | 87.0 / 145.1 | 86.5 / 135.6 | 103.3 / 145.3 | 94.9 / 127.2 |
| Free Vulkan after load, Bowie / Crockett | about 3.3 GiB combined | 4.23 / 3.94 GiB | 0.86 / 2.47 GiB | 6.54 / 6.59 GiB |

Generation and prompt rates are token-weighted over every request in the
battery (per-request means were 38.6, 12.4, and 64.6 tok/s). The TTFT proxy is
server-side prompt-evaluation time plus the first generated token, per request;
the means are pulled up by a few cold ~19K-token prefills, which is why the
medians are so much lower. The candidate columns were therefore not measured the
same way as the standardized 18,982-token probe behind the production
reference, and the two should not be compared as exact figures. Production's
reference figures come from [the performance-profile bake-off](qwen36-performance-profiles.md)
(Moderate profile) and [the Q4_K_L promotion record](qwen36-q4kl-promotion.md);
its pass rate is the promotion-time result under the earlier timeout policy.
Free Vulkan is the GTT heap (12.83 GiB per node) sampled right after load,
before any request; the 31B's figure is at the smaller 65,536 context.

#### Per-task results

| Task | Gemma 4 26B-A4B | Gemma 4 31B | `gpt-oss-20b` (control) |
| --- | ---: | ---: | ---: |
| `err_python_env` | 100% (144 s) | 100% (230 s) | 100% (59 s) |
| `err_replay_patch` | 100% (124 s) | 100% (254 s) | 100% (45 s) |
| `err_ambiguous_edit` | 100% (160 s) | 100% (287 s) | 100% (59 s) |
| `err_case_search` | 100% (93 s) | 100% (237 s) | 100% (44 s) |
| `err_hidden_search` | 0% (65 s) | 0% (211 s) | 0% (43 s) |
| `err_big_output` | 100% (82 s) | 100% (261 s) | 100% (93 s) |
| `err_multi_dir` | 33% (126 s) | 100% (486 s) | 100% (50 s) |
| `err_inline_script` | 100% (116 s) | 100% (216 s) | 100% (43 s) |
| `err_big_file_read` | 100% (327 s) | 33% (572 s) | 67% (391 s) |

Cells are ok% with the mean wall time per trial in parentheses.

`err_hidden_search` scored 0/3 on all three models, including the control, and
was the only task with no 3/3 anywhere. It is the task Qwen3.6 Q4_K_L's single
failure fell on at promotion, while Qwen3.8-27B UD-Q3_K_XL passed it 3/3 on both
profiles, so it is not uniformly hard; why these three models miss it was not
investigated. The other misses are isolated: `err_multi_dir` for the 26B-A4B,
and `err_big_file_read` for the 31B (its one timeout was
`err_big_file_read` repetition 0, and its mean of 572 s shows the other trials
ran close to the limit) and the control. At three repetitions per task, one or
two trials is inside the noise, so 22/27 versus 23/27 is not a ranking between
these models.

The 3-trial no-tool arithmetic eval scored 0/3 for all three models under the
exact-match grader. In all nine trials the final answer was `RESULT=323`, but
reasoning text or a scanner warning line was captured before it, so the strict
match failed. As with Qwen3.8, that is output-format leakage, not an arithmetic
error.

#### Thermals and memory

The dense 31B was the hardest run on the hardware: Bowie peaked at 85.0°C CPU and
86.0°C GPU, past the 80°C limits configured for both, with about 102 W average
PPT sustained across a 146-minute battery, and it left only 0.86 GiB free
Vulkan memory on Bowie even at the reduced context. The 26B-A4B stayed under 80°C
(77.6°C CPU, 78.0°C GPU) but still ran about 5°C hotter than production's
72.0 / 73.0°C on the same profile. Crockett stayed at or below 75°C throughout,
and Bowie ran hotter than Crockett for every model, consistent with the
structural split asymmetry described in
[the Qwen3.8 thermal analysis](#thermals-and-the-bowiecrockett-asymmetry). As
before, the telemetry recorded temperatures and PPT but not GPU clocks, so any
throttling effect on throughput was not observed.

#### Not tested and why

- **Granite 4.2 30B Q4_K_M:** Crockett froze hard about 90 seconds into the
  115,000-context load, during the RPC tensor upload. Nothing was logged before
  the freeze (no OOM, kernel panic, or amdgpu error); the last kernel message was
  a clocksource watchdog timeout. The node stayed unreachable until it was
  cold-restarted almost six hours later. The cause was not verified,
  and the test was not retried, so there is no Granite result.
- **Gemma 3 27B Q4_K_M:** not run; the harness skipped it while Crockett was
  down, and it was not pursued afterwards.

#### Comparison and verdict

**Neither Gemma 4 model is recommended for production.** Both scored 22/27,
below Qwen3.6 Q4_K_L's 26/27 at promotion (under a different timeout policy, so
not an exact comparison), and neither beats `gpt-oss-20b`, which ran under the
same conditions here and remains the fast/light alternative: it was faster
(65.4 against 39.1 tok/s), quicker per trial (92 against 137 s), and scored
23/27.

- **26B-A4B** is the more interesting of the two. It is a MoE like production,
  completed with no timeouts at the full 115,000 context, and stayed under the
  80°C limits, but at 39.1 tok/s it generates well below production's standardized
  56.2 tok/s and 137 s per trial is about 40% longer than production's 97.4 s.
  It does not displace production or `gpt-oss-20b`.
- **31B** is not viable on this hardware: it could not hold 115,000 context, it
  generated about 12 tok/s, a single trial averaged 306 s, and it ran Bowie past
  its thermal limits with almost no memory headroom.

The two Gemma 4 models tied on accuracy, but the dense 31B was about 3.2 times
slower in generation (12.2 against 39.1 tok/s) and 2.2 times slower per trial
than the 4B-active MoE. That points the same way as Qwen3.8-27B, a dense model, against the A3B production
model on this hardware, but each pair differs in more than parameter count, so it is
suggestive rather than isolated experimentally. Production stays on Qwen3.6-35B-A3B
Q4_K_L with the Moderate profile.

Comparability caveats: this used a 600-second timeout with early stop disabled
while earlier bake-offs used 180 or 300 seconds with early stop, so pass rates
are not directly comparable; generation and TTFT were measured per request during
the agent battery rather than with the standardized probe; the 31B ran at a
smaller context than the other models; and each task has only three repetitions
of one run per model. The `gpt-oss-20b` control scored 23/27 here against 21/27
in the original bake-off; that difference reflects a different timeout policy,
context size, and date, and the original figure remains its official entry.

#### Operational notes

- **Stop production before downloading on Bowie.** With production resident the
  node has about 0.3 GB of free RAM, and the kernel OOM-killed the Hugging Face
  downloader twice. Stopping production first and using the classic HTTP
  download path (`HF_HUB_DISABLE_XET=1`) avoided it.
- **Crockett's RPC cache grows with every model tried.** The worker runs with
  `--cache`, and each candidate adds several GB of tensor files to Crockett's
  disk; the cache had reached 156 GB of its 236 GB before the daily cleanup timer
  removed the files older than one day. Files created by each test were removed
  afterwards by modification time.
- **After a cold restart Crockett's GPU enumerated as `card1`, not `card0`.** Any
  check that reads `/sys/class/drm/card0/...` (for example to confirm the sclk
  profile) fails there until it uses a glob.
- **Production was stopped for about ten and a half hours**, roughly six of
  them waiting for Crockett to recover, and was then restored and
  verified: `llama-server` active on 8080, `/v1/models` returning 200 at
  n_ctx 115,200, a live completion succeeding, and Moderate confirmed on both
  nodes.

The sections below preserve the earlier Qwen3-Coder/Hermes-4 fit campaign and
post-cooling history. They are historical evidence, not the current model
recommendation.

Tested 2026-08-22 on the tagged `baseline-moderate-40cu` cluster. No clocks,
voltage points, CU routing, kernel, Mesa, firmware, CPU cores, network settings,
or memory settings were changed. The temporary llama-server overrides were
removed after testing and the Ansible-managed 8K router service was restored.

## Baseline TTM note

The live kernel command line on both boards contained
`ttm.pages_limit=3014656 ttm.page_pool_size=3014656` throughout this campaign.
The settings were originally applied manually on August 22, 2026 and were left
untouched during model testing. They are now explicitly adopted by the
`bc250_hardware` role as the reproducible 11.5 GiB GTT baseline; the benchmark
results therefore describe the managed 12 GiB Vulkan-visible configuration.

## Method

Viable models used llama.cpp commit
`d775b8967a46d8beb110d444aa3b8938179e0dd8`, Bowie local Vulkan, Crockett RPC
Vulkan, 65,536 tokens, Q8_0 K/V cache, one slot, all layers offloaded, and
automatic layer split. A fixed 18,976-token prompt measured prompt processing;
a separate fixed prompt generated 1,024 tokens. Identical synthetic agent tasks
covered Ansible, systemd, constrained shell, ordered infrastructure diagnosis,
and schema-only tool output. Telemetry sampled both boards once per second.

The long prompt is intentionally more thermally demanding than the earlier
short-prompt known-good benchmark. Reported device allocation is GTT plus VRAM
from DRM counters. Weight allocation is derived from the RPC tensor cache on
Crockett and the remaining GGUF bytes on Bowie.

## Artifacts

| Model | Repository and filename | Bytes | SHA-256 |
| --- | --- | ---: | --- |
| Qwen3-Coder-30B-A3B-Instruct Q4_K_M | `unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF`, `Qwen3-Coder-30B-A3B-Instruct-Q4_K_M.gguf` | 18,556,689,568 | `fadc3e5f8d42bf7e894a785b05082e47daee4df26680389817e2093056f088ad` |
| Hermes-4-14B Q6_K | `bartowski/NousResearch_Hermes-4-14B-GGUF`, `NousResearch_Hermes-4-14B-Q6_K.gguf` | 12,121,937,856 | `f7156c9ad8e9a0a4e01792714edb81424882507f7929a4d00f25689ccdb29552` |
| Hermes-4-14B Q8_0 | same repository, `NousResearch_Hermes-4-14B-Q8_0.gguf` | 15,698,534,336 remote | Not downloaded; model context cap already disqualified it |
| Qwen3-Coder-Next UD-IQ1_S | `unsloth/Qwen3-Coder-Next-GGUF`, `Qwen3-Coder-Next-UD-IQ1_S.gguf` | 21,508,749,344 | `98c98964d9dbc8aaba3153abe2aca35f6202a867e9e3ba2568b982621815d4ce` |

The incomplete Q8_0 transfer was removed after disqualification. Model files
are deployment artifacts and are not part of this repository.

## Results

| Candidate | Loaded | True 64K | Load | Weight allocation Bowie / Crockett | Total Vulkan allocation Bowie / Crockett | Free Vulkan Bowie / Crockett | Prompt tok/s | Generation tok/s | Stream first event | Peak temperature Bowie / Crockett | Peak PPT Bowie / Crockett | Result |
| --- | --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| Qwen3-Coder-30B Q4_K_M, auto | Yes | **Yes** | 104 s | 8.79 / 8.49 GiB | 10.20 / 10.62 GiB | 1,837 / 1,409 MiB | **377.96** | **77.89** | 3.34 s | 67 / **90°C** | 135.31 / 131.09 W | Best overall, but Crockett throttled |
| Hermes-4-14B Q6_K | Yes | **No** | 74 s | Not retained | 8.71 / 8.31 GiB | 3,366 / 3,781 MiB | — | — | — | — | — | Server capped the slot to 40,960; not viable |
| Hermes-4-14B Q8_0 | No | **No** | — | — | — | — | — | — | — | — | — | Not attempted after the shared 40,960 model cap was proven |
| Qwen3-Coder-Next UD-IQ1_S, auto | Yes | **Yes** | 133 s | 10.57 / 9.46 GiB | 10.70 / 10.54 GiB | 1,329 / 1,493 MiB | 348.47 | 56.50 | 3.68 s | 66 / **88°C** | 141.13 / 135.03 W | Viable but tight, slow, aggressively quantized, and throttled |

Host `MemAvailable` immediately after load was 3.86/3.47 GiB for Qwen3-Coder
30B (Bowie/Crockett), 5.18/5.85 GiB for Hermes Q6, and 3.48/3.79 GiB for
Qwen3-Coder-Next. Minimum observed Bowie host headroom during the tests was
about 2.73 GiB for Qwen3-Coder-30B auto and 1.96 GiB for Coder-Next.

Both Qwen runs transferred substantial data over the backend, proving RPC
participation. Qwen3-Coder-30B telemetry observed approximately 801 MB received
and 596 MB transmitted on Bowie; Qwen3-Coder-Next observed approximately
1.04 GB received and 1.28 GB transmitted. Bowie stayed at 1750 MHz. Crockett
stepped down to 1550–1750 MHz for Qwen3-Coder-30B and 1600–1750 MHz for
Coder-Next, confirming thermal throttling.

No GPU reset, VM/page fault, ring timeout, allocation failure, new MCE, or other
AMDGPU/kernel hardware error was found. Both governor services and the
coordinator/RPC services remained active.

## Post-cooling thermal qualification

The sustained Qwen3-Coder-30B automatic-split test was repeated on 2026-08-23
after Crockett received the physical cooling changes described in
[Hardware preparation](hardware-prep.md). The software baseline was unchanged:
llama.cpp commit `d775b8967a46d8beb110d444aa3b8938179e0dd8`, the same verified
Q4_K_M GGUF, 65,536 context, Q8_0 K/V, one slot, all layers offloaded, and no
manual tensor split. Telemetry sampled both nodes once per second.

The original private prompt artifact was not retained. The rerun therefore
used a deterministic synthetic payload with the same 18,976-token prompt count,
followed immediately by a separate 1,024-token generation request. This keeps
the thermal load, allocation, context, and generation length comparable, but
the prompt-throughput change is not a controlled model-quality comparison.

| Metric | Original | Post-cooling | Change |
| --- | ---: | ---: | ---: |
| Bowie peak temperature | 67°C | 61°C | -6°C |
| Crockett peak temperature | 90°C | **66°C** | **-24°C** |
| Bowie peak PPT | 135.31 W | 137.02 W | +1.71 W |
| Crockett peak PPT | 131.09 W | 128.74 W | -2.35 W |
| Prompt processing | 377.96 tok/s | 420.25 tok/s | +11.2%[^prompt-comparison] |
| Generation | 77.89 tok/s | 77.10 tok/s | -1.0% |
| Stream first event | 3.34 s | 3.71 s | +0.37 s |
| Vulkan allocation, Bowie / Crockett | 10.20 / 10.62 GiB | 10.23 / 10.62 GiB | Comparable |
| Free Vulkan, Bowie / Crockett | 1,837 / 1,409 MiB | 1,808 / 1,412 MiB | Comparable |
| Crockett thermal throttling | Yes | **No** | Eliminated |
| GPU resets, faults, RPC errors | None | None | No change |

[^prompt-comparison]: The post-cooling payload preserved the token count but
    not the unavailable original prompt contents, so the apparent prompt-rate
    improvement is not attributed to cooling. Generation remained within
    normal run-to-run variance.

Crockett averaged 62.5°C during the late prompt heat-soak window and remained
at 1750 MHz in every one-second workload sample. Bowie averaged 58.9°C over the
same window. No thermal event, severe clock drop, GPU reset, VM/page fault,
ring timeout, RPC disconnect, or backend timeout occurred. The five-degree
peak difference is sufficiently balanced for unattended Hermes Agent use under
this qualified workload.

## Split experiment

llama.cpp lists the RPC device before Bowie's local device for `--tensor-split`.
The initially attempted `55,45` therefore favored Crockett and left only 552 MiB
free; it was stopped before stress. Corrected `45,55` favored Bowie and left
980 MiB free on Bowie and 2,265 MiB on Crockett. It improved the 30B prompt rate
to 411.50 tok/s and generation to 78.72 tok/s, but Crockett still reached 91°C.
Because it reduces Bowie's safety margin without fixing thermals, automatic
split remains the recommendation.

## Agent-task observations

Qwen3-Coder-30B correctly diagnosed the `noexec` service failure, produced valid
schema-only tool JSON, maintained the no-default-route constraint, and generally
gave useful ordered diagnostics. It did not obey the Ansible task's exact JSON
format (Markdown fences), did not render the requested corrected task, added
`-daystart` to the constrained `find` command, and exhausted the 600-token limit
on the broad infrastructure plan.

Qwen3-Coder-Next produced the exact constrained `find` command and exact tool
JSON, correctly diagnosed `noexec`, and gave the stronger Ansible task-shaped
answer. It also used Markdown fences despite the exact-JSON request, mentioned
an unsupported `checksum_algorithm` option in commentary, and exhausted the
infrastructure-plan token limit. Its small-sample reasoning quality was
competitive and occasionally better, but UD-IQ1_S did not provide a reliability
advantage sufficient to offset its lower speed, tighter memory, and extreme
quantization.

Hermes-4 quality testing was not used for ranking because llama-server capped
the requested slot to the model's native 40,960-token context. A 40K quality
result would not establish suitability for the required 64K Hermes workload.

## Historical preliminary ranking

1. **Best overall Hermes backend:** Qwen3-Coder-30B-A3B-Instruct Q4_K_M with
   65,536 context, Q8_0 KV, and automatic split.
2. **Best performance:** Qwen3-Coder-30B-A3B-Instruct Q4_K_M.
3. **Best observed reasoning/tool-use detail:** Qwen3-Coder-Next narrowly on a
   few constrained tasks, but not enough to overcome its operational costs.
4. **Best memory efficiency:** Hermes-4-14B Q6_K, but only for workloads at or
   below 40,960 tokens; it is not a valid backend for this requirement.
5. **Best fallback at required context:** Qwen3-Coder-Next UD-IQ1_S, with the
   important thermal and memory-pressure warnings above.

At this preliminary stage, the result supported
**Qwen3-Coder-30B-A3B-Instruct Q4_K_M** as the default model selection, with
65,536 context, Q8_0 K/V cache, and automatic split. The original bake-off
established the model recommendation but exposed Crockett's cooling failure.
The post-cooling rerun eliminated that throttling without changing clocks,
voltage, power, thermal limits, or the software profile, so this configuration
is now sustained-thermally qualified for unattended Hermes workloads on the
documented hardware.

Hermes-4 Q6_K and Q8_0 are **NOT VIABLE FOR HERMES ON CURRENT HARDWARE AND
REQUIRED CONTEXT** because the server caps them to 40,960 tokens. Qwen3-Coder-
Next technically works at 64K but should not be the default because of its
aggressive quantization, smaller margins, slower generation, and throttling.
