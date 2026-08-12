# Molecule Quick Start - RHEL10-STIG

> Audience: anyone new to molecule testing with this Ansible Lockdown role. Read top-to-bottom the first time; come back to the reference tables later.

## Two scenarios live here: `default/` vs `ubi/`

This `molecule/` directory ships two scenarios. Pick the right one before running:

| | `molecule/default/` (canonical) | `molecule/ubi/` (smoke test) |
|---|---|---|
| **When to use** | Every QA cycle, before merging/pushing. This is the **gating scenario**. | Occasional sanity check against Red Hat's upstream UBI image (e.g. confirm role still applies on the registry image, not just on the Rocky variant). |
| **Container image** | `rockylinux/rockylinux:10-ubi-init` | `redhat/ubi10-init:latest` |
| **Container name** | `rhel10-stig-qa` | `rhel10-stig-ubi` |
| **`audit_git_version`** | not pinned in the scenario - derived from `benchmark_version` via `vars/audit.yml`; override at run time with `--extra-vars` | same - derived from `benchmark_version`, so it cannot drift |
| **`rhel10stig_disruption_high`** | `true` (exercises full code path) | `true` (same code path; scope is narrowed by the disabled-control list instead) |
| **Disabled controls** | none - all controls run | 26 toggles switched off in `converge.yml` because UBI's repos cannot install their packages (aide, audit, s-nail, firewalld, chrony, usbguard, fapolicyd, opensc, rsyslog-gnutls, libreswan, pkcs11-provider) or the subsystem is absent (SELinux config, crypto-policies) |
| **`fetch_audit_output`** | `true` (auto-copies audit JSONs out to `_temp_fetched_audits/`) | not set (audit results stay inside the container; `docker cp` them out manually) |
| **`prepare.yml` package set** | Full (audit, aide, chrony, rsyslog, logrotate, cronie, acl, kmod, dnf-plugins-core, python3-libselinux, python3-policycoreutils, python3-dnf-plugin-versionlock, tmux, git, procps-ng, openssl-pkcs11, opensc) | Subset; some packages aren't in UBI repos so the install task carries `failed_when: false` - a partial install is the expected outcome, not an error |
| **Maintenance** | Kept in lock-step with the role and audit repos every cycle | Updated less often; intentionally drift-tolerant |

**Rule of thumb:** if you don't know which to use, use `default/`. `ubi/` is for cases where you specifically want to validate against Red Hat's `redhat/ubi10-init` registry image rather than the Rocky variant.

The rest of this document focuses on the `default/` scenario. To run the `ubi/` scenario instead, swap `molecule destroy` -> `molecule -s ubi destroy` (and the same `-s ubi` flag on `converge`, `verify`).

## What this scenario does

Molecule spins up a throwaway Rocky 10 Docker container, applies the **whole** Ansible Lockdown RHEL 10 STIG role against it, then runs the paired goss audit (from the `RHEL10-STIG-Audit` repo) to see how many controls passed and how many still fail. It's the gating test you run before merging or pushing benchmark changes.

End-to-end you get:
1. A clean container booted into systemd.
2. Pre-remediation audit (baseline): records which controls are already failing before the role runs.
3. Role converge #1: applies all the STIG remediations.
4. Role converge #2: re-runs to confirm idempotency (zero changed tasks the second time).
5. Post-remediation audit: records the after-picture.
6. Pre and post audit JSON summaries automatically copied out to a directory beside the role for review.

If steps 3 and 4 both report `failed=0` and step 6 shows the post-audit failure count substantially lower than the pre-audit count, the role is shippable.

## Prerequisites

| Need | Why | How to check |
|---|---|---|
| Docker (Desktop or Engine), running | Molecule uses Docker to create the test container | `docker info` returns without error |
| Python 3.10+ | Ansible / Molecule are Python tools | `python3 --version` |
| Ansible venv with `ansible-core >= 2.19`, `molecule`, `molecule-plugins[docker]`, `docker`, `passlib` | Runtime deps for the test | `pip list \| grep -E 'ansible|molecule'` |
| `git` on the controller | Audit content is cloned from `RHEL10-STIG-Audit` | `git --version` |
| Local clones of **both** `RHEL10-STIG` AND `RHEL10-STIG-Audit` (or network access so the audit repo can be cloned at runtime) | The role pulls audit goss content from the audit repo during converge | `git -C <path-to>/RHEL10-STIG-Audit status` |

One-time venv setup (skip if you already have one):

