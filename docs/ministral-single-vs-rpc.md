# Ministral 3 8B: one BC250 versus two-node RPC

This test measures whether a second BC250 helps a small dense model that fits
on one board. The same `Ministral-3-8B-Instruct-2512-Q4_K_M` GGUF was run on
Bowie alone and on Bowie plus Crockett over the dedicated 2.5 GbE llama.cpp RPC
link. It is a performance characterization, not a change to the production
Qwen3.6 configuration.

## Test configuration

| Item | Value |
| --- | --- |
| Model SHA-256 | `5dbc3647eb563b9f8d3c70ec3d906cce84b86bb35c5e0b8a36e7df3937ab7174` |
| llama.cpp revision | `d775b8967a46d8beb110d444aa3b8938179e0dd8` |
| Performance profile | Moderate: 3500 MHz CPU, 1750 MHz GPU |
| Context and KV cache | 65,536; Q8_0 K and Q8_0 V; one slot |
| Offload | All 35 model layers; Vulkan; `layer` split |
| Prompt | Existing deterministic bake-off fixture, 20,514 evaluated tokens |
| Generation | 1,024 tokens, temperature 0, fixed seed, EOS ignored |
| Batch sizes | Pinned llama.cpp defaults; no batch override in either mode |
| Runs | One warmup plus three measured runs per configuration |

Prompt caching was disabled for every request. The single-node command omitted
`--rpc`; the two-node command added only `--rpc 10.250.0.2:50052`. Model,
prompt, context, KV types, sampling, llama.cpp build, fan curve, CU routing,
TTM configuration, and clocks were otherwise identical.

## Results

| Metric | Bowie only | Bowie + Crockett | Two-node change |
| --- | ---: | ---: | ---: |
| Prompt processing mean | 232.538 tok/s | 362.422 tok/s | **+55.85%** |
| Prompt processing median | 232.530 tok/s | 362.416 tok/s | +55.85% |
| Generation mean | **25.756 tok/s** | 18.151 tok/s | **-29.53%** |
| Generation median | **25.756 tok/s** | 18.148 tok/s | -29.54% |
| Generation scaling | 1.000x | **0.705x** | -7.605 tok/s |
| TTFT mean | 88.267 s | **56.654 s** | -35.82% |
| Total request mean | 127.986 s | **113.017 s** | -11.70% |
| Model load time | **11.076 s** | 23.374 s | +111.04% |
| Active GPU average PPT | **117.75 W** | 196.79 W combined | +67.13% |
| Prompt tok/s per active-GPU W | **1.975** | 1.842 | -6.75% |
| Generation tok/s per active-GPU W | **0.2187** | 0.0922 | -57.84% |

The synthetic request is heavily weighted toward its 20.5K-token prefill, so
two nodes reduced total time even though token-by-token generation became much
slower. Agent and chat workloads with shorter prompts or longer answers will
shift further in favor of the single node.

### Individual measured runs

| Mode | Run | Prompt tok/s | Generation tok/s | TTFT | Total |
| --- | ---: | ---: | ---: | ---: | ---: |
| Bowie | 1 | 232.522 | 25.756 | 88.273 s | 127.994 s |
| Bowie | 2 | 232.563 | 25.760 | 88.257 s | 127.972 s |
| Bowie | 3 | 232.530 | 25.754 | 88.270 s | 127.994 s |
| Two node | 1 | 362.410 | 18.148 | 56.657 s | 113.026 s |
| Two node | 2 | 362.416 | 18.159 | 56.653 s | 112.992 s |
| Two node | 3 | 362.438 | 18.146 | 56.652 s | 113.031 s |

## Memory and layer placement

The complete model and 65K Q8 KV cache fit on Bowie. The single-node load used
a 4,662 MiB Vulkan model buffer, 4,624 MiB KV buffer, and 164 MiB compute
buffer. Minimum measured GTT headroom was approximately 2.03 GiB.

Automatic two-node splitting placed approximately 52% of model-buffer bytes
on Bowie and 48% on Crockett. The KV buffers show the precise transformer-block
split: 16 of 34 blocks on Bowie and 18 on Crockett.

