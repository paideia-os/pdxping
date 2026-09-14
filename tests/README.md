# tests/ — pdxping test suite (M4)

**Milestone lineage.** M4 in `design/networking/r100-user-tools-plan.md`
§13.4 (paideia-os). Three issues under this milestone, all landed
together at v1.2.0:

- **#9 — M4-001** happy-path smoke: echo against a QEMU-reachable
  gateway (10.0.2.2).
- **#10 — M4-002** timeout-path smoke: unreachable host.
- **#11 — M4-003** elevate-denied-path smoke: fail-closed behavior +
  the exact fd-2 diagnostic.

## Why these are pipeline-replay drivers, not a literal `pdxping ...`
subprocess invocation

`pdxping` is a standalone ring-3 ELF with its own `_start` (unlike a
library, whose M4 test drivers can call its public API directly) — but
`_start` itself is not a callable, return-value-bearing function: every
path through it ends in a real `sys_exit` syscall. There is also no
QEMU-boot smoke-test harness in this repo (or, as of this landing,
anywhere in the R100 wave for a standalone argv-driven CLI tool) that
could exec the compiled binary with a real argv array, capture its
stdout, and report pass/fail back to a `u64`-returning caller.

Every driver in this directory instead does what `_start` itself does,
one call at a time: it builds a synthetic argv array, feeds it through
the SAME real, linked `ArgvParse::argv_parse`, then walks a loop shaped
identically to `src/entry.pdx`'s own `ep_real_grant` `--count` arm,
asserting on the same arithmetic that arm's `_er_emit_summary` helper
would print, and returns a `u64` status instead of the literal
"emit \<message\>, exit \<code\>" a real invocation would produce. This
exercises the actual argv-parsing logic `_start` runs (the SAME code,
not a mock) plus a faithful re-derivation of the loop/summary
arithmetic, just without the final `sys_write`/`sys_exit` framing —
matching `mount.pdxfs`'s own `tests/README.md` convention for the
identical CLI-tool-vs-library gap.

## Why `IcmpWire::icmp_wire_echo_one` is mocked locally, not overridden

`src/icmp_wire.pdx`'s real WEAK stub is a FIXED fixture: it always
returns `IW_ECHO_OK` at a hardcoded 1.5ms RTT. There is no way to
parameterize it into a timeout, and paideia-as's ELF emitter marks only
`Data`-kind symbols weak at link time (`PA10-007`,
`paideia-as-emitter-elf/src/writer.rs`) — function symbols are always
`Global`/strong, so two definitions of the same function name across
`.o` files is a link-time collision, not an override. `pdxping_happy_
smoke.pdx` and `pdxping_timeout_smoke.pdx` therefore each declare their
own distinctly-named local fixture (`_phs_mock_echo_ok` /
`_pts_mock_echo_timeout`) matching the real function's exact call
signature and return-value convention, used only within that driver's
own `run()`. `ElevateGate::pdxping_elevate_check_and_require`, by
contrast, is called directly (no mock) in `pdxping_elevate_denied_
smoke.pdx`: its real WEAK stub already always returns `EG_DENY`, the
exact behavior that driver needs to confirm.

## Return-code convention

Every driver exports `run() -> u64`: `0` means "every assertion in this
driver passed"; any nonzero value identifies which assertion failed
first (see each file's own header for its exact nonzero-code table). A
future test harness — either a proper QEMU-boot argv/subprocess
protocol once one exists for standalone CLI tools, or a thin `pkg`/
`shell`-hosted runner that links `src/` alongside these drivers — walks
the driver list and reports each nonzero return using the table in that
driver's own file header; the process exits 0 iff every driver returned
0. Matches `mount.pdxfs`'s own `tests/README.md` convention exactly.

## Why no link step happens here

`tools/build.sh` compiles every `tests/*.pdx` file to its own `.o`
independently (to verify it assembles), with no link step and no
execution — cross-module calls like `call argv_parse;` are
unresolved-external references in a test file's own object, resolved
only once a future consumer links `src/` alongside these drivers. Every
driver here is written against that same convention, matching every
cross-file call already in `src/` (e.g. `src/entry.pdx`'s own calls
into `argv_parse` / `pdxping_elevate_check_and_require` /
`icmp_wire_echo_one`).

## Files

- **`pdxping_happy_smoke.pdx`** (M4-001, #9) — drives `pdxping
  --count=3 10.0.2.2` through `argv_parse`, then a 3-iteration loop
  against `_phs_mock_echo_ok` (always succeeds, rtt=2ms). Asserts the
  "3 transmitted, 3 received, 0% loss" fingerprint by recomputing
  `received`/`loss_pct` via the same formula `Entry::_er_emit_summary`
  uses. Returns `0` on match, a distinct nonzero code per failing
  assertion otherwise.
- **`pdxping_timeout_smoke.pdx`** (M4-002, #10) — same shape, driving
  `pdxping --count=3 203.0.113.1` (RFC 5737 TEST-NET-3) against
  `_pts_mock_echo_timeout` (always times out). Asserts the
  "3 transmitted, 0 received, 100% loss" fingerprint.
- **`pdxping_elevate_denied_smoke.pdx`** (M4-003, #11) — calls the real
  `ElevateGate::pdxping_elevate_check_and_require` directly, asserts
  `EG_DENY`, then byte-compares `Entry::ep_msg_eacces`'s 24 bytes
  against a local expected `"[pdxping.M3-001 EACCES]\n"` literal. Also
  documents (without asserting, since no live call site exists yet — see
  that file's own header) that `PingRecord::PR_RESULT_NO_PERMISSION`
  (3) is the `result_code` a future audit-call-site fix on this path
  should use; `src/audit_wire.pdx`'s own header currently overstates
  that `AuditWire::pdxping_audit_echo` is called on this arm, when
  `src/entry.pdx`'s actual code does not call it there.

## What a full QEMU smoke matrix still needs

All three drivers are fully self-contained today: each exercises real,
non-stub logic (`argv_parse`, the real `pdxping_elevate_check_and_
require`) plus a faithful local re-derivation of the `--count` loop and
summary arithmetic, entirely through `.bss` and local fixtures, with no
real kernel dependency beyond the syscalls those bodies themselves issue
transparently. A future consumer tool (this repo's own equivalent of
`tools/run-qemu.sh`, once one exists for a standalone CLI tool the way
paideia-os's own kernel-level smoke harness exists for the kernel
image) linking `src/` alongside these three drivers and printing their
return codes is the only thing missing before these run under real QEMU
boot and against a real `10.0.2.2` / `203.0.113.1` network path.
