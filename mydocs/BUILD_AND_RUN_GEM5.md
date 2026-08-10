# Build & Run gem5 (Garnet)

Step-by-step: clone the gem5 fork, install the user-space build toolchain
(no sudo), compile `gem5.opt`, and run a Garnet on-chip-network configuration
on a host prepared per [REMOTE_SERVER_SETUP.md](REMOTE_SERVER_SETUP.md).

> **No passwords, private keys, host IPs, or other secrets are included.**
> Replace every `<placeholder>` with your own value.

---

## 1. Clone the fork

gem5 keeps its third-party sources (pybind11, etc.) vendored under `ext/` in
the repository itself — **no submodules** need initializing.

```bash
# remote
mkdir -p ~/work && cd ~/work
git clone git@github.com:hujian404/gem5.git gem5
cd ~/work/gem5

# add upstream as read-only remote (the GitHub SSH key has read access to
# public upstreams even though it can only push to your fork)
git remote add upstream git@github.com:gem5/gem5.git
git fetch upstream
git checkout stable
git log -1 --oneline     # confirm baseline commit
```

---

## 2. Build fixes

### 2.1 Already applied in this fork (no action needed)

**None.** This fork is a clean baseline of upstream `gem5/gem5` `stable`. The
build compiles successfully using only the host-environment workarounds in
§2.2 — no `SConstruct`, `SConscript`, or source files were modified.

### 2.2 Local-only fixes (re-apply on each fresh clone)

> These live in the working tree / user home only. The upstream commit is
> untouched. They are **not** part of any git repo (host-specific).

The host lacks `scons`, `pip`, and the `python3-dev` headers. We obtain them
user-space from the Ubuntu mirror with `apt-get download` (no sudo) +
`dpkg -x`, and write a small `python3-config` shim so gem5's `SConstruct` can
find Python.

#### 2.2.1 Obtain `python3-dev` headers (Python.h + multiarch pyconfig.h)

gem5 embeds Python and needs `Python.h` at compile time. The
`python3.10-dev` + `libpython3.10-dev` packages provide the headers; we
extract them into `~/.local/opt/python3-dev` and point the unversioned
`libpython3.10.so` symlink at the system shared library (which already
exists).

```bash
# remote
STAGE=/tmp/gem5-hostenv
mkdir -p "$STAGE" ~/.local/opt
cd "$STAGE"
apt-get download python3.10-dev libpython3.10-dev
rm -rf pydev && mkdir -p pydev
for d in *.deb; do dpkg -x "$d" pydev; done
mkdir -p ~/.local/opt/python3-dev
rm -rf ~/.local/opt/python3-dev/usr
cp -a pydev/usr ~/.local/opt/python3-dev/usr
# Point the dev .so symlink at the system libpython3.10.so.1 (already present).
mkdir -p ~/.local/opt/python3-dev/usr/lib/x86_64-linux-gnu
ln -sfn /usr/lib/x86_64-linux-gnu/libpython3.10.so.1 \
        ~/.local/opt/python3-dev/usr/lib/x86_64-linux-gnu/libpython3.10.so
# verify
ls ~/.local/opt/python3-dev/usr/include/python3.10/Python.h    # present
ls ~/.local/opt/python3-dev/usr/include/x86_64-linux-gnu/python3.10/pyconfig.h  # present
```

#### 2.2.2 Obtain `scons` (user-space)

```bash
cd "$STAGE"
apt-get download scons                       # scons 4.0.1 on Ubuntu 22.04
rm -rf scons && mkdir -p scons
for d in scons_*.deb; do dpkg -x "$d" scons; done
mkdir -p ~/.local/opt/scons
rm -rf ~/.local/opt/scons/usr
cp -a scons/usr ~/.local/opt/scons/usr
# verify
~/.local/opt/scons/usr/bin/scons --version | head -2
```

#### 2.2.3 `python3-config` shim (`~/.local/bin/python3-config`)

gem5's `SConstruct` (`config_embedded_python`) runs
`python3-config --ldflags --includes [--embed]`, keeps only `-l/-L/-I` flags,
then `CheckHeader('Python.h')` and `CheckPythonLib()`. The host has **no**
`python3-config`, so we provide one that emits the user-space paths. Two `-I`
paths are essential: `Python.h` lives in `include/python3.10/`, and the
`pyconfig.h` it `#include`s dispatches to the multiarch
`<x86_64-linux-gnu/python3.10/pyconfig.h>`, which resolves via the **parent**
`include/` directory (on the system `/usr/include` is a built-in default
include path; in user-space we must add `-I<dev>/include` explicitly).

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/python3-config <<'SH'
#!/usr/bin/env bash
# User-space python3-config shim for gem5 (no sudo / no python3-dev).
set -u
DEV="$HOME/.local/opt/python3-dev/usr"
PYINC="$DEV/include/python3.10"
PARENT="$DEV/include"
MULTI="$DEV/include/x86_64-linux-gnu/python3.10"
PYLIB="$DEV/lib/x86_64-linux-gnu"
INCFLAGS="-I$PYINC -I$PARENT -I$MULTI"
LIBFLAGS="-L$PYLIB -lpython3.10 -ldl -lm"
out=""
for a in "$@"; do
  case "$a" in
    --includes|--cflags)        out+=" $INCFLAGS" ;;
    --ldflags)                  out+=" $LIBFLAGS" ;;
    --embed)                    out+=" $INCFLAGS $LIBFLAGS" ;;
    --prefix|--exec-prefix)     out+=" $DEV" ;;
    --version)                  out+=" 3.10.12" ;;
    *) ;;
  esac
