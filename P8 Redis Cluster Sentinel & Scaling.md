## 📓 **PHẦN 8: Redis Cluster, Sentinel & Scaling – Mở rộng Redis cho High Availability và Load Balancing**

### 🎯 **Mục tiêu học phần**

Sau phần này, bạn sẽ:

1. Hiểu sự khác biệt giữa **Redis Standalone**, **Master–Replica**, **Sentinel**, và **Cluster**.
2. Biết cách thiết lập Redis **High Availability (HA)** và **Horizontal Scaling**.
3. Nắm cơ chế **failover tự động**, **partition dữ liệu**, và **connection pooling** trong Spring Boot.
4. Cấu hình demo Redis Sentinel + Cluster bằng Docker Compose.
5. Nắm được best practices khi triển khai Redis ở môi trường production thực tế.

---

## 🧠 1. Redis Deployment Modes

Redis có **4 chế độ vận hành chính**:

| Mô hình            | Đặc điểm                          | Ứng dụng thực tế          |
| ------------------ | --------------------------------- | ------------------------- |
| **Standalone**     | 1 Redis server duy nhất           | Dev/test local            |
| **Master–Replica** | 1 master, nhiều replica (chỉ đọc) | Tăng khả năng đọc, backup |
| **Sentinel**       | Hệ thống giám sát + failover      | High Availability         |
| **Cluster**        | Shard dữ liệu + tự động phân tán  | Scale lớn, hàng triệu key |

---

## 🧩 2. Mô hình 1 – Standalone

Đơn giản nhất: một instance Redis.

```
[Client] → [Redis Server]
```

**Ưu điểm:** đơn giản, dễ cài.
**Nhược điểm:** nếu server down → mất dữ liệu và downtime.

---

## 🧩 3. Mô hình 2 – Master–Replica

Redis hỗ trợ **replication** (nhân bản dữ liệu đọc-only sang node khác).

```
        ┌────────────┐
        │  Master    │  ← ghi
        └─────┬──────┘
              │
     ┌────────┴────────┐
     │                 │
┌────────────┐   ┌────────────┐
│ Replica 1  │   │ Replica 2  │ ← chỉ đọc
└────────────┘   └────────────┘
```

Cấu hình:

```bash
# replica.conf
replicaof 192.168.1.10 6379
```

Spring Boot tự động đọc từ master khi ghi và replica khi chỉ đọc nếu bạn cấu hình `LettuceReadFrom.REPLICA_PREFERRED`.

---

## 🧩 4. Mô hình 3 – Sentinel (High Availability)

Redis Sentinel là **cơ chế giám sát và tự động failover** khi master chết.

### 🧭 Cấu trúc

```
┌──────────┐
│ Sentinel │
└────┬─────┘
     │ Giám sát
     ▼
┌──────────┐      replication
│  Master  │ <-------------------┐
└────┬─────┘                     │
     │ failover                  │
     ▼                           │
┌──────────┐   ┌──────────┐
│ Replica1 │   │ Replica2 │
└──────────┘   └──────────┘
```

Nếu master down → Sentinel chọn replica mới làm master.

---

### 🧩 Docker Compose ví dụ (3 Sentinel + 1 Master + 1 Replica)

```yaml
version: "3"
services:
  redis-master:
    image: redis:7.2
    command: redis-server --appendonly yes
    ports: ["6379:6379"]

  redis-replica:
    image: redis:7.2
    command: redis-server --replicaof redis-master 6379
    ports: ["6380:6379"]

  sentinel1:
    image: redis:7.2
    command: >
      redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel.conf:/etc/redis/sentinel.conf
    ports: ["26379:26379"]
```

**sentinel.conf**

```bash
sentinel monitor mymaster redis-master 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
```

---

### ⚙️ Spring Boot kết nối Sentinel

```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes: localhost:26379
      password:
      timeout: 3000
```

Spring Boot sẽ tự động failover sang master mới khi Sentinel báo.

---

## 🧩 5. Mô hình 4 – Redis Cluster (Sharding & Scaling)

Redis Cluster cho phép chia dữ liệu thành **16384 hash slots**.

→ Mỗi node giữ một phần dữ liệu.

```
[Node1] slots 0–5460
[Node2] slots 5461–10922
[Node3] slots 10923–16383
```

### Lợi ích:

* Scale theo chiều ngang (horizontal).
* Không cần manual partition key.
* Có replication cho từng shard.

---

### 🧱 Kiến trúc Redis Cluster

```
      ┌────────────┐
      │   Node A   │ master + replica A'
      ├────────────┤
      │ slots 0–5460 │
      └────┬────────┘
           │
      ┌────▼────┐
      │ Node B  │ master + replica B'
      ├─────────┤
      │ slots 5461–10922 │
      └────┬────┘
           │
      ┌────▼────┐
      │ Node C  │ master + replica C'
      ├─────────┤
      │ slots 10923–16383 │
      └─────────┘
```

