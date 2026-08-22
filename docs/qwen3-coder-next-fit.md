# Qwen3-Coder-Next pre-bakeoff fit qualification

> Historical planning document. The subsequent 64K campaign is complete; see
> [Hermes model bake-off](hermes-model-bakeoff.md). UD-IQ1_S loaded and ran at
> 65,536 tokens but was not selected as the preferred Hermes backend.

`Qwen/Qwen3-Coder-Next` was evaluated as a coding-agent workload outside the
normal cluster deployment. The procedure below records the pre-test methodology
and should still be followed when repeating fit qualification on other boards.

The upstream model has 80 billion total parameters, approximately 3 billion
active parameters per token, 48 layers, and a native 262,144-token context.
The initial cluster test is deliberately limited to 8,192 tokens. See the
[Qwen model card](https://huggingface.co/Qwen/Qwen3-Coder-Next) and the
[Unsloth GGUF repository](https://huggingface.co/unsloth/Qwen3-Coder-Next-GGUF).

## Candidate order

The following sizes were planning estimates, not allocation estimates. Confirm
the exact remote byte count, revision, and SHA-256 before any repeat download
because publisher artifacts may be replaced.

| Order | Quantization | Approximate GGUF size | Decision |
| --- | --- | ---: | --- |
| 1 | `UD-IQ2_XXS` | 23.3 GB | Preferred initial quality target if measured headroom is sufficient |
| 2 | `UD-IQ1_S` | 21.5 GB | First fallback |
| 3 | `UD-TQ1_0` | 18.9 GB | Second fallback |
| Exclude initially | `Q2_K` | 29.2 GB | Evaluate only if measurements unexpectedly prove sufficient headroom |
| Exclude initially | Q3/Q4 variants | 28.5 GB and higher | Not practical without contrary measured evidence |

Do not select a quantization from GGUF size alone. Vulkan allocations can use
shared host memory on these integrated GPUs, so host and device figures are not
independent pools that may be added twice.

## Baseline measurements

After `site.yml` and `playbooks/90-validation.yml` pass, collect measurements
with services in their normal state:

```bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/91-qwen3-coder-next-fit.yml
```

For each node, retain:

- `MemTotal`, `MemAvailable`, swap use, and the normal idle service footprint.
- The capacity/free-memory report from `llama-cli --list-devices`.
- Vulkan heap sizes, budgets, and current usage from `vulkaninfo`.
- Current TTM limits and actual VRAM/GTT use, recorded separately from the
  Ansible output when the kernel exposes those counters.

Use the lower repeatable free-memory value after several clean boots, not the
single best result. Reserve operating-system and service headroom before
considering model weights.

## Runtime and context measurements

Once a candidate GGUF is manually staged on Bowie, first measure a local load
on each node and then the distributed load. Capture resident host memory,
Vulkan heap usage, llama.cpp logs, and kernel GPU faults immediately before and
after each load. The measured difference above the GGUF weight allocation is
the runtime overhead; do not substitute a fixed percentage.

Run the distributed candidate at `--ctx-size 8192` with the configured
quantized KV types first. Record the actual KV allocation reported by llama.cpp.
Repeat at 16,384 only after the 8K load and sustained inference are stable. Use
the observed 8K-to-16K delta to validate context-memory scaling rather than
assuming it is linear for this hybrid architecture.

For each node calculate:

```text
safe_budget = repeatable_usable_memory - OS_reserve - operational_reserve
required = assigned_model_allocation + measured_runtime_overhead
           + measured_context_state + transient_peak
headroom = safe_budget - required
```

A candidate qualifies only when both nodes retain positive, operationally safe
headroom during model load, prompt processing, and sustained generation. Report
the exact deficit on each node if it does not qualify. Do not change UMA size,
TTM settings, CU routing, clocks, kernel, or Mesa merely to erase that deficit.

## Split comparison and benchmark sequence

For the largest candidate that qualifies:

1. Start at an 8,192-token context using automatic `layer` allocation.
2. Verify both GPUs participate and capture per-node peak memory and RPC traffic.
3. Derive an explicit tensor split from the two measured safe device budgets,
   not total installed RAM.
4. Repeat the same prompt, seed, batch settings, and sampling parameters with
   the explicit split.
5. Compare load success, minimum headroom, prompt tokens/s, generation tokens/s,
   time to first token, backend traffic, temperatures, and kernel faults.
6. Prefer the highest-quality quantization that remains stable; do not choose a
   smaller quant solely because it loads more easily.
7. Attempt 16K only after the chosen 8K configuration passes sustained tests.

Record measurements under the gitignored `benchmarks/private/`. Promote a model
to inventory only after this qualification is complete; this plan does not
enable `llama_manage_model_download` or set `llama_model_filename`.
