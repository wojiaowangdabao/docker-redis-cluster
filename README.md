# Redis Cluster with Docker Compose

## 仓库说明

本仓库的核心目的是帮助开发者在Windows本地环境下，使用基于WSL2的Docker快速创建一个本地Redis集群，用于本地开发和调试。

该配置方案解决了Windows Docker环境下Redis集群的网络通信问题，提供了完整的配置指南和使用说明，确保开发者能够快速搭建可用的Redis集群环境。

## Windows Docker 配置注意事项

为了确保Redis集群能够正常工作，需要在Windows上进行以下Docker和WSL配置：

### 1. Docker 设置
- 打开 Docker Desktop
- 进入 Settings
- 选择 Resources
- 勾选 "Enable host networking"

### 2. WSL 设置
- 打开 Windows Subsystem for Linux (WSL) 设置
- 开启 "主机地址环回" 选项
- 将网络模式更改为 "Mirrored"
- 启用 "localhost 转发"

## 使用说明

### 启动集群
```bash
docker-compose up -d
```

### 停止集群
```bash
docker-compose down
```

### 手动创建redis集群
```bash
docker exec -it redis-1 sh -c "REDISCLI_AUTH=<REDIS_PASSWORD> redis-cli --cluster create <宿主机IP>:7001 <宿主机IP>:7002 <宿主机IP>:7003 <宿主机IP>:7004 <宿主机IP>:7005 <宿主机IP>:7006 --cluster-replicas 1 --cluster-yes"
```
将 `<宿主机IP>` 替换为您的实际宿主机IP地址，例如 `192.168.0.154`
将 `<REDIS_PASSWORD>` 替换为您的实际Redis密码

### 查看集群状态
```bash
docker exec -it redis-1 redis-cli -a <REDIS_PASSWORD> cluster nodes
```

### 查看集群信息
```bash
docker exec -it redis-1 redis-cli -a <REDIS_PASSWORD> cluster info
```

## 集群配置

- 3个主节点 (redis1, redis2, redis3)
- 3个从节点 (redis4, redis5, redis6)
- 每个节点都有独立的端口映射
- 使用密码认证保护

## 横向扩容集群

本章节将指导您如何向现有的Redis集群添加新节点，实现横向扩容。以下步骤基于实际操作示例，使用的IP地址为 `192.168.0.170`，密码为 `123456`。

### 1. 添加新节点配置

在 `docker-compose.yml` 中添加新节点的配置（本示例添加了redis7和redis8节点）：

```yaml
redis7:
  container_name: redis-7
  ports:
    - "7007:6379"
    - "17007:16379"
  volumes:
    - ./redis-data/7007:/data
  healthcheck:
    test: ["CMD", "redis-cli", "-h", "redis-7", "-a", "${REDIS_PASSWORD}", "ping"]
    interval: 3s
    retries: 5
    start_period: 2s
    timeout: 3s
  depends_on:
    - redis1
  command: >
    redis-server
    --cluster-enabled yes
    --cluster-config-file nodes.conf
    --cluster-node-timeout 5000
    --cluster-announce-ip ${IPV4}
    --cluster-announce-port 7007
    --cluster-announce-bus-port 17007
    --cluster-replica-no-failover yes
    --appendonly yes
    --save 300 1000
    --save 60 10000
    --requirepass ${REDIS_PASSWORD}
    --masterauth ${REDIS_PASSWORD}
    --maxmemory 64m
    --maxmemory-policy allkeys-lru
    --protected-mode no
    --bind 0.0.0.0
  <<: *redis-common

redis8:
  container_name: redis-8
  ports:
    - "7008:6379"
    - "17008:16379"
  volumes:
    - ./redis-data/7008:/data
  healthcheck:
    test: ["CMD", "redis-cli", "-h", "redis-8", "-a", "${REDIS_PASSWORD}", "ping"]
    interval: 3s
    retries: 5
    start_period: 2s
    timeout: 3s
  depends_on:
    - redis1
  command: >
    redis-server
    --cluster-enabled yes
    --cluster-config-file nodes.conf
    --cluster-node-timeout 5000
    --cluster-announce-ip ${IPV4}
    --cluster-announce-port 7008
    --cluster-announce-bus-port 17008
    --cluster-replica-no-failover yes
    --appendonly yes
    --save 300 1000
    --save 60 10000
    --requirepass ${REDIS_PASSWORD}
    --masterauth ${REDIS_PASSWORD}
    --maxmemory 64m
    --maxmemory-policy allkeys-lru
    --protected-mode no
    --bind 0.0.0.0
  <<: *redis-common
```

### 2. 启动新节点

使用 `docker-compose up -d` 启动新添加的节点：

```bash
docker-compose up -d redis7 redis8
```

### 3. 查看当前集群状态

在添加新节点前，先查看当前集群的节点信息：

```bash
docker exec -it redis-1 redis-cli -a 123456 cluster nodes
```

示例输出：
```
27326be2703e7c6efca71b22acbb2656ed899b0f 192.168.0.170:7004@17004 slave,nofailover cf77ca0ba55e10b40eac3da1642bd4aa41246943 0 1774894354000 3 connected
cf77ca0ba55e10b40eac3da1642bd4aa41246943 192.168.0.170:7003@17003 master,nofailover - 0 1774894355223 3 connected 10923-16383
16d51f4a1b52307ea2dcc14a2f998b4cf41353ea 192.168.0.170:7005@17005 slave,nofailover 23b986c61be8d4372ac5fe4dfdb62b0905956a22 0 1774894353531 1 connected
23b986c61be8d4372ac5fe4dfdb62b0905956a22 192.168.0.170:7001@17001 myself,master,nofailover - 0 0 1 connected 0-5460
aa7aed70b6c492be824a459bed267a9fe9c4370e 192.168.0.170:7002@17002 master,nofailover - 0 1774894354000 2 connected 5461-10922
0a394b054d34fb4b35850907ee193cde14deec38 192.168.0.170:7006@17006 slave,nofailover aa7aed70b6c492be824a459bed267a9fe9c4370e 0 1774894354033 2 connected
```

