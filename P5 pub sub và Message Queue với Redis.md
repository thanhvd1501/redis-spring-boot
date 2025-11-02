## 📔 **PHẦN 5: Pub/Sub và Message Queue với Redis – Realtime Communication, Notification, Chat và Background Job**

### 🎯 **Mục tiêu học phần**

Sau phần này, bạn sẽ:

1. Hiểu cơ chế **Publish/Subscribe** trong Redis — hoạt động như hệ thống **Event Bus**.
2. Biết cách **triển khai Pub/Sub với Spring Boot**.
3. Xây dựng **demo chat hoặc notification real-time** bằng Redis.
4. Biết dùng Redis cho **Message Queue (List/Stream)** để xử lý nền (background jobs).
5. Nắm **best practices & lỗi thường gặp** khi scale Pub/Sub.

---

## 🧠 1. Redis Pub/Sub là gì?

Redis hỗ trợ **cơ chế Publish/Subscribe (Pub/Sub)** cho phép **các ứng dụng trao đổi thông điệp theo kênh (channel)**.

### 🔸 Mô hình tổng quan

```
[Publisher] --> [Redis Channel] --> [Subscriber(s)]
```

* **Publisher**: gửi thông điệp đến một channel (ví dụ `"chat:room1"`).
* **Subscriber**: lắng nghe channel đó và nhận mọi tin nhắn mới.
* **Redis Server**: chỉ là “message broker”, **không lưu trữ** tin nhắn.

---

### 🧩 Ví dụ thực tế

* Ứng dụng chat (phòng chat → mỗi room là 1 channel).
* Gửi thông báo realtime đến frontend (VD: notification event).
* Microservice A publish event → service B nhận để xử lý.

---

## ⚙️ 2. Cấu hình Redis Pub/Sub trong Spring Boot

### 📁 `RedisPubSubConfig.java`

```java
package com.example.redis.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.MessageListener;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.listener.ChannelTopic;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;
import org.springframework.data.redis.listener.adapter.MessageListenerAdapter;

@Configuration
public class RedisPubSubConfig {

    public static final String TOPIC_NAME = "chatroom";

    // Đăng ký Listener container
    @Bean
    public RedisMessageListenerContainer redisContainer(RedisConnectionFactory connectionFactory,
                                                        MessageListenerAdapter listenerAdapter) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(listenerAdapter, new ChannelTopic(TOPIC_NAME));
        return container;
    }

    // Gắn listener tới service
    @Bean
    public MessageListenerAdapter listenerAdapter(MessageSubscriber subscriber) {
        return new MessageListenerAdapter(subscriber, "onMessage");
    }

    // Khai báo channel
    @Bean
    public ChannelTopic topic() {
        return new ChannelTopic(TOPIC_NAME);
    }
}
```

---

## 📨 3. Publisher – Gửi thông điệp

```java
package com.example.redis.pubsub;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.listener.ChannelTopic;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class MessagePublisher {

    private final RedisTemplate<String, Object> redisTemplate;
    private final ChannelTopic topic;

    public void publish(String message) {
        redisTemplate.convertAndSend(topic.getTopic(), message);
        System.out.println("📤 Sent message: " + message);
    }
}
```

---

## 📩 4. Subscriber – Nhận thông điệp

```java
package com.example.redis.pubsub;

import org.springframework.stereotype.Service;

@Service
public class MessageSubscriber {

    public void onMessage(String message, String channel) {
        System.out.printf("📥 Received message on channel [%s]: %s%n", channel, message);
    }
}
```

---

## 🌐 5. REST Controller để gửi tin nhắn

```java
package com.example.redis.controller;

import com.example.redis.pubsub.MessagePublisher;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/chat")
@RequiredArgsConstructor
public class ChatController {

    private final MessagePublisher publisher;

    @PostMapping("/send")
    public String sendMessage(@RequestParam String message) {
        publisher.publish(message);
        return "Message sent: " + message;
    }
}
```

---

## 🧪 6. Chạy thử demo

1️⃣ Khởi động ứng dụng Spring Boot.
2️⃣ Mở **2 terminal**:

* **Terminal A** chạy Redis CLI subscriber:

  ```bash
  redis-cli SUBSCRIBE chatroom
  ```
* **Terminal B** gửi HTTP request:

  ```bash
  curl -X POST "http://localhost:8080/chat/send?message=Hello+Redis!"
  ```

🎯 Kết quả:

* **Spring Boot log:**

  ```
  📤 Sent message: Hello Redis!
  📥 Received message on channel [chatroom]: Hello Redis!
  ```
* **Redis CLI:**

  ```
  "message"
  "chatroom"
  "Hello Redis!"
  ```

---

## 💬 7. ASCII Flow – Redis Pub/Sub

```
┌───────────────┐
│   Publisher   │
└───────┬───────┘
        │ convertAndSend("chatroom", msg)
        ▼
┌────────────────────────┐
│       Redis Server     │
│  (temporary channel)   │
└──────────┬─────────────┘
           │ broadcast
           ▼
┌────────────────────────┐
│     Subscriber(s)      │
│ onMessage(channel,msg) │
└────────────────────────┘
```

---

## 🧩 8. Pub/Sub vs Queue

| Đặc điểm | Pub/Sub                                | Message Queue                               |
| -------- | -------------------------------------- | ------------------------------------------- |
| Cơ chế   | Broadcast (phát đến nhiều subscriber)  | Point-to-point (1 consumer xử lý 1 message) |
| Lưu trữ  | Không lưu tin nhắn                     | Có thể lưu (List / Stream)                  |
| Dùng khi | Realtime notification, chat            | Job queue, async processing                 |
| Đảm bảo  | Không đảm bảo nhận lại nếu mất kết nối | Có thể đảm bảo delivery                     |

---

## 🧱 9. Redis Queue với List (Background Jobs)

### Ví dụ: thêm công việc vào hàng đợi

```java
redisTemplate.opsForList().rightPush("jobQueue", "Send email to user123");
```

### Worker xử lý nền

```java
while (true) {
    String job = (String) redisTemplate.opsForList().leftPop("jobQueue", 5, TimeUnit.SECONDS);
    if (job != null) {
        System.out.println("🚀 Processing job: " + job);
    }
}
```

> 💡 Có thể dùng Spring `@Scheduled` hoặc `@Async` để chạy worker nền.
> Redis List hoạt động như **FIFO Queue**, cực nhanh.

---

## 🧮 10. Redis Stream – Queue nâng cao (từ Redis 5+)

Nếu bạn muốn:

* Lưu lịch sử message
* Có **acknowledgement**
* Hỗ trợ **consumer group**

➡️ Hãy dùng **Redis Stream** (`XADD`, `XREADGROUP`, `XACK`)

Ví dụ:

```bash
XADD mystream * name "job1" status "pending"
XREADGROUP GROUP workers alice COUNT 1 STREAMS mystream >
```

Spring Boot hỗ trợ Stream qua **ReactiveRedisTemplate** hoặc **Spring Cloud Stream** (học ở phần nâng cao).

---

## 🧠 11. Bài tập nhỏ

**Bài 1:**

* Tạo endpoint `/notify` → gửi message “System maintenance at midnight”.
* Subscriber log ra màn hình.

**Bài 2:**

* Dùng Redis List làm job queue:

  * `/job/add?name=backup`
  * `/job/process` → pop và in job ra log.

**Bài 3:**

* Thử nhiều instance app cùng subscribe → xem Redis broadcast đến tất cả.

---

## ⚠️ 12. Sai lầm phổ biến

| Sai lầm                             | Hậu quả                                    |
| ----------------------------------- | ------------------------------------------ |
| Dùng Pub/Sub cho message quan trọng | Mất dữ liệu nếu subscriber ngắt kết nối.   |
| Dùng RedisTemplate sai kiểu dữ liệu | Gửi sai format, subscriber không đọc được. |
| Không đặt topic rõ ràng             | Xung đột message giữa các module.          |
| Không monitor channel               | Khó debug khi mất message.                 |

---

## 🌟 13. Best Practices

✅ Dùng Pub/Sub cho **realtime event, chat, notification**.
✅ Dùng Queue (List/Stream) cho **background processing**.
✅ Prefix rõ ràng: `"chat:room1"`, `"notif:system"`, `"job:email"`.
✅ Log mọi message nhận để kiểm tra.
✅ Dùng RedisInsight để quan sát channel và queue.
✅ Với hệ thống lớn → dùng **Redis Cluster hoặc Kafka** khi cần durability.

---

## 🎓 Kết luận

Bạn đã học:

* Cách Redis thực hiện Pub/Sub
* Cách triển khai trong Spring Boot với Publisher – Subscriber
* Khi nào dùng Pub/Sub, khi nào dùng Queue/List
* Cách mở rộng sang Stream để lưu message bền vững
