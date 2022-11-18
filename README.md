### 1. To run databases use command:
``` docker compose up -d ```

To access databases use credentials: ``` username: root, password: password ```

### 2. Setup master config with
```
server_id = 99 # server ID, random int
log_bin = /var/lib/mysql/my-bin.log # path to binlog
binlog_do_db = ReplicationDb # database name
```
It's possible to change settings in the mysq/master.cnf file.


### 3. Created a user with replication permissions on master
```
CREATE USER 'slave'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%';
FLUSH PRIVILEGES;
```

### 4. Create a table to replicate
```
CREATE TABLE MainData 
(
	Id INT AUTO_INCREMENT PRIMARY KEY,
	Col1 nvarchar(100),
    Col2 nvarchar(100),
    Col3 nvarchar(100)
);
```

Initialize with the test data
```
INSERT INTO MainData (Col1, Col2, Col3)
VALUE
	("Random Text", "1", "Column3"),
    ("Data for Column 1", "2", "Column3"),
    ("Another Text", "3", "Column3");
```

### 5. Set the lock for the whole database
```
use ReplicationDb;
FLUSH TABLES WITH READ LOCK;
```

### 6. Check master status
```
SHOW MASTER STATUS;
```
Output
```
	File	Position	Binlog_Do_DB	Binlog_Ignore_DB	Executed_Gtid_Set
	my-bin.000003	155	ReplicationDb
```

### 7. Create a dump
```
mysqldump -u root -p ReplicationDb > replicadb-dump.sql
```

### 8. Unlock all tables
```
UNLOCK TABLES;
```

### 9. Copy dump file to slave containers
```
docker cp master:/etc/mysql/dumps/replicadb-dump.sql ./mysql/dump.sql

docker cp ./mysql/dump.sql slave-1:/etc/mysql/replicadb-dump.sql
docker cp ./mysql/dump.sql slave-2:/etc/mysql/replicadb-dump.sql
```

### 10. Load data from the dump to slaves db (on a slave containers)
```
cd /etc/mysql/
mysql -u root -p ReplicationDb < replicadb-dump.sql
```

### 11. Set and upload config for slave databases
```
server_id = 98 # server ID, random int
relay-log = /var/log/mysql/my-relay-bin.log
log_bin = /var/lib/mysql/my-bin.log # binlog path on M
binlog_do_db = ReplicationDb # database name
```
To change configuration use musq/slaveX.cnf file

### 12. Set replica params to slave
```
CHANGE MASTER TO GET_MASTER_PUBLIC_KEY=1;

	CHANGE MASTER TO MASTER_HOST='master', MASTER_LOG_FILE = 'my-bin.000003', MASTER_LOG_POS = 155;
	START SLAVE USER='slave' PASSWORD='password';
```

### 13. Check replica status
```
SHOW SLAVE STATUS;
```
Output:
```
	Slave_IO_State: Waiting for master to send event
	Master_Host: master
	Master_User: slave
	Slave_IO_Running: Yes
	Slave_SQL_Running: Yes
```

## Effect After removing column on the slave
Was 
```
	1	Column1	Column2	Column3
	2	updated	Column2	Column3
	3	Column1	Column2	Column3
	4	Random Text	1	Column3
```
Delete
```
alter table MainData drop Col2;
```
After
```
	1	Column1	Column3
	2	updated	Column3
	3	Column1	Column3
	4	Random Text	Column3
	5	Data for Column 1	Column3
```