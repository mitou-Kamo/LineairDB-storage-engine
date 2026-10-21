# LineairDB MySQL Custom Storage Engine

## How to Build

```
git clone --recursive git@github.com:Tatzhiro/LineairDB-storage-engine.git
cd LineairDB-storage-engine
./build.sh
```

Initialize MySQL:
```bash
build/bin/mysqld --defaults-file=my.cnf --initialize-insecure
```

## Setting up semi-synchronous replication
On primary node, run the following:
```bash
build/bin/mysqld --defaults-file=my.cnf --daemonize
build/bin/mysql -u root  -e "
  CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'replpass';
  GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
  FLUSH PRIVILEGES;
  RESET MASTER;
"
```

After that, change the server-id in my_replica.cnf on replica nodes so that each node has a unique server-id.
```
[mysqld]
basedir=...
...

server-id=${UNIQUE_NUMBER}
...
```
Then, run the following on the replicas.
Make sure to set `PRIMARY` to the IP address of the primary node.
```bash
build/bin/mysqld --defaults-file=my_replica.cnf --daemonize
build/bin/mysql -u root  -e "
  CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'replpass';
  GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
  FLUSH PRIVILEGES;

  STOP REPLICA;
  RESET REPLICA ALL;
  CHANGE REPLICATION SOURCE TO
    SOURCE_HOST=$PRIMARY,
    SOURCE_PORT=3306,
    SOURCE_USER='repl',
    SOURCE_PASSWORD='replpass',
    SOURCE_AUTO_POSITION=1;
  CHANGE REPLICATION SOURCE TO SOURCE_CONNECT_RETRY = 2;
  START REPLICA;

  SET GLOBAL rpl_semi_sync_replica_enabled = 1;
  STOP REPLICA IO_THREAD;
  START REPLICA IO_THREAD;
```

Finally, run these commands on the primary node:
```bash
build/bin/mysql -u root  -e "
  SET GLOBAL rpl_semi_sync_source_enabled = 1;
  SET GLOBAL rpl_semi_sync_source_timeout = 4294967295;
  SET GLOBAL rpl_semi_sync_source_wait_for_replica_count = 1;
"
```

Add these to my.cnf as well so that semi-sync setup persists even when primary node gets restarted:
```
rpl_semi_sync_source_enabled = 1
rpl_semi_sync_source_timeout = 4294967295
rpl_semi_sync_source_wait_for_replica_count = 1
```


```bash
# verify semi-sync replication setup is successful.

# OK if the command returns the number of replicas
build/bin/mysql -u root  -e "SHOW STATUS LIKE 'Rpl_semi_sync_source_clients';"
# OK if ON
build/bin/mysql -u root  -e "SHOW STATUS LIKE 'Rpl_semi_sync_source_status';";
```


## Benchmark

see [bench/README.md]

## How to Contribute

## Acknowledgments

This work is partially based on results obtained from the project "Research and Development Project of the Enhanced Infrastructures for Post-5G Information and Communication Systems (JPNP20017)" by the New Energy and Industrial Technology Development Organization (NEDO).
