# TL;DR: reproduce the two-node BC250 cluster

This project builds a two-node AMD BC250 distributed llama.cpp Vulkan
inference backend with Ansible. Node 1 runs `llama-server`, uses its local GPU,
and exposes an OpenAI-compatible API. Node 2 contributes its GPU through
llama.cpp RPC over a dedicated 2.5 GbE link.

The tested hosts are named Bowie and Crockett, but those are only the author's
hostnames. A deployment may use other names if it updates the fixed inventory
keys and related checks consistently.

```text
Existing AI client / Hermes
             |
             | OpenAI-compatible API
             v
           Node 1
        llama-server
         /       \
   local GPU      RPC over dedicated 2.5 GbE
                    |
                  Node 2
                  Vulkan GPU
```

This page is a quick start and results summary. The linked detailed documents
remain authoritative.

## Hardware prerequisites

### Required concepts

- Two AMD BC250 boards for the full cluster.
- Suitable power delivery and cooling for sustained compute.
- Management Ethernet for SSH, administration, API access, and downloads.
- A dedicated 2.5 GbE-capable point-to-point connection for llama.cpp RPC.
- Storage for Fedora, the llama.cpp build, and GGUF models.
- A separate Linux computer to run Ansible.

### Tested physical implementation

| Component | Tested implementation |
| --- | --- |
| Power | One 500 W PSU powering both BC250 nodes |
| Main airflow | ARCTIC P12 Pro fans |
| Fan mounting | Custom 3D-printed mounts and static-pressure boosters/ducts |
| Stock heatsink | Center fins removed to accommodate the fan arrangement |
| APU interface | Honeywell PTM7950 |
| Other interfaces | Original thermal pads replaced with thermal putty |
| Mount pressure | Small plastic washers added to increase pressure slightly |
| Rear cooling | Aluminum heatsinks attached near the rear/backplate area |
| RPC networking | USB 3.x 2.5 GbE adapters on both nodes |

These details describe the tested boards, not a mandatory parts list. Assess
your own cooling, connectors, wiring, mounting pressure, and board condition.

