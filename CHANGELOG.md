# Changelog

All notable changes to `pdxping` are recorded here. Format: keep-a-
changelog-style, semver-ordered, newest first.

## [1.0.0] -- Wave AA cohort (2026-09-13)

Closes pdxping#1, pdxping#2, pdxping#3, pdxping#6, pdxping#7.

### Added

- **M1-001 (#1): scaffold + `caps.decl`.** Repo shape (`README.md`,
  `LICENSE` (MIT), `CHANGELOG.md`, `caps.decl`, `tools/build.sh`,
  `link.ld`, `manifest.pdxproj`, `manifest.pdxsig` source-form
  placeholder). `caps.decl` declares `KIND_USER` (mandatory) and a
  `KIND_ELEVATE_CHANNEL` placeholder row (0x191, optional -- see
  `src/elevate_gate.pdx`). `src/entry.pdx` (`Module Entry`, `_start`)
  and `src/argv_parse.pdx` (`Module ArgvParse`) land as empty-surface
  scaffolds wired together at M1-002.

- **M1-002 (#2): argv surface.** `src/argv_parse.pdx` parses
  `--count=N`, `--interval-ms=N`, `--timeout-ms=N`, `--dry-run`, and
  the trailing `<host>` positional. Defaults: `count=4`,
  `interval-ms=1000`, `timeout-ms=1000`. A missing `<host>`, an
  unrecognized `--flag`, or a malformed numeric value all return
  `ARGV_ERR` and `src/entry.pdx` prints a usage diagnostic to fd 2 and
  exits 2.

- **M1-003 (#3): `--dry-run` PingRecord preview.** `src/ping_record.pdx`
  (`Module PingRecord`) defines the wire shape: `{version:u32,
  op:u32, host_hash_lo:u64, host_hash_hi:u64, count:u64,
  timeout_ms:u64, reserved:[u64;11]}` = 128 bytes exactly (see that
  file's header for the arithmetic reconciliation against this
  issue's `reserved[10×u64]` shorthand, which sums to 120B, 8B short
  of the stated 128B target -- `reserved` is 11 `u64` words here, not
  10, to close that gap while keeping every other field as specified).
  `--dry-run` builds one `PingRecord@0.1` per `--count` iteration
  (`seq` = loop index) and writes a `PingRecord@0.1 dry-run ...`
  comment-annotated hex preview of the 128 bytes to fd 2 -- no network
  I/O, no elevate call, no real echo attempted.

- **M3-001 (#6): `R_NET_PRIVILEGED_PROTOCOL` elevate gate (fail-closed).**
  `src/elevate_gate.pdx` (`Module ElevateGate`) wires
  `ElevateClient::elevate_client_acquire` +
  `ElevateClient::elevate_client_require` for the
  `R_NET_PRIVILEGED_PROTOCOL` right ahead of the (not-yet-landed-in-
  this-repo) real `sys_icmp_echo` send path. `libpdx-elevate` is not
  in this repo's `deps:` (no broker-endpoint cap is provisioned to
  this tool, same gap rm/mount.pdxfs/umount.pdxfs document for their
  own `KIND_ELEVATE_CHANNEL` rows), so `elevate_check_and_require`
  is a WEAK stub that always returns `EG_DENY` -- fail-closed by
  construction, not by accident: even a live broker query would fail
  today, since the kernel-side gate
  (`cap_check_r_net_privileged_protocol`, R100-PREP-003 #2009) permits
  only the boot-witness/init context, not an arbitrary ring-3 caller.
  A non-`--dry-run` invocation calls this gate once before any echo
  attempt; on `EG_DENY` it writes `[pdxping.M3-001 EACCES]\n` to fd 2
  and exits 13. The `EG_GRANT` arm is real, wired code -- kept for the
  day a broker cap is provisioned -- but unreachable at this landing
  (M2's real `sys_icmp_echo` send is not implemented in this repo
  yet, so `EG_GRANT` currently falls through to the same "M2 not
  landed" exit-0 no-op `--dry-run` skips).

- **M3-002 (#7): `libpdx-audit` per-echo record (WEAK stub).**
  `src/audit_wire.pdx` (`Module AuditWire`) declares
  `pdxping_audit_echo(seq, result_code, rtt_ns) -> u64`, intended to
  call `audit_file_append("/system/audit/pdxping.log", ...)` once per
  echo *attempt* (not once per session) -- `src/entry.pdx` calls this
  once per `--dry-run` loop iteration (`result_code = PR_RESULT_DRY_RUN`)
  and once for the `EG_DENY` refusal (`result_code =
  PR_RESULT_NO_PERMISSION`). `libpdx-audit` is not in this repo's
  `deps:` (no cross-repo link path stitched in this cohort), so the
  body is a no-op `xor rax, rax; ret` returning `AUDIT_OK` (0) --
  same WEAK-stub shape and same "swap only this file's body" contract
  `pdxsock`'s own `src/audit_wire.pdx` documents for the identical gap.

### Known deferred substrate

- M2 (real `sys_icmp_echo`, sysno 103, RTT stats printer) is not part
  of this cohort -- tracked as a follow-up `pdxping.M2-001` issue.
  Kernel-side, the syscall itself landed at R100-PREP-003 (#2009)
  with a fail-closed `cap_check_r_net_privileged_protocol` gate that
  permits only the boot-witness/init context; a real userspace
  `pdxping` invocation would be refused `-EPERM` even if this repo's
  own elevate/audit stubs were made real, until the elevate-broker
  dispatch itself is wired kernel-side.
- `libpdx-elevate` and `libpdx-audit` are referenced by name in
  source comments but are NOT in `manifest.pdxproj`'s `deps:` list --
  both integration points are WEAK stubs (see M3-001/M3-002 above).
- Semantic-pipe emission of `PingRecord@0.1` (a real
  `sys_semantic_send`, distinct from `--dry-run`'s direct hex preview
  to fd 2) is deferred with M2.
