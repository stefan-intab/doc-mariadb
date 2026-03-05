
### Version
mariadb  Ver 15.1 Distrib 10.10.7-MariaDB, for debian-linux-gnu (x86_64) using  EditLine wrapper


#### Add MariaDB repo to APT

Pin to version 10.10.7:
```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version=mariadb-10.10.7
```

DOES NOT EXIST!

```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version=mariadb-10.11
```



### Restore volume

You need to restore your data to a volume on the VPS

1. Create volume 
2. Mount it, ie as /mnt/restore or /mnt/backup

Restore a specific file to /mnt/backup
```bash
rclone copy digitalocean:intabcloud-db-backups/backup/archive/full-intabcloud-backup_20260301.sql.gz /mnt/backup
```

----

### Data volume

You need a database data volume

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
### Restore backup

Run database restore using tmux (or similar).
```bash
sudo apt install tmux
tmux
```

Restore compressed full database backup file by streaming. It should create the database intabcloud.
Execute from where backup is located:

```bash
pv full-intabcloud-backup_20260301.sql.gz | zcat | mariadb -uroot
```

Notes:
- pv is pipe viewer - which monitors progress of data flowing through a pipeline
- With syntax above pv shows the compressed size
- Streaming is faster than decompressing to sql file and reading a 15-20x larger file
- 


----

To restore into on a new database server: 

1. Install MariaDB

2. Create database: 
2.1 Use sudo mysql -uroot -p
2.2 (In SQL): CREATE DATABASE intabcloud;

1. Move backup files to DB server using 'scp'

2. Restore data:
4.1 (In bash): gunzip -c structure_20241211.sql.gz | mysql -uroot -p intabcloud
4.2 (In bash): gunzip -c small-tables_20241211.sql.gz | mysql -uroot -p --init-command="SET FOREIGN_KEY_CHECKS = 0;" intabcloud
4.3 (In bash): gunzip -c large-tables_20241211.sql.gz | mysql -uroot -p intabcloud
4.4 (In bash): gunzip -c ws_sensor_data-1w_20241211.sql.gz | mysql -uroot -p intabcloud
# In some environments will gunzip without -c just unpack file. With -c data is always sent to "|"
# FOREIGN_KEY_CHECKS is normally included in SQL-dump but not with --compact

1. Server can go live
2. Restore full backup on another server and dump the ws_sensor data you need
3. Import that dump on server