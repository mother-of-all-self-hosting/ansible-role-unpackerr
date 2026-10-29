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

Unpackerr is a daemon which checks for completed downloads and extracts them, so that [Sonarr](https://sonarr.tv/), [Radarr](https://radarr.video/), [Lidarr](https://lidarr.audio/) and [Readarr](https://readarr.com/) (collectively "Starr apps") can import them. After a Starr app has imported the extracted files, Unpackerr deletes them again. It can also watch folders and extract whatever archives appear in them, without any Starr app involved.

Since version 1.0.0, Unpackerr has a web UI, which shows the extraction queue, the history and the logs, and on which the Starr apps, the watched folders and the notifications can be configured.

See the project's [documentation](https://unpackerr.zip/docs/introduction) to learn what Unpackerr does and why it might be useful to you.

## Upgrading from Unpackerr 0.16 or earlier

Unpackerr 1.0.0 changed how it is run, and so did this role. If you used the role before, note that:

- **The web UI is enabled by default**, and served at the hostname which you need to set with `unpackerr_hostname` (see [below](#adjusting-the-playbook-configuration)). If you would rather keep running Unpackerr without it, see [this section](#running-unpackerr-without-the-web-ui).
- Unpackerr now keeps a config file and its extraction history in the directory specified with `unpackerr_data_path` (`/unpackerr/data` by default), which the role mounts into the container. Settings saved on the web UI are written there. If you have set a log file with `UN_LOG_FILE`, Unpackerr keeps the history next to the log file instead.
- The `UN_*` environment variables you pass with `unpackerr_environment_variables_additional_variables` keep working, but the settings they hold are shown as locked on the web UI. You can keep them, or move the settings to the web UI by removing the variables.
- **Whisparr is no longer an app of its own.** If you configured it with `UN_WHISPARR_*` variables, configure it as a Radarr instance instead (`UN_RADARR_whisparr_URL`, `UN_RADARR_whisparr_API_KEY`, etc.), as described in the [upgrade guide for Whisparr](https://unpackerr.zip/docs/install/upgrading#whisparr). The role refuses to run with the old variables, as Unpackerr would silently ignore them.

Refer to [this page](https://unpackerr.zip/docs/install/upgrading) on the documentation for the complete list of changes.

## Prerequisites

Unpackerr does not download anything by itself. It needs:

- access to the directory your download client (e.g. [qBittorrent](https://www.qbittorrent.org/)) downloads into, at exactly the same path as the Starr apps see it at
- at least one Starr app to serve, and/or a folder to watch

If you are looking for Ansible roles for these, you can check out [ansible-role-qbittorrent](https://github.com/mother-of-all-self-hosting/ansible-role-qbittorrent), [ansible-role-sonarr](https://github.com/mother-of-all-self-hosting/ansible-role-sonarr), [ansible-role-radarr](https://github.com/mother-of-all-self-hosting/ansible-role-radarr) and [ansible-role-lidarr](https://github.com/mother-of-all-self-hosting/ansible-role-lidarr).

To serve the web UI at a hostname, this role relies on a [Traefik](https://traefik.io/) reverse-proxy server. If you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook), it is set up for you.

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

### Set the hostname

To enable the web UI you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
unpackerr_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

>[!NOTE]
> The `unpackerr_path_prefix` variable can be adjusted to host under a subpath (e.g. `unpackerr_path_prefix: /unpackerr`). The role passes it to Unpackerr as its URL base.

### Running Unpackerr without the web UI

If you do not need the web UI, you can turn off Unpackerr's HTTP server, which serves the web UI and its API. Unpackerr then works in the background, as versions before 1.0.0 did, and is configured with environment variables only. `unpackerr_hostname` does not need to be set in this case.

To do so, add the following configuration to your `vars.yml` file:

```yaml
unpackerr_webserver_enabled: false
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

### Set the password for the web UI (optional)

By default, Unpackerr generates a temporary password for the `admin` user on its first start, which you can change on the web UI after logging in. See [this section](#logging-in-to-the-web-ui) for details.

If you would rather set the password yourself, add the following configuration to your `vars.yml` file. Make sure to replace `YOUR_PASSWORD_HERE` with a password of at least 8 characters.

```yaml
unpackerr_environment_variables_un_webserver_ui_password: admin:YOUR_PASSWORD_HERE
```

A password set this way is applied every time Unpackerr starts, and cannot be changed on the web UI.

To let a reverse-proxy authenticate users instead, refer to [this page](https://unpackerr.zip/docs/install/web-ui#reverse-proxy) on the documentation. This requires the reverse-proxy to be trusted, as described below.

### Configuring trusted proxies (optional)

For Unpackerr to accept the original client address and HTTPS scheme which Traefik passes to it, the proxy's address or network needs to be trusted. Unpackerr then marks its login cookie as secure, and logs the real client addresses.

First, inspect the Docker network shared by Traefik and Unpackerr on the server:

```sh
docker network inspect NETWORK_NAME --format '{{ range .IPAM.Config }}{{ println .Subnet }}{{ end }}'
```

Replace `NETWORK_NAME` with that network's actual name. Keep in mind that only the proxy's address or the specific subnet it connects from should be trusted. Trusting a subnet also trusts other containers attached to it. For an external proxy, use its source address or subnet as seen by Unpackerr.

Then add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
# This is an example. Replace the value with the actual proxy subnet.
unpackerr_environment_variables_un_webserver_upstreams:
  - 172.20.0.0/24
```

### Connecting Unpackerr to the Starr apps

Unpackerr needs the URL and the API key of each Starr app it serves. You can find the API key on each Starr app under `Settings > General` in the `Security` section.

If your Starr apps are installed on the same server with the roles above, you can let Unpackerr talk to them directly over the container network, instead of going through their public hostnames. To do so, add the following configuration to your `vars.yml` file. The example connects Unpackerr to Sonarr and Radarr; add or remove entries for the Starr apps you use.

```yaml
# Lets Unpackerr reach the Starr apps by their container names
unpackerr_container_additional_networks_custom:
  - "{{ sonarr_container_network }}"
  - "{{ radarr_container_network }}"

# Starts the Starr apps whenever Unpackerr is started
unpackerr_systemd_wanted_services_list_custom:
  - "{{ sonarr_identifier }}.service"
  - "{{ radarr_identifier }}.service"
```

**Note**: if you use the MASH playbook, this is done for you for Sonarr, Radarr and Lidarr.

Then add the Starr apps on the web UI under `Settings`, with the URLs `http://sonarr:8989` and `http://radarr:7878` (or how you/your playbook named the services, e.g. `http://mash-sonarr:8989`). Each one has a `Test` button, which checks that Unpackerr can reach it with the API key, before saving it.

Alternatively, you can configure them with environment variables, as below. The settings configured this way are shown as locked on the web UI.

```yaml
unpackerr_environment_variables_additional_variables: |
  UN_SONARR_0_URL=http://{{ sonarr_identifier }}:{{ sonarr_container_http_port }}
  UN_SONARR_0_API_KEY=YOUR_SONARR_API_KEY_HERE
  UN_SONARR_0_PATHS_0={{ qbittorrent_download_bind_path }}
  UN_RADARR_0_URL=http://{{ radarr_identifier }}:{{ radarr_container_http_port }}
  UN_RADARR_0_API_KEY=YOUR_RADARR_API_KEY_HERE
  UN_RADARR_0_PATHS_0={{ qbittorrent_download_bind_path }}
```

Unpackerr reads every Starr app's queue every 2 minutes, and extracts the downloads in it which have completed. By default it only processes downloads made over BitTorrent. If you use a Usenet client as well, add `usenet,UsenetDownloadProtocol` to the protocols of the Starr app on the web UI, or add `UN_SONARR_0_PROTOCOLS=torrent,TorrentDownloadProtocol,usenet,UsenetDownloadProtocol` (and the same for the other Starr apps).

[Whisparr](https://whisparr.com/) speaks the API of Radarr, so add it as a Radarr instance, named `Whisparr`.

### Watching folders (optional)

Unpackerr can also extract every archive which appears in a folder, regardless of any Starr app. You can add a folder to watch on the web UI under `Settings`, or with environment variables. To have it watch a folder inside the download directory, add the following configuration to your `vars.yml` file:

```yaml
unpackerr_environment_variables_additional_variables: |
  UN_FOLDER_0_PATH={{ qbittorrent_download_bind_path }}/auto_extract
```

If you have connected Unpackerr to Starr apps with environment variables as well, add this line to the same variable rather than defining it twice.

By default, the files extracted from an item in the watched folder are written to a new folder next to it, named after the item with an `_unpackerred` suffix, and nothing is deleted. Refer to [this page](https://unpackerr.zip/docs/install/configuration#watch-folders) on the documentation for the options to change that.

Unpackerr notices new items in a watched folder by filesystem events. If the folder is on network storage (e.g. NFS or CIFS), which may not deliver those events, have Unpackerr poll the folder instead, by setting its poll interval (e.g. to `1s`) on the web UI, or by adding `UN_FOLDER_0_INTERVAL=1s`.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `unpackerr_environment_variables_additional_variables` variable

Refer to [this page](https://unpackerr.zip/docs/install/configuration) on the documentation for a complete list of Unpackerr's config options that you can put in `unpackerr_environment_variables_additional_variables`.

>[!NOTE]
> Settings passed as environment variables take precedence over the config file that Unpackerr keeps in `unpackerr_data_path`, and cannot be changed on the web UI. The role always passes `UN_WEBSERVER_LISTEN_ADDR` and `UN_WEBSERVER_URLBASE`, as the reverse-proxy configuration depends on them, so set them with `unpackerr_webserver_enabled` / `unpackerr_container_http_port` and `unpackerr_path_prefix` instead. The web UI password and the trusted proxies have variables of their own too (`unpackerr_environment_variables_un_webserver_ui_password` and `unpackerr_environment_variables_un_webserver_upstreams`). The role refuses to run if any of these four settings is in `unpackerr_environment_variables_additional_variables`.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, Unpackerr starts working in the background. It checks the queues of the Starr apps every 2 minutes, and when it finds a completed download containing archives, it extracts them in place after a short delay (1 minute by default), so that the Starr app can import the extracted files. A few minutes after the import, Unpackerr deletes the extracted files again, leaving the original download as it was (so that it can keep seeding, for example).

The web UI becomes available at the specified hostname like `https://example.com`.

### Logging in to the web UI

The user is `admin`. Unless you have [set the password yourself](#set-the-password-for-the-web-ui-optional), Unpackerr generates a temporary password on its first start, and logs it once. To find it, log in to the server with SSH and run the command below (replace `unpackerr` with how you/your playbook named the service, e.g. `mash-unpackerr`):

```sh
journalctl -u unpackerr | grep 'Generated temporary UI password'
```

After logging in with it, change the password under `Settings > Web server`. The new password is kept (hashed) in the config file in `unpackerr_data_path`.

#### Resetting the password

If you have forgotten the password, stop the service, have Unpackerr write a new password into its config file, and start the service again. Replace `unpackerr`, the user, the path and the version as appropriate for your installation (they are the values of `unpackerr_identifier`, `unpackerr_uid` / `unpackerr_gid`, `unpackerr_data_path` and `unpackerr_container_image`):

```sh
systemctl stop unpackerr
docker run --rm --user 1000:1000 --mount type=bind,src=/unpackerr/data,dst=/config ghcr.io/unpackerr/unpackerr:v1.0.0 --reset
systemctl start unpackerr
```

The new password for the `admin` user is printed by the `docker run` command.

If you have set the password with `unpackerr_environment_variables_un_webserver_ui_password`, resetting it this way has no effect, as that password is applied every time Unpackerr starts. Change it in your `vars.yml` file instead.

### Metrics

Unpackerr can expose metrics for Prometheus at `/metrics`, if `UN_WEBSERVER_METRICS=true` is added to `unpackerr_environment_variables_additional_variables`. Scraping them requires an API key with the `system:metrics:read` permission, which can be created on the web UI under `Settings > Web server`. Refer to [this page](https://unpackerr.zip/docs/install/web-ui#api-keys-and-metrics) on the documentation for details.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu unpackerr` (or how you/your playbook named the service, e.g. `mash-unpackerr`). They can also be followed on the web UI.

On startup, Unpackerr logs the settings it is running with, including every Starr app and folder it has been configured with. If it cannot reach a Starr app, or the Starr app refuses the API key, an error is logged every time it tries to read the app's queue.

If Unpackerr logs `Could not persist config`, it cannot write to the directory specified with `unpackerr_data_path`, and the settings saved on the web UI are lost when it restarts. Make sure that the directory is owned by the user specified with `unpackerr_uid` / `unpackerr_gid`.

#### Increase logging verbosity

If you want to increase the verbosity, add the following configuration to your `vars.yml` file:

```yaml
unpackerr_environment_variables_un_debug: true
```
