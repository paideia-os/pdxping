# pdxping

ICMP echo CLI. Exercises the kernel's narrow-surface `sys_icmp_echo` syscall (no general raw-socket kind — a deliberate scope cut) and the new `R_NET_PRIVILEGED_PROTOCOL` elevate policy class. One echo per syscall, one blocking wait for the matching reply; `--count`, `--interval-ms`, `--timeout-ms`, `--dry-run` only. No traceroute, no MTU discovery, no flood mode — small on purpose.

## Spec

Full design lives in the paideia-os monorepo at
[`design/networking/r100-user-tools-plan.md`](https://github.com/paideia-os/paideia-os/blob/main/design/networking/r100-user-tools-plan.md)
(softarch's R100 user-tools plan). Section references in issues point into that document.

This repository is one of seven satellite repos that together deliver the
R100 wave: `libpdx-net`, `libpdx-url`, `pdxcurl`, `pdxping`, `pdxdig`,
`pdxsock`, `pdxtrust`.

## Status

v1.0.0 (Wave AA cohort, closes #1/#2/#3/#6/#7): scaffold + `caps.decl`,
full argv surface, `--dry-run` `PingRecord@0.1` preview, and the
`R_NET_PRIVILEGED_PROTOCOL` elevate gate + `libpdx-audit` per-echo
record -- both fail-closed WEAK stubs pending a real cross-repo link
path (see `CHANGELOG.md` "Known deferred substrate"). M2 (real
`sys_icmp_echo`, sysno 103) is a follow-up, not part of this cohort.

## Usage

```
pdxping [--count=N] [--interval-ms=N] [--timeout-ms=N] [--dry-run] <host>
```

Defaults: `count=4`, `interval-ms=1000`, `timeout-ms=1000`. At v1.0.0,
only `--dry-run` is real; a non-`--dry-run` invocation exits 13 with
`[pdxping.M3-001 EACCES]` on fd 2 (fail-closed elevate gate; see
`doc/pdxping.pdxdoc`).

## Build

```
bash tools/build.sh
```

Requires `paideia-as` >= 0.36.0 (see `tools/build.sh` header for
resolution order). Produces `build-out/pdxping.elf` /
`build-out/pdxping.bin`.

## License

MIT.