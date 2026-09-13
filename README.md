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

v1.1.0 (Wave AA cohort #1/#2/#3/#6/#7 + Wave BB follow-up #4/#5/#8):
scaffold + `caps.decl`, full argv surface, `--dry-run` `PingRecord@0.1`
preview + semantic-pipe emit, and a real (but still unreachable, see
below) `--count` real-echo loop with min/avg/max RTT + loss-percent
summary stats. The `R_NET_PRIVILEGED_PROTOCOL` elevate gate,
`libpdx-audit` per-echo record, `sys_icmp_echo` RTT capture, and
semantic-pipe emit are all fail-closed / no-op WEAK stubs pending real
cross-repo link paths and a kernel-side gate opening (see
`CHANGELOG.md` "Known deferred substrate"). Every non-`--dry-run`
invocation still exits 13 (elevate refused) today, so the real-echo
loop, while real and wired, never executes in practice yet.

## Usage

```
pdxping [--count=N] [--interval-ms=N] [--timeout-ms=N] [--dry-run] <host>
```

Defaults: `count=4`, `interval-ms=1000`, `timeout-ms=1000`. At v1.1.0,
only `--dry-run` is reachable; a non-`--dry-run` invocation exits 13
with `[pdxping.M3-001 EACCES]` on fd 2 (fail-closed elevate gate; see
`doc/pdxping.pdxdoc`). The real `--count` echo loop and its summary
line ("`N transmitted, M received, LOSS_PCT% loss, min/avg/max =
MIN/AVG/MAX ns`") are implemented but unreachable until the elevate
gate and the kernel-side `R_NET_PRIVILEGED_PROTOCOL` policy both open
up to ordinary ring-3 callers.

## Build

```
bash tools/build.sh
```

Requires `paideia-as` >= 0.36.0 (see `tools/build.sh` header for
resolution order). Produces `build-out/pdxping.elf` /
`build-out/pdxping.bin`.

## License

MIT.