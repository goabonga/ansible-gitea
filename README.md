# ansible-gitea

[![CI](https://github.com/goabonga/ansible-gitea/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/goabonga/ansible-gitea/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/goabonga/ansible-gitea.svg)](https://github.com/goabonga/ansible-gitea/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/goabonga/ansible-gitea/blob/main/LICENSE)
[![Ansible](https://img.shields.io/badge/ansible--core-%E2%89%A5%202.18-blue.svg)](https://docs.ansible.com/)
[![uv](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/uv/main/assets/badge/v0.json)](https://github.com/astral-sh/uv)

Ansible project that brings up a **self-hosted Gitea lab on your own
workstation**: two KVM/QEMU guests on an isolated libvirt network, one running
Gitea, the other running Gitea Actions agents (`act_runner`). Push a repository
with a `.gitea/workflows/` file and the jobs run locally - no GitHub, no cloud,
no account.

```text
   workstation (libvirt / qemu:///system)
   ├── network gitea-lab  192.168.170.0/24  NAT
   ├── gitea-server    192.168.170.10   Gitea 1.27 + PostgreSQL (systemd)
   │                                    https :443   ssh :2222
   │                                    step-ca :9000   dnsmasq :53
   └── gitea-runner-1  192.168.170.11   2 act_runner agents + Docker
```

The Gitea guest also runs the internal CA and resolver, which is what makes the
lab reachable by name over TLS. Turn that off
(`lab_services_enabled: false`) and the lab falls back to plain
`http://192.168.170.10:3000` - see
[Internal domain and TLS](#internal-domain-and-tls).

Everything is installed natively from upstream binaries - no container to
build, `systemctl status gitea` and `journalctl -u 'act_runner@*'` behave the
way you expect on a normal server.

## Requirements

- A Linux workstation with hardware virtualisation (`/dev/kvm`) and `sudo`
- [uv](https://docs.astral.sh/uv/) - it installs Ansible and the linters
- About 6 GiB of RAM and 10 GiB of disk for the two guests
- Internet access on the first run (cloud image, Gitea and runner binaries)

The `kvm-host` playbook installs libvirt, QEMU and the rest on the
workstation; nothing else has to be prepared by hand.

## Getting started

```bash
git clone https://github.com/goabonga/ansible-gitea.git
cd ansible-gitea

uv sync                                                  # Ansible + linters
uv run ansible-galaxy collection install -r requirements.yml

uv run ansible-playbook playbooks/site.yml --ask-become-pass
```

If you prefer a plain shell, `source .venv/bin/activate` once and drop the
`uv run` prefix from every command below.

The sudo password is only used on the workstation (packages, libvirt, guest
disks); the guests themselves are driven with a passwordless key that the
first play generates in `~/.local/share/ansible-gitea/`.

Roughly ten minutes later:

| What | Where | Credentials |
| --- | --- | --- |
| Gitea web UI | <https://gitea.internal/> | `gitea-admin` / `GiteaLab#2026` |
| Git over HTTPS | `https://gitea.internal/<owner>/<repo>.git` | same |
| Git over SSH | `ssh://git@gitea.internal:2222/<owner>/<repo>.git` | your lab key |
| Runners | *Site administration → Actions → Runners* | - |
| Organisation | <https://gitea.internal/lab> | owned by the admin |

With `lab_services_enabled: false`, that is `http://192.168.170.10:3000/` and
`ssh://git@192.168.170.10:2222/...` instead.

Those credentials are lab defaults sitting in
`inventory/group_vars/gitea.yml`. Change them there (or in a vault) before the
guest is reachable by anyone but you.

An organisation named `lab` is created on the first deployment, because
repositories under an organisation get their own Actions secrets and runner
scope. Add your own, or empty the list to keep the instance bare:

```yaml
# inventory/group_vars/gitea.yml
gitea_organizations:
  - username: lab
    full_name: Lab
    description: Working area for the local lab
    visibility: private   # or public
```

## Running a workflow

Gitea reads workflows from `.gitea/workflows/` and `DEFAULT_ACTIONS_URL` is set
to GitHub, so `uses:` references resolve upstream unchanged:

```yaml
# .gitea/workflows/demo.yml
name: demo
on: [push]

jobs:
  hello:
    runs-on: ubuntu-latest      # -> docker://catthehacker/ubuntu:act-24.04
    steps:
      - uses: actions/checkout@v4
      - run: echo "built on $(hostname) at $(date)"
```

Create a repository in the web UI, push that file, and one of the agents picks
the job up within a couple of seconds.

## Playbooks

| Playbook | What it does |
| --- | --- |
| `playbooks/site.yml` | The whole lab, in order |
| `playbooks/kvm-host.yml` | Workstation: packages, libvirt, lab keypair, NAT network |
| `playbooks/provision.yml` | Creates the guests and waits for cloud-init |
| `playbooks/services.yml` | Optional: step-ca, dnsmasq, workstation wiring |
| `playbooks/gitea.yml` | PostgreSQL, Gitea, admin user, runner token, TLS certificate |
| `playbooks/runners.yml` | Docker and the `act_runner` agents |
| `playbooks/destroy.yml` | Removes the guests, their disks, and the workstation changes |

Each play is independent, so a change to Gitea alone is:

```bash
uv run ansible-playbook playbooks/gitea.yml
```

Rebuild from scratch (the base cloud image is kept, so it takes seconds):

```bash
uv run ansible-playbook playbooks/destroy.yml --ask-become-pass
uv run ansible-playbook playbooks/site.yml --ask-become-pass
```

Add `-e lab_destroy_network=true` to remove the libvirt network as well.

`destroy.yml` also puts the **workstation** back the way it was, since the lab
writes two files outside its own guests:

| Reverted | How |
| --- | --- |
| `/etc/systemd/resolved.conf.d/lab-internal.conf` | Removed, then systemd-resolved restarted |
| `/usr/local/share/ca-certificates/lab-internal-root.crt` | Removed, then `update-ca-certificates --fresh` rebuilds the bundle and drops the `/etc/ssl/certs` symlink |
| `~/.local/share/ansible-gitea/root_ca.crt` | Removed |

Kept on purpose: the lab SSH keypair, the base cloud image, and the libvirt
network (unless you pass the flag above). A copy of the root certificate that
*you* imported into a browser's own store has to be removed there by hand -
`update-ca-certificates` does not reach into NSS profiles.

## Internal domain and TLS

One switch turns the lab from a set of IP addresses into a named, TLS-served
environment. It is on in this inventory:

```yaml
# inventory/group_vars/all.yml
lab_services_enabled: true   # false for a plain http lab
```

Two daemons on the Gitea guest do the work - no extra VM, and no port that
Gitea or PostgreSQL already uses:

| Piece | Where | What it gives you |
| --- | --- | --- |
| [step-ca](https://smallstep.com/docs/step-ca) | `gitea-server:9000` | A private CA. Gitea gets a certificate for `gitea.internal`, renewed every 30 min by a systemd timer while it has less than 8 h left. |
| dnsmasq | `gitea-server:53` | Authoritative for `*.internal` (`gitea.internal`, `ca.internal`), forwards the rest to libvirt. |

Gitea then serves **https://gitea.internal** on 443, the agents re-register
against that URL by themselves, and the root certificate is added to the trust
store of the guests and - unless you set `lab_configure_workstation: false` -
of the workstation, along with a systemd-resolved drop-in routing `~internal`
to the lab. Both workstation changes are reverted by `playbooks/destroy.yml`.

```bash
# From the workstation, once services.yml has run:
git clone https://gitea.internal/lab/my-repo.git
```

Either piece can be switched off on its own - `lab_step_ca_enabled`,
`lab_dnsmasq_enabled` - and each has its own tag:

```bash
uv run ansible-playbook playbooks/services.yml --tags dnsmasq
```

Worth knowing:

- **Browsers keep their own trust store.** Firefox needs
  `security.enterprise_roots.enabled=true` (or an import of
  `~/.local/share/ansible-gitea/root_ca.crt`); Chrome reads the NSS store,
  which `update-ca-certificates` does not populate.
- **The CA's root key never leaves the guest.** Certificate requests use
  single-use 10-minute tokens minted locally by `playbooks/gitea.yml`, and only
  the root *certificate* is fetched to the workstation.
- **Moving the CA and resolver to their own guest** is a one-line change:
  point `lab_services_host` at another host of the `lab` group. Everything
  else - the DNS records, the CA URL, the resolver drop-in - follows it.
- Switching the stack on or off changes Gitea's URL, so existing clones need
  their remote updated.

## Customising

Everything lives in the inventory; the roles only hold defaults.

| File | Typical change |
| --- | --- |
| `inventory/local.yml` | Guest sizing, addresses, extra runner hosts |
| `inventory/group_vars/all.yml` | Network plan, base image, lab paths, optional stack |
| `inventory/group_vars/gitea.yml` | Gitea version, ports, credentials, DNS records, CA names |
| `inventory/group_vars/runners.yml` | Runner version, agent count, labels |

Two frequent ones:

- **More job concurrency** - raise `act_runner_count` (agents on the existing
  guest) or add a host to the `runners` group with a free address and MAC.
- **Different image** - point `lab_image_url` and `lab_image_checksum_url` at
  another cloud image; anything cloud-init based and Debian-flavoured works.

Guests get their address from cloud-init rather than DHCP, so adding one only
requires an address inside `192.168.170.0/24` and outside the DHCP range
(`.100`–`.200`).

## Layout

```text
ansible-gitea/
├── ansible.cfg
├── inventory/
│   ├── local.yml              # the two guests and the workstation
│   └── group_vars/
├── playbooks/
├── roles/
│   ├── kvm_host/              # libvirt, lab keypair, NAT network
│   ├── vm/                    # cloud image overlay + NoCloud seed + domain
│   ├── gitea/                 # binary, PostgreSQL, app.ini, systemd, admin, TLS
│   ├── act_runner/            # binary, Docker, one systemd instance per agent
│   ├── step_cli/              # optional: the `step` CLI
│   ├── step_ca/               # optional: the internal certificate authority
│   ├── internal_dns/          # optional: dnsmasq serving *.internal
│   ├── internal_resolver/     # optional: systemd-resolved routing for *.internal
│   └── internal_ca_trust/     # optional: trust the internal root certificate
├── requirements.yml           # Galaxy collections
└── multicz.toml               # versioning and changelog
```

## Development

```bash
uv run yamllint --strict .
uv run ansible-lint
uv run ansible-playbook --syntax-check playbooks/*.yml
uv run pre-commit install      # pre-commit + commit-msg hooks
```

`ansible-lint` runs at its `production` profile, and the same gates run in CI
on Python 3.11 and 3.12.

## Versioning and release

Versions are bumped from
[Conventional Commits](https://www.conventionalcommits.org/) by
[multicz](https://github.com/goabonga/multicz). On every push to `main`, CI
computes the bump, writes the changelog, tags and creates the GitHub release.
Only changes under `roles/`, `playbooks/`, `inventory/`, `ansible.cfg`,
`requirements.yml` and `pyproject.toml` count as releasable. Maintainers do not
bump versions or edit the changelog by hand.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the workflow, the commit-message
convention, and the lint expectations. By participating you agree to the
[Code of Conduct](CODE_OF_CONDUCT.md).

Security issues: please follow the disclosure process in
[SECURITY.md](SECURITY.md).

## License

Distributed under the [MIT License](LICENSE).
