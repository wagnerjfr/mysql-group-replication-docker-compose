# Setting up MySQL Group replication with Docker Compose

This repository now provides two separate Docker Compose setups:

- `mysql-8.0/docker-compose.yml` for MySQL 8.0 (`mysql/mysql-server:8.0`)
- `mysql-9.6/docker-compose.yml` for MySQL 9.6 (`mysql:9.6`)

## Full article MySQL 8.0
### [Setting up MySQL Group replication with Docker Compose](https://medium.com/gitconnected/setting-up-mysql-group-replication-with-docker-compose-7639347545a2)

## Full article MySQL 9.6
### [Build a MySQL 9.6 Group Replication Cluster with Docker Compose](https://medium.com/itnext/build-a-mysql-9-6-group-replication-cluster-with-docker-compose-7badd985118f)

## Quickstart

Prerequisite: Docker and Docker Compose installed.

1. Start the MySQL 9.6 cluster:
   ```bash
   cd mysql-9.6 && docker compose up -d
   ```

2. Wait for all 3 nodes to show `Up (healthy)`:
   ```bash
   docker compose ps
   ```

3. Bootstrap the group (run once on node1):
   ```bash
   docker compose exec node1 mysql -uroot -pmypass \
     -e "SET @@GLOBAL.group_replication_bootstrap_group=ON;" \
     -e "CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED BY 'replpass';" \
     -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';" \
     -e "FLUSH PRIVILEGES;" \
     -e "CHANGE REPLICATION SOURCE TO SOURCE_USER='repl', SOURCE_PASSWORD='replpass' FOR CHANNEL 'group_replication_recovery';" \
     -e "START GROUP_REPLICATION;" \
     -e "SET @@GLOBAL.group_replication_bootstrap_group=OFF;" \
     -e "SELECT * FROM performance_schema.replication_group_members;"
   ```

4. Join node2 and node3 to the group:
   ```bash
   for N in 2 3; do
     docker compose exec node$N mysql -uroot -pmypass \
       -e "STOP GROUP_REPLICATION;" \
       -e "RESET BINARY LOGS AND GTIDS;" \
       -e "CHANGE REPLICATION SOURCE TO SOURCE_USER='repl', SOURCE_PASSWORD='replpass' FOR CHANNEL 'group_replication_recovery';" \
       -e "START GROUP_REPLICATION;"
   done
   ```

5. Verify group membership:
   ```bash
   docker compose exec node1 mysql -uroot -pmypass \
     -e "SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE FROM performance_schema.replication_group_members;"
   ```

6. Clean up when done:
   ```bash
   docker compose down -v
   ```

Full step-by-step explanations for MySQL 8.0 and 9.6 are in the linked articles above.

***P.S.** Take a look at this [tutorial](https://medium.com/@wagnerjfr/setting-up-mysql-group-replication-with-mysql-docker-images-f5eedd44fa2b) and check how to setup MySQL Group Replication with Docker containers.*
