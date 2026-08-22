# Performance tuning and benchmarks

Keep all tuning per board. Establish stock Fedora results before changing CU
routing, clocks, voltage, memory split, or kernel behavior.

The validated private deployment profile is `moderate`, managed independently
of CU routing through `bc250_performance_profile`,
`bc250_cpu_governor_enabled`, and `bc250_gpu_governor_enabled`. The public
example inventory leaves both governors disabled and selects `stock`. Do not
enable or promote a profile without a separate per-board stability and inference
benchmark.

## TTM inference aperture

The validated reproducible baseline enables `bc250_ttm_enabled` with
`bc250_ttm_pages_limit: 3014656` and `bc250_ttm_page_pool_size: 3014656`.
These are fixed, validated memory-capacity settings rather than part of a clock
profile. They provide an 11.5 GiB GTT ceiling and approximately 12 GiB total
Vulkan-visible capacity when combined with the BC250's 512 MiB hardware VRAM.
The GTT backing is shared system RAM, not dedicated VRAM, so memory headroom
must still be measured for every model and context.

The Fedora default would be approximately 7.9 GiB Vulkan-visible capacity per
node on these systems. Do not raise the managed limits to fit a model. Changing
or disabling them requires a separate memory-pressure, model-load, and kernel-
fault qualification.

The public example keeps `bc250_ttm_enabled: false`. Enabling it adopts the
documented values; it must not be treated as a harmless generic default.

## CU qualification

Detect the factory topology, then add one WGP pair at a time. For each candidate
run Vulkan correctness, a short llama.cpp comparison, sustained inference,
thermal monitoring, and kernel-log checks. A board may stabilize below 40 CUs;
Bowie and Crockett need not use identical tables. Boot persistence is a manual
promotion after repeated warm and cold validation.

## Cluster benchmark matrix

Use the same pinned build, GGUF, prompt, context, seed, batch sizes, and sampling
parameters for:

1. Bowie local Vulkan only.
2. Crockett local Vulkan only.
3. Two-node RPC with automatic `layer` allocation.
4. Two-node RPC with measured explicit tensor proportions.
5. Two-node `row` split.
6. Experimental `tensor` split only after the stable cases.

Test a small model, a model near one node's practical memory limit, a model that
needs combined memory, and a long-context workload. Record cold/warm load time,
prompt-processing tokens/s, generation tokens/s, time to first token, request
latency, backend throughput, memory, clocks, temperatures, throttling, and GPU
faults. Preserve raw private results under the gitignored `benchmarks/private/`.

## Qwen3-Coder-Next historical fit plan

The [fit qualification plan](qwen3-coder-next-fit.md) records the analysis that
preceded the completed bake-off. `Qwen3-Coder-Next-UD-IQ1_S` subsequently ran at
65,536 tokens but was not preferred over Qwen3-Coder-30B Q4_K_M. GGUF size alone
remains an invalid fit calculation on this integrated-GPU architecture because
Vulkan and host processes contend for the same physical RAM.
