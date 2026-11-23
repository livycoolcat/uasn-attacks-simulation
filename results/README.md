# Results

Saved simulation output for the report. Experiment CSVs from
`scripts/run_experiments.py` and graphs from `scripts/analyze.py` also go here
(for example `results/blackhole.csv` and `results/figures/`).

## Fix 1: original vs fixed new-packet test in DBR

Aqua-Sim NG's DBR decided "this packet is new from my own application" by a
forward count of zero. The source never increments that count, so the
source's neighbours also saw zero, added a second, empty DBR header and read
packet ID 0 for every packet. DBR forwards each ID once, so packets 2-50 were
dropped as duplicates. The fix tests header direction instead (the channel
marks every received copy UP). See the main README for details.

Same layout, no attackers, 50 packets from source to sink:

| Seed | Original test: delivered | Original: tx per delivered | Fixed test: delivered | Fixed: tx per delivered |
|---|---|---|---|---|
| 1 | 1 / 50 | 102 | 50 / 50 | 37.8 |
| 2 | 1 / 50 | 115 | 50 / 50 | 62.9 |
| 3 | 1 / 50 | 118 | 50 / 50 | 59.5 |

Files:

- `fix1_before.txt`: full output of `./ns3 run "UasnDbr --RngRun=1 --legacyNewPacketTest=1"`
- `fix1_after.txt`: full output of `./ns3 run "UasnDbr --RngRun=1"`
- `fix1_comparison.csv`: CSV lines for seeds 1-3, both versions (`legacyNewPacketTest` column: 1 = original, 0 = fixed)

To reproduce, from `~/ns3-workspace/ns-allinone-3.41/ns-3.41`:

```bash
./ns3 run "UasnDbr --RngRun=1 --legacyNewPacketTest=1"   # original
./ns3 run "UasnDbr --RngRun=1"                           # fixed
```

## Pilot: silent blackhole vs depth spoofing (Nov 20)

`pilot_depthspoof.csv`: 30 seeds x {blackhole, depthspoof} x 4 placements x
{5, 10, 20, 30}% x `suppressedIsFinal` {0, 1}, default 300 x 300 x 500 m box,
100 relays, 100 m range. A pilot to check that the spoofing attack behaves
sensibly, not final results. Each DBR variant has its own baseline rows (the
`suppressedIsFinal` column), so filter on it before averaging.
