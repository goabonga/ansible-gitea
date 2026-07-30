# ansible-gitea

[![CI](https://github.com/goabonga/ansible-gitea/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/goabonga/ansible-gitea/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/goabonga/ansible-gitea.svg)](https://github.com/goabonga/ansible-gitea/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/goabonga/ansible-gitea/blob/main/LICENSE)
[![Ansible](https://img.shields.io/badge/ansible--core-%E2%89%A5%202.18-blue.svg)](https://docs.ansible.com/)
[![uv](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/uv/main/assets/badge/v0.json)](https://github.com/astral-sh/uv)

Ansible project that brings up a **self-hosted Gitea lab on your own
workstation**: two KVM/QEMU guests on an isolated libvirt network, one running
Gitea, the other running Gitea Actions agents (`act_runner`). Push a repository
with a `.gitea/workflows/` file and the jobs run locally — no GitHub, no cloud,
no account.

```text
   workstation (libvirt / qemu:///system)
   ├── network gitea-lab  192.168.170.0/24  NAT
   ├── gitea-server    192.168.170.10   Gitea 1.27 + PostgreSQL (systemd)
   │                                    http :3000   ssh :2222
   └── gitea-runner-1  192.168.170.11   2 act_runner agents + Docker
```

Everything is installed natively from upstream binaries — no container to
build, `systemctl status gitea` and `journalctl -u 'act_runner@*'` behave the
way you expect on a normal server.

## Requirements

- A Linux workstation with hardware virtualisation (`/dev/kvm`) and `sudo`
- [uv](https://docs.astral.sh/uv/) — it installs Ansible and the linters
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
| Gitea web UI | <http://192.168.170.10:3000/> | `gitea-admin` / `GiteaLab#2026` |
| Git over SSH | `ssh://git@192.168.170.10:2222/<owner>/<repo>.git` | your lab key |
| Runners | *Site administration → Actions → Runners* | — |
| Organisation | <http://192.168.170.10:3000/lab> | owned by the admin |

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

Each guest boots a thin qcow2 overlay on the shared base image, so adding one
costs seconds and a few MiB.

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
| `playbooks/services.yml` | Optional: the internal domain |
| `playbooks/gitea.yml` | PostgreSQL, Gitea, admin user, organisations, runner token |
| `playbooks/runners.yml` | Docker and the `act_runner` agents |
| `playbooks/destroy.yml` | Removes the guests, their disks and their seeds |

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

## Internal domain

One switch gives the lab its own DNS, so guests answer by name instead of by
address. It is on in this inventory:

```yaml
# inventory/group_vars/all.yml
lab_services_enabled: true   # false for a plain address-based lab
```

A dnsmasq on the Gitea guest is authoritative for `*.internal`
(`gitea.internal` for now) and forwards everything else to libvirt. It has its
own tag:

```bash
uv run ansible-playbook playbooks/services.yml --tags dnsmasq
```

Nothing points at it yet — the machines that should ask it come next.

## Customising

Everything lives in the inventory; the roles only hold defaults.

| File | Typical change |
| --- | --- |
| `inventory/local.yml` | Guests: sizing, addresses, groups |
| `inventory/group_vars/all.yml` | Network plan, base image, lab paths |
| `inventory/group_vars/gitea.yml` | Gitea version, ports, credentials, organisations |
| `inventory/group_vars/runners.yml` | Runner version, agent count, labels |

**More job concurrency** — raise `act_runner_count` (agents on the existing
guest) or add a host to the `runners` group with a free address and MAC.

**Different image** — point `lab_image_url` and `lab_image_checksum_url` at
another cloud image; anything cloud-init based and Debian-flavoured works.

**Reusing this skeleton for another project** — clone it, then rename the lab
(`lab_network_name`, `lab_network_bridge`, `lab_state_dir`) and the project
itself (`pyproject.toml`, `multicz.toml`, the README and the badges).

## Layout

```text
ansible-gitea/
├── ansible.cfg
├── inventory/
│   ├── local.yml              # the guests and the workstation
│   └── group_vars/
├── playbooks/
├── roles/
│   ├── kvm_host/              # libvirt, lab keypair, NAT network
│   ├── vm/                    # cloud image overlay + NoCloud seed + domain
│   ├── internal_dns/          # optional: dnsmasq serving *.internal
│   ├── gitea/                 # binary, PostgreSQL, app.ini, systemd, admin
│   └── act_runner/            # binary, Docker, one systemd instance per agent
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
