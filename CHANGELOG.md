# Changelog

All notable changes to this role are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the role uses
[semantic versioning](https://semver.org/) (`0.y.z` until the interface is
declared stable).

## [0.1.0] - 2026-10-16

### Added

- Harden OpenSSH on RHEL-family 9 and 10: key-only authentication, no root
  login, `AllowGroups`, shorter login grace time, fewer authentication attempts,
  forwarding disabled, Ed25519/RSA host keys, idle timeout and a warning banner.
- Optional firewalld restriction of SSH to trusted sources.
- Lockout guard that stops the role before any change if no user could still
  log in, and a guard that refuses to disable every login method.
- Molecule scenarios (`default`, `custom`) for Rocky Linux 10.2 and 9,
  pre-commit hooks and a GitHub Actions workflow.
