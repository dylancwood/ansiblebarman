Role Name
=========

This role installs and configures PG Barman (http://www.pgbarman.org/) from the postgresql apt repo. This role will not work with non-Debian Linux distros, as it relies on `apt-get`.

Requirements
------------

None

Role Variables
--------------

All attributes listed in the default configuration file can be set via role variables.
Please see `defaults/main.yml` for a full list of variables.
Here is a list of tricky variables with a detailed description:

* **barman_home_creation_user** is the user that will create the `barman_home` directory if it does not exist. This is useful if using mounted network storage that is root-squashed.

* **barman_uid** is the user id to be used when creating the `barman_user` (optional)

* **barman_groups** is a comma-separated list of groups to be assinged to the `barman_user` (optional)

* **barman_group** is the primary group name to be assigned to the `barman_user` (optional)`

* **barman_ssh_private_key_path** is the pathname to a file containing the private ssh key to be installed for the barman user. (optional)

* **barman_ssh_public_key_path** is the pathname to a file containing the public ssh key to be installed for the barman user. (optional)

* **barman_ssh_authorized_key_path** is the pathname to a file containing the public ssh key to be used by the postgres instance when syncing WAL archives via rsync. This will be appended to the `authorized_keys` file if it exists. (optional)

* **barman_upstreams** is a list of upstream server specifications. Here is an annotated example:
``` yml
barman_upstreams:
  - name: main #required
    description: 'Main PG server' #optional
    hostname: prod_db.example.com #required
    postgres_user: postgres #optional default = postgres
    dbname: postgres #optional default = postgres
    port: 5432 #optional default = 5432
    retention_policy: 'REDUNDANCY 2' #optional
    minimum_redundancy: 1 #optional default = 1
    backup_cron_interval: '0 0 * * 0' #optional will not run automatic backups if not specified
```

* **barman_pg_pass_content** is the content that should be inserted into the `barman_user`'s .pgpass file. This is the safest way to use md5 or password authentication when connecting to the upstream postgres servers. Look at https://wiki.postgresql.org/wiki/Pgpass for more info. Example: 
```yml
barman_pg_pass_content: "*:*:*:barman_bot:{{ lookup('file', '/path/to/file/containing/password.pwd') }}"
```

Tips
------------

## Periodic checks and backups
The Barman package installed by this script configures `/etc/cron.d/barman` to run `barman cron` every minute.
In addition, you may specify a `backup_cron_interval` (e.g. `0 0 * * 0` === Weekly on Sunday at midnight) for each upstream server to be backed up.

## Configuring WAL shipping on PG servers
This playbook will not set up WAL shipping for you. This is because the WAL shipping needs to be configured on the Postgres instance that is being backed up. To achieve this on my own system, I use the `ansible_hostname` as the `name` for each upstream server, and add the following lines in my `group_vars/all` file:
```yml
barman_home: /var/lib/barman # even though this is the default, it is necessary to declare it here to make it available to other roles
barman_hostname: barman.example.com
```
I then reference that variable in my role-specific variable definitions (e.g. `group_vars/postgres-barman-upstream` to set up WAL shipping:
```yml
postgresql_wal_level: archive # minimal, archive, hot_standby, or logical
postgresql_archive_mode: on
postgresql_archive_command: 'rsync -a %p barman@{{barman_hostname}}:{{barman_home}}/{{ansible_hostname}}/wal/%f'
```

## Working with NFS storage
I ran into the following error while getting things working:
*IOError: [Errno 37] No locks available*

To resolve this, I needed to use local flock locks on my network storage:
```
mount /path/to/dir host:/path/to/dir -t nfs -o 'local_locks=flock'
```

Example Playbook
----------------

```yml
- hosts: servers
  roles:
     - role: dylancwood.ansible-barman
  vars: 
    barman_ssh_private_key_path: "~/.ssh/id_rsa.postgres"
    barman_ssh_public_key_path: "~/.ssh/id_rsa.postgres.pub"
    barman_ssh_authorized_keys_path: "~/.ssh/id_rsa.postgres.pub" 
    barman_pg_pass_content: "*:*:*:barman_bot:{{ lookup('file', '/secret/path/to/barman_bot.pwd') }}"
    barman_backup_options: concurrent_backup 
    barman_upstreams: 
      - name: 'productiondb'
        description: 'Production Database: be careful'
        hostname: 'productiondb.example.com'
        postgres_user: 'barman_bot'
        dbname: postgres
        retention_policy: 'REDUNDANCY 2'
        backup_cron_interval: '0 0 * * 0' #optional will not run automatic backups if not specified
```

Note that the `postgres_user` matches the username section of the `barman_pg_pass_content` var.
         

License
-------

MIT

Author Information
------------------

Feel free to reach out via Github Issues or Pull Requests

