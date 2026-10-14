# RHEL10STIG

## Based on STIG V1R2 - 2026 October - Community updates

- **279 task names did not match the benchmark title.** The text after `PATCH |` or `AUDIT |` is what
  an operator sees in playbook output and what a reviewer matches against the STIG, and it had
  drifted across 123 controls. Most were stale wording the benchmark has since revised, such as
  RHEL-10-600420 saying `logon attempts` where the benchmark says `login attempts`, and
  RHEL-10-500810 still listing the old syscall set although the audit rule template already writes
  `renameat2`. The remainder were hyphenation and trailing periods. Names now carry the benchmark
  title with its quotation marks removed, matching the audit content; the severity prefix, the
  `PATCH`/`AUDIT`/`WARN` verb and the trailing step qualifiers that distinguish tasks within a block
  are all preserved. No task logic, tag, condition or module argument is touched

- **`rhel10stig_sssd` was referenced but never defined.** `rhel10stig_sssd_config_certmap` builds the
  certmap stanza from `rhel10stig_sssd.domain` and `rhel10stig_sssd.rulename`, neither of which
  existed anywhere in the role, so enabling `rhel10stig_configure_sssd` failed RHEL-10-701280 with an
  undefined variable. The defect was invisible because the toggle defaults to false. The variable is
  now defined with the benchmark's own example values, `domains` derives from it rather than being
  hardcoded to a different domain, and a stray double quote that would have been written into
  `/etc/sssd/sssd.conf` after the rule name is removed along with the quotes around `matchrule`,
  which the benchmark does not use

- **RHEL-10-700830 applied Ctrl-Alt-Del through `gsettings` at run time.** The setting did not
  survive a reboot, the task re-applied an already-disabled binding on every run, and it failed once
  the binding was locked. Its audit step also invoked `gsettings set` with no value, which cannot
  succeed. The control now writes `logout` into `/etc/dconf/db/<db>.d/00-disable-CAD` with
  `community.general.ini_file` and notifies the existing `Update dconf` handler, matching how
  RHEL-10-700840 and the other GNOME controls in this role already work

- **the benchmark facts block guarded on a variable that is never set.** `tasks/main.yml` tested
  `post_audit_summary`, but nothing in the role assigns it; the template it renders consumes
  `post_audit_results`, which only `tasks/post_remediation_audit.yml` sets. Both clauses of the guard
  therefore collapsed to "run whenever the local audit fact is absent", and the template then failed
  with `post_audit_results is undefined` on any host where the post-remediation audit had not run.
  The guard now tests `post_audit_results`, and the template omits the `audit_summary` line rather
  than failing when it is undefined

- **the audit stage blocked the molecule idempotence gate.** Running the pre- and post-remediation
  audits executes goss, and the fetch and copy steps move the resulting files, so they report changed
  on every pass. The default scenario enables both `run_audit` and `fetch_audit_output`, so the gate
  could not go green no matter what the remediation tasks did. The exclusion tag is applied at the
  four points where the stage is imported or consumed, not on the individual commands: tagging only
  the commands skipped the audit run while leaving its capture task to read an output file that was
  never produced. `Show Audit Summary` is included because the `audit_results` default interpolates
  `pre_audit_results`, which only the audit stage sets

- **the molecule workflow now gates on idempotence.** It previously ran
  `create -> converge -> verify -> destroy`, deliberately skipping the idempotence step because
  converge #2 was a known non-zero-change run. The causes that note named are fixed in this
  release: the harness set the root password with an unpinned salt so the hash differed every run,
  and the pre- and post-remediation audit scans and the fetch step carried no exclusion tag.
  Measured from a cold `destroy` plus `converge`: `ok=494 changed=154 failed=0`, then idempotence
  `ok=457 changed=0 failed=0`. A second converge changing anything is now a regression rather than
  an expected outcome

- **molecule and Repo QA now run on the self-hosted runner**, the same one the pipeline workflows
  already use, rather than a GitHub-hosted one, so neither depends on GitHub-hosted runner minutes.
  Those pipelines are AWS and tofu runs that need no container runtime, so the molecule workflow
  probes for a usable Docker daemon with `docker info` and installs one from the distribution's
  repositories only when that probe fails. The runner must still permit privileged containers with
  `cgroupns: host` and a `/sys/fs/cgroup` bind mount, which no workflow step can grant
- **neither workflow uses `actions/setup-python` any more.** That action resolves an interpreter
  from a tool cache or from `actions/python-versions`, and a self-hosted Rocky host has neither, so
  it fails with `The version '3.12' with architecture 'x64' was not found`. Both workflows now
  select the interpreter the runner already ships, 3.12 or 3.11 and never 3.13 because both
  converge plays need the stdlib crypt module, and install into a job-scoped virtualenv so the
  shared runner's system Python is left alone. The molecule destroy step is guarded on molecule
  being present, so a failure before the virtualenv exists no longer surfaces as a misleading
  `command not found`
- **the molecule prepare bootstrap was not idempotent, despite saying it was.** Both scenarios ran
  `dnf install -y python3 sudo iproute openssh-server openssh-clients` through `raw` on every
  create. On dnf5 that is not a no-op for an already installed package: when the repositories
  carry a newer build than the image ships, it tries to resolve an upgrade, reports that it cannot
  install the best candidate, and exits non-zero, which fails the whole create. The bootstrap now
  asks dnf only for packages `rpm -q` reports absent

