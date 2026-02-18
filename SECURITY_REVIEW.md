# SeedSigner OS Security Review (quick static audit)

Date: 2026-02-18
Scope: repository-level static review for obvious backdoors/exploit vectors.

## Executive summary

I did **not** find a clear intentional backdoor in this repository.

I did find several **security-relevant risks** that should be addressed:

1. **Hardcoded root password in all board defconfigs** (high severity if a shell/console can be reached).
2. **Supply-chain flexibility without authenticity verification** for the SeedSigner app source repo/branch (medium severity in compromised CI/build environments).
3. **Unnecessary tooling in production images** (`netcat`, `crond`) that increases post-compromise utility (defense-in-depth issue).
4. **`mdev` hotplug script robustness issues** (quoting/condition style) that can cause fragile behavior and potential misuse in unusual environments.

## Findings

### 1) Hardcoded root password in image configs
Multiple configs set:

- `BR2_TARGET_GENERIC_ROOT_PASSWD="passworDT"`

This appears in production and dev defconfigs. If an attacker gets any local shell path (UART/JTAG/console escape/service bug), a known root password significantly lowers the bar to full compromise.

### 2) Build process allows unpinned app repo/branch overrides
`opt/build.sh` permits custom `--app-repo` and `--app-branch`, then clones directly with `git clone --depth 1`.

Risk: if build automation or operator environment is compromised/misconfigured, malicious code can be silently pulled into images. There is no commit pinning/signature verification step for the application checkout in this script.

### 3) Extra tooling included in production defconfigs
Production defconfigs include `BR2_PACKAGE_NETCAT=y` and busybox has `CONFIG_CROND=y`.

Not a backdoor by itself, but these capabilities increase attacker options after foothold, and are often removed in hardened minimal images unless strictly required.

### 4) `mdev` script hardening opportunities
`mdev` auto-runs `/etc/mdev/mdev.sh` on `mmcblk0p1` events. Script uses unquoted tests/vars and `==` in `/bin/sh` context.

This is mostly a robustness issue (word-splitting/edge behavior) and should be hardened with POSIX-safe style (`=` and quoting) to reduce surprises.

## Recommended remediations

1. Remove static root password from production images:
   - disable password login entirely, or
   - lock root account (`!`), and/or
   - enforce random/unique secrets only in explicitly-dev builds.
2. Pin SeedSigner app checkout to an immutable commit and verify expected hash/signature in `build.sh`.
3. Re-evaluate whether `netcat` and `crond` are necessary in non-dev builds; remove if not required.
4. Harden `mdev.sh` with strict/quoted shell checks and explicit mount options.
5. Consider adding a CI policy check to fail builds on hardcoded credentials or non-pinned source fetches.

## Notes on backdoor assessment

The reviewed startup scripts (`S02seedsigner`, `S02mdev`), overlay files, and package definitions did not show obvious hidden callbacks, remote control hooks, or obfuscated payload logic during this pass.
