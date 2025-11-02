## 📗 **PHẦN 2: Kết nối Redis trong Spring Boot**

### 🎯 **Mục tiêu học phần**

Sau khi học xong phần này, bạn sẽ biết:

1. Cách **kết nối Redis** với Spring Boot (cấu hình, dependency, test connection).
2. Phân biệt **RedisTemplate vs StringRedisTemplate**.
3. Cách **cấu hình serializer** (JSON, JDK, String).
4. Thực hành CRUD dữ liệu Redis bằng Java.
5. Biết debug và monitor Redis trong môi trường thật.

---

## 🧩 1. Chuẩn bị dự án Spring Boot

### ⚙️ **Thêm dependency trong `pom.xml`**

```xml
<dependencies>
    <!-- Spring Boot Starter Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring Data Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- JSON Serializer (nếu cần dùng Jackson) -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>

    <!-- Lombok (giúp code ngắn gọn) -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

> 💡 **Spring Boot 3.x** mặc định sử dụng **Lettuce** làm Redis client (thread-safe, non-blocking).
> Bạn **không cần thêm Jedis** trừ khi có lý do đặc biệt (Lettuce an toàn hơn trong môi trường multi-thread).

---

## 🧱 2. Cấu hình Redis trong `application.yml`

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ""
      timeout: 2000ms
      database: 0
```

> Nếu bạn chạy Redis bằng Docker Compose, có thể đặt `host: redis` (tên service).

---

## 🧰 3. Tạo cấu hình RedisConfig

```java
package com.example.redis.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        // Kết nối đến Redis (host, port lấy từ application.yml)
        return new LettuceConnectionFactory();
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        // 🔸 Key serializer = String
        template.setKeySerializer(new StringRedisSerializer());

        // 🔸 Value serializer = JSON (Jackson)
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer(new ObjectMapper()));

        // 🔸 Hash key/value serializer
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());

        template.afterPropertiesSet();
        return template;
    }
}
```

### 🔍 Giải thích:

* `LettuceConnectionFactory`: tạo kết nối Redis thread-safe.
* `RedisTemplate`: là **API chính** thao tác dữ liệu Redis (CRUD, hash, list, v.v.).
* `StringRedisSerializer`: chuyển key sang chuỗi dễ đọc.
* `GenericJackson2JsonRedisSerializer`: serialize object thành JSON để Redis lưu.

---

## 🧠 4. RedisTemplate vs StringRedisTemplate

| Tiêu chí            | RedisTemplate                      | StringRedisTemplate     |
| ------------------- | ---------------------------------- | ----------------------- |
| Kiểu dữ liệu        | `<String, Object>`                 | `<String, String>`      |
| Serializer mặc định | JDK serialization (không đọc được) | String                  |
| Dùng khi            | Muốn lưu Object (User, Product...) | Lưu text, counter, flag |
| Ưu điểm             | Linh hoạt, lưu JSON                | Đơn giản, dễ debug      |
| Nhược điểm          | Cần serializer                     | Chỉ dùng được String    |

> ✅ **Khuyến nghị:** Dùng `RedisTemplate<String, Object>` với JSON serializer để dễ debug và lưu object phức tạp.

---

## 🧪 5. Ví dụ CRUD thực tế với RedisTemplate

### 📁 Entity

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

### 📂 Service

```java
package com.example.redis.service;

import com.example.redis.model.User;
import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserRedisService {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String PREFIX = "user:";

    public void save(User user) {
        redisTemplate.opsForValue().set(PREFIX + user.getId(), user);
    }

    public User findById(String id) {
        return (User) redisTemplate.opsForValue().get(PREFIX + id);
    }

    public void delete(String id) {
        redisTemplate.delete(PREFIX + id);
    }
}
```

### 📡 Controller

```java
package com.example.redis.controller;

import com.example.redis.model.User;
import com.example.redis.service.UserRedisService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/users")
@RequiredArgsConstructor
public class UserController {

    private final UserRedisService userRedisService;

    @PostMapping
    public String save(@RequestBody User user) {
        userRedisService.save(user);
        return "User saved to Redis!";
    }

    @GetMapping("/{id}")
    public User get(@PathVariable String id) {
        return userRedisService.findById(id);
    }

    @DeleteMapping("/{id}")
    public String delete(@PathVariable String id) {
        userRedisService.delete(id);
        return "User deleted from Redis!";
    }
}
```

---

## 🧪 6. Kiểm thử bằng Postman hoặc curl

### ➕ Thêm user

```bash
curl -X POST http://localhost:8080/users \
-H "Content-Type: application/json" \
-d '{"id":"1","name":"James","age":30}'
```

### 🔍 Lấy user

```bash
curl http://localhost:8080/users/1
```

### ❌ Xoá user

```bash
curl -X DELETE http://localhost:8080/users/1
```

---

## 🔎 7. Kiểm tra trong Redis CLI

```bash
127.0.0.1:6379> keys *
"user:1"

127.0.0.1:6379> get user:1
"{\"@class\":\"com.example.redis.model.User\",\"id\":\"1\",\"name\":\"James\",\"age\":30}"
```

---

## 📊 8. Flow hoạt động (ASCII Diagram)

```
[Client] 
   ↓ POST /users
[Controller]
   ↓
[UserRedisService.save()]
   ↓
[RedisTemplate.opsForValue().set("user:1", user)]
   ↓
[Redis Server]
   ↓
(JSON lưu trong bộ nhớ)
```

---

## 🧠 9. Bài tập nhỏ

**Bài tập 1:**

* Thêm endpoint `/users/all` để lấy toàn bộ user đang lưu trong Redis.

**Bài tập 2:**

* Thêm TTL cho user (sẽ học kỹ ở phần Caching). Gợi ý:

  ```java
  redisTemplate.opsForValue().set(key, value, Duration.ofMinutes(10));
  ```

---

## ⚠️ 10. Sai lầm phổ biến

| Sai lầm                                         | Hậu quả                                                   |
| ----------------------------------------------- | --------------------------------------------------------- |
| Không cấu hình serializer                       | Redis lưu nhị phân khó đọc → khó debug.                   |
| Dùng cùng 1 RedisTemplate cho nhiều kiểu object | Deserialize sai class.                                    |
| Không đặt prefix key                            | Dễ trùng key giữa các module (VD: “user:1” vs “order:1”). |
| Dùng Jedis trong multi-thread                   | Gây lỗi connection leak. Lettuce an toàn hơn.             |

---

## 🌟 11. Best Practices

✅ Dùng `RedisTemplate<String, Object>` + JSON serializer.
✅ Prefix key theo module: `user:`, `order:`, `session:`.
✅ Mỗi domain có thể tạo `RedisTemplate` riêng nếu cần.
✅ Kiểm tra Redis connection bằng `PING` trong `@PostConstruct`.
✅ Dùng **RedisInsight** để xem key/value trực quan.

---

## 🎓 Kết luận

Phần này bạn đã nắm toàn bộ:

* Cách cấu hình và kết nối Redis vào Spring Boot
* Dùng `RedisTemplate` thao tác CRUD
* Phân biệt `RedisTemplate` và `StringRedisTemplate`
* Thực hành REST API và kiểm tra trực tiếp trên Redis CLI
