## 📘 **PHẦN 1: Tổng quan Redis & Cách hoạt động**

### 🎯 **Mục tiêu học phần**

Sau khi hoàn thành phần này, bạn sẽ:

* Hiểu Redis là gì, vì sao nó nhanh đến vậy.
* Nắm được mô hình kiến trúc của Redis (memory-based, single-thread, persistence).
* Biết Redis lưu trữ dữ liệu dưới dạng nào (key-value, hash, list, set…).
* Phân biệt được Redis và database truyền thống.
* Chuẩn bị môi trường để cài đặt Redis (local hoặc Docker).

---

## 🧠 1. Redis là gì?

**Redis (REmote DIctionary Server)** là một **in-memory data store** — tức là dữ liệu được lưu **trực tiếp trong RAM**, giúp tốc độ truy cập **nhanh hơn hàng chục đến hàng trăm lần** so với database lưu trên disk như PostgreSQL hoặc MySQL.

Redis thường được gọi là **“data structure server”**, vì nó không chỉ lưu “key-value” thuần túy mà còn hỗ trợ **các cấu trúc dữ liệu nâng cao** như:

* String (chuỗi ký tự)
* Hash (giống Map<String, Object>)
* List (danh sách có thứ tự)
* Set (tập hợp không trùng lặp)
* Sorted Set (tập hợp có sắp xếp)
* Stream, Bitmap, HyperLogLog (các loại nâng cao)

---

## ⚙️ 2. Vì sao Redis nhanh?

Redis cực nhanh nhờ 4 yếu tố chính:

| Yếu tố                         | Giải thích                                                                |
| ------------------------------ | ------------------------------------------------------------------------- |
| 💾 **In-Memory**               | Dữ liệu được lưu trong RAM, không phải disk → tốc độ microseconds.        |
| 🧵 **Single-threaded**         | Redis chạy trên 1 luồng duy nhất → không cần lock, tránh race condition.  |
| 🧠 **Cấu trúc dữ liệu tối ưu** | Redis implement các data structure ở mức C code → hiệu suất cực cao.      |
| 📡 **Protocol đơn giản**       | Giao tiếp client-server qua RESP (Redis Serialization Protocol), cực nhẹ. |

> 💡 **So sánh:**
>
> * PostgreSQL: lưu trữ bền vững, truy vấn phức tạp (SQL).
> * Redis: tốc độ cực cao, lý tưởng cho cache, session, counter, queue.

---

## 🏗️ 3. Redis Architecture – Kiến trúc tổng quan

```
+-------------+         +-------------+
|  Client App | <-----> | Redis Server|
+-------------+         +-------------+
       ↑                       |
       | (TCP 6379)            |
       |                       v
       |             +----------------------+
       |             | In-Memory Data Store |
       |             |   Key → Value        |
       |             +----------------------+
       |                       |
       |                       v
       |              +--------------------+
       |              | Persistence (RDB / AOF) |
       |              +--------------------+
```

### 🔸 **Memory-based store**

Redis lưu dữ liệu chính trong RAM để đạt tốc độ cực nhanh.

### 🔸 **Persistence (RDB / AOF)**

Redis có thể **ghi lại dữ liệu ra đĩa** để tránh mất mát khi tắt máy:

| Cơ chế                        | Cách hoạt động                               | Ưu điểm           | Nhược điểm                  |
| ----------------------------- | -------------------------------------------- | ----------------- | --------------------------- |
| **RDB (Redis Database File)** | Snapshot toàn bộ DB định kỳ (VD: mỗi 5 phút) | Nhanh, nhẹ        | Mất dữ liệu giữa 2 snapshot |
| **AOF (Append Only File)**    | Ghi log từng lệnh write                      | Không mất dữ liệu | File lớn, ghi chậm hơn      |

> ⚙️ Thực tế: Hầu hết dự án doanh nghiệp bật **cả hai (RDB + AOF)** để cân bằng tốc độ và an toàn.

---

## 🧩 4. Redis Data Model (Key-Value nâng cao)