```bash
python3 -m venv <path-to-your-ansible-venv>
source <path-to-your-ansible-venv>/bin/activate
pip install 'ansible-core>=2.19' 'molecule>=24' 'molecule-plugins[docker]' docker passlib
```

## Quick start

```bash
# Every time
source <path-to-your-ansible-venv>/bin/activate
cd <path-to>/RHEL10-STIG

# Full gating pair
molecule destroy && molecule converge && molecule converge && molecule verify
```

The whole sequence takes 15-25 minutes on a modern laptop. Network is needed (clone of audit content + dnf installs) for the first run; subsequent runs reuse the cached Docker image.

## What each command does

| Command | What happens |
|---|---|
| `molecule destroy` | Removes any leftover container from a previous run. Safe to run on a clean system. |
| `molecule converge` (first) | (a) pulls / starts the `rockylinux/rockylinux:10-ubi-init` container, (b) runs `prepare.yml` to install supporting packages, (c) clones the audit content from `RHEL10-STIG-Audit`, (d) runs the pre-remediation audit, (e) applies the full role, (f) runs the post-remediation audit, (g) fetches both audit JSONs to your local `_temp_fetched_audits/` folder. |
| `molecule converge` (second) | Re-runs the role against the already-remediated container. The Ansible `PLAY RECAP` should show `changed=0` (or a small handful of acceptable re-renders) - that's the **idempotency check**. |
| `molecule verify` | Final assertions defined in the scenario (light by default for this role; the real validation is in the audit JSONs). |

## Where the audit results land

After `molecule converge` completes, look at:

```
<working-tree>/_temp_fetched_audits/
  rhel10-stig-qa-RHEL10-STIG-v<ver>_pre_scan_<timestamp>.json
  rhel10-stig-qa-RHEL10-STIG-v<ver>_post_scan_<timestamp>.json
```

The JSONs are goss output - each has a `results: []` array where every entry has a `successful: true|false` field. Compare pre vs. post:

```bash
# How many controls passed before remediation?
jq '[.results[] | select(.successful==true)] | length' <pre_scan>.json

# How many passed after?
jq '[.results[] | select(.successful==true)] | length' <post_scan>.json
```

If the second number is significantly higher than the first, the role is working as intended.

## How to read `PLAY RECAP`

After each converge, Ansible prints a one-line summary like:

```
PLAY RECAP ********************************************************
rhel10-stig-qa : ok=238  changed=73  unreachable=0  failed=0  skipped=547  rescued=0  ignored=0
```

| Counter | What it means |
|---|---|
| `ok` | Tasks that ran and finished successfully (no state change needed, or already in desired state) |
| `changed` | Tasks that modified the system (installed a package, edited a file, etc.) |
| `failed` | Tasks that errored out. **MUST BE 0** for the run to be considered passing. |
| `skipped` | Tasks gated off by `when:` conditions (often because `system_is_container: true` skips container-incompatible work) |
| `rescued` / `ignored` | Block-level error handling - typically 0 |

**Idempotency expectation:** the second `converge` should show `changed=0` (or near zero) - the system was already in the desired state. A high `changed` count on the second run means a task is not idempotent and needs fixing.

## Why some audit failures are expected (even after the role passes)

The post-scan will still show some controls failing. Most fall into three buckets that aren't role bugs - they're inherent to containerized testing:

1. **SSH-config controls** (banner, ciphers, KexAlgorithms, MACs, idle timeouts, login grace time): these check `/etc/ssh/sshd_config`, which doesn't exist because `openssh-server` is intentionally NOT installed (running sshd inside Docker is an anti-pattern; containers use `docker exec` for shell access). The role's `when: rhel10stig_sshd_required` gate handles this correctly on real hosts.

2. **User-state controls** (FIPS-hashed passwords, password lifetime, dotfile audit): need real interactive users (UID >= 1000) with shadow entries and home directories. Containers only have system accounts (`chrony`, `polkitd`, `tss`, etc.), so the controls can't satisfy their preconditions.

3. **Kernel / mount controls**: any control gated by `not system_is_container` is intentionally skipped during remediation (auditd, FIPS, kernel sysctls, mount options) because containers share the host kernel. Audit still runs the goss tests, so they report as failing.

None of these warrant fixing the role. They're the expected delta between a containerized test and a real RHEL 10 host.

## Common gotchas

