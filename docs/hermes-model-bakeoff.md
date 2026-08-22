# Hermes model bake-off

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

## Ranking and recommendation

1. **Best overall Hermes backend:** Qwen3-Coder-30B-A3B-Instruct Q4_K_M with
   65,536 context, Q8_0 KV, and automatic split.
2. **Best performance:** Qwen3-Coder-30B-A3B-Instruct Q4_K_M.
3. **Best observed reasoning/tool-use detail:** Qwen3-Coder-Next narrowly on a
   few constrained tasks, but not enough to overcome its operational costs.
4. **Best memory efficiency:** Hermes-4-14B Q6_K, but only for workloads at or
   below 40,960 tokens; it is not a valid backend for this requirement.
5. **Best fallback at required context:** Qwen3-Coder-Next UD-IQ1_S, with the
   important thermal and memory-pressure warnings above.

Use **Qwen3-Coder-30B-A3B-Instruct Q4_K_M** as the default model selection, with
65,536 context, Q8_0 K/V cache, and automatic split. However, neither true-64K
candidate passed the long-prompt run without Crockett exceeding the 80°C target
and throttling. Treat the selection as functionally correct but not yet
thermally qualified for unattended sustained Hermes workloads. Thermal-interface
replacement, a rear heatsink, and increased physical separation from Bowie are
planned, followed by a repeat of the same benchmark. Do not raise clocks,
voltage, power, or thermal limits to mask this pending cooling work.

Hermes-4 Q6_K and Q8_0 are **NOT VIABLE FOR HERMES ON CURRENT HARDWARE AND
REQUIRED CONTEXT** because the server caps them to 40,960 tokens. Qwen3-Coder-
Next technically works at 64K but should not be the default because of its
aggressive quantization, smaller margins, slower generation, and throttling.
