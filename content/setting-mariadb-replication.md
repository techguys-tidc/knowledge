+++ 
title = 'Setting MariaDB Replication' 
+++
- [MariaDB Replication](#mariadb-replication)
  - [Master \& Replica](#master--replica)
  - [Master](#master)
  - [Replica](#replica)
  - [Switch master to Slave](#switch-master-to-slave)

# MariaDB Replication

## Master & Replica
1. Install tool
```sh
wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb
sudo dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb
sudo apt update
sudo apt install qpress mariadb-backup -y
```
2. Edit file config
```ssh
sudo nano /etc/mysql/my.cnf
```
```cnf
...
[mysqld]
bind-address={local ip}
server_id={uniq int}
log_bin=binlog
gtid_domain_id={uniq init}
gtid_strict_mode=1
gtid_ignore_duplicates=1
```
3. Restart MariaDB
```sh
sudo systemctl restart mariadb

```

4. Check config
```sh
sudo mariadb
```
```sql
select @@server_id,@@bind_address,@@log_bin,@@gtid_strict_mode,@@gtid_domain_id;
exit
```

## Master
```sh
sudo mariadb
```
```sql
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
SHOW MASTER STATUS;
show global variables like'gtid_current_pos';

+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| gtid_current_pos | 1-1-3 |
+------------------+-------+

exit
```
```sh
sudo mariabackup --defaults-file=/etc/mysql/my.cnf --backup --compress \
--target-dir=/tmp/full_backup --user=root \
--password=password --backup --compress --parallel=4
sudo scp -i ~/.ssh/id_rsa -rp /tmp/full_backup user@{replica ip}:/tmp
```

## Replica
1. Stop MariaDB
```sh
sudo systemctl stop mariadb
sudo ps -ef| grep mariadb | grep -v grep
```
2. Backup old data
```sh
DATADIR=$(grep datadir /etc/mysql/mariadb.conf.d/50-server.cnf | cut -d'=' -f2)
cd $DATADIR
cd ..
sudo mv mysql mysql_old
```

3. Restore data from master
```sh
sudo mv /tmp/full_backup/ mysql/
MEMORY=$(grep -w innodb_buffer_pool_size /etc/mysql/mariadb.conf.d/50-server.cnf | grep -v "http" | cut -d'=' -f2 | cut -d' ' -f2)
echo $MEMORY
sudo mariabackup --decompress --parallel=4 --remove-original --use-memory=$MEMORY --target-dir=mysql
sudo mariabackup --prepare --use-memory=$MEMORY --target-dir=mysql
sudo chown -R mysql:mysql mysql
sudo systemctl start mariadb
```

4. Setting and Start Slave
```sh
sudo mariadb
change master to master_host='{master ip}', master_port=3306, master_user='repl', master_password='password', master_connect_retry=10, master_use_gtid=slave_pos;
SET GLOBAL gtid_slave_pos = '{master gtid_current_pos}';
start slave;
show slave status \G
exit
```
