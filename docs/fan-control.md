# BC250 fan control

The BC250 exposes an NCT6686D-compatible embedded controller at ISA address
`0xa20`. Fedora's in-tree `nct6683` driver provides read-only sensors. Writable
PWM support uses upstream `nct6687d` pinned at commit
`4864fd681346119cf17417f82934a8ce05d88ff6`.

The implementation was adapted from the current BC250 SteamOS Real Toolkit,
which builds the same driver with `force=true`. This Fedora deployment uses a
checksum-pinned source file, builds against the running stock kernel, exposes
only the qualified channel with `fan_mask`, and manages loading and control
through Ansible. Rerun the fan playbook after booting a new kernel so the
module is built for it.

## Safety gates

Cooling is part of the normal `site.yml` dependency chain as
`playbooks/15-cooling.yml`, immediately after base configuration and before CU,
TTM, performance, Vulkan, llama.cpp, and inference services. It processes one
board at a time and fails the deployment unless the configured channel has
already been positively qualified and live PWM/RPM validation succeeds.

For qualification or maintenance, the dedicated playbook hard-fails unless
exactly one host is selected:

```bash
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/25-fan-control.yml --check --diff --limit bowie
ansible-playbook -i inventories/production/hosts.yml \
  playbooks/25-fan-control.yml --limit bowie
```

Before setting `bc250_fan_control_channel_validated: true`, record the fan's
firmware-mode RPM, briefly command a higher PWM value, confirm the corresponding
tachometer increase, and restore `pwmN_enable=2` in an unconditional cleanup
path. Repeat on the second board only after restoring the first.

The qualified production boards both use channel 2, labeled `Pump Fan`:

| Node | Firmware RPM | Full-PWM RPM | Result |
| --- | ---: | ---: | --- |
| Bowie | 1716 | 3108 | channel 2 confirmed |
| Crockett | 1697 | 3045 | channel 2 confirmed |

Do not copy this assignment to an untested board.

`bc250_cooling_bypass: true` is an explicit diagnostic bypass only. It skips
cooling setup, marks the host non-production-qualified, and the cluster role
refuses to enable RPC or coordinator services while the bypass is active.

## Automatic curve and fail-safe behavior

The conservative curve is 45% at 40°C, 55% at 55°C, 70% at 65°C, 90% at
75°C, and 100% at 80°C. The controller interpolates using the hottest GPU edge
temperature, updates every two seconds, and logs temperature, commanded fan
percentage, and RPM every ten seconds.

On a normal stop or controller error, it returns the EC to standard hwmon
automatic mode (`pwm2_enable=2`). The systemd unit repeats that restoration in
`ExecStopPost` and restarts after failures. A zero tachometer reading is a
controller failure. A systemd watchdog also terminates a stalled controller so
`ExecStopPost` can return the EC to firmware mode before restart.

```bash
sudo journalctl -u bc250-fan-control.service -f
```

The 2026-08-24 Qwen3.6 long-context qualification reached 63°C/2214 RPM on
Bowie and 66°C/2285 RPM on Crockett. Both GPUs remained at the existing
1750 MHz Moderate profile. No clocks, CU routing, TTM limits, kernel, Mesa, or
thermal thresholds changed during qualification.
