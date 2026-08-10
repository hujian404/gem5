# Test Results

How to run the gem5 Garnet synthetic-traffic test and the results captured on
the reference host. The build is covered in
[BUILD_AND_RUN_GEM5.md](BUILD_AND_RUN_GEM5.md); host/SSH setup in
[REMOTE_SERVER_SETUP.md](REMOTE_SERVER_SETUP.md).

> **No passwords, private keys, host IPs, or other secrets are included.**
> Replace every `<placeholder>` with your own value.

Prerequisites: [REMOTE_SERVER_SETUP.md](REMOTE_SERVER_SETUP.md) and
[BUILD_AND_RUN_GEM5.md](BUILD_AND_RUN_GEM5.md) must be done (i.e.
`~/work/gem5-garnet/gem5/build/Garnet_standalone/gem5.opt` exists).

---

## 1. Run the Garnet synthetic-traffic test

The Garnet synthetic-traffic config must be run with the **single-protocol**
`Garnet_standalone` build (see
[BUILD_AND_RUN_GEM5.md §3](BUILD_AND_RUN_GEM5.md) for why the multi-protocol
`ALL` build cannot run the legacy `configs/ruby/` path directly).

```bash
cd ~/work/gem5-garnet/gem5
export PATH="$HOME/.local/bin:$HOME/.local/opt/scons/usr/bin:$PATH"
build/Garnet_standalone/gem5.opt \
  -d ~/work/gem5-garnet/results/uniform_random_1M_inj0.1 \
  configs/example/garnet_synth_traffic.py \
  --num-cpus=16 --num-dirs=16 \
  --network=garnet --topology=Mesh_XY --mesh-rows=4 \
  --sim-cycles=1000000 --injectionrate=0.1 \
  --synthetic=uniform_random
# stats → ~/work/gem5-garnet/results/uniform_random_1M_inj0.1/stats.txt
```

> **Topology name.** In gem5 v25 the mesh topology module is `Mesh_XY.py`
> (class `Mesh_XY`), not the legacy `Mesh`. `configs/ruby/Ruby.py`'s
> `create_topology` does `getattr(import_module("topologies.<topo>"), "<topo>")`,
> so the `--topology` value must equal the class name. Available topologies
> live under `configs/topologies/` (`Mesh_XY`, `Mesh_westfirst`,
> `MeshDirCorners_XY`, `CustomMesh`, `Crossbar`, `CrossbarGarnet`, `Pt2Pt`,
> `Cluster`).

> **Output discipline.** Always use `-d ~/work/gem5-garnet/results/<name>` to
> keep all run outputs under `results/`. Never omit `-d` — that would write to
> `m5out/` inside the source tree and clutter the repo. The naming convention
> is `<pattern>_<cycles>cyc_inj<rate>` (e.g. `tornado_1M_inj0.1`).

### 1.1 Single smoke test

```bash
cd ~/work/gem5-garnet/gem5
build/Garnet_standalone/gem5.opt \
  -d ~/work/gem5-garnet/results/smoke_10k_inj0.1 \
  configs/example/garnet_synth_traffic.py \
  --num-cpus=16 --num-dirs=16 --network=garnet --topology=Mesh_XY --mesh-rows=4 \
  --sim-cycles=10000 --injectionrate=0.1 --synthetic=uniform_random
```

### 1.2 Sweep traffic patterns

```bash
cd ~/work/gem5-garnet/gem5
for pattern in uniform_random tornado bit_complement bit_reverse bit_rotation \
               neighbor shuffle transpose; do
  build/Garnet_standalone/gem5.opt \
    -d ~/work/gem5-garnet/results/${pattern}_1M_inj0.1 \
    configs/example/garnet_synth_traffic.py \
    --num-cpus=16 --num-dirs=16 --network=garnet --topology=Mesh_XY --mesh-rows=4 \
    --sim-cycles=1000000 --injectionrate=0.1 --synthetic=$pattern
done
# view all results
grep 'average_flit_latency' ~/work/gem5-garnet/results/*/stats.txt
```

Statistics are written to `<outdir>/stats.txt` (use `-d <outdir>`); the run
also prints `info: Garnet version 3.0` and exits with
`Exiting @ tick <N> because Network Tester completed simCycles`.

---

## 2. What the test exercises

