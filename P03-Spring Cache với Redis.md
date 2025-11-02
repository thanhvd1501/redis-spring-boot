## 📙 **PHẦN 3: Spring Cache với Redis – @Cacheable, @CacheEvict, @CachePut, TTL & Cache-Aside Pattern**

### 🎯 **Mục tiêu học phần**

Sau phần này, bạn sẽ:

1. Hiểu cách Spring Boot tích hợp **Redis làm cache layer**.
2. Biết cách dùng **@Cacheable, @CacheEvict, @CachePut**.
3. Biết mô hình **Cache-Aside Pattern** (read-through / write-through).
4. Cấu hình **TTL**, **cache manager**, và **multi-cache**.
5. Tránh lỗi cache stale, cache miss, memory leak.

---

## 🧠 1. Tổng quan Spring Cache

Spring Boot cung cấp một abstraction (`org.springframework.cache`) để bạn **thêm cache mà không cần đổi logic code**.
Redis chính là 1 implementation của tầng cache đó.

```
[Client] → [Controller] → [Service (@Cacheable)] → [Redis Cache] → [Database]
```

---

### ⚙️ Annotation chính

| Annotation       | Mô tả                                                                                           |
| ---------------- | ----------------------------------------------------------------------------------------------- |
| `@Cacheable`     | Khi gọi method → nếu đã có cache → trả từ Redis. Nếu chưa → gọi method thật, rồi lưu vào cache. |
| `@CachePut`      | Luôn thực thi method và **cập nhật cache** với kết quả mới.                                     |
| `@CacheEvict`    | Xóa cache khi dữ liệu thay đổi (update/delete).                                                 |
| `@EnableCaching` | Kích hoạt Spring Cache system.                                                                  |

---

## 🧱 2. Cấu hình Redis Cache trong Spring Boot

### 📂 `application.yml`

```yaml
spring:
  cache:
    type: redis
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms
      database: 0
```

### 📄 `RedisCacheConfig.java`

```java
package com.example.redis.config;

import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.interceptor.SimpleCacheErrorHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import java.time.Duration;

@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {

        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)) // TTL mặc định 10 phút
                .disableCachingNullValues()
                .serializeValuesWith(
                        RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())
                );

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(config)
                .build();
    }

    @Bean
    public SimpleCacheErrorHandler cacheErrorHandler() {
        return new SimpleCacheErrorHandler();
    }
}
```

> ✅ `@EnableCaching` bật Spring caching system.
> ✅ `RedisCacheManager` là lớp chịu trách nhiệm lưu/truy xuất cache từ Redis.
> ✅ TTL đảm bảo cache tự hết hạn (tránh chiếm RAM mãi mãi).

---

## 💾 3. Ví dụ: Cache dữ liệu người dùng

### 📁 Model

```java
package com.example.redis.model;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private String id;
    private String name;
    private int age;
}
```

### 📂 Repository (mô phỏng DB thật)

```java
package com.example.redis.repository;

import com.example.redis.model.User;
import org.springframework.stereotype.Repository;

import java.util.HashMap;
import java.util.Map;

@Repository
public class UserRepository {
    private static final Map<String, User> DB = new HashMap<>();

    static {
        DB.put("1", new User("1", "James", 30));
        DB.put("2", new User("2", "Lily", 25));
    }

    public User findById(String id) {
        simulateSlowQuery();
        return DB.get(id);
    }

    public void save(User user) {
        DB.put(user.getId(), user);
    }

    public void delete(String id) {
        DB.remove(id);
    }

    private void simulateSlowQuery() {
        try {
            Thread.sleep(2000); // mô phỏng query DB chậm
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

---

### 📂 Service (nơi thêm annotation cache)

```java
package com.example.redis.service;

import com.example.redis.model.User;
import com.example.redis.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.cache.annotation.*;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository repository;

    @Cacheable(value = "users", key = "#id")
    public User getUser(String id) {
        System.out.println("🚀 Truy vấn DB thật với id: " + id);
        return repository.findById(id);
    }

    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        repository.save(user);
        return user;
    }

    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(String id) {
        repository.delete(id);
    }
}
```

---

### 📡 Controller

```java
package com.example.redis.controller;

