## 📖 **PHẦN 9: Debug, Monitoring & Optimization – Giám sát, Phân tích Hiệu năng & Tối ưu Redis trong Spring Boot**

### 🎯 **Mục tiêu học phần**

Sau phần này, bạn sẽ:

1. Biết cách **debug và kiểm tra lỗi Redis** trong Spring Boot.
2. Biết **monitor** Redis bằng RedisInsight, `redis-cli`, và metrics trong Spring Actuator.
3. Hiểu nguyên nhân gây **cache miss, memory leak, slow command**, và cách khắc phục.
4. Biết tối ưu **TTL, eviction policy, pipeline, batching, và connection pool**.
5. Làm chủ checklist tối ưu Redis cho production.

---

## 🧠 1. Debug Redis trong Spring Boot

### 🔹 Cách kiểm tra Redis có đang hoạt động

Trong `@PostConstruct`:

```java
@Autowired private StringRedisTemplate redisTemplate;

@PostConstruct
public void checkRedisConnection() {
    String pong = redisTemplate.getConnectionFactory().getConnection().ping();
    System.out.println("✅ Redis connection test: " + pong);
}
```

> Nếu bạn thấy “✅ Redis connection test: PONG” → Redis OK.

---

### 🔹 Bật log cho RedisTemplate

```yaml
logging:
  level:
    org.springframework.data.redis: DEBUG
```

Spring log sẽ hiển thị các thao tác `SET`, `GET`, `DEL`, `EXPIRE` giúp bạn thấy Redis flow trong thời gian thực.

---

### 🔹 Kiểm tra lỗi phổ biến

| Lỗi                                    | Nguyên nhân                       | Cách xử lý                                   |
| -------------------------------------- | --------------------------------- | -------------------------------------------- |
| `Connection refused`                   | Redis chưa chạy / port sai        | Kiểm tra `redis-cli ping`, `application.yml` |
| `Cannot deserialize`                   | Sai serializer                    | Dùng `GenericJackson2JsonRedisSerializer`    |
| `TimeoutException`                     | Redis overload hoặc network delay | Tăng `timeout` và pool                       |
| `OOM command not allowed`              | Hết RAM, không có eviction        | Giới hạn TTL hoặc cấu hình eviction policy   |
| `LOADING Redis is loading the dataset` | Redis đang restore snapshot       | Kiểm tra RDB load time                       |

---

## 🧰 2. Monitor bằng Redis CLI

### ⚙️ Kiểm tra trạng thái tổng quan

```bash
redis-cli INFO
```

Một số nhóm quan trọng:

* **Server**: version, uptime
* **Clients**: số kết nối hiện tại
* **Memory**: tổng RAM, peak usage
* **Persistence**: RDB/AOF status
* **Stats**: số request/s
* **Keyspace**: số key trong từng DB

Ví dụ:

```
# Memory
used_memory_human: 72.45M
used_memory_peak_human: 85.12M
evicted_keys: 0
expired_keys: 231
```

---

### 📊 Kiểm tra cache key

```bash
keys *
ttl user:1
get cache:product:123
```

> ⚠️ Không nên dùng `KEYS *` trong production (O(N)), thay bằng `SCAN`.

---

### 🚀 Real-time monitor

```bash
redis-cli MONITOR
```

Hiển thị tất cả lệnh đang chạy:

```
1627909320.892342 [0 127.0.0.1:58392] "SET" "user:1" "{\"id\":\"1\"}"
```

---

## 📈 3. RedisInsight – GUI Monitoring

