[![Build Status](https://travis-ci.org/wagnerjfr/mysql-group-replication-docker-compose.svg?branch=master)](https://travis-ci.org/wagnerjfr/mysql-group-replication-docker-compose)

# Seeting up MySQL Group replication with Docker Compose

The below steps show how to setup [MySQL Group Replication](https://dev.mysql.com/doc/en/group-replication.html) with [Docker Compose](https://docs.docker.com/compose/) using [mysql/mysql-server:8.0](https://hub.docker.com/r/mysql/mysql-server) Docker images.

Other ways to setup Group Replication using Docker containers:
- https://github.com/wagnerjfr/mysql-group-replication-docker
- https://github.com/wagnerjfr/mysql-group-replication-binaries-docker

## Step 1: Starting the containers
After clonning the project, run the following command:
```
docker-compose up -d
```
Wait for the State `Up (healthy)` in all 3 containers.
```
docker-compose ps
```
A similar output should appear:
```console
Name               Command                  State                     Ports
-----------------------------------------------------------------------------------------
node1   /entrypoint.sh mysqld --se ...   Up (healthy)   0.0.0.0:3307->3306/tcp, 33060/tcp
node2   /entrypoint.sh mysqld --se ...   Up (healthy)   0.0.0.0:3308->3306/tcp, 33060/tcp
node3   /entrypoint.sh mysqld --se ...   Up (healthy)   0.0.0.0:3309->3306/tcp, 33060/tcp
```

## Step 2: Bootstrap the group
Run the command below so **node1** will bootstrap the group:
```
docker-compose exec node1 mysql -uroot -pmypass \
  -e "SET @@GLOBAL.group_replication_bootstrap_group=1;" \
  -e "create user 'repl'@'%';" \
  -e "GRANT REPLICATION SLAVE ON *.* TO repl@'%';" \
  -e "flush privileges;" \
  -e "change master to master_user='root' for channel 'group_replication_recovery';" \
  -e "START GROUP_REPLICATION;" \
  -e "SET @@GLOBAL.group_replication_bootstrap_group=0;" \
  -e "SELECT * FROM performance_schema.replication_group_members;"
```
Expected output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
| group_replication_applier | da26f305-19c5-11e9-a728-0242ac1e0002 | node1       |        3306 | RECOVERING   | SECONDARY   | 8.0.13         |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
```
`MEMBER_STATE` should appear on `RECOVERING` or `ONLINE` state after some seconds.

## Step 3: Join other nodes to group
Execute the command below to configure the other nodes and join the to group:
```
for N in 2 3
do docker-compose exec node$N mysql -uroot -pmypass \
  -e "change master to master_user='repl' for channel 'group_replication_recovery';" \
  -e "START GROUP_REPLICATION;"
done
```

## Step 4: Check the group
Execute:
```
for N in 1 2 3
do docker-compose exec node$N mysql -uroot -pmypass \
  -e "SHOW VARIABLES WHERE Variable_name = 'hostname';" \
  -e "SELECT * FROM performance_schema.replication_group_members;"
done
```

The output expected is:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| hostname      | node1 |
+---------------+-------+
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
| group_replication_applier | da26f305-19c5-11e9-a728-0242ac1e0002 | node1       |        3306 | ONLINE       | PRIMARY     | 8.0.13         |
| group_replication_applier | da4f5551-19c5-11e9-b4e5-0242ac1e0003 | node2       |        3306 | ONLINE       | SECONDARY   | 8.0.13         |
| group_replication_applier | da6bca74-19c5-11e9-b9f3-0242ac1e0004 | node3       |        3306 | ONLINE       | SECONDARY   | 8.0.13         |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| hostname      | node2 |
+---------------+-------+
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
| group_replication_applier | da26f305-19c5-11e9-a728-0242ac1e0002 | node1       |        3306 | ONLINE       | PRIMARY     | 8.0.13         |
| group_replication_applier | da4f5551-19c5-11e9-b4e5-0242ac1e0003 | node2       |        3306 | ONLINE       | SECONDARY   | 8.0.13         |
| group_replication_applier | da6bca74-19c5-11e9-b9f3-0242ac1e0004 | node3       |        3306 | ONLINE       | SECONDARY   | 8.0.13         |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| hostname      | node3 |
+---------------+-------+
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
| group_replication_applier | da26f305-19c5-11e9-a728-0242ac1e0002 | node1       |        3306 | ONLINE       | PRIMARY     | 8.0.13         |
| group_replication_applier | da4f5551-19c5-11e9-b4e5-0242ac1e0003 | node2       |        3306 | ONLINE       | SECONDARY   | 8.0.13         |
| group_replication_applier | da6bca74-19c5-11e9-b9f3-0242ac1e0004 | node3       |        3306 | ONLINE       | SECONDARY   | 8.0.13         |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+
```

## Step 5: Clean up
Run `docker-compose down` in the same path where docker-compose.yml file is located.
