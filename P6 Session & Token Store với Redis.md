## 📕 **PHẦN 6: Session & Token Store với Redis – Lưu phiên đăng nhập, xác thực JWT, và Single Sign-On (SSO)**

### 🎯 **Mục tiêu học phần**

Sau phần này, bạn sẽ:

1. Hiểu cách Spring Boot dùng Redis để **lưu session người dùng**.
2. Biết cách dùng Redis làm **token store** cho JWT hoặc OAuth2.
3. Xây dựng ví dụ **đăng nhập → lưu session → kiểm tra session → logout.**
4. Biết thiết lập **TTL, invalidate, auto-logout**.
5. Nắm best practices khi dùng Redis cho authentication, và tránh lỗi memory leak hoặc stale session.

---

## 🧠 1. Vì sao dùng Redis cho Session?

Trong ứng dụng web truyền thống, session được lưu:

* Trong **RAM của server (Tomcat session)** → chỉ dùng được trên 1 server.
* Khi scale lên nhiều server → session mất (vì load balancer chuyển request sang node khác).

=> Redis giải quyết bằng cách **lưu session tập trung (distributed session store)**.
Tất cả node đều truy cập cùng session qua Redis.

```
Client → Node1   ─┐
                  │   [Shared Redis Session Store]
Client → Node2   ─┘
```

---

## ⚙️ 2. Kích hoạt Redis HTTP Session

Thêm dependency:

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

Spring Session sẽ **tự động thay thế cơ chế session mặc định của servlet container** bằng Redis.

---

## 📄 3. Cấu hình `application.yml`

```yaml
spring:
  session:
    store-type: redis
    timeout: 30m   # TTL session 30 phút
  data:
    redis:
      host: localhost
      port: 6379
      database: 1
```

> Redis DB số 1 riêng cho session, tránh trộn với cache hoặc queue.

---

## ⚙️ 4. Tạo cấu hình `RedisHttpSessionConfig.java` *(tùy chọn)*

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class RedisHttpSessionConfig {
    // 1800s = 30 phút
}
```

Khi bật, Spring tự tạo `RedisOperationsSessionRepository`.

Mỗi session sẽ được lưu với key dạng:

```
spring:session:sessions:<session-id>
```

---

## 🧩 5. Demo: Đăng nhập – Lưu session vào Redis

### 📁 UserService

```java
@Service
public class AuthService {
    private final Map<String, String> USERS = Map.of(
            "admin", "123",
            "james", "abc"
    );

    public boolean login(String username, String password) {
        return USERS.containsKey(username) && USERS.get(username).equals(password);
    }
}
```

### 📂 Controller

```java
@RestController
@RequestMapping("/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    public String login(@RequestParam String username,
                        @RequestParam String password,
                        HttpSession session) {
        if (authService.login(username, password)) {
            session.setAttribute("user", username);
            return "✅ Logged in as " + username + " | sessionId=" + session.getId();
        }
        return "❌ Invalid credentials";
    }

    @GetMapping("/me")
    public String getProfile(HttpSession session) {
        String user = (String) session.getAttribute("user");
        return (user != null)
                ? "👤 Current user: " + user
                : "⚠️ No active session";
    }

    @PostMapping("/logout")
    public String logout(HttpSession session) {
        session.invalidate();
        return "👋 Logged out successfully";
    }
}
```

---

## 🧪 6. Kiểm thử

1️⃣ **Login:**

```bash
curl -X POST "http://localhost:8080/auth/login?username=james&password=abc"
# ✅ Logged in as james | sessionId=a4b07de5...
```

2️⃣ **Kiểm tra session hoạt động:**

```bash
curl -b "JSESSIONID=a4b07de5..." http://localhost:8080/auth/me
# 👤 Current user: james
```

3️⃣ **Xem trên Redis:**

```bash
127.0.0.1:6379> keys spring:session:sessions*
1) "spring:session:sessions:a4b07de5..."
127.0.0.1:6379> ttl spring:session:sessions:a4b07de5...
(integer) 1780
```

4️⃣ **Logout:**

```bash
curl -X POST -b "JSESSIONID=a4b07de5..." http://localhost:8080/auth/logout
# 👋 Logged out successfully
```

➡️ Redis key bị xoá ngay khi logout.

---

## 🧮 7. Token Store (JWT, API Tokens)

Redis cũng rất phù hợp để **lưu token ngắn hạn** (thay vì session ID).

Ví dụ: JWT login API

### 📁 TokenService

```java
@Service
@RequiredArgsConstructor
public class TokenService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void storeToken(String token, String username, long ttlSeconds) {
        redisTemplate.opsForValue().set("token:" + token, username, Duration.ofSeconds(ttlSeconds));
    }

    public boolean validate(String token) {
        return redisTemplate.hasKey("token:" + token);
    }

    public void revoke(String token) {
        redisTemplate.delete("token:" + token);
    }
}
```

### 📂 AuthController (phiên bản JWT)

```java
@PostMapping("/token-login")
public String tokenLogin(@RequestParam String username,
                         @RequestParam String password) {
    if (authService.login(username, password)) {
        String token = UUID.randomUUID().toString();
        tokenService.storeToken(token, username, 3600); // TTL 1h
        return "✅ Token issued: " + token;
    }
    return "❌ Invalid credentials";
}

