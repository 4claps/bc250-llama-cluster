# Hardware preparation

The BC250 is **not ready for sustained inference in stock form**. Before
attempting this project, read the
[AMD BC250 Community Documentation](https://elektricm.github.io/amd-bc250-docs/),
particularly its guidance on power, cooling, BIOS configuration, and hardware
setup.

These boards were designed for a very different environment from a normal PC
or home server. Proper cooling and power delivery must be addressed before
putting them under sustained inference workloads.

> [!CAUTION]
> This repository documents a configuration tested on **my hardware**, not a
> minimum parts list or a universally safe modification guide. BC250 boards
> vary, especially when enabling additional Compute Units or changing low-level
> performance settings. I cannot guarantee that the same physical changes,
> cooling, power delivery, CU routing, aggressive TTM configuration, or
> overclocking/undervolting will be safe or stable on another board. Hardware
> modification can cause equipment damage or data loss; proceed at your own
> risk.

## My hardware configuration

My current physical setup includes:

- One **500 W power supply** powering both BC250 nodes
- **ARCTIC P12 Pro** high-static-pressure cooling fans
- Custom **3D-printed fan mounts and static-pressure boosters/ducts**
- The middle fins of the BC250 heatsinks removed to accommodate the cooling
  arrangement
- **Honeywell PTM7950** thermal interface material on each APU
- The original thermal pads replaced with **thermal putty**
- Small **plastic washers** added to the heatsink mounting hardware to increase
  mounting pressure slightly and improve component contact
- An additional **aluminum heatsink attached to the rear/backplate area** for
  additional GPU-side heat dissipation
- **USB 3.x 2.5 GbE adapters** on both nodes for the dedicated llama.cpp RPC
  backend network

This list describes what I did to these two boards. It is not a recommendation
to copy the power-supply sizing, remove heatsink fins, increase mounting
pressure, or attach a rear heatsink. A power supply's wattage rating alone does
not establish that its connectors, wiring, and 12 V capacity are suitable.
Likewise, excessive mounting pressure or poor heatsink contact can damage a
board. Follow the Community Documentation and assess each installation on its
own merits.

The dedicated 2.5 GbE adapters are separate from the normal management
interfaces. They provide the point-to-point network used for llama.cpp RPC
traffic between the coordinator and worker.

## Cooling matters

Sustained AI inference is a very different workload from simply booting the
board or running short tests. Long-context prompt processing can keep the GPU,
memory subsystem, and power-delivery components heavily loaded for extended
periods.

Do not assume that a BC250 which appears stable at idle or during short
benchmarks has adequate cooling for sustained inference.

Before enabling performance tuning or unattended workloads:

- Verify heatsink contact.
- Replace degraded thermal interface material where necessary.
- Provide strong airflow through the heatsink.
- Monitor temperatures under sustained load.
- Verify that power connectors and wiring are not overheating.
- Qualify each board individually before enabling persistent 40-CU routing.

During the original sustained 64K Qwen3-Coder test, Crockett reached
approximately 90°C and throttled while Bowie peaked at 67°C. After the physical
cooling remediation listed above, the equivalent 18,976-token prompt plus
1,024-token generation qualification peaked at 66°C on Crockett and 61°C on
Bowie. Crockett's late-prompt heat-soak temperature averaged 62.5°C, both GPUs
held 1750 MHz, and neither node throttled or produced a GPU, kernel, RPC, or
network error. The 24°C reduction qualifies this physical setup for the tested
sustained Hermes workload.

Cooling qualification now also includes the Ansible-managed NCT6686D fan
controller, positive PWM/tachometer channel identification on each board, a
fail-safe automatic curve, and continuous PWM/RPM telemetry. The later Qwen3.6
profile characterization peaked at 64/66°C on Bowie/Crockett under the
recommended Moderate/1750 profile. Strong/1850 and Aggressive/2000 were also
stable, but remain optional characterized profiles rather than unattended
defaults. See [Fan control](fan-control.md) and
[Qwen3.6 performance profiles](qwen36-performance-profiles.md).

My hardware configuration is an example, not a universal BC250 cooling
specification or proof that another board is safe under the same workload.
