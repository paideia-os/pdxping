# pdxping -- mirror push (M5-002, #13)

**Status: placeholder scaffolding.** This document describes the
intended mirror-push pipeline for `pdxping` release artifacts. No real
push to `pkgs.paideia-os` is performed by this repo, this document, or
any tooling landed at v1.2.0 -- the push itself is blocked on
release-infra R32 (the mirror-hosting / push-pipeline infrastructure),
which has not landed anywhere in the paideia-os organization as of this
writing. Nothing in this repo's `tools/build.sh` invokes a push of any
kind; this file exists so the intended shape is documented and
reviewable ahead of R32 landing, not to claim a working pipeline exists
today.

## Intended pipeline (once release-infra R32 lands)

1. **Tag.** A release engineer (or an automated release-line job) tags
   the repo at the version being shipped (e.g. `v1.2.0`), matching
   `manifest.pdxproj`'s own `version` field and `manifest.pdxsig`'s
   `source-tag`.
2. **Build.** `tools/build.sh` runs against the tagged commit, producing
   `build-out/pdxping.elf` / `build-out/pdxping.bin`.
3. **Manifest completion.** The release tool reads `manifest.pdxsig`'s
   source form (this repo, repo root), recomputes every `<BLAKE3-*>`
   hash placeholder from the tagged working tree, and writes a
   completed binary `manifest.pdxsig` (see that file's own header for
   the exact artifact list and section layout).
4. **Dual sign.** `paideia-pq-sign::sign_release_artifact` is invoked
   once with the paideia-release-line Ed25519 secret key and once with
   the ML-DSA-65 secret key (hardware-backed / KMS-custody per
   `design/02-development-environment.md` §1164, never repo-resident).
   Both detached signatures are appended to the binary manifest;
   verification is AND-semantics (both must verify).
5. **Push.** The compiled artifacts (`pdxping.elf`, `pdxping.bin`), the
   signed `manifest.pdxsig`, `doc/pdxping.pdxdoc`, and this repo's
   `LICENSE`/`README.md`/`CHANGELOG.md` are pushed to the mirror at:

   ```
   https://pkgs.paideia-os/main/pdxping/<version>/
   ```

   e.g. `https://pkgs.paideia-os/main/pdxping/1.2.0/manifest.pdxsig`
   for this release, matching `manifest.pdxproj`'s own `release:
   mirror_target` field.
6. **Verification.** A consumer resolving `pdxping` from the mirror
   verifies the dual signature against the paideia-release-line public
   keys before trusting the artifact, per the signature scheme
   `manifest.pdxsig`'s own `[signatures]` block documents
   (`hybrid-ed25519+ml-dsa-65`, `paideia-pq-hybrid-v1`).

## Why this is blocked, not merely undone

`pkgs.paideia-os` is named throughout this repo's own release tooling
(`manifest.pdxproj`'s `release: mirror_target`, `manifest.pdxsig`'s
header) as the target host, but no repo in the paideia-os organization
-- this one included -- has, as of this landing, a working push
mechanism: no CI/CD runs against `pdxping` (paideia-os itself never
uses GitHub Actions; verification here is local-only, same posture),
no release-line signing key is repo-resident (by design -- see step 4
above), and no `pkgs.paideia-os` ingest/hosting endpoint has been
confirmed reachable from any tool in this wave. Release-infra R32 is
the tracked round for standing up that infrastructure end to end
(tagging automation, KMS/TPM-backed signer invocation, and the mirror's
own ingest API); until it lands, every satellite repo in the R100 wave
ships this same scaffolding-only mirror-push document rather than a
working script, so the intended shape is reviewed and agreed on ahead
of the infrastructure existing to run it.

## What R32 needs to provide before this becomes real

1. A confirmed, reachable `pkgs.paideia-os` ingest endpoint (protocol,
   auth model, and the exact directory layout under
   `main/<pkg>/<version>/`).
2. A release-line signer invocation path this repo's build pipeline can
   call without ever holding the Ed25519/ML-DSA-65 secret material
   itself (hardware-backed in CI, or a cloud KMS pending the PQ-KMS
   availability check `design/02-development-environment.md` §1164
   tracks as a TODO).
3. A `tools/mirror-push.sh` (or equivalent) script in THIS repo that
   performs steps 2-5 above against the real endpoint from (1) and the
   real signer from (2) -- not written at this landing, since writing
   it against an endpoint and signer that do not exist yet would be
   exactly the kind of fabricated-plausible-call this repo's other
   WEAK stubs (`src/elevate_gate.pdx`, `src/icmp_wire.pdx`) document
   avoiding.

## References

- `manifest.pdxproj` (this repo) -- `release: mirror_target` field.
- `manifest.pdxsig` (this repo) -- the signed-manifest source form this
  pipeline completes and signs.
- `design/02-development-environment.md` §1140, §1164 (paideia-os) --
  hybrid signature rationale and release-line key custody.
- `design/networking/r100-user-tools-plan.md` §13.4 (paideia-os,
  pdxping#13) -- the M5-002 milestone this document closes.
