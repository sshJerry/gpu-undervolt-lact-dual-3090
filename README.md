# gpu-undervolt-lact-dual-3090

True undervolting for two RTX 3090s under a continuous vLLM workload. A lift of
[sshJerry/gpu-undervolt](https://github.com/sshJerry/gpu-undervolt) ideas into
permanent, crowned V/F curves via the
[LACT](https://github.com/ilya-zlobintsev/LACT) daemon, layered on top of
Zagitta's [gpu-unlock](https://github.com/Zagitta/gpu-unlock) kernel module.

The GPU unlock doubles tensor core throughput on GeForce and this setup keeps
the extra compute paid for at roughly 10 percent less voltage, which shows up
as fewer watts and lower temps with identical tokens/sec.

## Stack, in order of what runs when

1. **gpu-unlock 2.0.0** (DKMS module, lives in the initramfs). Patches the
   tensor speed selectors before the NVIDIA driver binds the card:

   ```
   journalctl -k -b | grep gpu_unlock: unlocked
   pci 0000:07:00.0: gpu_unlock: unlocked
   pci 0000:08:00.0: gpu_unlock: unlocked
   ```

   Opt in from the kernel command line with `gpu_unlock.targets=all`.

2. **LACT 0.10.1 headless daemon** (`lactd`), one `nvidia_gpu_vf_curve` per
   card. Each curve is "crowned": positive clock offsets on the low voltage
   points so a target clock is reachable at a lower voltage, flattened above
   it so the driver settles there. Applied at all times, no rules, no power
   caps. Idle is untouched because points below the crown are stock, so the
   cards still fall to P8.

3. **Fan curves per card** in the same daemon, 30 percent floor to 100 percent
   at 75 C.

## Hardware

| Card | Board | PCI | VBIOS | Role |
|------|-------|-----|-------|------|
| RTX 3090 2-fan | Dell OEM (subvendor 1028:3880) | 08:00.0 | 94.02.26.80.80 | bottom slot, cooling limited |
| RTX 3090 3-fan | Gigabyte (subvendor 1458:4043) | 07:00.0 | 94.02.26.88.3C | top slot |

- Driver 595.84, kernel 7.0.14-19-pve (Proxmox VE), headless host.
- The bottom card exhausts into the top card's intake, so the top card is the
  one that runs hot. In practice the Dell on the bottom showed the
  higher temps on its weaker 2-fan cooler, which is why it runs the deepest
  crown (lower settle clock and voltage) and the 3-fan Gigabyte gets a slightly
  lighter crown.
- Workload: vLLM, tensor parallel over both cards, MTP speculative decoding,
  long agentic contexts, decode is memory bandwidth bound, prefill is compute
  heavy and is where the unlock pays.

## Results

| Metric | Dell (bottom) | Gigabyte (top) |
|--------|---------------|----------------|
| Load voltage | 737 mV | 750 mV |
| Load clock | 1590-1605 MHz, pinned | 1605-1620 MHz, pinned |
| Load power | 245-268 W | 264-272 W |
| Load temp | 66-74 C | 65-74 C |
| Idle power / temp / fan | 14 W / 33 C / 32 percent | 29 W / 41 C / 0 percent |

Context for those numbers:

- At these clocks the stock curve pairs are roughly 820-825 mV, so both cards
  now run about 9-11 percent below stock voltage at the same clock.
- Before tuning, the cards chased the stock curve at 345 W class and 74-79 C
  while power limiting. Now the same decode does 245-272 W and tokens/sec is
  unchanged because decode is bandwidth bound. Prefill keeps most of the
  doubled tensor rate.
- nvidia-smi does not expose voltage, read it through the daemon:

  ```bash
  stat() { ( printf '%s\n' "{\"command\":\"device_stats\",\"args\":{\"id\":\"$ID\"}}"; sleep 1 ) \
    | nc -N -U /run/lactd.sock | jq -c '{v:.data.voltage.gpu, mhz:.data.clockspeed.gpu_clockspeed, w:.data.power.current, gt:.data.temps.GPU.current}'; }
  ```

- A single sample is noisy, take several during a sustained stretch.

## Regenerating a crown curve

Never replay an old offset map blindly, the driver rebases the curve each
boot, so generate from the current snapshot:

```bash
ID=10DE:2204-1458:4043-0000:07:00.0   # or the 1028 card
TARGET=1605                            # settle clock
FLOOR=1400                             # first stock point raised to TARGET

CURVE=$( printf '%s\n' "{\"command\":\"device_clocks_info\",\"args\":{\"id\":\"$ID\"}}" | nc -N -U /run/lactd.sock \
| jq -c --argjson t $TARGET --argjson f $FLOOR \
  '[.data.table.value.gpu_vf_curve[] | {key:(.index|tostring),
    value:{clockspeed_offset: (if .base_freq >= $f then ($t - .base_freq) else 0 end)}} 
    | select(.value.clockspeed_offset != 0)] | from_entries' )

req() { printf '%s\n' "$1" | nc -N -U /run/lactd.sock; }
req "{\"command\":\"get_gpu_config\",\"args\":{\"id\":\"$ID\"}}" | jq -c .data > /tmp/cfg.json
jq -nc --argjson cfg "$(cat /tmp/cfg.json)" --argjson curve "$CURVE" \
  '{command:"set_gpu_config", args:{id:"'"$ID"'", config:($cfg + {nvidia_gpu_vf_curve:$curve})}}' > /tmp/set.json
{ printf '%s\n%s\n' "$(cat /tmp/set.json)" \
    '{"command":"confirm_pending_config","args":{"command":"confirm"}}'; sleep 2; } \
  | nc -N -U /run/lactd.sock
```

The daemon persists confirmed curves into `/etc/lact/config.yaml` itself, both
crown maps live in this repo next to it.

## Monitoring and failure signature

```bash
nvidia-smi dmon -s pucvmet          # sbecc / dbecc are the columns to keep at 0
journalctl -k --no-pager | grep -i xid
```

Known behaviors documented along the way:

- Per-clock NVLM offsets alone (`gpu_clock_offsets`) act as underclocking on a
  power capped workload, not undervolting, so this setup went the curve route.
- OEM cooler note: the Dell 2-fan card sits at its cooler's equilibrium no
  matter what point it gets, hence it carries the deeper crown while the
  3-fan Gigabyte takes a slightly lighter one. The copper-limited bottom card
  sets the pace for both.

## Files in this repo

- `etc/lact/config.yaml`, fan curves plus both crowned maps, drop in at
  `/etc/lact/config.yaml`, `systemctl restart lactd`
- `gb_crown.txt` / `dell_crown2.txt` generated crown offset maps
- See also
  [Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock) for the tensor
  unlock this is stacked on
