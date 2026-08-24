# Qwen3.6 BC250 performance-profile characterization

Tested 2026-08-24 on the two-node Fedora 44 cluster. This characterization
compares the standard Moderate, Strong, and Aggressive profiles from the
upstream BC250 community toolkit. It does not define new frequencies or
voltage points, and it does not make either higher profile a production
default.

## Test platform

- Two AMD BC250 nodes: Bowie as coordinator/node 1 and Crockett as RPC
  worker/node 2.
- Validated 40/40 CU routing and approximately 12 GiB Vulkan-visible memory on
  each board.
- Stock Fedora 44 kernel, Mesa, and RADV.
- Dedicated 2.5 GbE llama.cpp RPC backend.
- Qualified Ansible-managed BC250 fan control.
- llama.cpp revision `d775b8967a46d8beb110d444aa3b8938179e0dd8`.
- `Qwen3.6-35B-A3B-Q4_K_M.gguf`, SHA-256
  `671e47e0ec53c665d048b98c3ecbfd5236b5ca9c3e02ed19fc8f81f7b85140c7`.
- 65,536-token context, Q8_0 K and V caches, one slot, all layers offloaded,
  and automatic layer split across Bowie local Vulkan and Crockett RPC
  Vulkan.

All tests used the same model, payload, llama.cpp configuration, CPU target,
CU routing, TTM configuration, network, and fan curve. Only the selected GPU
performance profile changed.

## Upstream profiles

The definitions below come from BC250 SteamOS Real Toolkit commit
`37f49f2d5837997a1c1998bf3502b648425e6bb8`. They are community-tooling
profiles, not curves invented by this project. All use a 3500 MHz CPU target
and an 80°C limit.

| Profile | GPU safe points, MHz/mV |
| --- | --- |
| Moderate/1750 | 500/700, 1000/800, 1175/850, 1500/900, 1600/910, 1700/920, 1750/925 |
| Strong/1850 | 500/700, 1000/800, 1175/850, 1500/900, 1600/910, 1700/920, 1850/930 |
| Aggressive/2000 | 500/700, 1000/800, 1175/850, 1500/900, 1600/910, 1700/920, 1850/930, 2000/960 |

## Standardized benchmark

The deterministic workload used an 18,982-token prompt, generated 1,024
tokens, ran one warmup, and retained three measured runs per profile.

| Profile | Prompt mean | Prompt median | Generation mean | Generation median | TTFT mean | Request mean |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Moderate/1750 | 587.458 tok/s | 587.519 tok/s | 51.361 tok/s | 51.363 tok/s | 32.347 s | 52.267 s |
| Strong/1850 | 612.437 tok/s | 612.458 tok/s | 52.452 tok/s | 52.466 tok/s | 31.028 s | 50.534 s |
| Aggressive/2000 | 649.288 tok/s | 648.929 tok/s | 54.029 tok/s | 54.024 tok/s | 29.269 s | 48.205 s |

Individual measured runs were tightly grouped:

| Profile/run | Prompt tok/s | Generation tok/s | TTFT | Request |
| --- | ---: | ---: | ---: | ---: |
| Moderate 1 | 587.519 | 51.363 | 32.344 s | 52.263 s |
| Moderate 2 | 587.192 | 51.344 | 32.362 s | 52.289 s |
| Moderate 3 | 587.662 | 51.376 | 32.336 s | 52.250 s |
| Strong 1 | 612.577 | 52.466 | 31.020 s | 50.521 s |
| Strong 2 | 612.458 | 52.471 | 31.027 s | 50.526 s |
| Strong 3 | 612.275 | 52.418 | 31.036 s | 50.554 s |
| Aggressive 1 | 648.929 | 53.992 | 29.285 s | 48.233 s |
| Aggressive 2 | 650.021 | 54.071 | 29.235 s | 48.155 s |
| Aggressive 3 | 648.914 | 54.024 | 29.287 s | 48.225 s |

## Thermal, power, and fan results

| Profile/node | GPU avg/peak | CPU avg/peak | PPT avg/peak | PWM avg/peak | Fan avg/peak |
| --- | ---: | ---: | ---: | ---: | ---: |
| Moderate Bowie | 57.56/64°C | 61.01/63.1°C | 92.52/139.88 W | 152.6/173 | 1,978/2,234 RPM |
| Moderate Crockett | 59.92/66°C | 63.52/65.8°C | 97.01/139.35 W | 158.1/184 | 2,010/2,312 RPM |
| Strong Bowie | 59.44/65°C | 62.95/65.0°C | 100.29/145.88 W | 157.8/184 | 2,034/2,316 RPM |
| Strong Crockett | 61.02/68°C | 65.08/67.4°C | 95.69/145.65 W | 164.1/194 | 2,069/2,395 RPM |
| Aggressive Bowie | 62.52/71°C | 67.34/70.0°C | 106.95/165.59 W | 167.2/204 | 2,138/2,531 RPM |
| Aggressive Crockett | 64.25/72°C | 68.57/71.2°C | 111.59/167.06 W | 177.0/214 | 2,204/2,654 RPM |

