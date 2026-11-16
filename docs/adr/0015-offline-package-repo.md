# ADR-0015: Offline package repository on hdd-vg

- Status: Accepted
- Date: 2026-11-09
- Module: 0.2
- Related: ADR-0002 (provisional access), ADR-0012

## Context

The host is deliberately offline until Module 0.5 and is not registered
with Red Hat. Software still has to be installed on it (the virtualization
stack in this module, more in every later one). The only package source is
the RHEL 9.5 install DVD image, written to a USB stick. The image carries
complete BaseOS and AppStream repositories.

## Options

| Option | Problem |
|---|---|
| Mount the stick each time | A headless host depends on removable media being present |
| Copy to the root filesystem | About 12 GB on a 50 GiB root that should stay small |
| Copy to `hdd-vg/bulk` | `dnf` then depends on `/srv/bulk` being mounted |

## Decision

Copy `BaseOS` and `AppStream` to `/srv/bulk/repo/rhel9.5/` and point
`/etc/yum.repos.d/rhel-dvd.repo` at them with `file://` URLs. Signature
checking stays on, using the key the OS installed at
`/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release`.

Repository IDs: `dvd-baseos`, `dvd-appstream`.

## Consequences

- Installs work with nothing plugged in. About 12.7 GB of `hdd-vg/bulk`
  is used.
- If `/srv/bulk` is not mounted, every `dnf` operation fails with a curl
  "couldn't read a file" error. The runbook lists this under failure
  handling.
- The repository is frozen at 9.5 GA content. There are no security
  updates while the host is offline. Accepted while the host has no
  network; it must be revisited the day it gets one.
- The copy must be made after `/srv/bulk` is mounted, otherwise the files
  land on root and are hidden by the mount.
- `dnf` prints "not registered" notices on every run. They are expected.

## What would change this decision

Network access. Then either register the host and use the Red Hat CDN, or
serve a synchronised mirror from inside the environment and remove this
file-based repository.
