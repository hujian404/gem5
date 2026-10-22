# gem5 — User Documentation

This folder contains user-contributed documentation for building and running
**gem5** (with a focus on the **Garnet** on-chip network model at
`src/mem/ruby/network/garnet/`) on a remote Linux server **without sudo
privileges**, using only user-space tools.

It captures every workaround needed on a stock Ubuntu 22.04 host that ships
**without** `scons`, `pip`, or the `python3-dev` headers — read the "Why" notes
if you hit the same errors on a different machine.

> **No passwords, private keys, host IPs, or other secrets are included.**
> Replace every `<placeholder>` with your own value.

## Documents

| Document | Purpose |
|----------|---------|
| [REMOTE_SERVER_SETUP.md](REMOTE_SERVER_SETUP.md) | Host & toolchain assumptions, SSH access (local → remote, remote → GitHub) |
| [BUILD_AND_RUN_GEM5.md](BUILD_AND_RUN_GEM5.md) | Clone, host-environment workarounds, compile `gem5.opt`, run Garnet |
| [TEST_RESULTS.md](TEST_RESULTS.md) | How to run the Garnet synthetic-traffic test, expected results |
| [UPSTREAM_SYNC_2026-10-22.md](UPSTREAM_SYNC_2026-10-22.md) | v26.0 upstream merge, compatibility changes, validation and update procedure |

## Recommended reading order

1. [REMOTE_SERVER_SETUP.md](REMOTE_SERVER_SETUP.md) — one-time host & SSH setup.
2. [BUILD_AND_RUN_GEM5.md](BUILD_AND_RUN_GEM5.md) — install the user-space
   toolchain (scons + python3-dev headers + `python3-config` shim), compile
   `gem5.opt`, and run a Garnet NoC config.
3. [TEST_RESULTS.md](TEST_RESULTS.md) — run the Garnet synthetic-traffic test
   and check the expected network statistics.

## What is already fixed in this fork

This fork is a **clean baseline** of upstream `gem5/gem5` on the `stable`
branch (see the baseline commit recorded in
[REMOTE_SERVER_SETUP.md](REMOTE_SERVER_SETUP.md)). **No source-level or
build-system changes were needed** to compile gem5 on the reference host — the
build succeeds using host-environment workarounds only.

The current upstream baseline is **v26.0.0.0**, commit
`fed0e07e930fecf1e73dce94dd44e69a5bf70c46`, merged on 2026-10-22.
The fork's changes relative to that upstream tree are confined to `mydocs/`.
Normal merges preserve the fork's existing documentation commits; updating
this fork does not require rebasing or force-pushing shared history.

The workarounds needed to build (`python3-dev` headers, `scons`, the
`python3-config` shim, `~/.bashrc` env exports) are environment-level and live
**outside** any git repo — they must be re-applied on each fresh host. See
[BUILD_AND_RUN_GEM5.md §2.2](BUILD_AND_RUN_GEM5.md).

## Directory layout

All gem5 source and Garnet experiment files are consolidated under
`~/work/gem5-garnet/`:

- **Source**: `~/work/gem5-garnet/gem5/` (gem5 checkout, branch `stable`)
- **Binary**: `~/work/gem5-garnet/gem5/build/Garnet_standalone/gem5.opt`
  (rebuild after updating; see the dated validation report)
- **Results**: `~/work/gem5-garnet/results/<pattern>_<threshold>ticks_inj<rate>/`
- **Build logs**: `~/work/gem5-garnet/build-log.txt` (Garnet_standalone) and `~/work/gem5-garnet/logs/gem5-build-all.log` (ALL)
- **v26 validation**: `~/work/gem5-garnet/logs/v26-garnet-build.log` and
  `~/work/gem5-garnet/results/v26-sync-20261022/`

The existing `build/ALL/gem5.opt` and old result directories are historical
artifacts until explicitly rebuilt or rerun. Updating Git does not update
previously compiled binaries.

> **Timing correction.** Despite its name, `--sim-cycles` is compared with
> `curTick()` in the upstream synthetic tester. With the default 1 GHz tester
> clock, 2 GHz Ruby clock and 1 THz tick frequency, `--sim-cycles=1000000`
> means 1 microsecond: 1,000 tester cycles and 2,000 Ruby cycles. Use
> `500000000` for about 1,000,000 Ruby cycles at the default Ruby clock.
> Historical `*_1M_inj0.1` directories retain their original names.

See [REMOTE_SERVER_SETUP.md §6](REMOTE_SERVER_SETUP.md) for the full directory
tree, and [BUILD_AND_RUN_GEM5.md §1.1](BUILD_AND_RUN_GEM5.md) for the layout
after clone + build + run.

> **Output discipline.** All run commands use `-d ~/work/gem5-garnet/results/<name>`
> to keep results organized and the source tree clean. Never omit `-d` — that
> would write to `m5out/` inside the source tree.

## Scope of "gem5 garnet"

The Garnet network is gem5's on-chip network (NoC) model, part of the Ruby
memory system, located at
[`src/mem/ruby/network/garnet/`](../src/mem/ruby/network/garnet/). It models
routers (input/output units, switch allocator, crossbar, routing unit),
network interfaces, network links, and flit-level flow control. We exercise it
with the synthetic-traffic configuration
[`configs/example/garnet_synth_traffic.py`](../configs/example/garnet_synth_traffic.py),
which injects traffic patterns (uniform_random, tornado, bit_complement, …)
into a Garnet mesh and reports flit latency / throughput statistics.

The recommended build for running the synthetic-traffic config is
`build/Garnet_standalone/gem5.opt` (single-protocol preset
`build_opts/Garnet_standalone`), which the legacy
`configs/example/garnet_synth_traffic.py` config targets directly. The
comprehensive `build/ALL/gem5.opt` (all ISAs + protocols) is also available as
a general-purpose baseline; see [BUILD_AND_RUN_GEM5.md §3](BUILD_AND_RUN_GEM5.md)
for the distinction.
