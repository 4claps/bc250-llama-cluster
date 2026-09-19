# Qwen3.6-35B-A3B production quant promotion: Q4_K_M to Q4_K_L

Completed 2026-09-09. The production Hermes backend model was promoted from
`Qwen3.6-35B-A3B-Q4_K_M` to `Qwen3.6-35B-A3B-Q4_K_L`. In the same change
window the llama-server context size was raised from 65,536 to 100,000
(`llama-server` rounds the slot allocation up to `n_ctx` 100096). The context
size was raised again to 115,000 (`n_ctx` 115200) after this record; the figures
below describe the 100,000-token state at promotion.

This is a straight quant swap on the same `ggml-org/Qwen3.6-35B-A3B-GGUF`
family, the same pinned llama.cpp revision
`d775b8967a46d8beb110d444aa3b8938179e0dd8`, the same Moderate/1750 profile,
Q8_0 K/V cache, one slot, all layers offloaded, and automatic layer split
across Bowie local Vulkan and Crockett RPC Vulkan.

## What changed

| Item | Q4_K_M (previous) | Q4_K_L (current) |
| --- | --- | --- |
| Filename | `Qwen3.6-35B-A3B-Q4_K_M.gguf` | `Qwen3.6-35B-A3B-Q4_K_L.gguf` |
| GGUF size, bytes | 20,419,565,568 | 22,662,526,592 |
| SHA-256 | `671e47e0ec53c665d048b98c3ecbfd5236b5ca9c3e02ed19fc8f81f7b85140c7` | `49418000a889fef4b4353f36f598ad5a668c6ae374ac791bf923f76395a1bceb` |
| Quantization | Q4_K throughout | Q8_0 for embedding and output weights, Q4_K elsewhere |
| llama-server `--ctx-size` | 65536 | 100000 (`n_ctx` 100096) |

The SHA-256 above is `sha256sum` computed against the deployed file on Bowie at
`/var/lib/llama.cpp/models/Qwen3.6-35B-A3B-Q4_K_L.gguf`.

## Bake-off result

Re-ran the private Hermes bake-off harness against the live production endpoint
after the swap: the 27-trial error-task battery, the 3-trial no-tool arithmetic
eval, and the standardized streaming performance probe (one warmup plus three
measured runs, 18,982-token prompt, 1,024 generated tokens).

| Metric | Q4_K_L | Q4_K_M baseline |
| --- | ---: | ---: |
| Generation, tok/s (median) | 52.21 | 51.72 |
| Time to first token (median) | 34.15 s | ~32 s |
| Prompt processing, tok/s (median) | 557.0 | 588.9 |
| Error-task battery | 26/27 | 23/27 (original bake-off policy) |
| No-tool arithmetic eval | 3/3 | not run in original bake-off |

Verdict: performance parity within normal run-to-run and thermal variance, and
accuracy parity. No regression was observed. Q4_K_L is confirmed as the new
production model.

Generation throughput is marginally faster. Time to first token and prompt
processing are slightly slower but well inside the spread seen between repeated
Q4_K_M runs. The extra Q8_0 precision on the embedding and output tensors is
the motivation for the swap; it costs about 2.2 GB of GGUF size and a little
Vulkan headroom.

### Context-size note

The 65,536 to 100,000 increase was applied in the same window and is included
in the deployed configuration the bake-off measured. It was not separately
memory-qualified as an isolated change; watch Bowie and Crockett free Vulkan
memory and host `MemAvailable` under long-context production load.

## Known limitation carried over (not a Q4_K_L regression)

The `err_hidden_search` task fails about one run in three. The model's search
tooling returns only `docs/ops.md` and never descends into a hidden `.secrets/`
directory to find `rotation.cfg`, which the task's success check also requires.
This was reproduced at both the original 300-second task timeout and a
500-second rerun (158-second runtime, so not a timeout). It is a tool and
search-coverage gap that is independent of the quantization change and was
present on Q4_K_M. The single failure in the 26/27 Q4_K_L battery is this task.
