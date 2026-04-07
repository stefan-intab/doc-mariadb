## Restoring a DB server similar to intab-vm2 on Iver


### Version



### Setup timezone
Old intabcloud source code seems to be compatible when DB-server has local TZ configured.

Set time zone to CET/CEST:
```bash
timedatectl set-timezone Europe/Stockholm
```

>If MariaDB is already installed and running, you need to restart it


----
#### Add MariaDB repo to APT

>Currently there is limited support of MariaDB for Debian 13, use Debian 12 for now!

```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version=mariadb-10.11
```

Try 11.8:

```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version=mariadb-11.8
```

-----
#### Install MariaBD

```bash
sudo apt install mariadb-server mariadb-client
```

----
#### Setup MariaDB Config

In this repository there is a file: `mariadb.conf` 
Copy the content and paste it into a new file on server:

```bash
nano /etc/mysql/mariadb.conf.d/70-intab.cnf
```

To activate it: 

```bash
sudo systemctl restart mariadb
sudo systemctl status mariadb
mariadbd --help --verbose | head -n 30
```

----
#### Verify time zone

```sql
SELECT 
  @@system_time_zone,
  @@global.time_zone,
  @@session.time_zone,
  NOW(),
  UTC_TIMESTAMP(),
  TIMEDIFF(NOW(), UTC_TIMESTAMP());
```
q


----
#### Restore volume

You need to restore your data to a volume on the VPS

1. Create volume 
2. Mount it, ie as /mnt/restore or /mnt/backup

Restore a specific file to /mnt/backup
```bash
rclone copy digitalocean:intabcloud-db-backups/backup/archive/full-intabcloud-backup_20260301.sql.gz /mnt/backup
```

----
#### Data volume

You probably need a database data volume large enough to handle restoring data

1. Create volume
2. Mount it as /mnt/data
3. Stop MariaDB: `systemctl stop mariadb`
4. Move location of db:
   1. Assuming MariaDB uses /var/lib/mysql
   2. `sudo rsync -av /var/lib/mysql/ /mnt/data/mysql/`
   3. `sudo chown -R mysql:mysql /mnt/data/mysql`
5. Tell MariaDB to use new datadir:
   1. Edit `/etc/mysql/mariadb.conf.d/50-server.cnf`
   2. Change: `datadir = /var/lib/mysql`
   3. To: `datadir = /mnt/data/mysql`
6. Start MariaDB: `sudo systemctl start mariadb`
7. Verify by executing: `mariadb -e "SELECT @@datadir;"`


----
### Restore


#### Restore FULL backup

Run database restore using tmux (or similar).
```bash
sudo apt install tmux
tmux
```

Restore compressed full database backup file by streaming. It should create the database intabcloud.
Execute from where backup is located:

```bash
pv full-intabcloud-backup_20260321.sql.zst | zstdcat | mariadb -uroot
```

Notes:
- pv is pipe viewer - which monitors progress of data flowing through a pipeline
- With syntax above pv shows the compressed size
- Streaming is faster than decompressing to sql file and reading a 15-20x larger file
- 

----
#### Restore structure, tables and 1 week of ws_sensor_data




----
### Handle errors

One solution: **Start from scratch**

Stop DB: `sudo systemctl stop mariadb`
Remove all DB files`sudo rm -rf /mnt/data/mysql/*`
Reinitialize: `sudo mariadb-install-db --user=mysql --datadir=/mnt/data/mysql`
or possibly: `sudo mysqld --initialize-insecure --user=mysql --datadir=/mnt/data/mysql`

Start MariaDB: `sudo systemctl start mariadb`

Test login: `mariadb -uroot`
List databases: `SHOW DATABASES;`


----

## Restore into on a new database server (Partial) 

1. Setup volumes
For old intabcloud - about 1.5 TB is needed:
Follow steps for creating a network volume for `/mnt/data`

2. Install MariaDB

3. Copy backups to VPS using rclone

4. Restore database to VPS

**Create database**

```bash
# Login
sudo mariadb -uroot -p
```

```sql
CREATE DATABASE intabcloud;
```

**Restore DB structure**
```bash
zstd -dc packed-file.sql.zst | mariadb -uroot -p intabcloud
# or
zstdcat packed-file.sql.zst | mariadb -uroot -p intabcloud
```

**Restore small tables**
```bash
zstdcat small-tables_xxxxxxxxxx.sql.zst | mariadb -uroot -p --init-command="SET FOREIGN_KEY_CHECKS = 0;" intabcloud
```

**Restore large tables**
```bash
zstdcat large-tables_xxxxxxxxxx.sql.zst | mariadb -uroot -p intabcloud
```

**Restore large tables**
```bash
zstdcat ws_sensor_data-1w_xxxxxxxxxx.sql.zst | mariadb -uroot -p intabcloud
```
> Note: use zstdcat or zstd -dc (decompress and (c) to stdout so it can be piped to mariadb)

1. Setup users 

Add intabadmin as non-sudo user (will run backups rclone etc):
```bash
sudo adduser intabadmin
```
cd .
**Store password in Bitwarden**

Also add sudo user(s) with ssh key

**/mnt/backup** should be owned by intabadmin
**crontab scripts** for sqldump should be executed by intabadmin

----

**Create SQL users:**
Login to MariaDB:
`mariadb -uroot -p`

```sql
-- Create user for apache/php
CREATE USER IF NOT EXISTS 'intabcloud'@'10.110.0.%' IDENTIFIED BY 'intabcloud';
GRANT SELECT, INSERT, UPDATE, DELETE, SHOW VIEW ON intabcloud.* TO 'intabcloud'@'10.110.0.%';

-- Backup user (socket)
CREATE USER IF NOT EXISTS 'backup'@'localhost' IDENTIFIED BY 'backup';
GRANT
  SELECT,                 -- read all tables
  SHOW VIEW,              -- dump views
  EVENT,                  -- dump events
  TRIGGER,                -- dump triggers
  -- PROCESS,                -- read metadata; not needed?
  -- RELOAD,                 -- FLUSH TABLES; not needed?
  -- LOCK TABLES,            -- for --lock-tables; not needed?
  SHOW DATABASES         -- list all databases
ON *.* TO 'backup'@'localhost';

-- Account to for remote usage (through ssh)
CREATE USER 'readonly'@'127.0.0.1' IDENTIFIED BY 'readonly';
GRANT SELECT, TRIGGER, SHOW VIEW ON intabcloud.* TO 'readonly'@'127.0.0.1';

FLUSH PRIVILEGES;
```

6. Server can go live
7. Restore full backup on another server and dump the ws_sensor data you need
8. Import that dump on server