done
echo "${out# }"
SH
chmod +x ~/.local/bin/python3-config
```

#### 2.2.4 Persist the environment in `~/.bashrc`

```bash
# append (idempotent guarded block)
cat >> ~/.bashrc <<'EOF'

# === gem5 host env (no sudo) ===
export PATH="$HOME/.local/bin:$HOME/.local/opt/scons/usr/bin:$PATH"
export PYTHONPATH="$HOME/.local/opt/scons/usr/lib/python3/dist-packages${PYTHONPATH:+:$PYTHONPATH}"
# python3-config wrapper (in ~/.local/bin) points gem5 at user-space Python
# dev headers + system libpython3.10.so.1.
# === end gem5 host env ===
EOF
```

For a non-interactive SSH session (`ssh host 'bash -s'`), `~/.bashrc` is **not**
sourced automatically — set `PATH`/`PYTHONPATH` explicitly at the top of such
scripts, or run `source ~/.bashrc` first.

#### 2.2.5 Verify the toolchain

```bash
export PATH="$HOME/.local/bin:$HOME/.local/opt/scons/usr/bin:$PATH"
export PYTHONPATH="$HOME/.local/opt/scons/usr/lib/python3/dist-packages:$PYTHONPATH"
which scons && scons --version | head -2          # scons 4.0.1
which python3-config && python3-config --includes  # -I.../python3.10 -I.../include -I.../x86_64-linux-gnu/python3.10
# smoke test: compile+link a tiny Python-embedding program
printf '#include <Python.h>\n#include <stdio.h>\nint main(){printf("%%s\\n", Py_GetVersion()); return 0;}\n' > /tmp/pytest.c
gcc /tmp/pytest.c $(python3-config --includes) $(python3-config --ldflags --embed) -o /tmp/pytest && /tmp/pytest
```

> Note: a pre-existing `~/.local/bin/make` wrapper (Verilator PCH workaround,
> only intercepts `V*.mk` arguments) is harmless for gem5 — scons invokes
> `g++` directly, and the wrapper passes through all other invocations.

---

## 3. Compile

### 3.1 Recommended build for Garnet: `build/Garnet_standalone/gem5.opt`

The Garnet synthetic-traffic config (`configs/example/garnet_synth_traffic.py`)
uses the legacy `configs/ruby/Garnet_standalone.py` path, which requires a
**single-protocol** build. Build the dedicated preset:

```bash
cd ~/work/gem5
export PATH="$HOME/.local/bin:$HOME/.local/opt/scons/usr/bin:$PATH"
export PYTHONPATH="$HOME/.local/opt/scons/usr/lib/python3/dist-packages:$PYTHONPATH"
scons build/Garnet_standalone/gem5.opt -j$(nproc)
```

`build_opts/Garnet_standalone` sets `PROTOCOL="Garnet_standalone"` (single
protocol), so the SLICC controllers (`L1Cache_Controller`, etc.) are available
at top level and no `--protocol` flag is needed at run time. On a 112-core host
this builds in ~5 minutes.

Expected key configure lines (proving the Python workaround is in effect):

```
Info: Using Python config: python3-config
Checking for C header file Python.h... yes
```

Expected final output: the `build/Garnet_standalone/gem5.opt` binary is
produced, with the last scons line `scons: done building targets.` and exit
code 0.

```bash
ls -la build/Garnet_standalone/gem5.opt
build/Garnet_standalone/gem5.opt --build-info | grep -i version
```

### 3.2 Optional comprehensive build: `build/ALL/gem5.opt`

`build/ALL` compiles every ISA and every Ruby protocol (including
`RUBY_PROTOCOL_Garnet_standalone`) into one binary. It is useful as a
general-purpose baseline but, because it sets `USE_MULTIPLE_PROTOCOLS=y`, it
**cannot run the legacy `garnet_synth_traffic.py` config directly** (the
controllers are namespaced and `configs/ruby/Garnet_standalone.py` expects them
at top level — see [TEST_RESULTS.md §4.1](TEST_RESULTS.md)). To use the `ALL`
binary for Garnet, use the new gem5 standard-library API instead of the legacy
config. For the classic synthetic-traffic workflow, use §3.1.

```bash
scons build/ALL/gem5.opt -j$(nproc)     # ~15 min on a 112-core host
```

---

## 4. Run prerequisites (Garnet)

No additional prerequisites beyond §2.2. The Garnet synthetic-traffic config
is pure Python using gem5's own modules (`m5`, `common.Options`, `ruby.Ruby`)
— no extra `pip` packages are required.

Optional packages gem5 auto-detects: protobuf (present), HDF5 (absent — gem5
disables HDF5 support, which is fine for Garnet runs).

---

## 5. Run the Garnet synthetic-traffic test

```bash
cd ~/work/gem5
build/Garnet_standalone/gem5.opt configs/example/garnet_synth_traffic.py \
  --num-cpus=16 --num-dirs=16 \
  --network=garnet --topology=Mesh_XY --mesh-rows=4 \
  --sim-cycles=1000000 --injectionrate=0.1 \
  --synthetic=uniform_random
