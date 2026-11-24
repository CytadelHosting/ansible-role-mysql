# Ansible Role: MySQL

[![CI](https://github.com/geerlingguy/ansible-role-mysql/actions/workflows/ci.yml/badge.svg)](https://github.com/geerlingguy/ansible-role-mysql/actions/workflows/ci.yml)

Installs and configures MySQL or MariaDB server on RHEL/CentOS or Debian/Ubuntu servers.

Cytadel fork of geerlingguy/ansible-role-mysql. Debian keeps the package log directory mode `02750`. `mysql_secure_installation: false` leaves root authentication, `/root/.my.cnf`, anonymous users and the `test` database untouched. See `README-synchro-upstream.md`.

## Requirements

No special requirements; note that this role requires root access, so either run it in a playbook with a global `become: true`, or invoke the role in your playbook like:

```yaml
- hosts: database
  roles:
    - role: geerlingguy.mysql
      become: true
```

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
mysql_user_home: /root
mysql_user_name: root
mysql_user_password: root
```

The home directory inside which Python MySQL settings will be stored, which Ansible will use when connecting to MySQL. This should be the home directory of the user which runs this Ansible role. The `mysql_user_name` and `mysql_user_password` can be set if you are running this role under a non-root user account and want to set a non-root user.

```yaml
mysql_root_home: /root
mysql_root_username: root
mysql_root_password: root
```

The MySQL root user account details.

```yaml
mysql_root_password_update: false
mysql_secure_installation: ~
```

Whether to force update the MySQL root user's password. By default, this role will only change the root user's password when MySQL is first configured. You can force an update by setting this to `true`.

`mysql_secure_installation` gates root authentication changes, `/root/.my.cnf`, removal of remote root accounts, anonymous users and the `test` database. Left null, it follows upstream: run on package install or when `mysql_root_password_update` is true. Set it to `false` on an already installed server.

```yaml
mysql_log_dir_mode: "02750"   # Debian default in this fork, "0755" elsewhere
```

Mode of the directory containing `mysql_log_error`. Debian packages ship `/var/log/mysql` as `02750` (setgid, group `adm`). Override per host if needed.

> Note: If you get an error like `ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)` after a failed or interrupted playbook run, this usually means the root password wasn't originally updated to begin with. Try either removing  the `.my.cnf` file inside the configured `mysql_user_home` or updating it and setting `password=''` (the insecure default password). Run the playbook again, with `mysql_root_password_update` set to `yes`, and the setup should complete.

> Note: If you get an error like `ERROR 1698 (28000): Access denied for user 'root'@'localhost' (using password: YES)` when trying to log in from the CLI you might need to run as root or sudoer.

```yaml
mysql_enabled_on_startup: true
```

Whether MySQL should be enabled on startup.

```yaml
mysql_config_file: *default value depends on OS*
mysql_config_include_dir: *default value depends on OS*
mysql_config_mariadb_include_dir: /etc/mysql/mariadb.conf.d   # Debian and Arch only
```

The main my.cnf configuration file and the default include directory. Debian and Arch also read `/etc/mysql/mariadb.conf.d`, exposed as `mysql_config_mariadb_include_dir`. RedHat keeps a single directory (`/etc/my.cnf.d`).

```yaml
overwrite_global_mycnf: true
```

Whether the global my.cnf should be overwritten each time this role is run. Setting this to `false` tells Ansible to only create the `my.cnf` file if it doesn't exist. This should be left at its default value (`true`) if you'd like to use this role's variables to configure MySQL.

```yaml
mysql_skip_mycnf_management: false
mysql_config_file_link_to_mariadb_cnf: true   # default: true on Debian-family + MariaDB, false elsewhere
mysql_config_file_manage: true                # only read when the link is disabled
mysql_config_file_link_backup: true
mysql_config_mariadb_cnf: /etc/mysql/mariadb.cnf   # default: mariadb.cnf next to mysql_config_file
```

Who owns `my.cnf` (`tasks/config-file.yml`):

| Condition | What the role does with `my.cnf` |
|---|---|
| `mysql_skip_mycnf_management: true` | Nothing at all: the file in place is left as is. |
| link enabled (default on Debian + MariaDB) | Makes `my.cnf` a symlink to `mariadb.cnf` (see below). `mysql_config_file_manage` is ignored. |
| link disabled, `mysql_config_file_manage: true` | Renders `my.cnf` from `my.cnf.j2` (historical behaviour). |
| link disabled, `mysql_config_file_manage: false` | Nothing: `my.cnf` is not managed. |

`mysql_config_include_files` are always deployed afterwards.

With the link (Debian MariaDB layout: `/etc/mysql/my.cnf` links to `mariadb.cnf`, which includes `conf.d/` and `mariadb.conf.d/`):

- a missing `mariadb.cnf` is restored with the Debian default content (`templates/mariadb.cnf.j2`); an existing one is never overwritten;
- a link that already leads to `mariadb.cnf` is left alone, including the Debian alternatives chain (`my.cnf -> /etc/alternatives/my.cnf -> mariadb.cnf`);
- a regular `my.cnf` is copied to `my.cnf.bak-<date>` (`mysql_config_file_link_backup`), then replaced by a relative link; a link to another file is repointed;
- the global settings of `my.cnf.j2` (`mysql_max_connections`, `mysql_key_buffer_size`, ...) are **not deployed**: either enable `mysql_mariadb_includes_manage` (see below) so the same variables drive the package include files, or set them from `mysql_config_include_files`, typically in `mariadb.conf.d` as `90-*.cnf` so they are read after the package's `50-server.cnf`.

To keep rendering `my.cnf` from the template on a Debian MariaDB host, set `mysql_config_file_link_to_mariadb_cnf: false`. To keep a hand-made `my.cnf` without looking at it, set `mysql_skip_mycnf_management: true`.

```yaml
mysql_config_include_files: []
```

A list of files that should override the default global my.cnf. Each item requires `src`.

- `type`: `template` (default, Jinja) or `file` (copied as-is). Lookup paths are unchanged: templates are resolved from the role or playbook `templates/` directory, files from `files/`.
- `dest_dir`: `conf.d` (default, `mysql_config_include_dir`), `mariadb.conf.d` (`mysql_config_mariadb_include_dir`), or an absolute directory.
- `dest_name`: destination filename. Default is the source basename with a trailing `.j2` removed. `foo.cnf.j2` becomes `foo.cnf`. A name without `.j2` is kept as-is, so existing entries do not change path.
- `force`: replace an existing destination on every run. Default `false`.

```yaml
mysql_config_include_files:
  - src: path/relative/to/playbook/file.cnf
  - src: path/relative/to/playbook/tuning.cnf.j2
    dest_dir: mariadb.conf.d
    dest_name: 90-tuning.cnf
    force: true
  - src: path/relative/to/playbook/pmm.cnf
    type: file
    dest_dir: conf.d
```

#### MariaDB package includes driven by variables

```yaml
mysql_mariadb_includes_manage: false   # default: nothing changes
mysql_mariadb_includes_version: ''     # default: '10.11' below MariaDB 11.6, '11.8' from 11.6
mysql_mariadb_includes_exclude: []     # e.g. ['60-galera.cnf']
```

Debian + MariaDB only. With `mysql_mariadb_includes_manage: true` the role deploys in `mariadb.conf.d` the include files of the official MariaDB package, rendered from `templates/mariadb_includes/<set>/`. Every commented option of a package file is uncommented, with the value of its variable, **only when the project sets that variable** (host_vars, group_vars, `-e`). Otherwise the line stays exactly as in the package, still commented, with the package default. A rendering with no variable is therefore **byte-identical** to the package file.

"Set by the project" is not `is defined`: the role defaults define nearly every `mysql_*` variable. The role records the variables coming from the inventory in `mysql_explicit_vars` (`tasks/explicit-vars.yml`, before any role `set_fact`). A variable set to `none` or `''` keeps the package line.

```yaml
# host_vars: key_buffer_size and max_connections become active, the rest stays as the package ships it
mysql_mariadb_includes_manage: true
mysql_key_buffer_size: 512M
mysql_max_connections: 801
```

| File | Content |
|---|---|
| `50-server.cnf`, `50-client.cnf`, `50-mysqld_safe.cnf`, `60-galera.cnf` | Package files. Every commented option is driven by a variable (list in `defaults/main.yml`), and so is every **active** option the package sets (`pid-file`, `bind-address`, `expire_logs_days`, `character-set-server`, `collation-server`): the package value stays unless the project sets the variable. |
| `50-mysql-clients.cnf` (10.11) / `50-mariadb-clients.cnf` (11.8) | Package files, no option to drive (kept for completeness, the package renamed it in 11.x). |
| `70-cytadel-extra.cnf` | Role-owned. Carries the `my.cnf.j2` settings the package include files do not know (`port`, `sql_mode`, `ignore_db_dirs`, `skip-log-bin`, `table_definition_cache`, memory and InnoDB settings, `[mysqldump]`...). An option the package already drives **in the same section** is never repeated here; the same option name in **another section** is written here with its own variable (see below). |
| `80-cytadel-monitoring.cnf` | Role-owned, `performance_schema` and monitoring (Zabbix, PMM): `mysql_performance_schema_enabled`, `mysql_performance_schema_instrument`, `mysql_performance_schema_consumer_statements_digest`, `mysql_innodb_monitor_enable`, `mysql_log_slow_rate_limit`. **Deployed only if the project sets one of them**: no empty file on the other hosts. |

Same variable names as `my.cnf.j2` wherever the option exists there (`mysql_max_connections`, `mysql_table_open_cache`, `mysql_log`...), otherwise `mysql_<option>`. `table_cache` is a deprecated alias of `table_open_cache`: the active line is written `table_open_cache`. Options duplicated in `60-galera.cnf` use a `mysql_galera_` prefix so they do not leak into the server group.

**One variable per (section, option).** The same option name in two sections is two distinct settings (the last file read wins per option, whatever the section), so each one has its own variable and a variable never writes two sections:

| Option | Section | Variable |
|---|---|---|
| `bind-address` | `[mysqld]` / `[mariadbd]` · `[galera]` | `mysql_bind_address` · `mysql_galera_bind_address` |
| `binlog_format` | `[mysqld]` (70) · `[galera]` | `mysql_binlog_format` · `mysql_galera_binlog_format` |
| `innodb_flush_log_at_trx_commit` | `[mysqld]` (70) · `[galera]` | `mysql_innodb_flush_log_at_trx_commit` · `mysql_galera_innodb_flush_log_at_trx_commit` |
| `max_allowed_packet` | `[mysqld]` · `[mysqldump]` | `mysql_max_allowed_packet` · `mysql_mysqldump_max_allowed_packet` |
| `ssl_cert`, `ssl_key` | `[mysqld]` · `[client]` | `mysql_ssl_cert`, `mysql_ssl_key` · `mysql_client_ssl_cert`, `mysql_client_ssl_key` |

**Binary log.** Each option lives in exactly one file, so a project include must not repeat it: `log_bin` (`mysql_log_bin`) and `max_binlog_size` (`mysql_max_binlog_size`) are written by `50-server.cnf`, `log-bin-index` (`mysql_log_bin_index`), `binlog_format` (`mysql_binlog_format`) and `binlog_expire_logs_seconds` (`mysql_binlog_expire_logs_seconds`) by `70-cytadel-extra.cnf`. Only the variables set by the project are written (the `ROW` default of `mysql_binlog_format` is ignored, it keeps serving `my.cnf.j2`).

If a project wants the same value everywhere, it sets both variables. Mind that the server reads `[galera]` after `[mysqld]` within `60-galera.cnf` but `70-cytadel-extra.cnf` is read last: when both `[mysqld]` (70) and `[galera]` are set to different values, 70 wins.

**Value kinds of 70 and 80.** Booleans are written `ON` / `OFF` (`mysql_performance_schema_enabled: true`), `mysql_disable_log_bin: true` writes the bare option `skip-log-bin` (nothing when false), a list repeats the option (`mysql_ignore_db_dirs: [lost+found, tmp]`, several `performance-schema-instrument`). `mysql_general_log_file` sets only the log file, the general log stays off (`mysql_log` keeps enabling it and still names the file when `mysql_general_log_file` is not set).

**Retire a file.** The role never deletes a file by itself. A project that moves a static include to these variables lists it in `mysql_config_include_files_absent` (visible in `--check --diff`, restart notified, refused if the same file is also deployed):

```yaml
mysql_config_include_files_absent:
  - { dest_dir: conf.d, dest_name: custom-local.cnf }
  - { dest_dir: mariadb.conf.d, dest_name: 90-old.cnf }
```

Two template sets cover the differences between releases: `10.11` (Debian 12, every 10.x) and `11.8` (Debian 13, reference). They differ by the server group (`[mysqld]` / `[mariadbd]`), `character-set-server` (set active by the 10.11 package only), and the clients file name. To add a release, copy the package files into a new `templates/mariadb_includes/<version>/`, replace the commented options by `{{ m.opt(...) }}` / `{{ m.flag(...) }}` calls and the active ones by `{{ m.act(...) }}` (`_macros.j2`), and register the files in `vars/main.yml`.

The role list is merged with `mysql_config_include_files` (the project list, with the project's own templates and files). On the same destination the project entry wins, and `90-*.cnf` files are read last:

```yaml
mysql_mariadb_includes_manage: true
mysql_config_include_files:
  - src: mysql/includes/90-site.cnf.j2     # resolved from the playbook templates/
    dest_dir: mariadb.conf.d
    force: true
```

Requirements: `my.cnf` linked to `mariadb.cnf` or left untouched (the `my.cnf.j2` template does not include `mariadb.conf.d`); the role fails before touching any configuration file otherwise. The package files are dpkg conffiles: an upgrade keeps the role version and writes the new package one as `*.dpkg-dist`.

```yaml
mysql_databases: []
```

The MySQL databases to create. A database has the values `name`, `encoding` (defaults to `utf8`), `collation` (defaults to `utf8_general_ci`) and `replicate` (defaults to `1`, only used if replication is configured). The formats of these are the same as in the `mysql_db` module.

You can also delete a database (or ensure it's not on the server) by setting `state` to `absent` (defaults to `present`).

```yaml
mysql_users: []
```

The MySQL users and their privileges. A user has the values:

- `name`
- `host` (defaults to `localhost`)
- `password` (can be plaintext or encrypted—if encrypted, set `encrypted: true`; defaults to `omit`)
- `encrypted` (defaults to `false`)
- `priv` (defaults to `*.*:USAGE`)
- `append_privs` (defaults to `false`)
- `state` (defaults to `present`)
- `case_sensitive` (defaults to `false`)
- `update_password` (defaults to `always`)
- `tls_requires` (defaults to `{}`)
- `plugin` (defaults to `omit`)
- `plugin_auth_string` (defaults to `omit`)
- `plugin_hash_string` (defaults to `omit`)

The formats of these are the same as in the `mysql_user` module.

```yaml
mysql_packages:
  - mysql
  - mysql-server
```

(OS-specific, RedHat/CentOS defaults listed here) Packages to be installed. In some situations, you may need to add additional packages, like `mysql-devel`.

```yaml
mysql_enablerepo: ""
```

(RedHat/CentOS only) If you have enabled any additional repositories (might I suggest geerlingguy.repo-epel or geerlingguy.repo-remi), those repositories can be listed under this variable (e.g. `remi,epel`). This can be handy, as an example, if you want to install later versions of MySQL.

```yaml
mysql_python_package_debian: python3-mysqldb
```

(Ubuntu/Debian only) If you need to explicitly override the MySQL Python package, you can set it here. Set this to `python-mysqldb` if using older distributions running Python 2.

```yaml
mysql_port: "3306"
mysql_bind_address: '0.0.0.0'
mysql_datadir: /var/lib/mysql
mysql_socket: *default value depends on OS*
mysql_pid_file: *default value depends on OS*
```

Default MySQL connection configuration.

```yaml
mysql_log_file_group: mysql *adm on Debian*
mysql_log: ""
mysql_log_error: *default value depends on OS*
mysql_syslog_tag: *default value depends on OS*
```

MySQL logging configuration. Setting `mysql_log` (the general query log) or `mysql_log_error` to `syslog` will make MySQL log to syslog using the `mysql_syslog_tag`.

```yaml
mysql_slow_query_log_enabled: false
mysql_slow_query_log_file: *default value depends on OS*
mysql_slow_query_time: 2
```

Slow query log settings. Note that the log file will be created by this role, but if you're running on a server with SELinux or AppArmor, you may need to add this path to the allowed paths for MySQL, or disable the mysql profile. For example, on Debian/Ubuntu, you can run `sudo ln -s /etc/apparmor.d/usr.sbin.mysqld /etc/apparmor.d/disable/usr.sbin.mysqld && sudo service apparmor restart`.

```yaml
mysql_key_buffer_size: "256M"
mysql_max_allowed_packet: "64M"
mysql_table_open_cache: "256"
...
```

The rest of the settings in `defaults/main.yml` control MySQL's memory usage and some other common settings. The default values are tuned for a server where MySQL can consume 512 MB RAM, so you should consider adjusting them to suit your particular server better.

```yaml
mysql_disable_log_bin: false
```

This variable should be set to `true` if you don't need replication, or otherwise don't need a log of all MySQL's activity. If you leave it at the default value, disk space may be consumed at an alarming rate on highly-utlilized database servers!

```yaml
mysql_server_id: "1"
mysql_max_binlog_size: "100M"
mysql_binlog_format: "ROW"
mysql_expire_logs_days: "10"
mysql_replication_role: ''
mysql_replication_master: ''
mysql_replication_user: {}
```

Replication settings. Set `mysql_server_id` and `mysql_replication_role` by server (e.g. the master would be ID `1`, with the `mysql_replication_role` of `master`, and the slave would be ID `2`, with the `mysql_replication_role` of `slave`). The `mysql_replication_user` uses the same keys as individual list items in `mysql_users`, and is created on master servers, and used to replicate on all the slaves.

`mysql_replication_master` needs to resolve to an IP or a hostname which is accessable to the Slaves (this could be a `/etc/hosts` injection or some other means), otherwise the slaves cannot communicate to the master.

If the replication master has different IP addresses where you are running ansible and where the mysql replica is running, you can *optionally* specify a `mysql_replication_master_inventory_host` to access the machine (e.g. you run ansible on your local machine, but the mysql master and replica need to communicate on a different network)

```yaml
mysql_hide_passwords: false
```

Do you need to hide tasks' output which contain passwords during the execution ?

### systemd drop-in (service limits)

`mysql_systemd_dropin_manage: true` writes **one** drop-in, `<mysql_daemon>.service.d/<mysql_systemd_dropin_name>`, with `LimitNOFILE` (`mysql_systemd_open_file_limit`) and `OOMScoreAdjust` (`mysql_systemd_oom_score_adjust`). Off by default: nothing is written. The variable names are those of the former `cytadel_mysql` role, whose drop-in task has to be switched off when this one is on (two roles writing the same file would overwrite each other).

Our file is `override.conf`, the name `systemctl edit` writes by default. systemd reads the drop-ins in alphabetical order and the last one wins, so a second file in the directory would silently beat ours. Keeping a single drop-in is enforced in three layers:

| Layer | Variable | What it does |
|---|---|---|
| Converge | `mysql_systemd_dropin_purge: true` | Removes every other file of the directory at each run (`*.conf` read by systemd, and leftovers such as `*.bak`), except `mysql_systemd_dropin_keep` (default: `migrated-from-my.cnf-settings.conf`, created empty by the Debian package and recommended by the package comments). In `--check --diff` the removals are listed. |
| Naming | `mysql_systemd_dropin_name: override.conf` (default) | Same name as the one `systemctl edit` writes: `systemctl cat mariadb` shows our header, and a hand edit lands in our file. A hand edit is effective until the next run, which reverts it or fails on the verify step. |
| Verify | `mysql_systemd_dropin_verify: true` (default) | Reloads systemd, asks `systemctl show` what it really loaded and **fails** when `LimitNOFILE` / `OOMScoreAdjust` differ from the variables, when a reload is pending, or (purge on) when a foreign drop-in is still read, e.g. `mysql.service.d/` which is an alias of `mariadb.service.d/`. Skipped in check mode. |

A limit only applies to the process at its next start: `mysql_systemd_dropin_restart` (default `false`) restarts the service when the drop-in changes. Leave it off on a production database and restart in a maintenance window. Nothing technical stops root from running `systemctl edit`; to be told about it, run the role in `--check --diff` on a schedule (the removal shows up as a change) and, outside Ansible, watch the directory with auditd (`-w /etc/systemd/system/mariadb.service.d -p wa`).

### MariaDB usage

This role works with either MySQL or a compatible version of MariaDB. On RHEL/CentOS 7+, the mariadb database engine was substituted as the default MySQL replacement package. No modifications are necessary though all of the variables still reference 'mysql' instead of mariadb.

#### Ubuntu 14.04+ MariaDB configuration

On Ubuntu, the package names are named differently, so the `mysql_package` variable needs to be altered. Set the following variables (at a minimum):

    mysql_packages:
      - mariadb-client
      - mariadb-server
    mysql_daemon: mariadb

## Dependencies

If you have `ansible` installed (e.g. `pip3 install ansible`), none.

If you have only installed `ansible-core`, be sure to require `community.mysql` in your `collections/requirements.yml` or install it manually with `ansible-galaxy collection install community.mysql`.

## Example Playbook

    - hosts: db-servers
      become: true
      vars_files:
        - vars/main.yml
      roles:
        - { role: geerlingguy.mysql }

*Inside `vars/main.yml`*:

    mysql_root_password: super-secure-password
    mysql_databases:
      - name: example_db
        encoding: latin1
        collation: latin1_general_ci
    mysql_users:
      - name: example_user
        host: "%"
        password: similarly-secure-password
        priv: "example_db.*:ALL"

## License

MIT / BSD

## Author Information

This role was created in 2014 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