import com.example.redis.model.User;
import com.example.redis.service.UserService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/cache-users")
@RequiredArgsConstructor
public class UserCacheController {

    private final UserService userService;

    @GetMapping("/{id}")
    public User getUser(@PathVariable String id) {
        return userService.getUser(id);
    }

    @PutMapping
    public User updateUser(@RequestBody User user) {
        return userService.updateUser(user);
    }

    @DeleteMapping("/{id}")
    public String deleteUser(@PathVariable String id) {
        userService.deleteUser(id);
        return "Deleted!";
    }
}
```

---

## 🧪 4. Demo hoạt động thực tế

### 🧭 Flow (Cache-Aside Pattern)

```
┌─────────────┐
│  Client API │
└──────┬──────┘
       │
       ▼
┌──────────────────────────────┐
│ Service (getUser)            │
│ @Cacheable("users", key=id)  │
└─────────────┬────────────────┘
              │
   Cache MISS │ Cache HIT
              │
              ▼
     [Redis Cache] <----> [Database]
```

---

### 🧪 Test bằng curl

**Lần 1 – Cache miss (truy DB thật):**

```bash
curl http://localhost:8080/cache-users/1
# Server log: 🚀 Truy vấn DB thật với id: 1
# Redis: lưu cache users::1
```

**Lần 2 – Cache hit (trả từ Redis, nhanh):**

```bash
curl http://localhost:8080/cache-users/1
# Không còn log DB
# Redis trả kết quả ngay lập tức
```

**Xóa cache:**

```bash
curl -X DELETE http://localhost:8080/cache-users/1
```

**Cập nhật cache:**

```bash
curl -X PUT http://localhost:8080/cache-users \
-H "Content-Type: application/json" \
-d '{"id":"1","name":"James Updated","age":31}'
```

---

## 🧮 5. TTL và Cache Manager nâng cao

Bạn có thể cấu hình TTL riêng cho từng cache:

```java
@Bean
public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
    Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();
    cacheConfigs.put("users",
            RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(5)));

    cacheConfigs.put("products",
            RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(1)));

    return RedisCacheManager.builder(connectionFactory)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
}
```

> 👉 Mỗi loại cache có thời gian sống khác nhau giúp tối ưu RAM và độ tươi dữ liệu.

---

## 🧠 6. Bài tập nhỏ

**Bài 1:**
Thêm cache cho `getAllUsers()` → lưu toàn bộ danh sách user 1 phút.

**Bài 2:**
Khi gọi `deleteUser(id)` → tự động xóa cả cache danh sách tổng (`allUsers`).

**Bài 3:**
Kiểm tra tốc độ lần đầu và lần sau bằng `System.currentTimeMillis()`.

---

## ⚠️ 7. Sai lầm phổ biến

| Sai lầm                            | Hậu quả                           |
| ---------------------------------- | --------------------------------- |
| Không thêm `@EnableCaching`        | Cache annotation không hoạt động. |
| Cache key trùng giữa các module    | Dữ liệu bị ghi đè sai.            |
| Không `@CacheEvict` sau khi update | Trả về dữ liệu cũ (cache stale).  |
| Không TTL                          | RAM bị chiếm vĩnh viễn.           |
| Cache null value                   | Gây lỗi khó debug.                |

---

## 🌟 8. Best Practices

✅ Dùng prefix rõ ràng (`users::1`, `products::123`).
✅ Luôn có TTL phù hợp.
✅ Dùng `@CacheEvict(allEntries=true)` khi cập nhật danh sách lớn.
✅ Đặt cache ở **service layer**, không phải controller.
✅ Test cache hit/miss bằng `redis-cli MONITOR`.
✅ Dùng **RedisInsight** để xem TTL, key, size.

---

## 🎓 Kết luận

Sau phần này bạn đã:

* Hiểu cơ chế Spring Cache + Redis CacheManager
* Biết dùng các annotation cache chính xác
* Hiểu Cache-Aside pattern (cực phổ biến trong microservice)
* Biết cấu hình TTL và tránh lỗi cache cũ