---

### ⚙️ Cấu hình Spring Boot kết nối Redis Cluster

```yaml
spring:
  data:
    redis:
      cluster:
        nodes:
          - 127.0.0.1:7000
          - 127.0.0.1:7001
          - 127.0.0.1:7002
      timeout: 3000
```

> Spring Boot sử dụng **LettuceClusterClient** để tự động route key đến đúng slot.

---

## 🧮 6. Connection Pool Tuning (Lettuce & Jedis)

| Tham số                                | Giải thích          | Khuyến nghị |
| -------------------------------------- | ------------------- | ----------- |
| `spring.redis.lettuce.pool.max-active` | Số kết nối tối đa   | 16–64       |
| `spring.redis.lettuce.pool.max-idle`   | Kết nối idle tối đa | 8–32        |
| `spring.redis.lettuce.pool.min-idle`   | Kết nối tối thiểu   | 2–8         |
| `spring.redis.timeout`                 | Timeout đọc/ghi     | 3–5 giây    |

---

## 🔍 7. Kiểm tra cluster info

```bash
redis-cli -c -p 7000 cluster info
redis-cli -c -p 7000 cluster nodes
```

Kết quả:

```
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_known_nodes:6
```

---

## 🧠 8. Khi nào nên dùng mô hình nào?

| Trường hợp                  | Khuyến nghị        |
| --------------------------- | ------------------ |
| Dev, local học tập          | Standalone         |
| Production nhỏ, yêu cầu HA  | Sentinel           |
| Production lớn, traffic cao | Cluster            |
| Chỉ cần đọc dự phòng        | Master–Replica     |
| Muốn vừa scale vừa HA       | Cluster + Sentinel |

---

## ⚡ 9. ASCII Flow – Scaling Redis Cluster

```
Client Request
    ↓
Hash(key) % 16384 → SlotID
    ↓
Find node responsible for SlotID
    ↓
Redis Cluster routes request
```

> Redis client (như Lettuce) tự định tuyến — không cần middleware.

---

## 🧩 10. Spring Boot Docker Compose full example

```yaml
version: "3.9"
services:
  redis-node-1:
    image: redis:7.2
    command: redis-server --port 7000 --cluster-enabled yes --cluster-config-file nodes.conf --appendonly yes
    ports: ["7000:7000"]

  redis-node-2:
    image: redis:7.2
    command: redis-server --port 7001 --cluster-enabled yes --cluster-config-file nodes.conf --appendonly yes
    ports: ["7001:7001"]

  redis-node-3:
    image: redis:7.2
    command: redis-server --port 7002 --cluster-enabled yes --cluster-config-file nodes.conf --appendonly yes
    ports: ["7002:7002"]
```

Tạo cluster:

```bash
docker exec -it redis-node-1 redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  --cluster-replicas 0
```

---

## 🧮 11. Bài tập nhỏ

**Bài 1:**
Thiết lập Redis Sentinel local (1 master, 1 replica, 1 sentinel).
Tắt master và xem replica lên thay thế.

**Bài 2:**
Chạy Redis Cluster 3 node.
Lưu nhiều key khác nhau → xem key phân tán trên các slot.

**Bài 3:**
Đo tốc độ query khi bật pool connection Lettuce (benchmark bằng `redis-benchmark`).

---

## ⚠️ 12. Sai lầm phổ biến

| Sai lầm                                  | Hậu quả                     |
| ---------------------------------------- | --------------------------- |
| Không hiểu khác biệt Sentinel vs Cluster | Cấu hình sai, mất HA.       |
| Dùng Jedis trong multi-thread            | Connection leak hoặc block. |
| Không bật AOF/RDB trong Cluster          | Mất dữ liệu khi node chết.  |
| TTL không đồng bộ giữa node              | Dữ liệu stale.              |
| Không kiểm tra replication lag           | Replica trả dữ liệu cũ.     |

---

## 🌟 13. Best Practices

✅ Redis Cluster khi dữ liệu > vài GB hoặc cần scale.
✅ Sentinel nếu chỉ cần HA nhỏ.
✅ Dùng **Lettuce** (thread-safe, async, reactive).
✅ Dùng **connection pool** và **timeout hợp lý**.
✅ Monitor bằng **RedisInsight** hoặc **Prometheus + Grafana**.
✅ Cấu hình persistence (RDB + AOF).
✅ Kiểm thử failover định kỳ.

---

## 🎓 Kết luận

Bạn đã học:

* Kiến trúc Redis trong môi trường phân tán (HA, Cluster, Sentinel).
* Cách setup Docker Compose cho từng mô hình.
* Cách Spring Boot tự động failover / routing key.
* Best practices để scale Redis production an toàn và hiệu quả.