- **the reboot-skipped warning reported changed on every run.** The task only prints a message, but
  carried `changed_when: true`, so any host where a reboot is required and `skip_reboot` is set
  reported a change on every pass (ansible-lockdown/RHEL10-STIG#9). The molecule gate did not catch
  it because the enclosing block is now gated on `change_requires_reboot`, which is false in the
  test container, so the task never ran there. It is display-only with no notify, so it is now
  `changed_when: false`

- **the molecule scenarios could never pass idempotence.** Both converge playbooks set a root
  password in a pre-task with `password_hash('sha512')`, which generates a fresh salt on every
  invocation. The resulting hash differed each run, so the `user` module reported changed every time
  regardless of anything the role did. The salt is now pinned, which makes the task genuinely
  idempotent rather than excluding it from the gate. This is scenario plumbing for a throwaway
  container and is not a credential used anywhere else

- **RHEL-10-200692 and RHEL-10-700670 did not apply on a first run.** Both are gated on `postfix`
  appearing in `ansible_facts['packages']`, but the role gathers those facts once before prelim and
  installs postfix later, in RHEL-10-200690. On a freshly provisioned host the gate was therefore
  false and both controls silently skipped, applying only if the role was run a second time. The
  package facts are now refreshed immediately after the install, and only when it reports changed.
  This surfaced through the idempotence gate, where the two controls showed as changed on the second
  pass having been skipped on the first

- **the benchmark version string moves from `v1r2` to `v1.2.0`.** `benchmark_version` in
  `defaults/main/main.yml` is the single source: `audit_git_version` derives the audit branch from
  it as `benchmark_{{ benchmark_version }}`, the goss bridge template stamps it into every scan
  report, and the pre- and post-scan output filenames embed it. The paired audit content is
  published on a new `benchmark_v1.2.0` branch; the previous `benchmark_v1r2` branch is left in
  place and unchanged, so a role still on the old string keeps resolving and nothing is cut over by
  this change alone. The dotted form matches the convention the Ubuntu audit content already uses.

  **These two must move together.** The audit branch has to exist before a run with
  `audit_content: git` can fetch it, so the audit side ships first.

- **audit configuration is now overridable from inventory.** The audit settings lived in
  `vars/audit.yml`, loaded from `tasks/prelim.yml` with `include_vars`. That is variable precedence
  level 18, which outranks inventory `group_vars` and `host_vars` and play vars, so every knob the
  file documented as site-settable - `audit_content`, `get_audit_binary_method`,
  `audit_bin_copy_location`, `audit_conf_source`, `audit_conf_dest`, `audit_log_dir`,
  `audit_cmd_timeout`, `audit_max_concurrent`, `audit_bin_url`, `audit_bin_validate_certs` - was
  silently discarded unless passed as `-e`. A site setting `audit_content: copy` to run air-gapped
  got `git` anyway and the role attempted a clone. `defaults/main.yml` is now `defaults/main/main.yml`
  and the audit settings are `defaults/main/audit.yml`, at defaults precedence, where inventory
  overrides them as the comments always said it should. The `include_vars` task is gone. The audit
  toggles that were already in `defaults` (`setup_audit`, `run_audit`, `audit_only`,
  `audit_run_heavy_tests`, `fetch_audit_output`, `audit_output_collection_method`,
  `audit_output_destination`) moved into the same file, so audit configuration sits in one place at
  one precedence level. Anyone currently passing these with `-e` is unaffected; anyone who edited
  `vars/audit.yml` in place will find their edit moved
- **the auditd immutable-rules handler now reaches its notify.** `Audit immutable fact` is an
  `ansible.builtin.debug` task carrying `notify: Reboot required`, with no `changed_when`. A debug
  task reports `ok` and never `changed`, so the notify could never fire. It now sets
  `changed_when: true`. To be precise about the effect: the reboot signal was **not** actually
  being lost, because `Stop auditd process` in the same `listen: Auditd checks` group already sets
  `change_requires_reboot: true` unconditionally. This removes a dead code path rather than
  restoring a lost signal, and the two overlapping signals should be reconciled - see the handler
  notes below
- **`PermitRootLogin` is matched by keyword rather than by value.** The `lineinfile` regexp
  enumerated `(yes|no|true|false)`, so against a host already set to `prohibit-password`,
  `without-password` or `forced-commands-only` the match failed and a second `PermitRootLogin`
  directive was appended instead of the existing one being corrected. The pattern now matches the
  keyword and any trailing value, with a word boundary so `PermitRootLoginFoo` is still excluded.
  Governing rule `RHEL-10-700620`
- **`rhel10stig_pam_rounds` is `100000`.** It shipped `5000`, while `RHEL-10-600720` and
  `RHEL-10-600700` both require `rounds=100000`. The consuming tasks are gated behind
  `rhel10stig_disruption_high`, default false, so a stock run was unaffected, which bounded the
  exposure without making the shipped value correct
- **the reboot block is gated on `change_requires_reboot`.** It carried no block-level condition,
  only per-task `skip_reboot` checks, so with `skip_reboot` false the role would attempt
  `ansible.builtin.reboot` even when nothing had requested one (addresses
  ansible-lockdown/RHEL10-STIG#9; thank you @karlg100)
- **the benchmark facts task is tagged `molecule-idempotence-notest`.** `compliance_facts.j2`
  recomputes `Benchmark_run_date` through `strftime` on every run, so the task always reported
  changed and the idempotence check could never pass (addresses ansible-lockdown/RHEL10-STIG#10;
  thank you @karlg100)
- `audit_git_version` can now be set in a molecule scenario or from inventory. It was previously
  inert for the same precedence reason, and both scenarios carried a comment telling the reader to
  use `--extra-vars` instead. That workaround is no longer needed and the comments say so
- `.qa_baseline.json` repointed at `defaults/main/main.yml`. The two `jinja[spacing]` findings it
  carried are fixed rather than recorded, as they already were on the public role: `item.path}}` in
  `tasks/Cat2/RHEL-10-700xxx.yml` gains its missing space, and the space before each closing
  parenthesis in the passwd-data loop in `tasks/Cat2/RHEL-10-400xxx.yml` is removed. Both entries
  leave the baseline, so a new spacing finding in either file fails the gate.
- the Repo QA workflow pin moves from checker release 2.8.3 to 2.8.4. 2.8.3 resolves the role's
  defaults as the single path `defaults/main.yml` and exits 1 when that file is absent, so the
  directory split above failed the workflow before any check ran. 2.8.4 resolves either a
  `main.yml` file or a `main/` directory. No check behaviour changes: 2.8.4 run with the workflow's
  own arguments, `--strict` included, reports the same 13 passes and 2 user skips
- the changelog file is now `CHANGELOG.md` rather than `Changelog.md`, and the README link was
  updated to match. Repository paths are case-sensitive on the forge, so the old link would have
  returned a 404 after the rename
- **every pre-commit hook is now pinned to its current upstream release.** `ansible-lint` moved
  from `v26.4.0` to `v26.9.0` and `gitleaks` from `v8.30.0` to `v8.30.1`; the remaining three
  (`pre-commit-hooks` `v6.0.0`, `detect-secrets` `v1.5.0`, `yamllint` `v1.38.0`) were already
  current and were checked rather than assumed. No task content changed
- **every audit report was stamped with the wrong benchmark revision.**
  `templates/lockdown_audit.yml.j2` carried `benchmark_version: 'v1r1'` as a literal while the role
  is on `v1r2`. Goss reads that value into its `benchmark_meta` block, so every pre- and
  post-remediation scan report this role produced was labelled with the previous revision. It now
  references `{{ benchmark_version }}`, so it tracks the role instead of drifting at each bump
- **a template source pointed at a file that does not exist.** `tasks/auditd.yml` requested
  `audit/98_auditd_exception.rules.j2`; the template on disk is
  `etc/audit/rules.d/98-auditd_exception.rules.j2`, a different directory and a hyphen where the
  source had an underscore. The task runs only when
  `rhel10stig_allow_auditd_uid_user_exclusions` is enabled, which is not the default, so the role
  failed at runtime only for sites that turned it on and no test exercised it. Verified by running
  that path with the toggle enabled: the task now completes and writes the exceptions file
- **collection requirements carry version floors.** `collections/requirements.yml` previously
  listed bare collection names with no constraint. It now sets `community.general >=11.4.0` and
  `ansible.posix >=2.2.2`, the versions this cycle was verified against, in the form the Windows
  fleet already uses. Galaxy resolves a released version at or above the floor, so security fixes
  arrive without a maintenance bump and no build ever pulls unreleased code. A floor rather than an
  exact pin is deliberate: an exact pin goes stale and needs a person to raise it. For contrast, a
  `type: git` source with no `version` installs the collection's default branch, which moves
  between builds - on `ansible.posix` that currently resolves to `3.0.0`, a major version ahead of
  the `2.2.2` release. The file itself carries no explanatory comments; this entry is the record.
  `community.crypto` is removed from both the requirements file and `meta/main.yml` - no task,
  handler or template in this role calls it
- **the molecule QuickStart no longer misdescribes the scenarios.** Five corrections: the
  `benchmark_version` source file, the `prepare.yml` package list (it named `openssl-pkcs11` where
  the package is `pkcs11-provider`, and omitted six others), the claim that `openssh-server` is not
  installed (both scenarios install it; sshd is simply never started), and the host-vars table,
  which listed five of the seven keys actually set and did not mention that `converge.yml` re-sets
  them as play vars and wins
- **known, not fixed in this release.** `tasks/prelim.yml:230` registers
  `prelim_auditd_immutable_check` with no `when:` gate, so it is always defined. That makes
  `handlers/main.yml:239` (`when: prelim_auditd_immutable_check is not defined`) unreachable, so
  **auditd is never restarted by this role**, and it makes `Audit immutable fact` and
  `Stop auditd process` both fire unconditionally on a non-container host. `Stop auditd process`
  also stops nothing; its module is a `set_fact`. Recorded rather than changed, because fixing the
  gate changes when auditd restarts on live hosts and that needs its own test pass
- **known, not fixed in this release.** `RHEL-10-700800` is `command: echo "Run dconf update"` with
  `changed_when: false` and `notify: Update dconf`. A task that can never report changed never
  fires its handler, so the control is a no-op
- deliberately not renamed: existing entries in this file, which record what was true when written

## Based on STIG V1R2 - 2026 August - Contributing guide and README refresh

- replaced `CONTRIBUTING.rst` with `CONTRIBUTING.md`, carrying the current Ansible-Lockdown
  contributing guide. It adds the approved-contributor model for pull requests, keeps issues open to
  everyone, and retains the DCO 1.1 text and the GPG plus Signed-off-by requirements
- `README.md`: added a Contributing section pointing at `CONTRIBUTING.md`, and refreshed the
  Community Contribution section, which still described the previous open pull request workflow and
  so contradicted the new guide
- `README.md`: removed the decorative emoji from headings, and switched the social badge from
  `twitter.com` to `x.com`

## Based on STIG V1R2 - 2026 August - Company name updated to Quantum Sky

- the parent company name changed from Tyto Athene to Quantum Sky. Renamed in `meta/main.yml`, `vars/main.yml` and `LICENSE`. `vars/main.yml` feeds `file_managed_by_ansible`, so the header written into every templated file carries the new name; expect a one-off change on those files at the next run
- the Repo QA Checker pin moved from `2.8.1` to `2.8.3`. `2.8.1` expected the former parent company in `meta/main.yml`, so it failed this branch on its own rename; `2.8.3` expects the new name. `2.8.3` also fixes an output parser that had been dropping every `ansible-lint` finding whose rule carries a severity suffix, which surfaces 6 pre-existing findings here, 4 `complexity[tasks]` and 2 `jinja[spacing]`; they are recorded in `.qa_baseline.json` and no task content changed
- deliberately not renamed: existing entries in this file, which record what was true when written

## Based on STIG V1R2 - 2026 August - CI branch triggers

- the Molecule and Repo QA workflows carried different branch filters in the private role and in the public mirror - `latest` and `benchmark*` here, `devel` and `main` there - so the same file existed in two divergent forms and the overlay needed a hand edit each time. Both now list all four, which makes the file identical in both places and the overlay a no-op on it
- this is the convention the pipeline workflows already follow: `devel_pipeline_validation.yml` triggers on `devel` and `benchmark*`, `main_pipeline_validation.yml` on `main` and `latest`. Each names the public and private equivalent of the same stage, which is why those two files are shared unedited
- the `push` filter is now `main` and `latest`, matching the pair `main_pipeline_validation.yml` uses, rather than mixing the private trunk with the public integration branch. Pull requests into `devel` still run both workflows through the `pull_request` filter, so nothing is lost - only the post-merge push to `devel` no longer re-runs work the pull request already gated
- the `paths` filters, the weekly `schedule` and `workflow_dispatch` are unchanged

## Based on STIG V1R2 - 2026 August - CI action versions and Dependabot

- `actions/checkout` was split across two versions in this repo: `@v6.0.2` in the devel and main pipeline workflows against `@v7` in the molecule and Repo QA workflows. All seven references are now `@v7`
- `@v7` is a moving pointer rather than a frozen pin. Upstream advances the major tag onto each release, so `v7` and `v7.0.1` currently resolve to the same commit while `v7.0.0` does not. Referencing the major means future v7.x patches are picked up with no maintenance, and only a major would need a review. `actions/setup-python` and `actions/upload-artifact` already follow the same style here
- added `.github/dependabot.yml` enabling Dependabot for the `github-actions` ecosystem. All action updates are collapsed into a single grouped pull request on a monthly schedule, because each pull request in this repo runs both molecule scenarios plus the pipeline that provisions and destroys real infrastructure, so per-action pull requests would be disproportionately expensive
- Dependabot resolves version-like references only, so the actions referenced by branch are outside its reach and still need reviewing by hand: the AWS credentials action and first-interaction, both tracking a branch, and the three shared reusable workflows. A clean Dependabot run should not be read as the action supply chain being covered
- note for anyone adding to `.github/` in future: `.gitignore` excludes that directory, so a new file there needs a forced add even though the existing workflow files stage normally

## Based on STIG V1R2 - 2026 August - Alignment Strategy

- RHEL-10-400185: the PATCH task passed a quoted string as the file mode. The Jinja expression rendered `'go-rwx'` rather than `go-rwx`, with the single quotes inside the value, so `ansible.builtin.file` received something that is not a valid symbolic mode and the task failed on any host it actually ran against. The quotes are removed; both arms now render clean, `go-rwx` when the audit log group is `root` and `g-wx,o-rwx` otherwise. That the quotes were a typo rather than a convention is evident from the two sibling constructs in the role, RHEL-10-400190 and RHEL-10-500110, which use the same Jinja if/else for a mode without quoting either arm
- the fault was invisible to CI: the task loops over the result of a `find` for `*.log` under the audit log directory, and a container has no audit logs, so the loop body never ran and the mode was never evaluated. Every host with audit logs would have hit it. Verified by rendering both arms and checking the result against the symbolic-mode grammar rather than by a converge
- templates/boot/grub2/user.cfg.j2 was one of only two templates in the role without the `file_managed_by_ansible` header. Added. grub honours `#` comments, and the paired audit checks the file by pattern on the `GRUB2_PASSWORD=` line rather than by exact content, so the header is inert to the check - confirmed by rendering the template and re-running the matcher against the output
- templates/etc/issue.net.j2 is the other template without the header and deliberately keeps it that way. It renders the Standard Mandatory DOD Notice and Consent Banner for RHEL-10-700040 and is deployed to /etc/issue with /etc/issue.net symlinked to it, so prepending comment lines would alter banner text that has to match the mandated wording
- README: removed the emoji from the six remaining section headings and from the check-mode caution bullet, leaving the wording unchanged
- inlined three single-item `when:` lists that fitted on one line, in RHEL-10-700040, RHEL-10-700120 and the prelim audit-vars include. The remaining single-item `when:` blocks in the role hold multi-line `or` chains where a list entry is genuinely clearer, and are left alone

## Based on STIG V1R2 - 2026 August - Fact access moved to ansible_facts for ansible-core 2.24

- the role read facts through the injected top-level names (`ansible_distribution`, `ansible_os_family`, `ansible_distribution_major_version`, `ansible_virtualization_type`, `ansible_mounts`, `ansible_local`, `ansible_default_ipv4`). `INJECT_FACTS_AS_VARS` defaulting to `true` is deprecated and the behaviour is removed in ansible-core 2.24, at which point each of those names becomes undefined. Ten references across seven lines now use `ansible_facts['<name>']` in `tasks/main.yml`, `tasks/prelim.yml`, `defaults/main.yml` and `vars/main.yml`
- `tasks/main.yml` was already half-converted: the assert messages used `ansible_facts['distribution']` while the `that:` expression on the same task still used the injected names. The expression, the OS-vars include, the container `when:` and the local-facts lookup are now consistent with the messages
- deliberately left alone, because they are not facts and have no `ansible_facts` equivalent: `ansible_version` (magic variable), `ansible_connection` (connection variable), `ansible_user` and `ansible_become` (connection variables), `ansible_run_tags` (magic variable) and `ansible_facts_path` (this role's own default variable, despite the name). The deprecation warning cites the line holding `ansible_connection`, but that is only where the `when:` expression starts - the fact reference on the following line was the trigger. Converting `ansible_connection` would have silently disabled container detection
- verified by parsing the reflowed `that:` scalar back out of the YAML and comparing it to the intended expression, rather than by reading the diff, since the folded plain scalar carries a mix of single and double quotes. `ansible-playbook --syntax-check` exits 0 and `ansible-lint` reports no new findings
- `rhel10stig_custom_firewall_interface` was `"{{ ansible_facts['default_ipv4']['interface'] | list }}"`. The `interface` fact is a string, and Jinja's `list` filter applied to a string returns its characters, so the default evaluated to `['e', 't', 'h', '0']` rather than `['eth0']`. Its only consumer is RHEL-10-200532, which passes it to `ansible.posix.firewalld` as `interface: "{{ rhel10stig_custom_firewall_interface | join(',') }}"`, so firewalld was being handed `e,t,h,0` and the deny-all zone bound to four interface names that do not exist. The default is now a list literal, `"{{ [ansible_facts['default_ipv4']['interface']] }}"`, which yields `eth0` through the same `join(',')`. Keeping it a list preserves the ability to override with several interfaces, which still renders as `eth0,eth1`

## Based on STIG V1R2 - 2026 August - RHEL-10-700160 corrected to /tmp nosuid

- RHEL-10-700160 carries `SV-281240r1166672_rule`, which V1R2 defines as mounting `/tmp` with the `nosuid` option, but the task was written against `/var`. The title, the `item.mount` condition, the `notify` handler and the `path` have all been corrected to `/tmp`. No coverage is lost: `/var` with `nodev` is `SV-281241` and is handled by RHEL-10-700165, and no rule asks for `/var` with `nosuid`. The change closes a real gap, since `/tmp` was receiving `nodev` from RHEL-10-700150 and `noexec` from RHEL-10-700155 but never `nosuid`
- RHEL-10-700160 `opts` referenced `discovered_var_noexec`, which is registered nowhere in the role, so any host that actually reached the task failed on an undefined variable. It also referenced `discovered_var_nodev`, which RHEL-10-700165 registers 28 lines further down the same file, making it a forward reference. Both are replaced with `discovered_tmp_nodev` (registered by RHEL-10-700145) and `discovered_tmp_noexec` (registered by RHEL-10-700155), which are both registered earlier in the file and are the names the `/tmp` chain already uses
- Neither fault was reachable in a container: the `when` clause requires a separate mount that containers do not have, so the template was never evaluated and molecule stayed green throughout. Verified instead by a register-ordering sweep over the whole task file, where every `discovered_*` reference now resolves to a `register:` at a lower line number
- README "Local Testing" listed `molecule 4.0.1` plus the separate `molecule-docker`, `molecule-podman`, `molecule-vagrant` and `molecule-azure` plugins. Updated to match what the Molecule workflow installs: `ansible-core >= 2.19`, `molecule >= 25.0`, `molecule-plugins[docker]`, `docker` and `passlib`
- fixed the `end-of-file-fixer` pre-commit failure, which was red on every pull request rather than being specific to any one change. `templates/lockdown_audit.yml.j2` ended with a trailing blank line and `.qa_baseline.json` ended with `}` and no newline at all. The repo sets `autofix_prs: false`, so pre-commit.ci reported the failure instead of pushing a correction, meaning it would have failed indefinitely. Both files now end with exactly one newline and `.qa_baseline.json` still parses as JSON
- README: the Changelog link pointed at `./ChangeLog.md` while the tracked file is `Changelog.md`. Repository file paths are case-sensitive, so the link returned a 404 for every reader. The "complaint" typo in the opening heading was already correct on this branch, so only the link needed changing

## Based on STIG V1R2 - 2026 August - RHEL-10-600520 sudoers detection and remediation fixes

- RHEL-10-600520 AUDIT: the `find` scanned `/etc/sudoers.d` with `patterns: '*.conf'`, but sudoers.d entries carry no extension - sudo ignores any file with a dot in its name. The role already states this elsewhere: `tasks/prelim.yml` enumerates them with `find ... ! -name '*~' ! -name '*.*'`. The pattern therefore matched nothing, `matched` was always 0, and the disruption-high PATCH never ran. Replaced with `excludes: ['*~', '*.*']` to match the prelim convention
- RHEL-10-600520 AUDIT: dropped `read_whole_file: true` so `contains` is evaluated per line. With it enabled the regex is applied to the file as a single string, so a leading-anchor pattern is only ever tested against the first line and a file whose first line is a comment never matches
- RHEL-10-600520 PATCH: `line: '#\1'` with `backrefs: true` did not comment the offending line, it replaced it. The only capture group in the old pattern was the `(:ALL|)` alternation, so `\1` was `:ALL` or empty and `ALL ALL=(ALL) ALL` became a bare `#`, while `ALL ALL=(ALL:ALL) ALL` became `#:ALL`. The grant was neutralised but the original line was destroyed. The pattern now captures the whole line and makes the inner alternation non-capturing
- RHEL-10-600520 PATCH: the old pattern also matched lines it had already commented, so a second run produced `##...`. A leading `(?!\s*#)` guard makes re-runs a no-op. Both PATCH tasks are covered - the `/etc/sudoers.d` loop and the `/etc/sudoers` one gated on `not rhel10stig_disruption_high`
- RHEL-10-600520 WARN: the message read `.stdout_lines` from a register produced by `ansible.builtin.find`, which returns only `examined`, `files`, `matched` and `skipped_paths`. Changed to `files | map(attribute='path') | list`
- Verified against a live fixture rather than by inspection: an extension-less `/etc/sudoers.d` entry is now detected while a dotted one is correctly ignored, `ALL ALL=(ALL) ALL` becomes `#ALL ALL=(ALL) ALL` with its content intact, `root ALL=(ALL:ALL) ALL` is left alone as an authorized entry, an already-commented line is not commented again, and a second run reports no change
## Based on STIG V1R2 - 2026 August - Remove the dead auditd privileged-commands task

- removed the `POST | AUDITD | Ensure use of privileged commands is collected` task from `tasks/auditd.yml`, and the `rhel10stig_priv_command_excluded_mounts` default it was gated by. The task built its exclusion regex with `{%- set egrep_exclude = "(asdfmnop|{{ ... | join('|') }})" -%}`, which nests a `{{ }}` expression inside a `{% set %}` string. Jinja does not evaluate the inner expression there; it stays literal text. Verified against the real task on ansible-core 2.19.7 and 2.16.6, each with a passing control case, and with the variable both empty and populated - the rendered output is identical in every combination, so this never worked on any supported version rather than regressing at some point
- the practical effect was threefold: `rhel10stig_priv_command_excluded_mounts` was inert, so setting it did nothing; the unsubstituted `join('|')` collided with the surrounding single quotes so the bare `|` became a shell pipe and `) }})` ran as a command, which `failed_when: false` then swallowed; and `discovered_privileged_commands` was consumed by nothing at all. The audit rules template emits its privileged-command rules from a static path list, and no goss check in the paired audit repo referenced the discovery
- removing the variable is safe for anyone who set it in inventory precisely because it never had an effect. There is no behaviour change to the deployed auditd ruleset
- `.qa_baseline.json` regenerated: 7 findings down to 6. The `shell_pipefail` entry for `tasks/auditd.yml` is gone because its task is gone, and the Shell Pipefail Layout check now passes outright rather than being suppressed. The remaining 6 are the audit-template variable warnings. `.github/workflows/repo_qa.yml` comment updated to match the new count

## Based on STIG V1R2 - 2026 August - Molecule CI, Repo QA CI and lint hygiene

- added `.github/workflows/molecule.yml`: runs both scenarios (`default` on `rockylinux/rockylinux:10-ubi-init`, `ubi` on `redhat/ubi10-init`) end to end - create, converge, verify, destroy - on pull requests to `latest` and `benchmark*`, pushes to `latest`, a weekly schedule and manual dispatch, with `fail-fast: false` so one scenario's failure does not hide the other's result. `idempotence` is deliberately not run: converge #2 is a known non-zero-change run, so gating on it would be red for reasons that are not regressions. Goss audit JSONs upload as a build artifact but do not gate; the gate is converge finishing `failed=0`
- added `.github/workflows/repo_qa.yml`: runs the Ansible-Lockdown QA checker as a pass/fail gate on the same triggers. The checker is checked out at a pinned tag rather than pip-installed, because several of its checks load helpers from its unpackaged `scripts/` directory at run time and a module-only install would silently degrade three of them to no-ops. `--strict` is required for the gate to have teeth - most findings in this role are WARN severity and would otherwise exit 0
- added `.qa_baseline.json` recording the 7 findings that exist today, so the gate fails on new findings only. It is keyed per finding on file plus description, so a new finding inside an already-baselined check still fails. Six of the seven are audit-template variable warnings that name emitted goss keys rather than references; the seventh is `set -o pipefail` appearing after a Jinja `{%- set -%}` line that renders to nothing, so pipefail is in fact the first line the shell sees
- `molecule/default/molecule.yml` and `molecule/ubi/molecule.yml`: the container platform is now `${MOLECULE_DOCKER_PLATFORM:-linux/arm64}` instead of a hardcoded `linux/arm64`, so the same scenarios run unchanged on an Apple Silicon workstation and on amd64 CI runners
- `molecule/ubi/verify.yml` now imports `molecule/default/verify.yml` instead of duplicating it. The two files were byte-identical, and a duplicated verifier drifts silently. The file must still exist: a scenario with no `verify.yml` logs `Executed: Missing playbook` and exits 0, which reads as a passing verify step that asserted nothing
- `molecule/ubi/prepare.yml`: the supporting-package install used `ignore_errors: true`, which marks the task failed and continues. UBI repos genuinely do not carry every package in that list, so a partial install is the expected outcome rather than an error to swallow - changed to `failed_when: false`, which states that directly and clears the only ansible-lint error in the repo
- removed 8 `# noqa` directives that suppress nothing (16 -> 8). Each was verified empirically, not by inspection: every directive was stripped, ansible-lint re-run, and only those whose rule actually fires in that task block restored. ansible-lint output is unchanged apart from the `ignore_errors` fix above. Removed: `no-changed-when` x2 (handlers/main.yml), `command-instead-of-module` x1 each (tasks/Cat2/RHEL-10-400xxx.yml, tasks/Cat2/RHEL-10-700xxx.yml), `yaml[line-length]` x1 (tasks/post_remediation_audit.yml), `name[template]` x2 and `yaml[line-length]` x1 (tasks/pre_remediation_audit.yml). The surviving 8 all fire
- `vars/is_container.yml`: gated four service-activation controls that cannot start in a container - RHEL-10-200541 (chronyd, cannot adjust the clock), RHEL-10-200641 (rsyslog, fails against the container's journal socket), RHEL-10-200561 (usbguard, no USB subsystem) and RHEL-10-200602 (fapolicyd, fanotify unavailable; its sibling 200601 was already gated). The first two were hard converge failures surfaced by the new CI - 200541 on the Rocky scenario and 200641 on the UBI one - and neither was reachable before, because the scenarios had never been run end to end in CI
- `handlers/main.yml`: `Restart chronyd` and `Restart rsyslog` now carry `when: not system_is_container`, matching the seven handlers that already did. A handler gated only by its notifying control still fires when some other task notifies it
- `molecule/README_Molecule_QuickStart.md`: recorded the platform parameter and added the missing `verify.yml` row to the scenario file table. Also corrected three stale statements about the ubi scenario: two references to `ignore_errors` that the `failed_when` change above made wrong, and a `rhel10stig_disruption_high` row still claiming `false` when the scenario has set `true` since the V1R2 ubi scoping work. Added a row documenting the 26 controls that scoping disables, which was never written down

## Based on STIG V1R2 - 01 July 2026

### V1R1 -> V1R2 benchmark update

Updated to DISA RHEL 10 STIG Version 1, Release 2 (01 July 2026). No rules were added or removed (434 controls unchanged); this release is content revisions to 10 existing rules plus a correction of pre-existing SV/V-ID tag drift.

- Reconciled every task Vul_ID (`V-`) and Rule_ID (`SV-`) tag to the V1R2 XCCDF, correcting 41 tasks that carried shifted or leftover (RHEL 9) identifiers
- Bumped the DISA rule revision on the 10 rules revised in V1R2: 200530, 200611, 200691, 500040, 500410, 600730, 600750, 700750, 700920, 701130
- RHEL-10-200530: removed CCI-002314 and CCI-002322 (now CCI-000382)
- RHEL-10-700920: removed CCI-000057 (now CCI-001133)
- RHEL-10-700750: corrected the control description from 15 to 10 minutes of inactivity (idle-delay already enforced at 600 seconds)
- Reconciled all remaining task CCI tags to the V1R2 XCCDF: corrected 38 pre-existing drifted CCI sets (including malformed CCIs missing leading zeros on 000500/200600/200601 and over-applied CCI-000366) and replaced 33 `CCI-NA` placeholders with the mapped CCIs
- Replaced `NIST800-53R4_NA` with the real 800-53 control on 22 rules (derived from the role's own CCI->NIST mapping); 7 rules retain NA where the CCI has no unambiguous Rev4 mapping
- RHEL-10-300040 / 300060: corrected the title from "SSH client connections" to "SSH server connections"; RHEL-10-300070: "FIPS 140-2/140-3" -> "FIPS 140-3"
- RHEL-10-200070: corrected the CAT2 tag to CAT1 and the malformed SRG id GPOS-0042 -> GPOS-00042
- RHEL-10-200050: simplified to its actual requirement (remove tftp-server when TFTP is not required); dropped the inert/redundant `ExecStart` grep - secure-mode enforcement is handled by RHEL-10-800310
- RHEL-10-800310: gated the TFTP secure-mode drop-in on `rhel10stig_tftp_server_required` so it only runs when TFTP is required (matches 200050; avoids acting on a stale package fact)
- RHEL-10-600485: `find` `pattern` -> `contains` (content regex) and fixed the malformed `when` condition
- Handlers: removed the duplicate `Restart pcscd` and `Update aliases` handlers (finished the `listen:` migration) and dropped the wrong `sshd` package guard on the pcscd handler
- Tooling: pointed the molecule scenarios at the `benchmark_v1r2` audit branch, scoped the `ubi` scenario so it converges on the minimal UBI image, and fixed a README typo
- Tooling (follow-up): removed the inert `audit_git_version` pins from both molecule scenarios. `vars/audit.yml` loads `audit_git_version` via `include_vars` (precedence 18, which outranks molecule host/play vars) and derives it as `benchmark_{{ benchmark_version }}`, so a value set in the scenarios never took effect. Both scenarios now track the single `benchmark_version` source; override a QA branch at run time with `--extra-vars 'audit_git_version=<branch>'`. QuickStart README updated to match
- Added `rhel10stig_shell_executable` (default `/bin/bash`) in vars/main.yml and applied `args: executable` to the `ansible.builtin.shell` tasks that require a shell (pipe, glob, `&&`, or a shell builtin) so the interpreter is explicit and overridable to a POSIX shell such as dash or ash for container runs
- Converted single-command shell tasks that need no shell to `ansible.builtin.command` (RHEL-10-400100/400105/400500, 701250/701260, 200646, and the prelim sudoers-file find)
- Fixed RHEL-10-600540, which was missing the `|` block scalar so `set -o pipefail` folded into the `grep` line and the command never executed
- Converted `ansible.builtin.command` tasks that actually use shell features to `ansible.builtin.shell` so they work as intended (the command module was passing `|`/`>`/`&&`/globs as literal arguments): RHEL-10-200562, RHEL-10-200602, RHEL-10-200692, RHEL-10-400145, RHEL-10-400205, RHEL-10-400215
- RHEL-10-400285: dropped the `2>/dev/null` stderr redirect and kept the task as `ansible.builtin.command` (Ansible captures stderr natively, so no shell is needed)
- Standardized every remaining `ansible.builtin.shell` task on the multi-line block-scalar (`|`) form with `set -o pipefail` as the first line, so the convention is uniform across the role; non-piped shell tasks (glob, `&&`, redirect) now carry `set -o pipefail` too, where it is a harmless no-op
- `benchmark_version` updated to `v1r2`
- Corrected the goss binary link in `README.md` and `molecule/README_Molecule_QuickStart.md` from `goss-org/goss` to the krameff fork (`krameff/goss`), matching the migrated audit binary source (the prose links were missed when the goss source moved to krameff)
- Fixed the README Technical Dependencies section: `Python3.8` -> `Python 3.9+`, `python-def` (typo) -> `python3-dnf`, `libselinux-python` -> `python3-libselinux` (matches meta/main.yml min_ansible_version 2.16.1 and current RHEL 10 packages)
- RHEL-10-701250 / 701260: stopped commenting out `ExecStart`/`ExecStartPre` in the RPM-owned base units `/usr/lib/systemd/system/{emergency,rescue}.service` (modifying vendor-owned unit files, which `rpm -Va` integrity checks flag). The controls now deploy a reset-style systemd override drop-in (templated `emergency.service.conf` / `rescue.service.conf` with an `ExecStart=` reset followed by the sulogin `ExecStart`) under `/etc/systemd/system/*.service.d/`, leaving the vendor units untouched; DISA's check accepts the setting in the drop-in. Paired with the RHEL10-STIG-Audit goss relaxation
- RHEL-10-600010: the control set `set superusers=` in `/etc/grub.d/01_users` but never updated the `password_pbkdf2 root ${GRUB2_PASSWORD}` line, so a non-root `rhel10stig_grub_superuser` was left with no bootloader password. Added a `replace` that aligns the `password_pbkdf2` user to `rhel10stig_grub_superuser` (matches the RHEL9-STIG fix for the same bug). Also removed the unused `templates/etc/grub.d/01_users.j2` - it was orphaned (no task deployed it) and carried typos

## Based on STIG V1R1 March 2026

### July 2026 - Common alignment and QA cycle

Benchmark version unchanged (V1R1). Alignment cycle to bring the role in line with current Lockdown conventions.

- Converted `ansible_facts.<key>` dot access to bracket notation (`ansible_facts['<key>']`) across tasks and templates
- Added `set -o pipefail` to piped shell tasks so pipeline failures propagate
- Restructured audit variables:
  - Audit toggle flags moved to defaults/main.yml
  - Audit plumbing variables moved to vars/audit.yml
  - Added new `audit_bin_validate_certs` variable
  - Renamed the audit template to lockdown_audit.yml.j2
- Migrated handlers to the `ansible.builtin.systemd_service` module
- Added `when: not system_is_container` guards to handlers where required
- Switched the /tmp remount to an include-based approach
- Adopted the `discovered_*` register variable naming convention
- Slimmed the prelim tasks
- Added molecule scenarios: default and ubi
- Aligned metadata, branding, and file hygiene with Lockdown standards
- Corrected `vars/is_container.yml`: fixed the variable prefix to `rhel10stig_` (the previous `rhel_10_` prefix matched nothing, so the file was inert) and curated it to only the container-inapplicable controls (FIPS, separate filesystems, bootloader/kernel command line, auditd service, firewalld, SELinux context, systemd-unit and audit-storage controls that cannot run in a container)
- Gated the bootloader (grub rebuild / FIPS grubby) and NetworkManager handlers so container runs skip actions that require a real bootloader or absent services
- Updated the audit binary source to the krameff goss fork and bumped goss to v0.5.0
- Corrected copy-pasted SV and V identifiers on RHEL-10-500730 and RHEL-10-500740
- Fixed no-op handler notifications (auditd restart, reboot required) and corrected the RHEL-10-200730 task title to reference `pkcs11-provider`
- Added molecule `verify.yml` assertions and made the default and ubi scenarios runnable on RHEL 10
- Removed public-mirror-only workflows from this private repo (`export_badges_public`, `add_repo_issue_to_gh_project`)

Initial