```

> **Topology name.** In gem5 v25 the mesh topology is `Mesh_XY` (file
> `configs/topologies/Mesh_XY.py`, class `Mesh_XY`), not the legacy `Mesh`.
> See [TEST_RESULTS.md §4.2](TEST_RESULTS.md).

Key options (see `configs/example/garnet_synth_traffic.py`):

| Option | Default | Meaning |
|--------|---------|---------|
| `--synthetic` | `uniform_random` | traffic pattern: uniform_random, tornado, bit_complement, bit_reverse, bit_rotation, neighbor, shuffle, transpose |
| `--injectionrate` | `0.1` | packets/cycle/node (0–1) |
| `--sim-cycles` | `1000` | number of simulation cycles (use ≥ 1,000,000 for stable stats) |
| `--num-cpus` / `--num-dirs` | — | number of nodes (keep equal for a square mesh) |
| `--network` | `garnet` | Ruby network model |
| `--topology` | `Mesh_XY` | network topology (must match a `configs/topologies/<name>.py` class) |
| `--mesh-rows` | — | rows of the mesh (e.g. 4 for 16 nodes) |

The run prints `info: Garnet version 3.0`, exits with
`Exiting @ tick <N> because Network Tester completed simCycles`, and writes
Garnet network statistics (average flit latency / network latency / queueing
latency, flits injected/received) to `stats.txt` (use `-d <outdir>` to choose
the output directory). See [TEST_RESULTS.md](TEST_RESULTS.md) for captured
results.

---

## 6. Quick reference — full run from scratch

Assuming [REMOTE_SERVER_SETUP.md](REMOTE_SERVER_SETUP.md) is done:

```bash
# One-time env (put in ~/.bashrc)
export PATH="$HOME/.local/bin:$HOME/.local/opt/scons/usr/bin:$PATH"
export PYTHONPATH="$HOME/.local/opt/scons/usr/lib/python3/dist-packages:$PYTHONPATH"

# 1) Clone (see §1)
mkdir -p ~/work && cd ~/work
git clone git@github.com:hujian404/gem5.git gem5 && cd gem5
git remote add upstream git@github.com:gem5/gem5.git

# 2) Host toolchain (see §2.2): python3-dev headers + scons + python3-config shim

# 3) Compile the Garnet build (see §3.1)
scons build/Garnet_standalone/gem5.opt -j$(nproc)

# 4) Run Garnet synthetic traffic (see §5)
build/Garnet_standalone/gem5.opt -d ~/work/garnet-out \
  configs/example/garnet_synth_traffic.py \
  --num-cpus=16 --num-dirs=16 --network=garnet --topology=Mesh_XY --mesh-rows=4 \
  --sim-cycles=1000000 --injectionrate=0.1 --synthetic=uniform_random
# stats: ~/work/garnet-out/stats.txt
```

---

## 7. Troubleshooting

| Symptom | Fix section |
|---------|------------|
| `Can't find a suitable python-config` | §2.2.3 — install the `python3-config` shim on `PATH` |
| `Check failed for Python.h header` | §2.2.1 (headers) + §2.2.3 (shim `-I` paths) |
| `fatal error: x86_64-linux-gnu/python3.10/pyconfig.h` | §2.2.3 — shim must add `-I<dev>/include` (parent dir) |
| `Can't find a working Python installation` (link) | §2.2.1 — create `libpython3.10.so` symlink to system `.so.1` |
| `scons: command not found` | §2.2.2 — install scons user-space; ensure `~/.local/opt/scons/usr/bin` on `PATH` |
| `No module named pip` (trying to install scons) | §2.2.2 — use `apt-get download scons` instead of pip |
| `ImportError: No module named ...` (gem5 configs) | set `PYTHONPATH` per §2.2.4 (scons dist-packages) |
| GitHub HTTPS clone/push hangs | [SETUP §2.2](REMOTE_SERVER_SETUP.md) — use SSH (`git@github.com:...`) |
| `Warning: Couldn't find HDF5` | harmless — gem5 disables HDF5; not needed for Garnet |
