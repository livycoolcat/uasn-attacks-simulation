# Validation tests (hand-built topologies)

These check the attack code on a topology small enough to follow packet by
packet. `--trace=1` prints one `TRACE` line per DBR decision (QUEUE, SEND,
HEAR_SHALLOWER ... CANCELLED, SPOOF, BLACKHOLE_DROP). Node addresses in the
trace are relay index + 1 (relay 0 in the file is node 1); the source and sink
are the last two nodes.

## ladder.csv

Two columns (L at x=70, R at x=130), four layers. Adjacent layers hear each
other; layers two apart do not. Every layer has two candidate forwarders, so
DBR has redundancy at every hop: this is the case where a silent blackhole
should be hidden by DBR and a depth spoofer should not.

```
      SINK (z=300)
   L4(240)  R4(245)      nodes 7, 8
   L3(180)  R3(185)      nodes 5, 6
   L2(120)  R2(125)      nodes 3, 4
   L1(60)   R1(65)       nodes 1, 2
      SOURCE (z=0)
```

Run from the ns-3.41 folder:

```bash
L="--positions=$HOME/ns3-workspace/uasn-blackhole-attacks-simulation/tests/ladder.csv --width=200 --length=200 --depth=300 --range=100"
./ns3 run "UasnDbr $L"                                          # 50/50
./ns3 run "UasnDbr $L --attack=blackhole  --badList=2"          # silent at L2: 50/50 (hidden)
./ns3 run "UasnDbr $L --attack=blackhole  --badList=2,3"        # silent at L2+R2 (a cut): 0/50
./ns3 run "UasnDbr $L --attack=depthspoof --badList=2"          # ONE spoofer at L2: 0/50
./ns3 run "UasnDbr $L --attack=depthspoof --badList=6"          # spoofer next to the sink: 50/50
./ns3 run "UasnDbr $L --attack=depthspoof --badList=2 --packets=1 --trace=1"
```

Expected results (checked on ns-3.41, Nov 20):

| Run | Delivered | What the trace shows |
|---|---|---|
| no attack | 50/50 | both nodes of each layer forward; L2 loses both layer-1 copies to a collision |
| silent L2 | 50/50 | L2 drops, R2 forwards as before |
| silent L2+R2 | 0/50 | nothing crosses layer 2 |
| spoof L2 | 0/50 | L2 sends a copy claiming z=220; L3 and R3 had the packet queued and CANCEL it |
| spoof L4 | 50/50 | the spoofed copy collides at R4, so R4 still reaches the sink |

Things this test shows that matter for the report:

- **Collisions are real.** The broadcast MAC loses packets whose receptions
  overlap (each one lasts about 75 ms). On the ladder L2 never gets a copy, so
  silencing it changes nothing.
- **Same-depth nodes do not suppress each other.** With `DBR_MAX_DELAY = 0.2`
  and delta = R/2, the holding times of two nodes 5 m apart in depth differ by
  about 40 ms, which is less than the transmission time plus propagation, so
  both forward.
- **The spoofer reacts to any copy it hears, even one from a shallower node**
  (on the ladder, L2 first hears the packet from R2). It then kills the packet
  one layer further up.

## Counters printed by every run (also in the CSV)

| Column | Meaning |
|---|---|
| `cancelledBySpoof` | an honest node had the packet queued and cancelled it because of a spoofed copy |
| `spoofHeardFirst` | the spoofed copy arrived before the node had queued the packet |
| `cancelledHonest` | normal DBR suppression (cancelled by an honest shallower copy) |
| `requeuedAfterSuppress` | a node that had been suppressed later heard a deeper copy and queued the packet again |
| `requeueBlocked` | the same, but blocked because `--suppressedIsFinal=1` |

These counters use simulation-only knowledge (which nodes are attackers); no
real node could compute them.

## `--suppressedIsFinal`

Aqua-Sim's DBR only remembers packets a node has *sent*. A node that cancelled
a packet will queue it again if it later hears a copy from a deeper node.
Zuba et al. (2013) describe DBR as deciding on the *first* reception, so a
suppressed node stays quiet. `--suppressedIsFinal=1` gives that behaviour.
The default (0) is Aqua-Sim's original behaviour. The choice changes how strong
depth spoofing is (see `results/pilot_depthspoof.csv`), so report which one
you used, or both.
