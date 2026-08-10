# Remote Server Setup

One-time host and SSH setup for building gem5 (with the Garnet network model)
on a remote Linux server **without sudo privileges**. Once done, proceed to
[BUILD_AND_RUN_GEM5.md](BUILD_AND_RUN_GEM5.md).

> **No passwords, private keys, host IPs, or other secrets are included.**
> Replace every `<placeholder>` with your own value.

---

## 1. Host & toolchain assumptions

The reference host is a stock Ubuntu 22.04 machine with these system-wide
tools already installed (no sudo needed to *use* them):

| Tool        | Version used        | Notes                                                  |
|-------------|---------------------|--------------------------------------------------------|
| OS          | Ubuntu 22.04        | kernel 6.8, 112 CPUs, 1.5 TB RAM                       |
| git         | 2.34.1              | `/usr/bin/git`                                         |
| gcc / g++   | 11.4.0              | `/usr/bin/gcc`, `/usr/bin/g++`                         |
| clang       | 14.0.0              | `/usr/bin/clang` (optional)                            |
| make        | 4.3                 | `/usr/bin/make`                                        |
| cmake       | 3.22.1              | `/usr/bin/cmake` (not required by gem5)                |
| python3     | 3.10.12             | `/usr/bin/python3` (gem5 embeds this interpreter)      |
| protobuf    | 3.12.4              | `/usr/bin/protoc` + python `google.protobuf` (optional, gem5 auto-detects) |
| java        | OpenJDK 23.0.2      | `/usr/bin/java` (not required by gem5)                 |

**Not present on the host** (and installed user-space in
[BUILD_AND_RUN_GEM5.md §2.2](BUILD_AND_RUN_GEM5.md)):

- `scons` — gem5's build system.
- `pip` — not available (`python3 -m pip` → `No module named pip`).
- `python3-dev` headers — `Python.h` is **not** installed
  (`/usr/include/python3.10/Python.h` missing); gem5 requires it to embed
  Python.
- `python3-config` — not on `PATH`; gem5's `SConstruct` looks it up to obtain
  Python include/link flags.

The shared library `libpython3.10.so.1` **is** present system-wide
(`/usr/lib/x86_64-linux-gnu/libpython3.10.so.1`), so only the development
headers and the unversioned `.so` symlink are missing.

If your host is missing any of the system tools above, ask your administrator
to install them, or install a user-space copy under `~/.local`.

---

## 2. SSH access

### 2.1 Local machine → remote host (passwordless)

On your **local** machine, generate (or reuse) an ed25519 key and append its
public half to the remote's `~/.ssh/authorized_keys`:

```bash
# local
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ''   # if you don't already have one
ssh-copy-id -i ~/.ssh/id_ed25519.pub <user>@<remote-host>
# verify
ssh <user>@<remote-host> 'hostname; whoami'
```

### 2.2 Remote host → GitHub (over SSH, **not** HTTPS)

On the reference network **GitHub HTTPS (443) is blocked but SSH (22) works**,
so all `git clone` / `git push` must use the `git@github.com:...` form. A
dedicated GitHub key already exists on the remote and is pinned via
`~/.ssh/config`:

```bash
# remote (already done on the reference host)
ls ~/.ssh/id_ed25519_github         # the key
cat ~/.ssh/config
# Host github.com
#     HostName github.com
#     User git
#     IdentityFile ~/.ssh/id_ed25519_github
#     IdentitiesOnly yes
```

> If you are setting up a fresh host, create the key and register its public
> half at `https://github.com/settings/keys` (or as a repo deploy key on
> `<fork-repo>` → Settings → Deploy keys, with write access). For a long-term
> baseline you may prefer a per-repo key `~/.ssh/id_ed25519_gem5` with a
> `Host github-gem5` alias; the reference host reuses a shared GitHub key for
> simplicity.

Verify:

```bash
ssh -T git@github.com
# → "Hi <your-github-handle>! You've successfully authenticated..."
```

> **Tip (Windows users).** When driving the remote from PowerShell, prefer
> uploading a script with `scp` and running `ssh <host> 'bash -s' < script.sh`
> (or pipe through `tr -d '\r'` after the heredoc). PowerShell's `ssh.exe`
> silently strips inline double quotes and scripts saved with CRLF need CR
> stripping.

---

## 3. Network quirks to be aware of

1. **GitHub HTTPS (443) blocked, SSH (22) open.** → all git operations use SSH
   (`git@github.com:...`).
2. **No `python3-dev` / `scons` / `pip` system-wide.** → obtain them
   user-space via `apt-get download` + `dpkg -x` (the Ubuntu mirror is
   reachable). See [BUILD_AND_RUN_GEM5.md §2.2](BUILD_AND_RUN_GEM5.md).
3. **`Python.h`'s `pyconfig.h` dispatcher includes a multiarch header**
   (`<x86_64-linux-gnu/python3.10/pyconfig.h>`) that resolves via the parent
   include dir. On the system this works because `/usr/include` is a built-in
   default include path; in user-space the `python3-config` shim must add
   `-I<dev>/include` explicitly. See
   [BUILD_AND_RUN_GEM5.md §2.2.3](BUILD_AND_RUN_GEM5.md).

All are environmental and require no change to gem5 source.

---

## 4. Sanity check

Confirm on the remote:

```bash
git --version              # → 2.34+
gcc --version | head -1    # → gcc 11.4.0
python3 --version          # → Python 3.10.12
ssh -T git@github.com      # → "Hi <your-github-handle>!"
# confirm the host lacks scons / pip / python3-dev (expected):
command -v scons || echo "scons: not installed (will install user-space)"
python3 -m pip --version 2>&1 | head -1   # → No module named pip
ls /usr/include/python3.10/Python.h 2>&1   # → No such file (need python3-dev)
```

If all behave as above, the host is ready. Continue with
[BUILD_AND_RUN_GEM5.md](BUILD_AND_RUN_GEM5.md).

---

## 5. Baseline commit

The fork's baseline (the commit all experiments branch from) is the tip of
upstream `gem5/gem5` `stable` at the time of forking:

```text
branch:  stable
commit:  a9b193c59b55e9266adaf29bb71dee3329270dc3
subject: misc: add GitHub agentic workflow for labeling issues (#3373)
remote:  origin  = git@github.com:hujian404/gem5.git   (the fork)
         upstream = git@github.com:gem5/gem5.git        (read-only)
```

Recorded here for reproducibility; verify on a fresh clone with
`git log -1 --oneline`.
