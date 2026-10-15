<!--
SPDX-FileCopyrightText: 2018-2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer, although [20.04 may be problematic](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/ansible.md#supported-ansible-versions) if you run the Ansible playbook on it)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default, but some don't (e.g. Ubuntu 18.04) and require manual installation (something like `apt-get install python3`)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## Scenarios

There are three testing scenarios available.

### `default`

Tests a standard Unpackerr installation, and is where the substance of the suite lives.

It first runs the same container image with none of the role's configuration, to establish what an Unpackerr that nobody configured chooses for itself — it starts happily and idles, so nothing else in the scenario is allowed to rest on Unpackerr merely running. The scenario then checks that:

- the container runs as the configured user, read-only with every capability dropped and nothing published, and sees the download directory at the path the role mounts it at;
- the running process reports the version `unpackerr_version` pins, matching the image tag and the image's OCI version label;
- the user, timezone, watched folder and start delay Unpackerr is running with are the ones the role configured — all of them deliberately different from what Unpackerr picks for itself — and it reads nothing but its environment;
- a release folder holding an archive, moved into the watched folder, is extracted into a new folder next to it, as files owned by the role's user holding exactly what was packed;
- the service does not restart during a window longer than the unit's `RestartSec`.

### `sonarr`

Installs Sonarr with [ansible-role-sonarr](https://github.com/mother-of-all-self-hosting/ansible-role-sonarr) alongside Unpackerr, wired together the way [the documentation](../docs/configuring-unpackerr.md#connecting-unpackerr-to-the-starr-apps) describes. It checks that:

- Sonarr hands out its queue to its API key, and refuses anonymous callers;
- starting Unpackerr brings Sonarr up with it;
- Unpackerr reaches Sonarr by its container name and reads its queue with the API key the role handed it, while the same image holding a key Sonarr never issued is refused;
- the download path Unpackerr looks in is the one the role handed it, rather than the `/downloads` it falls back to on its own;
- Sonarr and Unpackerr see a folder placed into the download directory at the same path;
- neither service restarts during a window longer than the units' `RestartSec`.

### `uninstall`

Installs Unpackerr, lets it extract an archive, and then runs the role again with `unpackerr_enabled: false` — the uninstallation path that the other scenarios never touch. It checks that the systemd unit, its service file, the rendered `env` file, the base path, the container and the container network are all gone, and that the operator's data (the archive and what was extracted from it) is not.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
molecule test --scenario-name sonarr
molecule test --scenario-name uninstall
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Ubuntu 24.04
MOLECULE_DISTRO=ubuntu2404 molecule test --scenario-name default

# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default

# Debian 12
MOLECULE_DISTRO=debian12 molecule test --scenario-name default
```