The Garnet on-chip network model (`src/mem/ruby/network/garnet/`) is a
flit-level wormhole-router NoC. The synthetic-traffic config instantiates
`GarnetSyntheticTraffic` testers attached to a Ruby/Garnet mesh and injects a
chosen traffic pattern for `--sim-cycles` cycles. It exercises:

| Component | File | Role |
|-----------|------|------|
| `GarnetNetwork` | `GarnetNetwork.{hh,cc,py}` | sets up routers/links, collects stats |
| `Router` | `Router.{hh,cc}` | pipeline wakeup (Input/Output units, allocator, crossbar) |
| `InputUnit` / `OutputUnit` | `InputUnit.{hh,cc}` / `OutputUnit.{hh,cc}` | VC buffering, route compute, credits |
| `SwitchAllocator` | `SwitchAllocator.{hh,cc}` | SA-I (input VC arb) + SA-II (output port arb) |
| `CrossbarSwitch` | `CrossbarSwitch.{hh,cc}` | flit traversal |
| `RoutingUnit` | `RoutingUnit.{hh,cc}` | route computation (XY for `Mesh_XY`) |
| `NetworkInterface` | `NetworkInterface.{hh,cc}` | protocol↔network packet conversion, credit flow control |
| `NetworkLink` / `flit` / `flitBuffer` | `*.cc` | link traversal, flit storage |

---

## 3. Captured results (reference host)

Configuration: `build/Garnet_standalone/gem5.opt` (gem5 25.1.0.1, stable @
`a9b193c59b`), 16-node 4×4 `Mesh_XY`, `--network=garnet`,
`--sim-cycles=1000000`, `--injectionrate=0.1`. Ruby clock = 1 GHz
(`system.clk_domain.clock = 1000` ticks), global frequency 1 THz, so
**1 Ruby cycle = 1000 ticks**. Each run ~0.5 s of host time.

Simulation summary (uniform_random run):

```
simSeconds        0.000001        # 1 µs simulated
simTicks          1000000
finalTick         1000000
simFreq           1000000000000   # 1 THz
hostSeconds       0.52
hostTickRate      1923084         # ticks/s
hostMemory        736588          # bytes
```

Garnet network statistics across three traffic patterns (ticks; cycles in
parentheses, = ticks/1000):

| Traffic pattern | avg_flit_latency | avg_flit_network_latency | avg_flit_queueing_latency | avg_packet_latency | flits injected | flits received |
|-----------------|------------------|--------------------------|---------------------------|--------------------|----------------|----------------|
| `uniform_random` | 6042.10 (6.04) | 5042.10 (5.04) | 1000 (1.00) | 6023.29 (6.02) | 3740 | 3729 |
| `tornado`        | 4814.18 (4.81) | 3814.18 (3.81) | 1000 (1.00) | 4821.79 (4.82) | 3674 | 3654 |
| `bit_complement` | 7565.30 (7.57) | 6565.30 (6.57) | 1000 (1.00) | 7529.69 (7.53) | 3674 | 3637 |

All three runs exited with code 0 and the line
`Exiting @ tick 1000000 because Network Tester completed simCycles`.

### 3.1 Interpretation

- `average_flit_network_latency` ≈ hop count × (router pipeline + link)
  latency. For a 4×4 mesh with XY routing, the average hop distance is ~2.7;
  with a default 4-stage router pipeline + 1-cycle link this gives ~5 cycles
  for `uniform_random` — matching the observed 5042 ticks (5.04 cycles).
- `average_flit_queueing_latency` is 1 cycle (1000 ticks) across all patterns
  at this low injection rate (0.1) — little contention.
- `bit_complement` has the longest paths / highest contention → highest
  latency; `tornado` (minimal, low-contention) → lowest. This is the expected
  qualitative ordering for a Garnet mesh.

### 3.2 Verification run (smoke test)

A short `--sim-cycles=10000` run also completes with exit code 0 and emits
`info: Garnet version 3.0`, confirming the binary and config are functional;
flit counts are small at that duration, so the 1,000,000-cycle runs above are
used for the reported statistics.

---

## 4. Known issues / failure analysis

### 4.1 `ALL` build cannot run the legacy garnet config

**Symptom.** Running `garnet_synth_traffic.py` with the `build/ALL` binary:

```
ERROR: You must specify a protocol with --protocol <protocol>.
...
NameError: name 'L1Cache_Controller' is not defined
  at configs/ruby/Garnet_standalone.py: create_system
```

