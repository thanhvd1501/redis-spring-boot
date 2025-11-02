## 📚 **PHẦN 7: Rate Limiting & Distributed Lock – Giới hạn truy cập API và Khóa phân tán tránh Race Condition**

### 🎯 **Mục tiêu học phần**

Sau phần này, bạn sẽ:

1. Biết cách **giới hạn tần suất gọi API (Rate Limiting)** bằng Redis.
2. Hiểu cơ chế **Distributed Lock** – khóa phân tán giữa nhiều service để tránh race condition.
3. Thực hành với ví dụ **chống spam login, giới hạn request**, và **lock thao tác thanh toán/order**.
4. Nắm cách cài đặt **TTL, atomic counter, Lua script, Redisson Lock**.
5. Biết các lỗi phổ biến khi lock không được release và cách khắc phục.

---

## 🧠 1. Khái niệm tổng quan

### 🔸 Rate Limiting

Rate limiting dùng để **giới hạn số lần truy cập API trong một khoảng thời gian nhất định**
(VD: “chỉ cho phép 5 request mỗi 10 giây cho một IP hoặc user”).

→ Bảo vệ hệ thống khỏi spam, brute-force login, DDoS.

### 🔸 Distributed Lock

Khóa phân tán giúp **nhiều instance cùng chia sẻ tài nguyên an toàn** (VD: stock, balance, order).

→ Giúp tránh tình huống race condition khi nhiều server ghi cùng lúc.

---

## ⚙️ 2. Redis hỗ trợ Rate Limiting như thế nào?

Redis hỗ trợ **atomic counter + TTL**:

```
INCR key
EXPIRE key <seconds>
```

→ Khi `INCR` vượt quá giới hạn → chặn request.

Ví dụ:

```
key = "rate:login:ip:127.0.0.1"
count = INCR key
EXPIRE key 10
```

Nếu `count > 5` trong vòng 10 giây → reject request.

---

## 🧱 3. Triển khai Rate Limiter trong Spring Boot

### 📁 `RateLimiterService.java`

```java
package com.example.redis.ratelimit;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
@RequiredArgsConstructor
public class RateLimiterService {

    private final RedisTemplate<String, Object> redisTemplate;

    public boolean isAllowed(String key, int maxRequests, int windowSeconds) {
        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) {
            // Đặt TTL lần đầu
            redisTemplate.expire(key, Duration.ofSeconds(windowSeconds));
        }
        return count <= maxRequests;
    }
}
```

---

### 📡 Controller thử nghiệm

```java
package com.example.redis.controller;

import com.example.redis.ratelimit.RateLimiterService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import jakarta.servlet.http.HttpServletRequest;

@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class RateLimitController {

    private final RateLimiterService rateLimiter;

    @GetMapping("/login")
    public String loginAttempt(HttpServletRequest request) {
        String ip = request.getRemoteAddr();
        String key = "rate:login:" + ip;

        boolean allowed = rateLimiter.isAllowed(key, 5, 10); // max 5 lần / 10s

        if (!allowed) {
            return "🚫 Too many login attempts. Please try again later.";
        }
        return "✅ Login request accepted.";
    }
}
```

---

### 🧪 Test

```bash
curl http://localhost:8080/api/login
# Lặp lại 6 lần trong vòng 10 giây
```

Kết quả:

```
✅ Login request accepted.
✅ ...
🚫 Too many login attempts. Please try again later.
```

---

### 🧮 Check Redis key:

```bash
127.0.0.1:6379> keys rate:login*
1) "rate:login:127.0.0.1"
127.0.0.1:6379> ttl rate:login:127.0.0.1
(integer) 8
127.0.0.1:6379> get rate:login:127.0.0.1
"5"
```

---

## 🧩 4. Lua Script Rate Limiter (Atomic Operation nâng cao)

Redis Lua script đảm bảo **atomicity** – tránh lỗi khi nhiều instance cùng truy cập.

```lua
local current = redis.call("INCR", KEYS[1])
if tonumber(current) == 1 then
  redis.call("EXPIRE", KEYS[1], ARGV[1])
end
if tonumber(current) > tonumber(ARGV[2]) then
  return 0
else
  return 1
end
```

Java side:

```java
DefaultRedisScript<Long> script = new DefaultRedisScript<>();
script.setScriptText(luaScriptString);
script.setResultType(Long.class);

Long allowed = redisTemplate.execute(script, List.of(key),
        windowSeconds, maxRequests);
```

---

## 🔒 5. Distributed Lock (Khóa phân tán)

Redis lock hoạt động qua lệnh:

```
SET key value NX EX 10
```

* `NX`: chỉ set nếu key chưa tồn tại (lock mới).
* `EX 10`: TTL 10 giây (auto unlock nếu process crash).

