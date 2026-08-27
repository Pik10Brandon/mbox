# Optimized build and sandbox options

This guide builds **mbox** with compiler optimizations and explains the most
restrictive runtime option combination that the current implementation offers.

> [!WARNING]
> Mbox is an x86-64 Linux research prototype built around `ptrace` syscall
> interposition. It is not a container runtime or a hardened security boundary.
> Do not rely on it as the only protection for hostile code.

## Requirements

- An x86-64 Linux host whose security policy permits unprivileged
  `PTRACE_TRACEME`.
- A kernel that supports seccomp filters and `PTRACE_O_TRACESECCOMP` if you plan
  to use `-s`.
- A C build toolchain and OpenSSL development headers.

On Debian or Ubuntu:

```bash
sudo apt update
sudo apt install -y build-essential libssl-dev
```

The checked-in `configure` script is sufficient; rebuilding it with Autoconf or
Automake is not required.

## Optimized build

Run these commands from the repository root:

```bash
cd src
cp .configsbox.h configsbox.h
CFLAGS="-O2 -pipe" ./configure --prefix=/usr/local
make -j"$(nproc)"
```

`-O2` optimizes the resulting binary. `-pipe` can make compilation faster but
does not change runtime performance. `make -j"$(nproc)"` only parallelizes the
build.

## Validate the build

First check the command-line interface, then exercise both tracing modes:

```bash
./mbox -h
./mbox -i -- /bin/echo "ptrace mode works"
./mbox -s -i -- /bin/echo "seccomp-assisted mode works"
```

Both runtime checks must succeed without `sudo`. An error such as
`PTRACE_TRACEME doesn't work: Operation not permitted` means that the outer
container, sandbox, or host security policy denied ptrace. Fix that policy on a
development host instead of making root execution the normal setup.

The legacy integration suite can also exercise filesystem behavior in both
modes:

```bash
./testall.sh
./testall.sh -s
```

Some tests invoke optional or obsolete external tools, including `gvim` and the
old `pip search` command. Review an individual failure before treating a full
suite failure as an mbox regression.

## Choose runtime options

| Option | What it does | What it does not do |
| --- | --- | --- |
| `-s` | Uses a seccomp/BPF filter so selected syscalls trigger ptrace handling. This is primarily a tracing-performance option. | It does not add a filesystem or network policy by itself. |
| `-n` | Rejects creation of non-local sockets. | It does not create a network namespace, block Unix-domain sockets, or revoke inherited file descriptors. |
| `-i` | Skips the interactive review/commit session when the command exits. | It is not an isolation option. |
| `-p FILE` | Loads the experimental filesystem `hide`/`allow` rules described in `doc/NOTE.profile`. | The current loader does not enforce rules in the profile's `[network]` section. |

For a non-interactive run with filesystem redirection, restricted socket
creation, and seccomp-assisted tracing:

```bash
./mbox -s -n -i -- /bin/echo "sandbox smoke test"
```

If the host supports ptrace but not the seccomp tracing event, omit `-s`:

```bash
./mbox -n -i -- /bin/echo "sandbox smoke test"
```

These commands use the most restrictive general-purpose option combination
implemented by mbox, but the limitations above still apply.

## Optional installation

After validation, install the `mbox` binary under `/usr/local/bin`:

```bash
sudo make install
```
