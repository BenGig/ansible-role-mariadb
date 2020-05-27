# Ansible role: MariaDB

[![travis build status](https://img.shields.io/travis/fauust/ansible-role-mariadb?logo=travis)](https://travis-ci.org/fauust/ansible-role-mariadb)

Install and configure MariaDB Server on Debian/Ubuntu.

Optionally, this role also permits to:

- deploy a master/slave cluster;
- setup backups and rotation of the dumps.

## Requirements

The role use
[`mysql_user`](https://docs.ansible.com/ansible/latest/modules/mysql_user_module.html)
and
[`mysql_db`](https://docs.ansible.com/ansible/latest/modules/mysql_db_module.html)
ansible modules that depends on [PyMySQL](https://github.com/PyMySQL/PyMySQL) so
the following Python 2.7 or Python 3.X versions are needed.

For older Python versions, you may use
[MySQLdb](http://mysql-python.sourceforge.net/MySQLdb.html) but then the role
may not be idempotent (not tested).

## Fact gathering (performance)

Unless you want to use the MariaDB official repository (and then need
`ansible_distribution_version`, see [`tasks/setup.yml`](./tasks/setup.yml)) this
role does not require fact gathering.

## Role variables

Available variables are listed below, along with default values (see
[`defaults/main.yml`](./defaults/main.yml)):

### MariaDB version

```yaml
mariadb_use_official_repo: false
mariadb_use_official_repo_url: "http://ftp.igh.cnrs.fr/pub/mariadb/repo"
mariadb_use_official_repo_version: "10.4"
```

You may deploy the MariaDB Server version that comes with your distribution (Debian/Ubuntu) or
deploy the version packaged by the MariaDB Foundation.
You will find the repositories URL here:
<https://downloads.mariadb.org/mariadb/repositories/>

By default, we deploy the MariaDB Server version that comes with the distribution.

### MariaDB service activation and restart

```yaml
mariadb_enabled_on_startup: true
mariadb_can_restart: true
```

**Warning:** you may consider setting `mariadb_can_restart` to `false` on
production systems to prevent ansible runs to restart the MariaDB Server.

### General configuration

To populate the MariaDB Server configuration file, we use almost only raw
variables. This permits more flexibility and a very simple
[`templates/my.cnf.j2`](./templates/my.cnf.j2) file.

By default, some common and standard options are deployed based on MariaDB
Foundation package and it should be easy to change all of them.

#### Basic settings

```yaml
mariadb_config_file: "/etc/mysql/my.cnf"
mariadb_datadir: "/var/lib/mysql"
mariadb_port: "3306"
mariadb_bind_address: "127.0.0.1"
mariadb_unix_socket: "/run/mysqld/mysqld.sock"
```

```yaml
mariadb_basic_settings_raw: |
  user                  = mysql
  pid-file              = /run/mysqld/mysqld.pid
  socket                = {{ mariadb_unix_socket }}
  basedir               = /usr
  datadir               = {{ mariadb_datadir }}
  tmpdir                = /tmp
  lc-messages-dir       = /usr/share/mysql
  lc_messages           = en_US
  skip-external-locking
  port                  = {{ mariadb_port }}
  bind-address          = {{ mariadb_bind_address }}
```

#### Fine tuning

```yaml
mariadb_fine_tuning_raw: |
  max_connections         = 100
  connect_timeout         = 5
  wait_timeout            = 600
  max_allowed_packet      = 16M
  thread_cache_size       = 128
  sort_buffer_size        = 4M
  bulk_insert_buffer_size = 16M
  tmp_table_size          = 32M
  max_heap_table_size     = 32M
```

#### Query cache

```yaml
mariadb_query_cache_raw: |
  query_cache_size        = 16M
```

#### Logging

```yaml
mariadb_logging_raw: |
  log_error = /var/log/mysql/error.log
```

#### Character sets

```yaml
mariadb_character_sets_raw: |
  character-set-server = utf8mb4
  collation-server     = utf8mb4_general_ci
```

#### InnoDB

```yaml
mariadb_innodb_raw: |
  # InnoDB is enabled by default with a 10MB datafile in /var/lib/mysql/.
  # Read the manual for more InnoDB related options. There are many!
```

#### Mariadbdump

```yaml
mariadb_mysqldump_raw: |
  quick
  quote-names
  max_allowed_packet = 16M
```

### Databases management

```yaml
# Databases.
mariadb_databases: []
#   - name: db1
#     collation: utf8_general_ci
#     encoding: utf8
#     replicate: true|false
```

See: <https://docs.ansible.com/ansible/latest/modules/mysql_db_module.html>

### Users management

```yaml
# Users.
mariadb_users: []
#   - name: user
#     host: 100.64.200.10
#     password: password
#     priv: "*.*:USAGE/db1.*:ALL"
#     state: present|absent
```

See: <https://docs.ansible.com/ansible/latest/modules/mysql_user_module.html>

### Replication (optional)

Replication is only enabled if `mariadb_replication_role` has a value (`master` or
`slave`).

The replication setup on the slave use the GTID autopositionning
`master_use_gtid=slave_pos`. See:
<https://mariadb.com/kb/en/library/change-master-to/#master_use_gtid>

For the moment, we use an `SQL` command but in ansible 2.10, we should be able
to use the
[`mysql_replication`](https://docs.ansible.com/ansible/latest/modules/mysql_replication_module.html)
module (see
[`tasks/replication_slave.yml`](./tasks/replication_slave.yml#L09-L33) and
<https://github.com/ansible/ansible/pull/62648>).

```yaml
mariadb_replication_role: "" # master|slave
mariadb_replication_master_ip: ""
mariadb_server_id: "1"
mariadb_max_binlog_size: "100M"
mariadb_binlog_format: "MIXED"
mariadb_expire_logs_days: "10"

# Same keys as `mariadb_users` above.
# priv is set to "*.*:REPLICATION SLAVE" by default
mariadb_replication_user: []
```

### Backups (optional)

```yaml
# db dumps backup
mariadb_backup_db: false
mariadb_backup_db_cron_min: "50"
mariadb_backup_db_cron_hour: "00"
mariadb_backup_db_dir: "/mnt/backup"
mariadb_backup_db_rotation: "15"

# name of the database to dump
# (mandatory if mariadb_backup_db is set to true)
mariadb_backup_db_name: []
#   - db1
#   - db2
```

Database dump is done serially and compression (`gzip`) is done at the end to
avoid too long locks.

## Example playbook

```yaml
- hosts: db
  roles:
    - fauust.mariadb
```

## Lincense

GNU General Public License v3.0