| Symptom | Likely cause | Fix |
|---|---|---|
| `molecule converge` errors with "container not running" | Stale container from a previous interrupted run | Run `molecule destroy` first, then retry |
| Pre-task fails with "Failed to download metadata" | UBI image can't reach Red Hat repos, or no network | Check Docker network; some packages may be unavailable in UBI - that's expected (see `failed_when: false` in `prepare.yml`) |
| Converge #2 fails on a task that passed in #1 | Ansible 2.19+ struct-vs-string type-check tripping a `when:` clause that's "lucky" on the first run | Inspect the offending task's `when:` - look for quoted-string-as-boolean bugs |
| `Conditional result (True) was derived from value of type 'str'` | Ansible 2.19+ rejects when-clauses whose final value is a non-boolean string. Common bug: an `or "<some expression>"` where the RHS got wrapped in quotes by accident | Remove the quotes; the RHS should be a bare Jinja expression |
| `verify` step empty / passes trivially | Expected - this role's primary validation is the goss audit JSONs, not molecule's `verifier:` plays | Inspect the audit JSONs instead |

## Ansible version

RHEL 10 / Rocky 10 ship a current Python (3.12) with full `dnf` Python bindings, so Ansible 2.19+ runs cleanly on managed nodes - use any current Ansible venv.

## Reference: what's in the default scenario

| File | Purpose |
|---|---|
| `molecule.yml` | Driver: docker. Image: `rockylinux/rockylinux:10-ubi-init` on `${MOLECULE_DOCKER_PLATFORM:-linux/arm64}` (multi-arch image; the default keeps local Apple Silicon runs native, and CI exports `linux/amd64` for GitHub-hosted runners). Host vars listed in the next table. |
| `prepare.yml` | Bootstraps minimal Rocky 10 UBI via `raw` (python3, sudo, iproute, openssh), then installs supporting packages (`audit`, `aide`, `crypto-policies`, `firewalld`, `chrony`, `rsyslog`, `logrotate`, `cronie`, `acl`, `kmod`, `dnf-plugins-core`, `python3-libselinux`, `python3-policycoreutils`, `python3-dnf-plugin-versionlock`, `tmux`, `git`, `procps-ng`, `openssl-pkcs11`, `opensc`). Stubs `/etc/default/grub` so GRUB-tag tasks don't fail in the container. |
| `converge.yml` | Sets root password (so PAM tasks don't lock us out), includes the role. |
| `verify.yml` | Spot-checks a container-applicable subset of the hardening. Its real job is to stop `molecule verify` being a silent no-op: a scenario with no `verify.yml` logs `Executed: Missing playbook` and exits 0, which reads as a pass. `molecule/ubi/verify.yml` imports this one rather than duplicating it. The authoritative validation remains the Goss audit run during converge. |

## Reference: host vars (in `molecule/default/molecule.yml`)

| Override | Default | Purpose |
|---|---|---|
| `audit_output_destination` | `<working_dir>/_temp_fetched_audits/` (relative to `MOLECULE_PROJECT_DIRECTORY`) | Where fetched audit JSONs land. Pinned outside the role tree so they don't pollute system paths or git. |
| `rhel10stig_disruption_high` | `true` (molecule default) | Exercises high-disruption code paths (account locks, password lifetime resets, fapolicyd rules, sysctl reloads). Safe in throwaway containers; risky on real hosts. Defaults to `false` in `defaults/main.yml` for production safety. |
| `fetch_audit_output` | `true` | Auto-copies audit JSONs out of the container to `audit_output_destination`. |
| `system_is_container` | `true` | Gates container-incompatible tasks (auditd, FIPS, kernel sysctls, mount changes, GRUB, etc.) - see `vars/is_container.yml`. |
| `skip_reboot` | `true` | Suppresses reboot handlers (containers can't reboot). |

## Overriding the audit branch

The audit branch is **not** set in the molecule scenarios. It derives from `benchmark_version` in `defaults/main.yml`, which `vars/audit.yml` expands to `benchmark_{{ benchmark_version }}` and loads via `include_vars`. So bumping `benchmark_version` moves both scenarios in lock-step - there is no per-scenario pin to keep in sync (or to drift).

If you're QA'ing a feature branch on `RHEL10-STIG-Audit` that hasn't been merged yet, override at run time with extra-vars:

```bash
molecule converge -- --extra-vars 'audit_git_version=<your-qa-branch>'
```

`include_vars` (precedence 18) outranks molecule play/host vars, so a value set in `molecule.yml`/`converge.yml` would be ignored - `--extra-vars` (precedence 22) is the only way to override the audit branch at run time.

## Where to ask for help

- This role's open issues: `github.com/ansible-lockdown/RHEL10-STIG/issues`
- Ansible Lockdown Discord (linked from the role README badge)
- Molecule docs: `molecule.readthedocs.io`
- Goss output format: `github.com/krameff/goss`
