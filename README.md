# BC250 distributed llama.cpp Vulkan cluster

> [!CAUTION]
> A BC250 board is not ready for sustained inference in stock form. Before
> targeting hardware, read [Hardware preparation](docs/hardware-prep.md) and
> the linked BC250 Community Documentation. Physical modifications and
> low-level tuning can damage hardware and are undertaken at the operator's
> own risk.

## Why this project exists

The AMD BC250 is an unusual compute board originally produced for
cryptocurrency mining. It uses an AMD APU derived from PlayStation 5 hardware,
combining Zen 2 CPU cores, an RDNA 2-based GPU, and 16 GB of high-bandwidth
GDDR6 unified memory shared between the CPU and GPU. Community Linux support
has turned these surplus boards into inexpensive and surprisingly capable
compute platforms.

I have always liked the idea of the BC250, so when I found a decent local deal
on two of them, I picked them up. I initially built one into a complete system
with its own case and installed Bazzite, but after the initial experimentation
it mostly sat unused.

When I started experimenting with local AI, I realized the BC250 could be a
great candidate for inference. Its GPU, shared GDDR6 memory, Vulkan support,
and low cost made it an interesting platform to experiment with.

I first got a single BC250 working as a local AI system in my
[bc250-hermes-local-agent](https://github.com/4claps/bc250-hermes-local-agent)
project.

Once the single-node system was working, I knew it was time to see what two of
them could do together.

That became this project.

## Node naming

I use names from Texas history for devices throughout my homelab. In my
deployment, the two BC250 nodes are named **Bowie** and **Crockett**.

These are simply hostnames for my specific systems:

- **Bowie** = BC250 node 1 and the llama.cpp coordinator
- **Crockett** = BC250 node 2 and the llama.cpp RPC worker

These names are personal choices, not BC250 or llama.cpp terminology. The
current inventory and preflight safety checks do, however, use `bowie` and
`crockett` as the inventory host keys for this fixed two-node topology. To use
different names, update the inventory keys, group membership checks, and
related references consistently; changing only an operating-system hostname
is not sufficient.

Throughout the documentation, references to Bowie and Crockett mean my two
BC250 nodes.

## What this is

A reproducible Ansible deployment for a two-node AMD BC250 Fedora 44
llama.cpp Vulkan/RPC inference cluster. Bowie runs `llama-server` and its local
Vulkan GPU; Crockett contributes its Vulkan GPU through llama.cpp RPC over an
isolated 2.5 Gb Ethernet link.

## What this is not

- It is not an Ollama deployment.
- It does not use Docker or Podman for inference.
- It is not a generic GPU-clustering framework.
- Its validated hardware tuning is not guaranteed safe on another BC250.

## Networking

```text
management LAN                        backend 10.250.0.0/30

clients -> Bowie:8080                 Bowie: 10.250.0.1
           llama-server + Vulkan0             |
                                              | llama.cpp RPC only
                                      Crockett: 10.250.0.2:50052
                                                ggml-rpc-server + Vulkan0
```

The RPC listener is bound only to Crockett's backend address and firewalld
accepts it only from Bowie's backend address. The backend has no gateway, DNS,
or default route. llama.cpp RPC is unauthenticated and must not be exposed to an
untrusted network.

## Tested baseline

The validated private deployment used:

- Fedora 44 with the stock Fedora kernel and Mesa/RADV packages available on
  the validation date.
- Separately qualified 40/40 CU routing on each physical board.
- The `moderate` performance profile: 3500 MHz CPU target, scale `-22`, and a
  complete 500–1750 MHz GPU curve capped at 925 mV.
- 80°C GPU throttle and 75°C recovery thresholds.
- `ttm.pages_limit=3014656` and `ttm.page_pool_size=3014656`.
- Pinned llama.cpp commit `d775b8967a46d8beb110d444aa3b8938179e0dd8`.
- No CPU core unlock, UMA firmware change, custom kernel/Mesa, or experimental
  GFX1013 compute-queue patch.

The TTM value is exactly 3,014,656 4 KiB pages, or 11.5 GiB. Combined with the
BC250's approximately 512 MiB hardware VRAM, Vulkan reports approximately
12 GiB per node. The 11.5 GiB is GPU-addressable shared system RAM, **not**
dedicated VRAM. This aggressive inference-specific limit leaves materially less
RAM for Fedora and CPU-side work than the stock kernel behavior; monitor host
memory pressure for every model and context.

The public example inventory deliberately does **not** enable the 40-CU table,
performance governors, or TTM override. Those settings become reproducible only
after the operator qualifies each board and explicitly promotes them in an
ignored local production inventory.

## Results and model recommendation

The known-good Moderate/40-CU benchmark used `Qwen3-30B-A3B-Q4_K_M`, an 8,192
token context, automatic layer split, Bowie local Vulkan plus Crockett RPC, and
1,024 generated tokens:

| Metric | Result |
| --- | ---: |
| Prompt processing | 72.48 tokens/s |
| Generation | 77.77 tokens/s |
| Bowie peak | 1750 MHz, 49°C, 85.39 W PPT |
| Crockett peak | 1750 MHz, 66°C, 80.34 W PPT |

For Hermes Agent, the current recommendation is
`Qwen3-Coder-30B-A3B-Instruct-Q4_K_M` with a 65,536-token context, Q8_0 K/V
cache, both GPUs, and automatic layer split. It loaded and completed the 64K
tests. In the original sustained run, Crockett reached approximately 90°C and
throttled while Bowie peaked at 67°C. After the physical cooling changes in
[Hardware preparation](docs/hardware-prep.md), the equivalent 18,976-token
prompt plus 1,024-token generation qualification peaked at 61°C on Bowie and
66°C on Crockett. Both GPUs held 1750 MHz without thermal throttling, GPU
errors, resets, or RPC failures. Crockett therefore passes sustained thermal
qualification for this workload without a software-profile or thermal-limit
change.

See [the deployed-state record](docs/current-state.md) and the complete
[Hermes model bake-off](docs/hermes-model-bakeoff.md).

## Hardware safety

- BC250 harvesting varies by board. A disabled WGP may be defective.
- Never copy another board's validated WGP table. Test and record each physical
  board independently.
- Every CU test, persistence, and rollback write requires an explicit
  single-host `--limit`. Never route both boards in one operation.
- Overclocking and undervolting can cause computation errors, hangs, or hardware
  damage. Begin with stock settings and promote one named profile only after
  load, thermal, and kernel-error testing.
- UMA/CMOS changes can make a board unbootable and are outside this baseline.
- The TTM override aggressively shares RAM between CPU and GPU; a reported
  Vulkan capacity is not a guarantee that a model safely fits.
- Long-context inference raises sustained cooling requirements considerably.

The normal role defaults and public inventory are conservative. The known-good
profile documents what worked on two specific boards; it is not a universal
BC250 guarantee.

## Prepare a local inventory

The tracked `inventories/example` uses RFC 5737 documentation addresses,
`CHANGE_ME` NIC values, disabled CU persistence, an empty WGP table, stock
governors, and stock TTM behavior. It is intentionally unable to pass preflight
against real hosts.

Create the ignored local inventory and edit every host-specific value:

```bash
cp -a inventories/example inventories/production
$EDITOR inventories/production/group_vars/all.yml
$EDITOR inventories/production/group_vars/bc250_cluster.yml
$EDITOR inventories/production/host_vars/bowie.yml
$EDITOR inventories/production/host_vars/crockett.yml
```

Use it explicitly:

```bash
ansible-inventory -i inventories/production/hosts.yml --graph
ansible-playbook -i inventories/production/hosts.yml site.yml --syntax-check
ansible-playbook -i inventories/production/hosts.yml site.yml --check --diff --limit bowie
```

Do not enable CU persistence until `docs/operations.md`'s per-board test and
promotion workflow is complete.

## Install and validate

Install the tested collection versions, then run local-only checks against the
safe example inventory:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-inventory -i inventories/example/hosts.yml --graph
ansible-playbook -i inventories/example/hosts.yml site.yml --syntax-check
ansible-lint
yamllint .
```

`site.yml` performs preflight, Fedora configuration, BC250 hardware management,
networking, Vulkan installation, the pinned llama.cpp build, services, and
validation. Read [operations](docs/operations.md), [architecture](docs/architecture.md),
[performance tuning](docs/performance-tuning.md), and
[troubleshooting](docs/troubleshooting.md) before targeting hardware.

## Reproducibility boundary

llama.cpp and important BC250 source artifacts are pinned by commit and/or
checksum. Fedora itself is not an immutable image: `base_apply_updates: true`
allows normal Fedora package updates, including kernel, Mesa, firmware, and
userspace changes. This repository records a validated Fedora 44 configuration,
not a byte-for-byte frozen OS. Review and reboot one node at a time after
relevant updates, then repeat Vulkan, CU, RPC, thermal, and kernel-log
validation before updating the baseline record.

## License and credits

Original project code and documentation are licensed under the [MIT License](LICENSE).
Upstream ownership and links are recorded in [ACKNOWLEDGEMENTS.md](ACKNOWLEDGEMENTS.md).
