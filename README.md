[![Molecule](https://github.com/iamenr0s/ansible-role-apparmor-hardening/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-apparmor-hardening/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/apparmor_hardening) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-apparmor-hardening/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-apparmor-hardening)

# Ansible Role: Apparmor Hardening

> Standalone copy of `apparmor_hardening` from the [iamenr0s.linux_hardening](https://github.com/iamenr0s/linux-hardening-collection) collection. Variables keep the collection-wide `linux_hardening_*` prefix.

Audit-first AppArmor for the Debian family (Debian, Ubuntu). Other families are **skipped** with a message and `linux_hardening_apparmor_findings: []` (EL/Fedora: see `selinux_hardening`). It does not touch the bootloader and never reboots.

## What it checks
| Check | Remediation |
|---|---|
| `apparmor` and `apparmor-utils` installed | installed |
| AppArmor active in the kernel (`/sys/module/apparmor/parameters/enabled` and the securityfs interface) | **reported only**: `apparmor=0` / `security=` on the command line is a bootloader matter and needs a reboot |
| `apparmor.service` enabled and active | enabled; started when the kernel interface exists |
| At least one profile loaded | reported only |
| `cis_level2` only: no profile in complain mode | `aa-enforce` (needs consent, see below) |

## The enforce tier
A stock Debian 13 loads 110 profiles but ships many in **complain** mode on purpose (desktop apps, browsers, Xorg). Making them all enforcing is the `cis_level2` profile, and it can break the applications they confine, so remediation stops unless `linux_hardening_apparmor_confirm_enforce: true`. Leave some alone with `linux_hardening_apparmor_complain_allowed: [plasmashell]` (the parent name; its `//child` profiles follow it).

`aa-enforce` exits 0 even when it fails, so the role never trusts the exit code: it enforces by profile name, re-reads the live modes, then falls back to the profile *file* for names that are not a path (e.g. profiles whose executable is not installed). Anything still in complain mode is reported as `fixable: false`. Runs are idempotent: a change is counted only when a profile actually left complain mode.

## Modes
`audit` (default) is read-only and sets `linux_hardening_apparmor_findings` (`key, expected, actual, fixable`). `remediate` needs `linux_hardening_confirm_remediation: true`; after it only fixable findings fail the run.

## Example
```yaml
- hosts: debian_family
  roles:
    - role: iamenr0s.linux_hardening.apparmor_hardening
      vars:
        linux_hardening_mode: remediate
        linux_hardening_confirm_remediation: true
        linux_hardening_profile: cis_level2
        linux_hardening_apparmor_confirm_enforce: true
        linux_hardening_apparmor_complain_allowed: [plasmashell]
```

## Verification
Container scenarios `default` (cis_level1: packages and service), `audit`, `refuse`; EL containers check the skip. A container shares the host kernel and has no AppArmor securityfs, so kernel-level checks are reported there, not enforced. **VM scenario `vagrant`** (libvirt, `debian/trixie64`; `sg libvirt -c "molecule test -s vagrant"`): `cis_level2` refuses without consent, then enforces all 21 complain parents except the allowed one (including the `transmission-*` profiles that need the file fallback), the service is enabled and active, the audit is `COMPLIANT`, and a second run enforces 0. Tested: Debian 13 (container and VM), Ubuntu 24.04 (container), AlmaLinux 9 (skip). CI runs the container scenarios only.

## Rollback
`aa-complain <profile>` per profile (the profile files under `/etc/apparmor.d` are edited by `aa-enforce`, so dpkg may ask about them on upgrade).

## Known gaps
- Not tested on an Ubuntu VM, Debian 12, or with `AppArmor` disabled on the kernel command line (reported, not exercised).
- Does not write or tune profiles, check unconfined network services (`aa-unconfined`), or touch `unconfined`-flag profiles. CIS IDs not mapped.

CI & Release (maintainers)
----------------------------
A single workflow (`.github/workflows/molecule.yml`) runs lint and the full Molecule distro matrix on pushes to `main`, PRs, and `v*` tags. On `v*` tags, a `release` job publishes to Ansible Galaxy after all tests pass.

One-time setup:

```bash
# Galaxy publishing key (environment-scoped, get it from galaxy.ansible.com/ui/token)
gh secret set GALAXY_API_KEY --repo iamenr0s/ansible-role-apparmor-hardening

# Code scanning notifications (Slack webhook URL; for Discord append /slack to the webhook URL)
gh secret set SECURITY_ALERT_WEBHOOK --repo iamenr0s/ansible-role-apparmor-hardening
```

`.github/workflows/code-scanning-notify.yml` polls the code-scanning API every 6 hours and posts new or updated open alerts to that webhook (GitHub Actions cannot trigger on `code_scanning_alert` directly).

To release: tag a commit `vX.Y.Z` and push the tag — CI gates the Galaxy publish.

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the
local pipeline commands and pull request checklist. This project follows the
[Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## Security

See [SECURITY.md](SECURITY.md) — GitHub private vulnerability reporting, no
public issues for security bugs.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and release notes.

## License

This project is licensed under the [MIT License](LICENSE).

## Author Information

Author: iamenr0s

Galaxy: `iamenr0s.apparmor_hardening`
