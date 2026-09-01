# pdxping

ICMP echo CLI. Exercises the kernel's narrow-surface `sys_icmp_echo` syscall (no general raw-socket kind — a deliberate scope cut) and the new `R_NET_PRIVILEGED_PROTOCOL` elevate policy class. One echo per syscall, one blocking wait for the matching reply; `--count`, `--interval-ms`, `--timeout-ms`, `--dry-run` only. No traceroute, no MTU discovery, no flood mode — small on purpose.

## Spec

Full design lives in the paideia-os monorepo at
[`design/networking/r100-user-tools-plan.md`](https://github.com/paideia-os/paideia-os/blob/main/design/networking/r100-user-tools-plan.md)
(softarch's R100 user-tools plan). Section references in issues point into that document.

This repository is one of seven satellite repos that together deliver the
R100 wave: `libpdx-net`, `libpdx-url`, `pdxcurl`, `pdxping`, `pdxdig`,
`pdxsock`, `pdxtrust`.

## License

MIT.