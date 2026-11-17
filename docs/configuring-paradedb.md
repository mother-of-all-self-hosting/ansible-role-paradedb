<!--
SPDX-FileCopyrightText: 2019-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Hardy Erlinger
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020, 2021 Aaron Raimist
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2021 Marc Leuser
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024 Kim Brose
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up ParadeDB

This is an [Ansible](https://www.ansible.com/) role which installs [ParadeDB](https://www.paradedb.com) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

ParadeDB is a [PostgreSQL](https://www.postgresql.org) extension with a custom index for full-text and vector search, BM25 scoring, filters, facets, aggregations, and joins.

See the project's [documentation](https://www.paradedb.com/docs/reference/overview) to learn what ParadeDB does and why it might be useful to you.

## Adjusting the playbook configuration

To enable ParadeDB with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# paradedb                                                             #
#                                                                      #
########################################################################

paradedb_enabled: true

# Put a strong password below, generated with `pwgen -s 64 1` or in another way
paradedb_connection_password: ""

########################################################################
#                                                                      #
# /paradedb                                                            #
#                                                                      #
########################################################################
```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, the ParadeDB instance becomes available.

### Getting a database terminal

> [!WARNING]
> **Modifying the database directly (especially as services are running) is dangerous and may lead to irreversible database corruption.** When in doubt, consider [making a backup](#backing-up-a-database). You might also want to use a web user interface software like [Adminer](https://www.adminer.org/) and [pgAdmin](https://www.pgadmin.org/).

You can use the CLI installed to the directory specified with `paradedb_bin_path` (default: `/paradedb/bin/`) for getting interactive terminal access ([psql](https://www.postgresql.org/docs/current/app-psql.html)) to the ParadeDB server.

By default, this tool puts you in the `main` database, which contains nothing.

To see the available databases, run `\list` (or just `\l`).

To change to another database (for example `miniflux`), run `\connect miniflux` (or just `\c miniflux`).

You can then proceed to write queries. Example: `SELECT COUNT(*) FROM users;`

### Configuring Unix socket support

By default the role is configured to bind-mount the ParadeDB Unix socket directory to the host at `paradedb_run_path`. It can be disabled by setting `paradedb_container_unix_socket_enabled` to `false`.

It is also possible to control if helper CLI scripts should prefer Unix socket connections (when available) instead of TCP connection with the `paradedb_cli_use_unix_socket_enabled` variable. This does not affect server behavior.

### Importing database

### Prerequisites

The playbook supports importing ParadeDB dump files in **text** (e.g. `pg_dump > dump.sql`) or **gzipped** formats (e.g. `pg_dump | gzip -c > dump.sql.gz`).

Importing multiple databases (as dumped by `pg_dumpall`) is also supported.

Before doing the actual import, **you need to upload your ParadeDB dump file to the server** (any path is okay).

### Importing a dump file

To import, run this command (make sure to replace `SERVER_PATH_TO_POSTGRES_DUMP_FILE` with a file path on your server):

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=import-paradedb \
--extra-vars=server_path_paradedb_dump=SERVER_PATH_TO_POSTGRES_DUMP_FILE \
--extra-vars=paradedb_default_import_database=main
```

💡 **Notes**:

- `SERVER_PATH_TO_POSTGRES_DUMP_FILE` must be a file path to a ParadeDB dump file on the server (not on your local machine!)
- `paradedb_default_import_database` defaults to `main`, which is useful for importing multiple databases (for dumps made with `pg_dumpall`). If you're importing a single database (e.g. `miniflux`), consider changing `paradedb_default_import_database` to the name of the database (e.g. `miniflux`)
- after importing a large database, it's a good idea to run [an `ANALYZE` operation](https://www.postgresql.org/docs/current/sql-analyze.html) to make ParadeDB rebuild its database statistics and optimize its query planner. You can easily do this via the playbook by running `ansible-playbook -i inventory/hosts setup.yml --tags=run-paradedb-vacuum -e paradedb_vacuum_preset=analyze` (refer to [Vacuuming ParadeDB](#vacuuming-paradedb) for more details).

### Example playbook

```yaml
- hosts: servers
  roles:
    - role: galaxy/com.devture.ansible.role.systemd_docker_base

    - role: galaxy/paradedb

    - role: another_role
```

Example playbook configuration (`group_vars/servers` or other):

```yaml
paradedb_identifier: my-paradedb

paradedb_base_path: "{{ my_base_path }}/paradedb"

paradedb_container_network: "{{ my_container_container_network }}"

paradedb_uid: "{{ my_uid }}"
paradedb_gid: "{{ my_gid }}"

# If enabled, unix socket will be available at `{{ paradedb_run_path }}` on the host.
paradedb_container_unix_socket_enabled: true

paradedb_vacuum_default_databases_list: ["mydb", "anotherdb"]

paradedb_systemd_services_to_stop_for_maintenance_list: |
  {{
    (['my-service.service'])
  }}

paradedb_managed_databases: |
  {{
    [{
      'name': my_database_name,
      'username': my_database_username,
      'password': my_database_password,
    }]
    +
    [{
      'name': another_database_name,
      'username': another_database_username,
      'password': another_database_password,
    }]
  }}
```

## Maintenance

Table of contents:

- [Vacuuming ParadeDB](#vacuuming-paradedb), for when you wish to run a ParadeDB [VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html) (optimizing disk space)
- [Backing up ParadeDB](#backing-up-a-database), for when you wish to make a backup
- [Upgrading ParadeDB](#upgrading-paradedb), for upgrading to new major versions of ParadeDB. Such **manual upgrades are sometimes required**.
- [Tuning ParadeDB](#tuning-paradedb) to make it run faster

### Vacuuming ParadeDB

Deleting lots data from ParadeDB does not make it release disk space, until you perform a [`VACUUM` operation](https://www.postgresql.org/docs/current/sql-vacuum.html).

You can run different `VACUUM` operations via the playbook, with the default preset being `vacuum-complete`:

- (default) `vacuum-complete`: stops all services temporarily and runs `VACUUM FULL VERBOSE ANALYZE`.
- `vacuum-full`: stops all services temporarily and runs `VACUUM FULL VERBOSE`
- `vacuum`: runs `VACUUM VERBOSE` without stopping any services
- `vacuum-analyze` runs `VACUUM VERBOSE ANALYZE` without stopping any services
- `analyze` runs `ANALYZE VERBOSE` without stopping any services (this is just [ANALYZE](https://www.postgresql.org/docs/current/sql-analyze.html) without doing a vacuum, so it's faster)

**Note**: for the `vacuum-complete` and `vacuum-full` presets, you'll need plenty of available disk space in your ParadeDB data directory (e.g. `/mash/paradedb/data`). These presets also stop all services while the vacuum operation is running.

Example playbook invocations:

```sh
# Run the default `vacuum-complete` preset and restarts all services
ansible-playbook -i inventory/hosts setup.yml --tags=run-paradedb-vacuum

# Run the `analyze` preset with all services remaining operational at all times
ansible-playbook -i inventory/hosts setup.yml --tags=run-paradedb-vacuum -e paradedb_vacuum_preset=analyze
```

### Backing up a database

The script to dump the database (`dump-all`) is installed to the directory specified with `paradedb_bin_path`. It can dump the database to a path of your choosing.

To make a one-off back up of the current ParadeDB database, make sure it's running and then execute a command like this on the server (adjust the path to the binary, etc. to how you/your playbook specifies them, e.g. `mash-paradedb`):

```sh
/usr/bin/docker exec \
--env-file=/paradedb/env-paradedb-psql \
postgres \
/usr/local/bin/pg_dumpall -h postgres \
| gzip -c \
> /paradedb.sql.gz
```

Restoring a backup made this way can be done by [importing it](#importing-database).

>[!NOTE]
> To automatically make ParadeDB database backups on a fixed schedule, consider enabling dedicated services such as [docker-paradedb-backup-local](https://github.com/prodrigestivill/docker-paradedb-backup-local).

### Upgrading ParadeDB

This role is configured to stick to the major and minor version your existing ParadeDB installation is on until you manually upgrade it, as newer ParadeDB versions cannot start with data generated by older ParadeDB versions.

This role supports upgrading your existing ParadeDB installation with the following command:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=upgrade-paradedb
```

Running the command migrates all databases, users, etc. on the ParadeDB server.

**Running the command backs up the old ParadeDB data directory** automatically. When being backing up, the directory is by default renamed to `/paradedb/data-auto-upgrade-backup` (or how you/your playbook specifies the path, e.g. `/mash/paradedb/data-auto-upgrade-backup`). The auto-upgrade-backup directory stays around forever, until you **manually decide to delete it**.

There are several options which can be set to the command. See below for details.

#### Flag: `paradedb_auto_upgrade_backup_data_path`

To have the old directory renamed to a different path, pass a flag to the command above, like this: `--extra-vars="paradedb_auto_upgrade_backup_data_path=/another/disk/paradedb-before-upgrade"`

#### Flag: `paradedb_dump_dir`

As part of the upgrade, the database is dumped to `/tmp`, an upgraded and empty ParadeDB server is started, and then the dump is restored into the new server. To use a different directory for the dump, pass a flag to the command above, like this: `--extra-vars="paradedb_dump_dir=/directory/to/dump/here"`

#### Flag: `paradedb_dump_name`

To save disk space in `/tmp`, the dump file is gzipped on the fly at the expense of CPU usage. If you have plenty of space in `/tmp` and would rather avoid gzipping, you can explicitly pass a dump filename which doesn't end in `.gz`. Example: `--extra-vars="paradedb_dump_name=paradedb-dump.sql"`

### Tuning ParadeDB

ParadeDB can be [tuned](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server) to make it run faster. This is done by passing extra arguments to the ParadeDB process.

This role **already does some tuning by default**, which matches the [tuning logic](https://github.com/le0pard/pgtune/blob/master/src/features/configuration/configurationSlice.js) done by websites like <https://pgtune.leopard.in.ua/>. You can manually influence some of the tuning variables. These parameters (variables) are injected via the `paradedb_paradedb_process_extra_arguments_auto` variable.

Most users should be fine with the automatically-done tuning. However, you may wish to:

- **adjust the automatically-determined tuning parameters manually**: change the values for the tuning variables defined in the ParadeDB role's [default configuration file](../defaults/main.yml) (see `paradedb_max_connections`, `paradedb_data_storage` etc). These variables are ultimately passed to ParadeDB via a `paradedb_paradedb_process_extra_arguments_auto` variable

- **turn automatically-performed tuning off**: override it like this: `paradedb_paradedb_process_extra_arguments_auto: []`

- **add additional tuning parameters**: define your additional ParadeDB configuration parameters in `paradedb_paradedb_process_extra_arguments_custom`. See `paradedb_paradedb_process_extra_arguments_auto` defined in the ParadeDB role's [default configuration file](../defaults/main.yml) for inspiration

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu paradedb` (or how you/your playbook named the service, e.g. `mash-paradedb`).
