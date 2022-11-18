# MySQL Master-Slave replication
Replication is one of the database scaling techniques.
It implements frequent copying of data from a database in one computer or server to a database in another.

In this approach we choose one primary database server and call it Master. All data changes occur on it. 
The Slave server constantly copies all changes from the Master.
All data read requests are sent from the application to the Slave server. Thus, Master server is responsible for changing data, and Slave for reading.

### To run infrustucture use command:
``` docker compose up -d ```

To access databases use credentials: ``` username: root, password: password ```
It possible to change password for root user in the ```.env``` file.

# How to setup replication?

### 1. Setup master config
```
server_id = 99 # server ID, random int
log_bin = /var/lib/mysql/my-bin.log # path to binlog
binlog_do_db = ReplicationDb # database name
```
In our case we set the settings in the ```mysq/master.cnf``` file. To change them - update the file.

### 2. Create a user with replication permissions on master
```
CREATE USER 'slave'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%';
FLUSH PRIVILEGES;
```

### 3. Create a table to replicate and fill it with data
```
CREATE TABLE MainData 
(
	Id INT AUTO_INCREMENT PRIMARY KEY,
	Col1 nvarchar(100),
    Col2 nvarchar(100),
    Col3 nvarchar(100)
);

INSERT INTO MainData (Col1, Col2, Col3)
VALUE
	("Random Text", "1", "Column3"),
    ("Data for Column 1", "2", "Column3"),
    ("Another Text", "3", "Column3");
```

### 4. Make a dump from the master database
Set the lock for the whole database
```
use ReplicationDb;
FLUSH TABLES WITH READ LOCK;
```

Create a dump
```
mysqldump -u root -p ReplicationDb > replicadb-dump.sql
```

Unlock all tables
```
UNLOCK TABLES;
```

### 5. Check master status
Run the command
```
SHOW MASTER STATUS;
```
Output
```
+------------------+----------+-------------------+------------------+
| File             | Position | Binlog_Do_DB      | Binlog_Ignore_DB |
+------------------+----------+-------------------+------------------+
| mysql-bin.000003 |      155 | ReplicationDb     |                  |
+------------------+----------+-------------------+------------------+
```

### 6. Upload dump to the slaves
Copy the dump file to slave containers
```
docker cp master:/etc/mysql/dumps/replicadb-dump.sql ./mysql/dump.sql

docker cp ./mysql/dump.sql slave-1:/etc/mysql/replicadb-dump.sql
docker cp ./mysql/dump.sql slave-2:/etc/mysql/replicadb-dump.sql
```

Load data from the dump to slaves db (on a slave containers)
```
cd /etc/mysql/
mysql -u root -p ReplicationDb < replicadb-dump.sql
```

### 7. Set config for slave databases
```
server_id = 98 # server ID, random int
relay-log = /var/log/mysql/my-relay-bin.log
log_bin = /var/lib/mysql/my-bin.log # binlog path on M
binlog_do_db = ReplicationDb # database name
```
In our case we add settings to the ```musq/slaveX.cnf``` file. 
To update the settings - change the file.

### 8. Start the Slave
```
CHANGE MASTER TO GET_MASTER_PUBLIC_KEY=1;

CHANGE MASTER TO MASTER_HOST='master', MASTER_LOG_FILE = 'my-bin.000003', MASTER_LOG_POS = 155;
START SLAVE USER='slave' PASSWORD='password';
```
Setting MASTER_PUBLIC_KEY to 1 could not sute for the production, but it was required to make possible to connect slave to master.

### 9. Check replica status
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