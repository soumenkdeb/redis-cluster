# Redis Cluster with Podman

6-node Redis cluster (3 primary + 3 replica) managed with podman-compose, with RedisInsight UI.

## Requirements

- podman
- podman-compose

## Start

```bash
podman-compose up -d
```

### Initialize the cluster (run once, first time only)

`podman-compose up` starts 6 independent Redis instances — they don't know each other yet. This command joins them into a cluster, distributes the 16384 hash slots across the 3 primaries, and assigns the 3 replicas. Cluster state is persisted in `nodes.conf` inside each container's data volume, so this step is **not needed after restarts**.

```bash
podman exec -it redis-node-1 redis-cli --cluster create \
  172.20.0.11:6379 \
  172.20.0.12:6380 \
  172.20.0.13:6381 \
  172.20.0.14:6382 \
  172.20.0.15:6383 \
  172.20.0.16:6384 \
  --cluster-replicas 1 --yes
```

You'll see slot allocation output ending with `[OK] All 16384 slots covered.` — cluster is ready.

## RedisInsight

Access UI at `http://localhost:5540`.

Add the cluster:
1. Click **+ Add Redis Database**
2. Select **Cluster** as the connection type (not Standalone/Manual — using Standalone causes `MOVED` errors)
3. Host: `172.20.0.11`, Port: `6379`
4. Click **Add Redis Database**

RedisInsight auto-discovers all cluster nodes.

## Nodes

| Container    | IP           | Port | Bus Port |
|-------------|--------------|------|----------|
| redis-node-1 | 172.20.0.11 | 6379 | 16379    |
| redis-node-2 | 172.20.0.12 | 6380 | 16380    |
| redis-node-3 | 172.20.0.13 | 6381 | 16381    |
| redis-node-4 | 172.20.0.14 | 6382 | 16382    |
| redis-node-5 | 172.20.0.15 | 6383 | 16383    |
| redis-node-6 | 172.20.0.16 | 6384 | 16384    |

## Remote Deployment (spare laptop / remote host)

### 1. Open firewall on remote host

```bash
# firewalld
sudo firewall-cmd --add-port=5540/tcp --permanent
sudo firewall-cmd --reload

# ufw
sudo ufw allow 5540/tcp
```

### 2. Access RedisInsight remotely

```
http://<remote-host-ip>:5540
```

Add the cluster in UI same as above (uses internal container IPs — works because RedisInsight runs in the same podman network).

### 3. External Redis client access (optional)

By default `cluster-announce-ip` is set to internal container IPs (`172.20.0.x`). External clients get redirected to these unreachable addresses.

To allow external Redis client connections, update every `--cluster-announce-ip` in `podman-compose.yml` to the remote host's LAN IP, then open Redis and bus ports:

```bash
# firewalld
sudo firewall-cmd --add-port=6379-6384/tcp --permanent
sudo firewall-cmd --add-port=16379-16384/tcp --permanent
sudo firewall-cmd --reload

# ufw
sudo ufw allow 6379:6384/tcp
sudo ufw allow 16379:16384/tcp
```

## Stop

```bash
podman-compose down
```