| Placement | Bowie only | Two-node Bowie | Two-node Crockett |
| --- | ---: | ---: | ---: |
| Model buffer | 4,662 MiB | 2,432 MiB | 2,230 MiB |
| KV buffer | 4,624 MiB | 2,176 MiB | 2,448 MiB |
| Compute buffer | 164 MiB | 164 MiB | 164 MiB |
| llama.cpp projected free Vulkan memory | 2,819 MiB | 7,497 MiB | 7,427 MiB |
| Measured GTT use | 9.48 GiB | 4.55 GiB | 4.40 GiB |
| Minimum GTT headroom | 2.03 GiB | 6.95 GiB | 7.10 GiB |

The split therefore provides substantial memory headroom, but this model does
not need it at the tested context.

## Utilization, RPC, power, and cooling

Stock Fedora does not provide a working `gpu_busy_percent` counter on BC250.
Utilization evidence here uses each Vulkan process's cumulative
`drm-engine-gfx` time, together with sustained 1750 MHz clocks, PPT, network
traffic, and allocation logs.

| Metric | Bowie only | Two-node Bowie | Two-node Crockett |
| --- | ---: | ---: | ---: |
| GFX-engine active, full measured window | **97.2%** | 52.2% | 58.8% |
| GFX-engine active, prompt stage | 97.9% | 73.0% | 85.1% |
| GFX-engine active, generation stage | **96.3%** | 31.4% | 33.0% |
| Average PPT | 117.75 W | 95.39 W | 101.41 W |
| Peak PPT | 142.94 W | 141.34 W | 142.30 W |
| Average GPU temperature | 65.2 C | 58.1 C | 59.9 C |
| Peak GPU temperature | 69 C | 67 C | 69 C |
| Average fan PWM / RPM | 180.5 / 2,279 | 153.5 / 1,986 | 160.6 / 2,034 |
| Peak fan PWM / RPM | 194 / 2,500 | 184 / 2,380 | 199 / 2,439 |

The two-node run transferred approximately 4.54 GB over Bowie's backend
interface during the three measured requests. Average bidirectional interface
traffic was about 107 Mbit/s, far below the nominal 2.5 GbE link rate. Raw link
bandwidth was not saturated. The generation regression instead points to
per-token RPC latency, synchronization, and graph-split overhead: both GPUs
were active only about one third of generation time.

The lower two-node temperatures are consequently explained by both workload
distribution and idle gaps. They are not evidence that the two GPUs could
automatically produce more decode throughput under this layer-split RPC
configuration.

## Sustained generation

A separate warmed 2,048-token request checked for progressive changes. The
stream combined some token events, so the checkpoints are approximate equal
segments rather than four exact token quarters.

| Stream segment | Bowie only | Bowie + Crockett |
| --- | ---: | ---: |
| Early | 25.904 tok/s | 18.202 tok/s |
| Middle | 25.624 tok/s | 18.074 tok/s |
| Late | 25.285 tok/s | 17.884 tok/s |
| Full 2,048-token server timing | 25.451 tok/s | 17.979 tok/s |

Both modes slowed slightly as generation progressed, but the relative result
did not change: RPC remained approximately 29% slower for sustained decode.

## Stability and conclusion

Both modes held 1750 MHz without thermal throttling. There were no GPU resets,
ring timeouts, GPU page faults, Vulkan errors, RPC failures, or fan-controller
safety events.

For this 8B dense model, use **one BC250** when generation speed, efficiency,
load time, and total electrical cost matter. Two-node RPC is useful only for a
strongly prefill-dominated request or when its extra memory capacity is
required. It improves prompt processing substantially, but is not worthwhile
for ordinary Ministral chat or agent decoding.

The historical SteamOS/Ollama result of approximately 47 tok/s is useful only
as context. This pinned Fedora/llama.cpp Vulkan test measured 25.76 tok/s on one
BC250, about 45% lower, but software stack, model revision, quantization,
context, and measurement method may differ; it is not an apples-to-apples
regression claim.

After testing, both nodes were restored to the production Moderate profile,
Bowie's Qwen3.6 service, Crockett's RPC service, and the managed fan-control
services. TTM settings remained unchanged and no temporary benchmark unit was
left active.