| Profile | Combined average PPT | Combined observed peak-sum PPT |
| --- | ---: | ---: |
| Moderate/1750 | 189.53 W | 279.23 W |
| Strong/1850 | 195.98 W | 291.53 W |
| Aggressive/2000 | 218.53 W | 332.65 W |

Peak-sum PPT adds each board's individually observed peak. It is neither a
direct PSU wall-power measurement nor proof that both peaks occurred
simultaneously.

## Relative scaling

| Change from Moderate | Strong/1850 | Aggressive/2000 |
| --- | ---: | ---: |
| GPU clock | +5.71% | +14.29% |
| Prompt throughput | +4.25% | +10.53% |
| Generation throughput | +2.12% | +5.20% |
| TTFT | -4.08% | -9.52% |
| Total request | -3.32% | -7.77% |
| Combined average PPT | +3.41% | +15.30% |
| Combined peak-sum PPT | +4.40% | +19.13% |
| Bowie average fan RPM | +2.81% | +8.07% |
| Crockett average fan RPM | +2.98% | +9.66% |

Aggressive relative to Strong increased clock by 8.11%, prompt throughput by
6.02%, and generation by 3.01%. TTFT fell 5.67% and request time fell 4.61%,
while average PPT rose 11.51% and peak-sum PPT rose 14.10%. Inference
performance—especially token generation—does not scale linearly with GPU
clock.

## Performance per watt

Using combined average GPU PPT:

| Profile | Prompt tok/s/W | Generation tok/s/W |
| --- | ---: | ---: |
| Moderate/1750 | 3.100 | **0.2710** |
| Strong/1850 | **3.125** | 0.2676 |
| Aggressive/2000 | 2.971 | 0.2472 |

Strong is marginally best for prompt/prefill efficiency. Moderate is best for
generation efficiency. Aggressive loses approximately 8.8% generation
efficiency relative to Moderate.

## Real Hermes workflow

The same small `FAN_CHAIN_OK` terminal-tool fixture succeeded at all profiles.

| Profile | Elapsed | API calls | Tool calls | Tool errors |
| --- | ---: | ---: | ---: | ---: |
| Moderate/1750 | 23.19 s | 2 | 1 | 0 |
| Strong/1850 | 22.51 s | 2 | 1 | 0 |
| Aggressive/2000 | 23.17 s | 2 | 1 | 0 |

Strong saved 0.68 seconds. Aggressive was effectively identical to Moderate
and slower than Strong. For a task this small, agent and tool overhead dominate
the synthetic throughput difference.

## Stability and memory placement

Every requested clock was sustained. No thermal throttling, GPU reset, ring
timeout, GPU page fault, Vulkan error, llama.cpp RPC failure, or fan-controller
safety event occurred. Vulkan/GTT placement remained essentially unchanged:

| Node | Allocated | Free |
| --- | ---: | ---: |
| Bowie | 9.543 GiB | 1.957 GiB |
| Crockett | 9.723 GiB | 1.777 GiB |

## Verdicts

### Moderate/1750: recommended 24/7 production profile

Moderate has the best generation performance per watt, the lowest electrical
and cooling burden, excellent temperatures, no throttling, and no meaningful
real-Hermes disadvantage. It remains the production default.

### Strong/1850: optional balanced-performance profile

Strong was stable, slightly faster, and relatively efficient: 4.25% better
prefill and a 3.32% shorter synthetic request for 3.41% more average PPT. Its
0.68-second Hermes improvement is not enough to replace Moderate for normal
unattended operation.

### Aggressive/2000: optional short-duration performance profile only

Aggressive was stable and measurably faster for prefill-heavy work, but
generation improved only 5.20% while average PPT rose 15.30%, peak-sum PPT rose
19.13%, fan speed rose roughly 8–10%, and GPU temperatures rose roughly 4–5°C.
It did not improve the tested Hermes workflow. With both nodes sharing the
documented 500 W PSU arrangement, Aggressive is not recommended for unattended
24/7 service.

After characterization, both nodes were restored to the exact Moderate CPU
3500 MHz/GPU 1750 MHz configuration. Fan control, Bowie `llama-server`,
Crockett RPC, Qwen3.6 health, and the final Hermes fixture all passed; no
rollback timer or temporary experiment configuration remained.