### 4. 将新节点加入集群

#### 4.1 添加主节点 (redis7)

```bash
docker exec -it redis-1 redis-cli -a 123456 --cluster add-node 192.168.0.170:7007 192.168.0.170:7001
```

#### 4.2 获取新主节点的ID

```bash
docker exec -it redis-1 redis-cli -a 123456 cluster nodes | grep redis-7
```

示例输出：
```
ec76e23ebe39a653f72d98832d6886fe73fb7aee 192.168.0.170:7007@17007 master,nofailover - 0 1774894393171 0 connected
```

这里的节点ID是：`ec76e23ebe39a653f72d98832d6886fe73fb7aee`

#### 4.3 添加从节点 (redis8) 并关联到主节点

```bash
docker exec -it redis-1 redis-cli -a 123456 --cluster add-node 192.168.0.170:7008 192.168.0.170:7001 --cluster-slave --cluster-master-id ec76e23ebe39a653f72d98832d6886fe73fb7aee
```

### 5. 重新分配槽位

将槽位从现有主节点重新分配给新的主节点，以实现数据分片的均衡。

```bash
docker exec -it redis-1 redis-cli -a 123456 --cluster reshard 192.168.0.170:7001 --cluster-from all --cluster-to ec76e23ebe39a653f72d98832d6886fe73fb7aee --cluster-slots 4096 --cluster-yes
```

参数说明：
- `--cluster-from all`: 从所有现有主节点分配槽位
- `--cluster-to <节点ID>`: 槽位分配的目标节点ID
- `--cluster-slots <数量>`: 分配的槽位数量
- `--cluster-yes`: 自动确认所有操作

### 6. 验证扩容结果

#### 6.1 查看集群节点状态

```bash
docker exec -it redis-1 redis-cli -a 123456 cluster nodes
```

示例输出（扩容后）：
```
27326be2703e7c6efca71b22acbb2656ed899b0f 192.168.0.170:7004@17004 slave,nofailover cf77ca0ba55e10b40eac3da1642bd4aa41246943 0 1774894747000 3 connected
ab861dfc039b6dc07c27252f93bf8e580cd9d9f8 192.168.0.170:7008@17008 slave,nofailover ec76e23ebe39a653f72d98832d6886fe73fb7aee 0 1774894747000 8 connected
ec76e23ebe39a653f72d98832d6886fe73fb7aee 192.168.0.170:7007@17007 master,nofailover - 0 1774894747661 8 connected 0-1364 5461-6826 10923-12287
cf77ca0ba55e10b40eac3da1642bd4aa41246943 192.168.0.170:7003@17003 master,nofailover - 0 1774894748268 3 connected 12288-16383
16d51f4a1b52307ea2dcc14a2f998b4cf41353ea 192.168.0.170:7005@17005 slave,nofailover 23b986c61be8d4372ac5fe4dfdb62b0905956a22 0 1774894747259 1 connected
23b986c61be8d4372ac5fe4dfdb62b0905956a22 192.168.0.170:7001@17001 myself,master,nofailover - 0 0 1 connected 1365-5460
aa7aed70b6c492be824a459bed267a9fe9c4370e 192.168.0.170:7002@17002 master,nofailover - 0 1774894746621 2 connected 6827-10922
0a394b054d34fb4b35850907ee193cde14deec38 192.168.0.170:7006@17006 slave,nofailover aa7aed70b6c492be824a459bed267a9fe9c4370e 0 1774894748064 2 connected
```

#### 6.2 查看集群信息

```bash
docker exec -it redis-1 redis-cli -a 123456 cluster info
```

示例输出：
```
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:8
cluster_size:4
cluster_current_epoch:7
cluster_my_epoch:1
cluster_stats_messages_ping_sent:4138
cluster_stats_messages_pong_sent:4136
cluster_stats_messages_sent:8274
cluster_stats_messages_ping_received:4129
cluster_stats_messages_pong_received:4138
cluster_stats_messages_meet_received:7
cluster_stats_messages_received:8274
total_cluster_links_buffer_limit_exceeded:0
```

### 7. 扩容后的集群结构

扩容后，集群包含：
- **4个主节点**：redis1 (7001)、redis2 (7002)、redis3 (7003)、redis7 (7007)
- **4个从节点**：redis4 (7004)、redis5 (7005)、redis6 (7006)、redis8 (7008)
- 每个主节点都有一个从节点提供故障转移保护
- 所有16384个槽位已均匀分配到4个主节点

## 注意事项

1. **槽位分配**：根据实际需求调整分配的槽位数量，通常应尽量均匀分配
2. **节点命名**：保持节点命名的一致性，便于管理
3. **数据迁移**：槽位重新分配过程会自动迁移数据，无需手动干预
4. **集群状态**：在执行扩容操作前，确保集群状态为`ok`
5. **网络配置**：确保新节点能够与现有集群节点正常通信
6. **资源限制**：根据服务器资源情况调整新节点的内存限制等参数