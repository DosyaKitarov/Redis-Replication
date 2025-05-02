# Redis Replication with TLS and Sentinel Configuration

This project sets up a Redis replication with TLS encryption and Sentinel for high availability using Docker Compose.

## Features
- **Redis Master-Slave Replication**: A master node with two replicas (slaves).
- **TLS Encryption**: Secure communication between Redis nodes and clients.
- **Redis Sentinel**: High availability and automatic failover.
- **Dockerized Setup**: Easy deployment using Docker Compose.

## Directory Structure
```
redisTLS/
├───config
│   ├───redis-master    
│   │   └───mount         # Directory for dynamically updated Redis master configuration
│   ├───redis-sentinel1       
│   │   └───mount         # Directory for dynamically updated Sentinel 1 configuration
│   ├───redis-sentinel2       
│   │   └───mount         # Directory for dynamically updated Sentinel 2 configuration
│   ├───redis-sentinel3       
│   │   └───mount         # Directory for dynamically updated Sentinel 3 configuration
│   ├───redis-slave1
│   │   └───mount         # Directory for dynamically updated Redis slave 1 configuration
│   └───redis-slave2
│       └───mount         # Directory for dynamically updated Redis slave 2 configuration
├── redis-tls/              # TLS certificates
├── data/                   # Data directories for Redis nodes
├── docker-compose.redis.yml # Docker Compose file
└── clear.sh                # Script to clear and copy configuration files
```

### Configuration Management
Redis dynamically updates its configuration files during runtime. The `mount` directories store these dynamically updated configurations. However, the base configuration files in the `config` directory serve as a fallback for recovery scenarios. For example, if the master node fails and a slave node is promoted to master, the dynamically updated configuration in the `mount` directory reflects this change. The base configuration files remain unchanged and can be used to reset or reinitialize the setup if needed.

## Prerequisites
- Docker and Docker Compose installed.
- TLS certificates placed in the `redis-tls/` directory:
  - `redis.crt`: Certificate for Redis nodes.
  - `redis.key`: Private key for Redis nodes.
  - `ca.crt`: Certificate Authority file.

## Usage

### 1. Write Configurations
Place the following `redis.conf` and `sentinel.conf` files in the respective directories, including the `mount` directories.

#### redis.conf for Master
```
bind 0.0.0.0
port 0
masterauth <password>
tls-port 6379
tls-cert-file /etc/redis/certs/redis.crt
tls-key-file /etc/redis/certs/redis.key
tls-ca-cert-file /etc/redis/certs/ca.crt
tls-cluster yes
tls-auth-clients yes
requirepass <password>
tls-protocols "TLSv1.2 TLSv1.3"
tls-replication yes
repl-diskless-load on-empty-db
```

#### redis.conf for Slaves
```
bind 0.0.0.0
port 0
replicaof redis-master 6379
masterauth <password>
tls-port 6379
tls-cert-file /etc/redis/certs/redis.crt
tls-key-file /etc/redis/certs/redis.key
tls-ca-cert-file /etc/redis/certs/ca.crt
tls-cluster yes
tls-auth-clients yes
requirepass <password>
tls-protocols "TLSv1.2 TLSv1.3"
tls-replication yes
repl-diskless-load on-empty-db
```

#### sentinel.conf
```
port 0
protected-mode no

sentinel monitor mymaster redis-master 6379 2
sentinel down-after-milliseconds mymaster 10000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
sentinel resolve-hostnames yes
sentinel announce-port 26379

tls-port 26379
tls-cert-file /etc/redis/certs/redis.crt
tls-key-file /etc/redis/certs/redis.key
tls-ca-cert-file /etc/redis/certs/ca.crt
tls-cluster yes
tls-auth-clients yes
requirepass <password>
tls-protocols "TLSv1.2 TLSv1.3"
sentinel auth-pass mymaster <password>
tls-replication yes
```

### 2. Start the Cluster
Run the following command to start the Redis cluster:
```bash
docker-compose -f docker-compose.redis.yml up -d
```

### 3. Verify the Setup
- Check the logs of the containers:
  ```bash
  docker logs redis-master
  docker logs redis-slave1
  docker logs redis-slave2
  docker logs redis-sentinel1
  docker logs redis-sentinel2
  docker logs redis-sentinel3
  ```
- Verify Sentinel status:
  ```bash
  docker exec -it redis-sentinel1 redis-cli -p 26379 -a <password> --tls --cacert /etc/redis/certs/ca.crt sentinel masters
  ```

### 4. Clear Configuration Files
To clear and copy configuration files, run:
```bash
./clear.sh
```

## Network Configuration
- **Redis Master**: `172.25.0.10`
- **Redis Slave 1**: `172.25.0.11`
- **Redis Slave 2**: `172.25.0.12`
- **Sentinels**: Use ports `26379`, `26380`, and `26381`.

## Security
- All communication is encrypted using TLS.
- Password authentication is enabled with `requirepass` and `masterauth`.

## Notes
- Update the `docker-compose.redis.yml` file if you need to change IP addresses or ports.
- Ensure the certificates in `redis-tls/` are valid and match your setup.

## Cleanup
To stop and remove all containers, run:
```bash
docker-compose -f docker-compose.redis.yml down
```