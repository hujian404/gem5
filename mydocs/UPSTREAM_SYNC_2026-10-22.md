# Upstream stable sync: 2026-10-22

## Source provenance and scope

| Item | Revision |
|------|----------|
| Previous fork tip | `892cad827f` |
| Original upstream baseline | `a9b193c59b55e9266adaf29bb71dee3329270dc3` |
| New upstream stable | `fed0e07e930fecf1e73dce94dd44e69a5bf70c46` |
| Upstream release | `v26.0.0.0` (release commit dated 2026-10-21) |
| Merge commit | `bc51bc6009af1e3d1271c1ccffbe8bf94947754c` |
| Local backup branch | `backup/stable-before-v26-sync-20261022` |

The upstream remote's default branch is `stable`. This update keeps that
branch as the experimental baseline rather than switching to `develop`.
There were 959 upstream commits reachable from the new tip but absent from
the old fork. The old upstream tree to new upstream tree diff spans 1,734
files, with 364,699 insertions and 185,414 deletions. Merge completed without
conflicts and retained the two existing documentation commits.

All fork-specific tree changes remain under `mydocs/`. Simulator source,
configuration scripts, and build rules match upstream. Old results remain
outside the checkout under `~/work/gem5-garnet/results/`.

Sources: [upstream release notes at the pinned revision][release],
[the upstream tree][tree], and [Garnet-related history][garnet-history].

## Changes relevant to learning and reproducing experiments

- **Toolchain:** v26 selects C++20 and requires Python 3.10+. The reference
  Ubuntu 22.04 host uses GCC 11.4.0, Python 3.10.12 and SCons 4.0.1 with the
  existing user-space development headers and `python3-config` shim.
- **Garnet dedicated links:** upstream `94d5aa38b4` completes per-vnet
  dedicated links for XY routing. `Mesh_XY` supports `--per-vnet-links`,
  while the routing unit tracks output direction per vnet and checks that
  the selected port exists. Shared links remain the default. To study this
  feature, explicitly select `--routing-algorithm=1`; simply naming the
  topology `Mesh_XY` does not change the routing-algorithm default of 0.
- **SimpleNetwork profiling:** upstream `46724147da` adds route-profile
  output through `--simple-trace-routes` in `configs/network/Network.py`.
  This feature belongs to SimpleNetwork; it is not a Garnet tracing option.
- **Simulation exits:** the synthetic tester now calls
  `exitSimulationLoopClassic` instead of the deprecated `exitSimLoop`.
  Its tick-based stop comparison remains unchanged.
- **Broader API migration:** standard-library processor constructors now
  require `clk_freq`; PyStats snapshots require one root; simulation-exit
  APIs and RISC-V profile selection have changed. See the pinned release
  notes before reusing older CPU, full-system, or statistics tutorials.
- **New study directions:** v26 adds CHI-TLM integration and CustomMesh
  cache hierarchies, RISC-V KVM and BF16 extensions, Arm SME2/SME2.1, and
  expanded AMD GPU support. These are upstream features, not validated by
  the standalone Garnet checks below.

## Timing and interpretation corrections

The option `--sim-cycles` is passed to `sim_cycles`, but the tester compares
it directly with `curTick()`. Always inspect the pinned source and the saved
configuration instead of relying on the option's help text.

The default configuration uses a 1 THz tick frequency, a 1 GHz tester clock
(`system.clk_domain.clock=1000`) and a 2 GHz Ruby clock
(`system.ruby.clk_domain.clock=500`). Thus:

| Stop threshold | Simulated time | Tester cycles | Ruby cycles |
|----------------|----------------|---------------|-------------|
| `10000` | 10 ns | 10 | 20 |
| `1000000` | 1 microsecond | 1,000 | 2,000 |
| `500000000` | 0.5 millisecond | 500,000 | 1,000,000 |
| `1000000000` | 1 millisecond | 1,000,000 | 2,000,000 |

Latencies in the Garnet statistics are ticks: divide by 500 for Ruby cycles
with this configuration. Default router and link latencies are each 1
cycle. Previously documented division by 1,000 gave tester cycles rather
than Ruby cycles. The original result directories retain their old names.

Injected minus received flits can still be in flight at termination. These
runs do not drain the network, so that difference is not a packet-loss
measurement. Neither a single injection rate nor a short run proves
steady state or establishes the saturation throughput.

## Validation on the reference host

The single-protocol `Garnet_standalone` rebuild passed with `-j32` and
`scons: done building targets.` The resulting binary reports **26.0.0.0**,
compiled `Oct 22 2026 23:25:15`, and is 873,405,152 bytes. All 11 runs below
completed successfully, reported `Network Tester completed simCycles`,
produced finite latency statistics, and received flits.

All runs use 16 testers, 16 directories, a 4x4 Mesh_XY, injection rate 0.1,
1 GHz tester and 2 GHz Ruby clocks. The eight short pattern checks use
1,000,000 ticks with routing algorithm 0 and 48 internal links.

| Traffic pattern | Avg flit latency (ticks) | Injected flits | Received flits |
|-----------------|--------------------------|----------------|----------------|
| uniform_random | 6042.102440 | 3740 | 3729 |
| tornado | 4814.176245 | 3674 | 3654 |
| bit_complement | 7565.301072 | 3674 | 3637 |
| bit_reverse | 6049.452055 | 3674 | 3650 |
| bit_rotation | 5412.127019 | 3674 | 3653 |
| neighbor | 4814.176245 | 3674 | 3654 |
| shuffle | 5450.095969 | 3674 | 3647 |
| transpose | 6067.745474 | 3674 | 3646 |