[RedisInsight](https://redis.io/insight/) là công cụ chính thức của Redis (Windows, macOS, Linux, Docker).

### 🔹 Chức năng:

* Xem keys, TTL, JSON format trực quan
* Phân tích top keys, memory usage
* Query analyzer (slow command detector)
* Cluster & replication visualization
* Memory profiler (key size & fragmentation)

### ⚙️ Cài bằng Docker:

```bash
docker run -d --name redisinsight -p 8001:8001 redis/redisinsight:latest
```

Truy cập: [http://localhost:8001](http://localhost:8001)

---

## ⚙️ 4. Monitor bằng Spring Boot Actuator

Thêm dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Cấu hình:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics
```

Spring Boot sẽ tự expose:

```
/actuator/health
/actuator/metrics/redis.connections.active
/actuator/metrics/redis.commands
```

---

## ⚡ 5. Tối ưu hiệu năng Redis

### 🔹 1. Dùng TTL hợp lý

Không TTL = key tồn tại mãi → chiếm RAM.

Ví dụ:

```java
redisTemplate.opsForValue().set("cache:user:1", user, Duration.ofMinutes(15));
```

---

### 🔹 2. Cấu hình Eviction Policy (Khi hết RAM)

Trong `redis.conf`:

```
maxmemory 512mb
maxmemory-policy allkeys-lru
```

| Chính sách     | Mô tả                  |
| -------------- | ---------------------- |
| noeviction     | Không xóa gì → lỗi OOM |
| allkeys-lru    | Xóa key ít dùng nhất   |
| volatile-ttl   | Xóa key sắp hết hạn    |
| allkeys-random | Xóa ngẫu nhiên         |

> ✅ **Khuyến nghị:** `allkeys-lru` trong production.

---

### 🔹 3. Sử dụng Pipeline (Batch command)

Pipeline giúp giảm round-trip time khi gửi nhiều lệnh cùng lúc.

```java
List<Object> results = redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
    for (int i = 0; i < 100; i++) {
        byte[] key = ("user:" + i).getBytes();
        connection.stringCommands().set(key, ("name:" + i).getBytes());
    }
    return null;
});
```

> Giảm latency khi ghi nhiều key (VD: preload cache).

---

### 🔹 4. Dùng Connection Pool (Lettuce)

```yaml
spring:
  data:
    redis:
      lettuce:
        pool:
          max-active: 32
          max-idle: 16
          min-idle: 4
          time-between-eviction-runs: 1s
```

> Lettuce thread-safe, non-blocking → an toàn hơn Jedis trong môi trường đa luồng.

---

### 🔹 5. Nén dữ liệu JSON

Khi lưu object lớn → nén giúp tiết kiệm RAM.

```java
byte[] compressed = Snappy.compress(json.getBytes());
redisTemplate.opsForValue().set("user:compressed", compressed);
```

Hoặc dùng `GenericJackson2JsonRedisSerializer` với custom `ObjectMapper` (loại bỏ metadata).

---

## 🧩 6. Detect Slow Commands

### 🔹 Xem slowlog

```bash
redis-cli slowlog get 10
```

Kết quả:

```
1) 1) (integer) 3
   2) (integer) 1698655420
   3) (integer) 15000
   4) 1) "keys" "*" 
```

`15000` microseconds = 15ms → xem xét tối ưu query.

---

## 🧮 7. Bài tập nhỏ

**Bài 1:**
Dùng RedisInsight xem key có TTL > 1 giờ → xác định key “quên” TTL.

**Bài 2:**
Benchmark Redis local:

```bash
redis-benchmark -t set,get -n 100000 -q
```

**Bài 3:**
Bật `MONITOR` → gọi API nhiều lần → xem Redis hoạt động thế nào.

---

## ⚠️ 8. Sai lầm phổ biến

| Sai lầm                          | Hậu quả                      |
| -------------------------------- | ---------------------------- |
| Không TTL hoặc eviction          | Redis đầy RAM → crash.       |
| Dùng `KEYS *` trong production   | Block toàn bộ Redis.         |
| Không monitor memory             | Không phát hiện leak sớm.    |
| Dùng serializer sai → cache miss | Object deserialize sai kiểu. |
| Dùng Jedis trong async           | Connection pool deadlock.    |

---

## 🌟 9. Best Practices Checklist

✅ TTL bắt buộc cho mọi cache key.
✅ Sử dụng eviction policy `allkeys-lru`.
✅ Giám sát memory, connection, latency.
✅ Bật persistence (AOF + RDB).
✅ Tránh `KEYS *`, `FLUSHALL` trong production.
✅ Dùng pipeline khi ghi nhiều key.
✅ RedisInsight + Actuator metrics để theo dõi.
✅ Test failover (Sentinel/Cluster) định kỳ.

---

## 🎓 Kết luận

Bạn đã nắm:

* Toàn bộ kỹ năng **debug, monitor và tối ưu Redis** trong Spring Boot.
* Cách theo dõi hiệu năng bằng Redis CLI, RedisInsight, và Actuator.
* Các kỹ thuật tăng tốc (TTL, LRU, pipeline, pool).
* Cách phát hiện & xử lý lỗi phổ biến.