| Kiểu dữ liệu | Ví dụ                               | Ứng dụng thực tế        |
| ------------ | ----------------------------------- | ----------------------- |
| String       | “name”: “James”                     | Cache đơn giản, token   |
| Hash         | “user:1” → {name: “James”, age: 30} | Thông tin người dùng    |
| List         | [msg1, msg2, msg3]                  | Message queue           |
| Set          | {“admin”, “editor”}                 | Danh sách quyền         |
| Sorted Set   | {“userA”: 10, “userB”: 20}          | Bảng xếp hạng (ranking) |
| Stream       | Dòng sự kiện (event stream)         | Log, chat stream        |

---

## 🧰 5. Cài đặt Redis (Local hoặc Docker)

### 🧩 Cách 1: Cài Redis trên Docker (Khuyến nghị)

```bash
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7.2 \
  redis-server --appendonly yes
```

Kiểm tra:

```bash
docker exec -it redis redis-cli
127.0.0.1:6379> set name "James"
OK
127.0.0.1:6379> get name
"James"
```

### 🧩 Cách 2: Cài trực tiếp

* Linux: `sudo apt install redis-server`
* Windows: dùng [Memurai](https://www.memurai.com/) hoặc WSL

---

## 🔍 6. Khi nào dùng Redis?

| Use Case          | Mô tả                             | Lợi ích                |
| ----------------- | --------------------------------- | ---------------------- |
| **Caching**       | Lưu dữ liệu truy vấn tạm thời     | Giảm tải DB, tăng tốc  |
| **Session Store** | Lưu session user (Spring Session) | Tốc độ và phân tán     |
| **Pub/Sub**       | Gửi – nhận thông điệp real-time   | Chat app, Notification |
| **Queue**         | Lưu hàng đợi công việc            | Task, background job   |
| **Rate Limiting** | Giới hạn số lần truy cập API      | Bảo vệ hệ thống        |
| **Counter**       | Đếm lượt truy cập, like, view     | Atomic & nhanh         |

---

## 🧭 7. Demo Flow: Redis trong kiến trúc web

```
[Client] ---> [Spring Boot API]
      ↓
    (check cache)
      ↓
  [Redis Cache Layer]
      ↓
  [PostgreSQL Database]
```

Luồng hoạt động “Cache-Aside” phổ biến:

1. API nhận request → kiểm tra cache Redis.
2. Nếu có → trả luôn (cache hit).
3. Nếu không → truy DB → lưu vào Redis (cache miss).

---

## 🧩 8. Bài tập thực hành nhỏ

### 🔹 Bài tập 1:

* Cài Redis bằng Docker.
* Kết nối `redis-cli` và thử các lệnh:

  ```bash
  set course "Spring Boot with Redis"
  get course
  del course
  ```

### 🔹 Bài tập 2:

* Tạo file `docker-compose.yml` để khởi động Redis cùng app Spring Boot (sẽ hướng dẫn ở phần 2).

---

## ⚠️ 9. Sai lầm phổ biến

| Sai lầm                               | Hậu quả                                                                              |
| ------------------------------------- | ------------------------------------------------------------------------------------ |
| Nghĩ Redis là database chính          | Redis **không nên** lưu dữ liệu lâu dài quan trọng (vì RAM đắt, dễ mất khi restart). |
| Không bật persistence                 | Mất dữ liệu khi tắt container/máy.                                                   |
| Dùng Redis như cache mà không đặt TTL | Dễ tràn RAM, gây OOM.                                                                |
| Không giám sát memory usage           | Redis có eviction policy, nếu không cấu hình → crash khi đầy bộ nhớ.                 |

---

## 🌟 10. Best Practices

✅ Sử dụng **Docker Compose** để quản lý Redis.
✅ Cấu hình **RDB + AOF** cho an toàn dữ liệu.
✅ Mọi cache nên có **TTL** hợp lý.
✅ Luôn monitor Redis bằng **RedisInsight** hoặc `INFO` command.
✅ Dùng prefix rõ ràng cho key (VD: `user:`, `cache:product:`).

---

## 🎓 Kết luận

Phần này giúp bạn hiểu nền tảng của Redis: từ kiến trúc, cấu trúc dữ liệu, cách cài đặt và ứng dụng.
Ở phần tiếp theo, ta sẽ **kết nối Redis với Spring Boot**, sử dụng **RedisTemplate, StringRedisTemplate**, và thiết lập **cấu hình caching thực tế**.