The three patterns measured in the old v25 report reproduce the saved
latencies and flit counts at this configuration. This is a narrow baseline
comparison, not a full cross-version regression suite.

Additional checks all use uniform_random:

| Check | Ticks | Routing algorithm | Internal links | Avg flit latency (ticks) | Injected / received flits |
|-------|-------|-------------------|----------------|--------------------------|---------------------------|
| Longer baseline | 1,000,000,000 | 0 | 48 | 6032.055925 | 3728783 / 3728749 |
| Explicit XY, shared links | 1,000,000 | 1 | 48 | 6042.102440 | 3740 / 3729 |
| Explicit XY, per-vnet links | 1,000,000 | 1 | 144 | 6012.335747 | 3740 / 3729 |

The longer run covers 1,000,000 tester cycles and 2,000,000 Ruby cycles.
It has average network latency 5032.034471 ticks and average queueing latency
1000.021455 ticks. The per-vnet case confirms that the new option produces
three dedicated internal links per direction rather than one shared link;
its short latency result is not a claim about area-normalized performance.

Host-local evidence, outside the source checkout:

- `logs/v26-garnet-build.log`: build output and dependency warnings.
- `logs/v26-build-info.txt`: binary version and build configuration.
- `logs/v26-sync-revisions.txt`: merge and upstream revisions used to build.
- `logs/v26-validation.log`: initial build and pattern-run script output.
- `logs/v26-validation-summary.json`: checked statistics and clock/link
  configuration for all 11 runs.
- `results/v26-sync-20261022/<run>/`: `stats.txt`, `config.ini`,
  `config.json`, and `stdout.log` for each run.

The longer run's output directory was corrected from an initial
`uniform_random_1M_ruby_cycles_inj0.1` label to
`uniform_random_2M_ruby_cycles_inj0.1` after verifying the Ruby clock. The
original command line and script completion label remain in the raw logs;
the threshold was always 1,000,000,000 ticks.

Capstone, libpng, HDF5 and zstd support are unavailable in this build.
They are optional features and were not needed by the Garnet tests.
The historical `build/ALL/gem5.opt` was not rebuilt or validated for v26.
CPU/full-system workloads, checkpoints, other protocols, saturation sweeps
and the full upstream regression suite were not exercised.

Documentation validation includes `git diff --check`, local Markdown file
link checks and a check that the fork's tree changes outside `mydocs/` are
empty relative to the pinned upstream revision. Build artifacts and raw
simulation results are not committed to the fork.

## Reproduce this validation

```bash
cd ~/work/gem5-garnet/gem5
export PATH="$HOME/.local/bin:$HOME/.local/opt/scons/usr/bin:$PATH"
export PYTHONPATH="$HOME/.local/opt/scons/usr/lib/python3/dist-packages${PYTHONPATH:+:$PYTHONPATH}"
scons build/Garnet_standalone/gem5.opt -j32
build/Garnet_standalone/gem5.opt --build-info

# Use a fresh output root for each campaign to preserve older results.
OUT="$HOME/work/gem5-garnet/results/v26-reproduction-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUT"
for pattern in uniform_random tornado bit_complement bit_reverse \
               bit_rotation neighbor shuffle transpose; do
  build/Garnet_standalone/gem5.opt \
    -d "$OUT/${pattern}_1M_ticks_inj0.1" \
    configs/example/garnet_synth_traffic.py \
    --num-cpus=16 --num-dirs=16 --network=garnet \
    --topology=Mesh_XY --mesh-rows=4 \
    --sim-cycles=1000000 --injectionrate=0.1 --synthetic="$pattern"
done

# One million tester cycles = two million Ruby cycles at the defaults.
build/Garnet_standalone/gem5.opt \
  -d "$OUT/uniform_random_2M_ruby_cycles_inj0.1" \
  configs/example/garnet_synth_traffic.py \
  --num-cpus=16 --num-dirs=16 --network=garnet \
  --topology=Mesh_XY --mesh-rows=4 \
  --sim-cycles=1000000000 --injectionrate=0.1 --synthetic=uniform_random
```

For dedicated per-vnet links, add `--routing-algorithm=1 --per-vnet-links`
and choose a separate output directory. This changes network resources, so
it is a separate experiment rather than a replacement baseline.

## Future updates without rewriting shared history

Start with a clean worktree. If there are local edits, review and preserve
them before merging; do not automatically discard or stash them.

```bash
cd ~/work/gem5-garnet/gem5
git status --short --branch
git fetch origin
git fetch upstream --tags
git switch stable
git merge --ff-only origin/stable
git branch "backup/stable-before-sync-$(date +%Y%m%d-%H%M%S)"
git merge --no-edit upstream/stable

# No output here means no fork-specific edits outside mydocs/.
git diff --name-only upstream/stable HEAD -- . ':!mydocs'
git merge-base --is-ancestor upstream/stable HEAD

# Rebuild and validate, then update the dated report and document index.
git diff --check
git add mydocs
git commit -m "docs: Update upstream sync and reproduction notes"
git push origin stable
```

Keep upstream fetch-only in this workflow. No force-push is needed. A
future upstream revision can change commands or timing semantics; recheck
the actual source instead of applying this report to an unpinned checkout.

[release]: https://github.com/gem5/gem5/blob/fed0e07e930fecf1e73dce94dd44e69a5bf70c46/RELEASE-NOTES.md
[tree]: https://github.com/gem5/gem5/tree/fed0e07e930fecf1e73dce94dd44e69a5bf70c46
[garnet-history]: https://github.com/gem5/gem5/commits/fed0e07e930fecf1e73dce94dd44e69a5bf70c46/src/mem/ruby/network/garnet