**Root cause.** `build_opts/ALL` sets `USE_MULTIPLE_PROTOCOLS=y`; in a
multi-protocol build the SLICC controller classes are namespaced
(`Garnet_standalone_L1Cache_Controller`) and the legacy
`configs/ruby/Garnet_standalone.py` expects them at top level. gem5 v25 also
prints `The Ruby.py file is deprecated. ... use the standard library`.

**Fix (host-side, no source change).** Build the single-protocol
`build/Garnet_standalone/gem5.opt` (`build_opts/Garnet_standalone`,
`PROTOCOL="Garnet_standalone"`) and run the config with that binary. No
`--protocol` flag is needed for a single-protocol build. The `ALL` build
remains useful as a comprehensive baseline for other configs (and for the new
standard-library API), but the legacy `garnet_synth_traffic.py` config targets
the single-protocol build.

### 4.2 `--topology=Mesh` not found

**Symptom.** `ModuleNotFoundError: No module named 'topologies.Mesh'`.

**Root cause.** gem5 v25 renamed `Mesh.py` → `Mesh_XY.py` (class `Mesh_XY`).

**Fix.** Use `--topology=Mesh_XY` (see §1 note).

No source modifications were made for either issue — both are addressed by the
correct build target / command-line option.

---

## 5. Troubleshooting

| Symptom | Fix |
|---------|-----|
| `NameError: L1Cache_Controller is not defined` | [BUILD §3](BUILD_AND_RUN_GEM5.md) — use `build/Garnet_standalone/gem5.opt`, not `ALL` |
| `ModuleNotFoundError: No module named 'topologies.Mesh'` | §1 / §4.2 — use `--topology=Mesh_XY` |
| `You must specify a protocol with --protocol` | [BUILD §3](BUILD_AND_RUN_GEM5.md) — use the single-protocol `Garnet_standalone` build |
| `Can't find a suitable python-config` (build) | [BUILD §2.2.3](BUILD_AND_RUN_GEM5.md) — install `python3-config` shim |
| `Check failed for Python.h header` (build) | [BUILD §2.2.1](BUILD_AND_RUN_GEM5.md) + §2.2.3 |
| `gem5.opt: command not found` | [BUILD §3](BUILD_AND_RUN_GEM5.md) — build first |
| GitHub push hangs | [SETUP §2.2](REMOTE_SERVER_SETUP.md) — use SSH |

---

## 6. Where things live

```
~/work/gem5-garnet/                              # unified experiment root
├── gem5/                                        # gem5 checkout (branch stable)
│   ├── SConstruct                              #   build entry (uses PYTHON_CONFIG)
│   ├── build_opts/Garnet_standalone            #   single-protocol preset (for garnet)
│   ├── build_opts/ALL                          #   comprehensive preset (all ISAs + protocols)
│   ├── build/Garnet_standalone/gem5.opt        #   built binary (~823 MB, gem5 25.1.0.1)
│   ├── build/ALL/gem5.opt                      #   comprehensive binary (optional, ~11 GB)
│   ├── configs/example/garnet_synth_traffic.py #   Garnet synthetic-traffic config
│   ├── configs/topologies/Mesh_XY.py           #   XY-routing mesh topology
│   ├── src/mem/ruby/network/garnet/            #   Garnet NoC model source
│   ├── mydocs/                                 #   this documentation
│   └── m5out/                                  #   (empty — always use -d to redirect)
├── results/                                     # experiment run outputs
│   ├── uniform_random_1M_inj0.1/               #   stats.txt + config.ini + stdout.log
│   ├── tornado_1M_inj0.1/
│   ├── bit_complement_1M_inj0.1/
│   └── smoke_10k_inj0.1/
├── logs/                                        # build logs
│   ├── gem5-build-all.log                      #   ALL build log
│   └── gem5-build-all.pid
├── build-log.txt                               # Garnet_standalone build log
└── build-log.pid

~/.local/opt/python3-dev/usr/                   # extracted python3.10-dev headers + .so symlink
~/.local/opt/scons/usr/                         # extracted scons 4.0.1
~/.local/bin/python3-config                     # BUILD §2.2.3 — Python-detect shim
~/.ssh/id_ed25519_github                        # SETUP §2.2 — GitHub SSH key
~/.ssh/config                                   # SETUP §2.2 — pin key to github.com
```
