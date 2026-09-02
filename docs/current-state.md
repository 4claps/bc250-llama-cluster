# Validated private deployment state

Last verified: 2026-08-24.

## Nodes and hardware

| Node | Role | Management | Backend | Active CUs | RAM | GPU memory target |
| --- | --- | --- | --- | --- | --- | --- |
| Bowie | llama.cpp coordinator | `10.0.0.170` | `10.250.0.1/30` | 40/40 | ~14 GiB | 512 MiB VRAM + ~11.5 GiB GTT |
| Crockett | llama.cpp RPC worker | `10.0.0.171` | `10.250.0.2/30` | 40/40 | ~14 GiB | 512 MiB VRAM + ~11.5 GiB GTT |

The baseline intentionally uses the Ansible-managed kernel arguments
`ttm.pages_limit=3014656` and `ttm.page_pool_size=3014656`. At 4 KiB per page,
the TTM limit is exactly 12,348,030,976 bytes (11.5 GiB) of GTT. Together with
512 MiB of hardware VRAM, this produces approximately 12.0 GiB of
Vulkan-visible capacity per node. GTT is GPU-addressable shared system RAM, not
additional dedicated VRAM, and remains in contention with Fedora and host
processes.

Without these arguments, the Fedora kernel's TTM initialization would use
approximately half of the installed system RAM and expose only about 7.9 GiB
total per node after adding hardware VRAM. The larger inference-specific limit
is required for the validated large-model and 64K-context workloads. Those
tests completed without allocation failures, VM faults, GPU resets, ring
timeouts, or new MCEs. The operator originally applied the two arguments
manually with `grubby --update-kernel=ALL` on August 22, 2026; the
`bc250_hardware` role now adopts the exact tested values through Fedora's
BLS-compatible mechanism so current and future kernel entries remain
reproducible. No UMA firmware setting is changed.

The 40-CU tables are enabled by `bc250-cu-live-manager.service` and were
verified after separate reboots. The rollback playbook remains available if a
board later proves unstable under longer workloads.

Both Fedora root logical volumes and their XFS filesystems consume the full
235.89 GiB `fedora` volume group. The base role performs this expansion only
when `base_expand_root_lvm_enabled` is true and free extents remain.

## Performance profile

Both nodes run the Ansible-managed `moderate` profile. The CPU SMU boot profile
is 3500 MHz, scale `-22`, and an 80°C maximum. The GPU governor uses the complete
upstream 500–1750 MHz safe-point curve, whose 1750 MHz point is 925 mV, with
80°C throttling and 75°C recovery. `bc250-smu-oc.service` and
`cyan-skillfish-governor-smu.service` are enabled and active on both nodes.
Because llama.cpp Vulkan compute does not reliably assert the graphics
`GUI_ACTIVE` bit used by upstream `busy-flag` sampling, the governor's upstream
D-Bus performance mode holds the approved 1750 MHz ceiling while retaining
thermal throttling.

The known-good, cache-cleared benchmark used `Qwen3-30B-A3B-Q4_K_M`, an 8,192
token context, automatic layer split, Bowie local Vulkan plus Crockett RPC
Vulkan, and 1,024 generated tokens:

| Metric | Stock | Moderate | Change |
| --- | ---: | ---: | ---: |
| Prompt processing | 48.31 tokens/s | 72.48 tokens/s | +50.03% |
| Generation | 71.47 tokens/s | 77.77 tokens/s | +8.81% |
| Bowie peak | 1500 MHz, 48°C, 78.9 W | 1750 MHz, 49°C, 85.39 W | Stable |
| Crockett peak | 1500 MHz, 65°C, 85.2 W | 1750 MHz, 66°C, 80.34 W | Stable |

No thermal throttling, GPU reset, VM fault, ring timeout, allocation failure, or
new MCE was observed during the Moderate run.

The later Qwen3.6 characterization compared the exact upstream Moderate/1750,
Strong/1850, and Aggressive/2000 profiles. Both higher profiles were stable,
but Moderate remains the recommended 24/7 setting because it has the best
generation efficiency and no meaningful real-Hermes disadvantage. See
[Qwen3.6 performance profiles](qwen36-performance-profiles.md).