@GetMapping("/token-validate")
public String validate(@RequestParam String token) {
    return tokenService.validate(token)
            ? "✅ Token valid"
            : "❌ Token expired or invalid";
}
```

---

## 🧭 8. ASCII Flow: Session vs Token

```
┌────────────┐         ┌──────────────┐
│   Browser  │         │   Mobile App │
└────┬───────┘         └────┬────────┘
     │                        │
     ▼                        ▼
 [Spring Boot API]         [Spring Boot API]
     │                        │
     ▼                        ▼
 [Redis Session Store]   [Redis Token Store]
     │                        │
  key=spring:session:...   key=token:<uuid>
```

---

## 🧠 9. TTL & Auto-Logout

Khi session hoặc token hết TTL → Redis tự xoá → user bị logout.

```java
spring:
  session:
    timeout: 15m
```

Hoặc token TTL:

```java
redisTemplate.opsForValue().set("token:...", username, Duration.ofMinutes(15));
```

---

## 🧮 10. Bài tập nhỏ

**Bài 1:**

* Dùng Redis để lưu token reset password (valid 10 phút).
* API `/reset/request`, `/reset/verify?token=`.

**Bài 2:**

* Dùng Redis hash `session:<id>` chứa `{username, loginTime, ip}`.

**Bài 3:**

* Thêm `@Scheduled` task kiểm tra session còn hoạt động (chạy mỗi 5 phút).

---

## ⚠️ 11. Sai lầm phổ biến

| Sai lầm                                | Hậu quả                                |
| -------------------------------------- | -------------------------------------- |
| Dùng cùng Redis DB cho cache + session | Dễ xoá nhầm dữ liệu hoặc lẫn TTL.      |
| Không đặt TTL cho token                | Token tồn tại mãi → rủi ro bảo mật.    |
| Không revoke token khi logout          | User vẫn truy cập API dù đã đăng xuất. |
| Dùng session cho stateless API         | Gây mất cân bằng giữa microservice.    |
| Không mã hoá dữ liệu nhạy cảm          | Dễ rò rỉ thông tin người dùng.         |

---

## 🌟 12. Best Practices

✅ Dùng Redis DB riêng cho session hoặc token.
✅ Prefix rõ ràng: `spring:session:`, `token:`, `reset:`.
✅ TTL ngắn: 15–60 phút.
✅ Xoá session/token khi logout hoặc timeout.
✅ Sử dụng HTTPS để tránh lộ token qua network.
✅ Redis Cluster hoặc Sentinel để đảm bảo session HA.

---

## 🎓 Kết luận

Bạn đã biết:

* Redis hoạt động như session store tập trung.
* Cách Spring Session lưu session vào Redis tự động.
* Cách lưu & xác thực JWT/token bằng Redis.
* TTL, auto-logout, và best practices bảo mật.