---

### 📁 LockService.java

```java
package com.example.redis.lock;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import java.time.Duration;

@Service
@RequiredArgsConstructor
public class LockService {

    private final RedisTemplate<String, Object> redisTemplate;

    public boolean acquireLock(String key, String value, long ttlSeconds) {
        Boolean success = redisTemplate.opsForValue()
                .setIfAbsent(key, value, Duration.ofSeconds(ttlSeconds));
        return Boolean.TRUE.equals(success);
    }

    public void releaseLock(String key, String value) {
        Object currentValue = redisTemplate.opsForValue().get(key);
        if (value.equals(currentValue)) {
            redisTemplate.delete(key);
        }
    }
}
```

---

### 📦 Demo: Lock thanh toán tránh double-order

```java
@RestController
@RequestMapping("/order")
@RequiredArgsConstructor
public class OrderController {

    private final LockService lockService;

    @PostMapping("/create")
    public String createOrder(@RequestParam String orderId) throws InterruptedException {
        String lockKey = "lock:order:" + orderId;
        String lockValue = UUID.randomUUID().toString();

        if (!lockService.acquireLock(lockKey, lockValue, 5)) {
            return "🚫 Another process is creating this order.";
        }

        try {
            // Giả lập xử lý order
            Thread.sleep(3000);
            return "✅ Order created successfully!";
        } finally {
            lockService.releaseLock(lockKey, lockValue);
        }
    }
}
```

Test:

```bash
# Mở 2 terminal chạy cùng lệnh
curl -X POST "http://localhost:8080/order/create?orderId=123"
```

Kết quả:

```
✅ Order created successfully!
🚫 Another process is creating this order.
```

---

## 🧭 6. ASCII Flow – Distributed Lock

```
┌────────────┐
│  Service A │
└────┬───────┘
     │ SETNX lock:order:123
     ▼
 [Redis Server]
     │
     ▼
┌────────────┐
│  Service B │ → tries SETNX same key
└────────────┘ → ❌ fail (key exists)
```

---

## 🧰 7. Redisson – Cách dễ hơn để dùng Distributed Lock

Redisson là thư viện Java hỗ trợ Lock API tiện dụng.

### 🧩 Thêm dependency

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.30.0</version>
</dependency>
```

### ⚙️ Cấu hình

```java
@Configuration
public class RedissonConfig {
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        return Redisson.create(config);
    }
}
```

### 🔒 Dùng Redisson lock

```java
@Service
@RequiredArgsConstructor
public class RedissonLockService {

    private final RedissonClient redisson;

    public void safeOrder(String orderId) {
        RLock lock = redisson.getLock("lock:order:" + orderId);
        try {
            if (lock.tryLock(1, 5, TimeUnit.SECONDS)) {
                System.out.println("✅ Processing order " + orderId);
                Thread.sleep(3000);
            } else {
                System.out.println("🚫 Could not acquire lock for " + orderId);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 🧠 8. Bài tập nhỏ

**Bài 1:**
Tạo middleware Spring AOP intercept tất cả endpoint `/api/*` và áp rate limit IP → 100 requests / minute.

**Bài 2:**
Viết `StockService` cập nhật tồn kho sản phẩm → dùng Distributed Lock để tránh race khi nhiều user đặt cùng lúc.

**Bài 3:**
Thử Redisson trên nhiều instance Spring Boot (Docker Compose với 2 container).

---

## ⚠️ 9. Sai lầm phổ biến

| Sai lầm                        | Hậu quả                                   |
| ------------------------------ | ----------------------------------------- |
| Không TTL cho lock             | Lock tồn tại mãi → deadlock.              |
| Release lock không đúng thread | Xóa lock của process khác → mất đồng bộ.  |
| Không atomic khi release       | Race condition khi unlock.                |
| Dùng rate limit sai key        | Giới hạn toàn hệ thống thay vì từng user. |
| Không chống replay             | Request lặp lại vẫn được xử lý.           |

---

## 🌟 10. Best Practices

✅ Dùng key pattern rõ ràng: `lock:order:`, `rate:ip:`
✅ TTL cho lock = max processing time + buffer.
✅ Dùng Lua hoặc Redisson để đảm bảo atomic release.
✅ Log chi tiết thời gian và IP rate-limit.
✅ Redis Cluster cho hệ thống phân tán.
✅ Dùng Redis `EXPIRE` để auto cleanup keys.

---

## 🎓 Kết luận

Bạn đã học:

* Cách cài Rate Limiter trong Redis bằng atomic counter.
* Cách tạo Distributed Lock tránh ghi trùng.
* Thực hành với RedisTemplate và Redisson.
* Các lỗi phổ biến khi lock và cách khắc phục.