The baseline retains the stock Fedora kernel and Mesa/RADV. Apart from the
documented TTM command-line limits, it has no UMA modification, CPU core unlock,
GFX1013 compute-queue patch, or other custom kernel/amdgpu change.

## llama.cpp deployment

- Commit: `d775b8967a46d8beb110d444aa3b8938179e0dd8`
- Build: Vulkan and RPC enabled, native upstream default, vCPU-based parallelism
- Active release: `/opt/llama.cpp/releases/d775b8967a46d8beb110d444aa3b8938179e0dd8`
- Bowie API: `http://10.0.0.170:8080`
- Crockett RPC: `10.250.0.2:50052`, backend-only
- Model split: automatic layer split over Bowie `Vulkan0` and Crockett RPC `Vulkan0`

The production model is intentionally not tracked by Git:

- Model: `Qwen3.6-35B-A3B-Q4_K_M`.
- Path on Bowie: `/var/lib/llama.cpp/models/Qwen3.6-35B-A3B-Q4_K_M.gguf`.
- SHA-256: `671e47e0ec53c665d048b98c3ecbfd5236b5ca9c3e02ed19fc8f81f7b85140c7`.
- Context: 65,536 tokens.
- KV cache: Q8_0 K and Q8_0 V, one slot.
- Offload: all layers, automatic `layer` split over Bowie local Vulkan and
  Crockett RPC Vulkan.
- Prompt-cache safety: `--cache-ram 0 --no-cache-idle-slots` disables the
  cross-request host-RAM prompt cache that caused an OOM during the bake-off;
  it does not disable the normal Q8_0 K/V cache.

The completed Hermes bake-off selected Qwen3.6 as the best overall backend.
`gpt-oss-20b` MXFP4 remains the useful fast/light alternative. The earlier
Qwen3-Coder-30B-A3B-Instruct Q4_K_M result is retained as a historical control,
not the production recommendation.

The later `Qwen3.6-35B-A3B-UD-Q4_K_XL` follow-up was stable at 65,536 context
but did not change production: it generated 2.46% more slowly than Q4_K_M,
left only about 1.1 GB free Vulkan memory per node, and triggered the Hermes
early-stop rule after two 180-second timeouts in its first nine trials. See the
[Hermes model bake-off](hermes-model-bakeoff.md) for the comparison.

The original 64K Qwen3-Coder-30B run was functionally successful, but Crockett
reached approximately 90°C and throttled while Bowie peaked at 67°C. After
thermal-interface replacement, a rear heatsink, and increased physical board
separation, the sustained qualification was repeated with the same model,
llama.cpp revision, 65,536 context, Q8_0 K/V, one slot, and automatic split.
Crockett peaked at 66°C, averaged 62.5°C during late-prompt heat soak, and held
1750 MHz without throttling. Bowie peaked at 61°C. Generation remained within
normal variance at 77.10 tokens/s, and no GPU reset, fault, RPC error, or
backend timeout occurred. Crockett now passes sustained thermal qualification.
The later qualified automatic fan-control implementation and Qwen3.6 profile
characterization supersede any earlier concern that fan response was unknown.

The coordinator is configured for the single declared production GGUF rather
than router-mode discovery. Its OpenAI-compatible API is bound to the
management address and permitted only from configured trusted management
CIDRs. It has no built-in API authentication in this deployment; network
firewall policy is the access-control boundary. Crockett's unauthenticated RPC
listener remains bound only to the dedicated backend address.

## Validation and operations

Run local repository checks:

```bash
ansible-inventory -i inventories/production/hosts.yml --graph
ansible-playbook -i inventories/production/hosts.yml site.yml --syntax-check
ansible-lint
```

Run live cluster validation:

```bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/90-validation.yml
```

Read current CU routing:

```bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/21-cu-live-status.yml
```

Rollback one board to stock dispatch if needed:

```bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/23-cu-live-rollback.yml --limit bowie \
  --tags cu_live_rollback
```

Never run CU write playbooks against both boards simultaneously. The normal
`site.yml` run does not rewrite live CU dispatch, but it preserves the enabled
persistence service and its per-host validated table. The validation role reads
the live topology and fails if it differs from that table.

The public repository defaults to the sanitized example inventory. Commands
against this private deployment require the ignored local production inventory
to be selected explicitly with `-i inventories/production/hosts.yml`.