> [!WARNING]
> A bare or stock BC250 is not automatically ready for unattended inference.
> Read the [BC250 Community Documentation](https://elektricm.github.io/amd-bc250-docs/)
> before starting. Cooling and power delivery matter, boards vary, and a
> disabled Compute Unit may be defective. Hardware modification, CU routing,
> undervolting, and overclocking can damage hardware or corrupt computation.
> This repository records what worked on two specific boards; proceed at your
> own risk. The project also builds on the
> [BC250 SteamOS Real Toolkit](https://github.com/rpf16rj/bc250-steamos-real-toolkit).

## Software prerequisites

### BC250 nodes

- Fedora 44, the validated operating system.
- SSH access and sudo privileges.
- Network access for Fedora packages and pinned source artifacts.

### Separate Ansible controller

Ansible is **not** expected to run on either BC250. Use another Linux computer
with SSH key access to both nodes.

On a Fedora controller:

```bash
sudo dnf install ansible python3-netaddr
ansible-galaxy collection install -r requirements.yml
```

## Network layout

| Network | Purpose |
| --- | --- |
| Management LAN | SSH, Ansible, package downloads, administration, and Bowie's OpenAI-compatible API |
| Dedicated backend | Point-to-point llama.cpp RPC only; no gateway, DNS, or default route |

The documented backend example is:

```text
Node 1: 10.250.0.1/30
Node 2: 10.250.0.2/30
RPC:    10.250.0.2:50052
```

Ansible resolves USB backend adapters by permanent MAC address instead of
depending on unstable Linux USB interface names. Crockett's unauthenticated
RPC listener is backend-only; never expose it to the management LAN or
internet. Bowie's API is restricted to configured trusted management CIDRs.

## Fresh-build order

1. Install Fedora 44 on both boards.
2. Configure management networking, SSH, and sudo access.
3. Prepare the separate Ansible controller.
4. Copy `inventories/example` to the ignored `inventories/production`.
5. Set management addresses, trusted API CIDRs, and each backend NIC's MAC.
6. Qualify each board's cooling and CU routing as described in the detailed
   documentation; never copy another board's WGP table blindly.
7. Run the production deployment:

   ```bash
   ansible-playbook -i inventories/production/hosts.yml site.yml
   ```

8. Ansible executes the dependency chain:

   ```text
   base -> BC250 detection -> fan/sensor driver -> fan telemetry
        -> fail-safe fan control -> cooling validation
        -> CU/TTM/performance -> backend network -> Vulkan -> llama.cpp
        -> RPC/coordinator services -> production model/API -> validation
   ```

9. Run the same playbook again. Persistent tasks should report `changed=0`
   and `failed=0`. The read-only bandwidth validation intentionally adds and
   removes a temporary firewall rule and may report those two runtime changes.
10. Check `/health` and `/v1/models`, then point an external
    OpenAI-compatible client at Node 1.

Cooling validation is mandatory. Production RPC and coordinator services are
not enabled when fan telemetry/control is unqualified or bypassed.

## Known-good production settings

| Item | Validated setting |
| --- | --- |
| OS | Fedora 44, stock Fedora kernel/Mesa/RADV |
| CU routing | 40/40, qualified separately on each board |
| TTM | `ttm.pages_limit=3014656`, `ttm.page_pool_size=3014656` |
| Vulkan-visible memory | Approximately 12 GiB per node |
| llama.cpp | `d775b8967a46d8beb110d444aa3b8938179e0dd8` |
| Production profile | Moderate: CPU 3500 MHz, GPU 1750 MHz, 80°C limit |
| Production model | Qwen3.6-35B-A3B Q4_K_M |
| Context and KV | 65,536 tokens, Q8_0 K/V, one slot |
| Split | All layers offloaded, automatic layer split |

Production llama-server arguments:

```text
--ctx-size 65536
--cache-type-k q8_0
--cache-type-v q8_0
--parallel 1
--gpu-layers all
--split-mode layer
--rpc 10.250.0.2:50052
--jinja
--cache-ram 0
--no-cache-idle-slots
```

`--cache-ram 0` and `--no-cache-idle-slots` prevent llama.cpp's cross-request
host-RAM prompt cache from accumulating unrelated large agent prompts and
exhausting coordinator memory. They do **not** disable the normal Q8_0 K/V
cache used by the active request.

## Hermes-focused model bake-off

All six candidates fit at 65,536 context with full GPU layer offload. The
losing models were rejected primarily for practical agent behavior or latency,
not inability to fit.

| Rank | Model | Practical success | Pass@3 | Mean task | Tool errors | Generation | Verdict |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | --- |
| 1 | Qwen3.6-35B-A3B Q4_K_M | 23/27 (85.2%) | 8/9 | 108.4 s | 2 | 51.72 tok/s | Best overall Hermes backend and quality |
| 2 | gpt-oss-20b MXFP4 | 21/27 (77.8%) | 8/9 | 70.9 s | 21 | 66.30 tok/s | Best speed, efficiency, and memory headroom |
| 3 | Qwen3-Coder-30B-A3B Q4_K_M | 19/27 (70.4%) | 9/9 | — | 25 | — | Capable control; weaker practical Hermes result |
| 4 | Qwen3.6-35B-A3B UD-Q4_K_XL | 5/9 (55.6%), early stop | Not reached | 143.6 s | 0 | 50.45 tok/s | Stable, but tight memory and excessive agent latency |
| 5 | Qwen3.8-27B Q4_K_M | 2/21 (9.5%) | — | Excessive | — | — | Not recommended |
| 6 | GLM-4.7-Flash Q4_K | Opening two tasks timed out | — | — | — | — | Not recommended |

Qwen3.6 won on real tool use, reliability, latency, and completed agent tasks,
not raw tokens per second alone. `gpt-oss-20b` remains the useful fast/light
alternative.
The larger UD-Q4_K_XL quant did not replace it: generation was 2.46% slower,
minimum free Vulkan memory fell to about 1.1 GB per node, and two of its first
nine Hermes tasks timed out.

## Performance-profile results

The same 18,982-token prompt, 1,024-token generation, model, split, cooling,
and software configuration were used for all profiles.

| Profile | Prompt | Generation | Request | Combined avg PPT | GPU peak node 1/2 | Verdict |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| Moderate/1750 | 587.458 tok/s | 51.361 tok/s | 52.267 s | 189.53 W | 64/66°C | **Recommended 24/7** |
| Strong/1850 | 612.437 tok/s | 52.452 tok/s | 50.534 s | 195.98 W | 65/68°C | Optional balanced-performance profile |
| Aggressive/2000 | 649.288 tok/s | 54.029 tok/s | 48.205 s | 218.53 W | 71/72°C | Short-duration performance profile only |

Aggressive raised GPU clock 14.29%, prompt throughput 10.53%, and generation
throughput only 5.20%, while combined average PPT rose 15.30%. Moderate remains
the best 24/7 profile because generation performance scaled poorly relative to
power and cooling cost.

## Real Hermes profile check

| Profile | Elapsed | API/LLM calls | Tool calls | Tool errors |
| --- | ---: | ---: | ---: | ---: |
| Moderate/1750 | 23.19 s | 2 | 1 | 0 |
| Strong/1850 | 22.51 s | 2 | 1 | 0 |
| Aggressive/2000 | 23.17 s | 2 | 1 | 0 |

All runs succeeded. Despite measurable synthetic gains, higher clocks provided
no meaningful improvement to this small real Hermes workflow.

## Cooling result

Before physical cooling work, Crockett reached approximately 90°C and
throttled. After hardware preparation it operated in the mid-60s under
qualified workloads without throttling. With final managed fan control:

- Moderate peaks: node 1 64°C, node 2 66°C.
- Aggressive peaks: node 1 71°C, node 2 72°C.
- No thermal throttling, GPU reset, Vulkan failure, RPC failure, or
  fan-controller safety event occurred.

Fan-driver installation, positive PWM/RPM identification, automatic fail-safe
control, and live validation are now required parts of the normal Ansible
build.

## If everything worked

A successful deployment provides:

- Two Fedora BC250 nodes, with 40 CUs only where each board passes individual
  qualification.
- Approximately 12 GiB Vulkan-visible memory per node.
- Managed cooling and fan telemetry.
- A private 2.5 GbE RPC path with no default route.
- `llama-server` plus local Vulkan on node 1.
- `ggml-rpc-server` plus Vulkan on node 2.
- Qwen3.6 at 65K context behind an OpenAI-compatible management-LAN API.
- Moderate/1750 as the unattended production profile.
- External clients such as Hermes connecting only to node 1.

Do not expect identical benchmark numbers. Results vary with the individual
boards, cooling, network, model revision, Fedora packages, and llama.cpp build.

## Quick API check

The validated deployment has no API-key authentication; restrict access with
the management firewall. Substitute your node 1 management address and port.

```bash
curl --fail http://<node1-management-ip>:<port>/health
curl --fail http://<node1-management-ip>:<port>/v1/models
```

```bash
curl --fail \
  --header 'Content-Type: application/json' \
  --data '{
    "model": "/var/lib/llama.cpp/models/Qwen3.6-35B-A3B-Q4_K_M.gguf",
    "messages": [{"role": "user", "content": "Reply with exactly API_READY."}],
    "temperature": 0,
    "max_tokens": 64,
    "chat_template_kwargs": {"enable_thinking": false}
  }' \
  http://<node1-management-ip>:<port>/v1/chat/completions
```

Disabling thinking is useful for short deterministic replies and forced tool
calls. Normal reasoning workflows can leave thinking enabled with an adequate
completion budget.

## Detailed documentation

- [Hardware preparation](hardware-prep.md)
- [Architecture](architecture.md)
- [Operations](operations.md)
- [Current validated state](current-state.md)
- [Performance tuning](performance-tuning.md)
- [Hermes model bake-off](hermes-model-bakeoff.md)
- [Qwen3.6 performance profiles](qwen36-performance-profiles.md)
- [Acknowledgements](../ACKNOWLEDGEMENTS.md)

Use this page as the front door. Use the detailed documents for safety gates,
inventory variables, qualification procedures, troubleshooting, and the full
evidence behind the recommendations.
