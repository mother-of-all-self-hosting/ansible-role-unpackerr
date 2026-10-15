<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Unpackerr

This is an [Ansible](https://www.ansible.com/) role which installs [Unpackerr](https://unpackerr.zip/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Unpackerr is a daemon which checks for completed downloads and extracts them, so that [Sonarr](https://sonarr.tv/), [Radarr](https://radarr.video/), [Lidarr](https://lidarr.audio/), [Readarr](https://readarr.com/) and [Whisparr](https://whisparr.com/) (collectively "Starr apps") can import them. After a Starr app has imported the extracted files, Unpackerr deletes them again. It can also watch folders and extract whatever archives appear in them, without any Starr app involved.

See the project's [documentation](https://unpackerr.zip/docs/introduction) to learn what Unpackerr does and why it might be useful to you.

## Prerequisites

Unpackerr has no web UI and does not download anything by itself. It needs:

- access to the directory your download client (e.g. [qBittorrent](https://www.qbittorrent.org/)) downloads into, at exactly the same path as the Starr apps see it at
- at least one Starr app to serve, and/or a folder to watch

If you are looking for Ansible roles for these, you can check out [ansible-role-qbittorrent](https://github.com/mother-of-all-self-hosting/ansible-role-qbittorrent), [ansible-role-sonarr](https://github.com/mother-of-all-self-hosting/ansible-role-sonarr), [ansible-role-radarr](https://github.com/mother-of-all-self-hosting/ansible-role-radarr) and [ansible-role-lidarr](https://github.com/mother-of-all-self-hosting/ansible-role-lidarr).

## Adjusting the playbook configuration

To enable Unpackerr with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# unpackerr                                                            #
#                                                                      #
########################################################################

unpackerr_enabled: true

########################################################################
#                                                                      #
# /unpackerr                                                           #
#                                                                      #
########################################################################
```

### Mount the download directory

Unpackerr finds the files to extract where the Starr app says they are. Since that is a path inside the Starr app's container, Unpackerr needs to see the download directory at exactly the same path. Refer to [this page](https://unpackerr.zip/docs/install/docker#data-mount) on the documentation for details.

If you download with qBittorrent installed with [ansible-role-qbittorrent](https://github.com/mother-of-all-self-hosting/ansible-role-qbittorrent) and have mounted its download directory into your Starr apps at the path qBittorrent sees it at, add the following configuration to your `vars.yml` file to do the same for Unpackerr:

```yaml
unpackerr_container_additional_volumes_custom:
  - type: bind
    src: "{{ qbittorrent_download_path }}"
    dst: "{{ qbittorrent_download_bind_path }}"
```

Unpackerr runs as the user specified with `unpackerr_uid` / `unpackerr_gid`, and it needs to be able to write to that directory, as it extracts the archives next to them. If you use the MASH playbook, Unpackerr, qBittorrent and the Starr apps all run as the same user by default, so no adjustment is needed.

### Connecting Unpackerr to the Starr apps

If your Starr apps are installed on the same server with the roles above, you can let Unpackerr talk to them directly over the container network, instead of going through their public hostnames.

To do so, add the following configuration to your `vars.yml` file. The example connects Unpackerr to Sonarr and Radarr; add or remove entries for the Starr apps you use.

```yaml
# Lets Unpackerr reach the Starr apps by their container names
unpackerr_container_additional_networks_custom:
  - "{{ sonarr_container_network }}"
  - "{{ radarr_container_network }}"

# Starts the Starr apps whenever Unpackerr is started
unpackerr_systemd_wanted_services_list_custom:
  - "{{ sonarr_identifier }}.service"
  - "{{ radarr_identifier }}.service"

unpackerr_environment_variables_additional_variables: |
  UN_SONARR_0_URL=http://{{ sonarr_identifier }}:{{ sonarr_container_http_port }}
  UN_SONARR_0_API_KEY=YOUR_SONARR_API_KEY_HERE
  UN_SONARR_0_PATHS_0={{ qbittorrent_download_bind_path }}
  UN_RADARR_0_URL=http://{{ radarr_identifier }}:{{ radarr_container_http_port }}
  UN_RADARR_0_API_KEY=YOUR_RADARR_API_KEY_HERE
  UN_RADARR_0_PATHS_0={{ qbittorrent_download_bind_path }}
```

Replace `YOUR_SONARR_API_KEY_HERE` and `YOUR_RADARR_API_KEY_HERE` with the API keys of the Starr apps, which you can find on each of them under `Settings > General` in the `Security` section.

Unpackerr reads every Starr app's queue every 2 minutes, and extracts the downloads in it which have completed. By default it only processes downloads made over BitTorrent. If you use a Usenet client as well, add `UN_SONARR_0_PROTOCOLS=torrent,TorrentDownloadProtocol,usenet,UsenetDownloadProtocol` (and the same for the other Starr apps).

### Watching folders (optional)

Unpackerr can also extract every archive which appears in a folder, regardless of any Starr app. To have it watch a folder inside the download directory, add the following configuration to your `vars.yml` file:

```yaml
unpackerr_environment_variables_additional_variables: |
  UN_FOLDER_0_PATH={{ qbittorrent_download_bind_path }}/auto_extract
```

If you have connected Unpackerr to Starr apps as well, add this line to the same variable rather than defining it twice.

By default, the files extracted from an item in the watched folder are written to a new folder next to it, named after the item with an `_unpackerred` suffix, and nothing is deleted. Refer to [this page](https://unpackerr.zip/docs/install/configuration) on the documentation for the options to change that.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can configure Unpackerr further with the `unpackerr_environment_variables_additional_variables` variable

Refer to [this page](https://unpackerr.zip/docs/install/configuration) on the documentation for a complete list of Unpackerr's config options. Each of them can be set with the environment variable listed there, which is what you can put in `unpackerr_environment_variables_additional_variables`.

>[!NOTE]
> Unpackerr is configured through environment variables only. This role does not mount a `/config` directory into the container, so Unpackerr neither reads nor creates an `unpackerr.conf` file.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

Unpackerr has no web UI. After running the command for installation, it starts working in the background: as soon as a Starr app lists a completed download containing archives in its queue, Unpackerr extracts them in place, so that the Starr app can import the extracted files. A few minutes after the import, Unpackerr deletes the extracted files again, leaving the original download as it was (so that it can keep seeding, for example).

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu unpackerr` (or how you/your playbook named the service, e.g. `mash-unpackerr`).

On startup, Unpackerr logs the settings it is running with, including every Starr app and folder it has been configured with. If it cannot reach a Starr app, or the Starr app refuses the API key, an error is logged every time it tries to read the app's queue.

### Increase logging verbosity

If you want to see more details in the logs, add the following configuration to your `vars.yml` file:

```yaml
unpackerr_environment_variables_additional_variables: |
  UN_DEBUG=true
```
