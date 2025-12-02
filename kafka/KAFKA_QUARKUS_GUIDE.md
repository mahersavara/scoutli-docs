# Kafka & Quarkus Integration Guide

Hướng dẫn tích hợp Kafka vào microservices Quarkus cho dự án Scoutli.

## 🎯 Tổng quan

### Kafka là gì?
Kafka là một **distributed streaming platform** - nền tảng cho event-driven architecture với khả năng:
- High-throughput, low-latency message broker
- Message persistence (lưu trữ events theo thời gian)
- Nhiều consumers có thể đọc cùng một message
- Horizontal scaling thông qua partitioning

### Kafka vs Traditional Message Queue

| Feature | Traditional MQ (RabbitMQ) | Kafka |
|---------|---------------------------|-------|
| Message persistence | Xóa sau khi đọc | Lưu lại theo retention policy |
| Consumers | 1 message → 1 consumer | 1 message → nhiều consumers |
| Pattern | Work Queue | Event Log / Pub-Sub |
| Use case | Task distribution | Event streaming, audit log |

### Kafka vs Async API

**Async API:**
```java
@Async
public CompletionStage<Result> process() {
    // Xử lý bất đồng bộ
    // ❌ Mất data khi server restart
}
```

**Kafka:**
```java
emitter.send(event);  // Gửi vào Kafka
// ✅ Event được persist, không mất khi restart
// ✅ Có thể replay events
// ✅ Multiple consumers có thể xử lý
```

## 🔑 Core Concepts

### 1. Topic
- Giống như "table" hoặc "category" để lưu events
- VD: `user-events`, `comment-events`, `discovery-events`

### 2. Partition
- Chia nhỏ topic để scale horizontally
- Messages cùng key → cùng partition (đảm bảo thứ tự)
- Mỗi partition là một ordered log

```
Topic: "comment-events"
┌─────────────────────────────────────┐
│ Partition 0: [msg1][msg2][msg3]... │
│ Partition 1: [msg4][msg5][msg6]... │
│ Partition 2: [msg7][msg8][msg9]... │
└─────────────────────────────────────┘
```

### 3. Producer
- Service gửi messages (events) vào Kafka
- VD: Comment Service emit `CommentCreatedEvent`

### 4. Consumer
- Service đọc messages từ Kafka
- VD: Notification Service đọc `CommentCreatedEvent` để gửi thông báo

### 5. Consumer Group
- Nhóm consumers cùng xử lý messages
- Mỗi partition chỉ được 1 consumer trong group xử lý
- Scale bằng cách thêm consumers vào group

### 6. Offset
- Vị trí của message trong partition
- Consumer track offset để biết đã đọc đến đâu
- Có thể "rewind" để đọc lại (replay events)

## 📦 Use Case: Comment Notification

### Flow truyền thống
```
User A post comment → Comment Service
                          ↓
                      1. Lưu comment vào DB
                      2. Tìm owner của Discovery
                      3. Tạo notification cho owner
                      4. Gửi email (nếu có)
                          ↓
                      Response về FE (chậm!)
```

**Vấn đề:**
- User phải chờ tất cả logic xong mới có response
- Comment Service phải biết về Notification logic
- Khó thêm side-effects mới (analytics, moderation...)

### Flow với Kafka
```
User A post comment → Comment Service
                          ↓
                      1. Lưu comment vào DB
                      2. Emit event "CommentCreated"
                          ↓
                      Response về FE ngay! ✅

                          ↓
            [Kafka Topic: comment-events]
                          ↓
        ┌─────────────────┴─────────────────┐
        ↓                                    ↓
   Notification Service              Analytics Service
   (tạo notification)                (track engagement)
```

**Lợi ích:**
- Response nhanh, user không chờ
- Services độc lập (loose coupling)
- Dễ dàng thêm consumers mới
- Events được lưu lại, có thể replay

## 🚀 Implementation Steps

### Bước 1: Setup Kafka với Docker

**File: `docker-compose.yml`**
```yaml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
```

**Chạy:**
```bash
docker-compose up -d

# Kiểm tra
docker-compose ps
docker-compose logs kafka
```

### Bước 2: Add Dependencies

**File: `pom.xml`**
```xml
<dependencies>
    <!-- Kafka với Reactive Messaging -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-smallrye-reactive-messaging-kafka</artifactId>
    </dependency>

    <!-- JSON serialization -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-resteasy-reactive-jackson</artifactId>
    </dependency>
</dependencies>
```

### Bước 3: Tạo Event Model

**File: `src/main/java/com/scoutli/events/CommentCreatedEvent.java`**
```java
package com.scoutli.events;

import java.time.LocalDateTime;

public class CommentCreatedEvent {

    public Long commentId;
    public Long discoveryId;
    public Long authorId;
    public String authorName;
    public String content;
    public Long discoveryOwnerId;
    public LocalDateTime timestamp;

    // Constructor mặc định (cần cho deserialization)
    public CommentCreatedEvent() {
    }

    // Constructor với params
    public CommentCreatedEvent(Long commentId, Long discoveryId,
                               Long authorId, String authorName,
                               String content, Long discoveryOwnerId) {
        this.commentId = commentId;
        this.discoveryId = discoveryId;
        this.authorId = authorId;
        this.authorName = authorName;
        this.content = content;
        this.discoveryOwnerId = discoveryOwnerId;
        this.timestamp = LocalDateTime.now();
    }

    @Override
    public String toString() {
        return "CommentCreatedEvent{" +
                "commentId=" + commentId +
                ", discoveryId=" + discoveryId +
                ", authorName='" + authorName + '\'' +
                '}';
    }
}
```

### Bước 4: Configure Kafka

**File: `src/main/resources/application.properties`**
```properties
# Kafka Bootstrap Server
kafka.bootstrap.servers=localhost:9092

# ========== PRODUCER (Comment Service gửi event) ==========
mp.messaging.outgoing.comment-created-out.connector=smallrye-kafka
mp.messaging.outgoing.comment-created-out.topic=comment-events
mp.messaging.outgoing.comment-created-out.value.serializer=io.quarkus.kafka.client.serialization.ObjectMapperSerializer

# ========== CONSUMER (Notification Service nhận event) ==========
mp.messaging.incoming.comment-created-in.connector=smallrye-kafka
mp.messaging.incoming.comment-created-in.topic=comment-events
mp.messaging.incoming.comment-created-in.value.deserializer=io.quarkus.kafka.client.serialization.ObjectMapperDeserializer
mp.messaging.incoming.comment-created-in.value.deserializer.type=com.scoutli.events.CommentCreatedEvent
mp.messaging.incoming.comment-created-in.group.id=notification-service
mp.messaging.incoming.comment-created-in.auto.offset.reset=earliest
```

**Giải thích:**
- `outgoing`: Producer configuration (gửi messages)
- `incoming`: Consumer configuration (nhận messages)
- `topic`: Tên topic trong Kafka (producer và consumer phải cùng topic)
- `group.id`: Consumer group name (quan trọng cho scaling)
- `auto.offset.reset=earliest`: Đọc từ đầu topic khi consumer mới join

### Bước 5: Implement Producer

**File: `src/main/java/com/scoutli/interaction/service/CommentService.java`**
```java
package com.scoutli.interaction.service;

import com.scoutli.interaction.entity.Comment;
import com.scoutli.interaction.entity.Discovery;
import com.scoutli.events.CommentCreatedEvent;
import io.smallrye.reactive.messaging.annotations.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

@ApplicationScoped
public class CommentService {

    @Inject
    @Channel("comment-created-out")  // Map với config trong application.properties
    Emitter<CommentCreatedEvent> eventEmitter;

    @Transactional
    public Comment createComment(Long discoveryId, Long userId,
                                  String content, String userName) {

        // 1. Validate và tìm Discovery
        Discovery discovery = Discovery.findById(discoveryId);
        if (discovery == null) {
            throw new RuntimeException("Discovery not found: " + discoveryId);
        }

        // 2. Tạo Comment entity và lưu vào DB
        Comment comment = new Comment();
        comment.discoveryId = discoveryId;
        comment.userId = userId;
        comment.content = content;
        comment.createdAt = LocalDateTime.now();
        comment.persist();

        // 3. Emit event vào Kafka
        CommentCreatedEvent event = new CommentCreatedEvent(
            comment.id,
            discoveryId,
            userId,
            userName,
            content,
            discovery.userId  // Owner của Discovery
        );

        // Send event (non-blocking)
        eventEmitter.send(event);

        System.out.println("✅ Comment saved (ID: " + comment.id + ") and event emitted to Kafka");

        // 4. Return comment ngay, không cần chờ event được xử lý
        return comment;
    }
}
```

### Bước 6: Implement Consumer

**File: `src/main/java/com/scoutli/interaction/consumer/NotificationConsumer.java`**
```java
package com.scoutli.interaction.consumer;

import com.scoutli.events.CommentCreatedEvent;
import com.scoutli.interaction.entity.Notification;
import io.smallrye.reactive.messaging.annotations.Blocking;
import org.eclipse.microprofile.reactive.messaging.Incoming;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;
import org.jboss.logging.Logger;

@ApplicationScoped
public class NotificationConsumer {

    private static final Logger LOG = Logger.getLogger(NotificationConsumer.class);

    @Incoming("comment-created-in")  // Map với config trong application.properties
    @Blocking  // Xử lý synchronous (không async)
    @Transactional
    public void handleCommentCreated(CommentCreatedEvent event) {

        LOG.infof("📬 Received event from Kafka: %s", event);

        // Tạo notification cho owner của Discovery
        createNotification(event);

        LOG.infof("✅ Notification created for user %d", event.discoveryOwnerId);
    }

    private void createNotification(CommentCreatedEvent event) {
        // Không tạo notification nếu user comment vào Discovery của chính mình
        if (event.authorId.equals(event.discoveryOwnerId)) {
            LOG.info("Skipping notification: author is the owner");
            return;
        }

        Notification notification = new Notification();
        notification.userId = event.discoveryOwnerId;
        notification.type = "COMMENT";
        notification.message = event.authorName + " đã comment vào Discovery của bạn: \""
                             + truncate(event.content, 50) + "\"";
        notification.relatedId = event.commentId;
        notification.relatedType = "COMMENT";
        notification.isRead = false;
        notification.createdAt = LocalDateTime.now();

        notification.persist();
    }

    private String truncate(String str, int maxLength) {
        if (str.length() <= maxLength) return str;
        return str.substring(0, maxLength) + "...";
    }
}
```

### Bước 7: Tạo REST Endpoint để Test

**File: `src/main/java/com/scoutli/interaction/resource/CommentResource.java`**
```java
package com.scoutli.interaction.resource;

import com.scoutli.interaction.dto.CreateCommentRequest;
import com.scoutli.interaction.entity.Comment;
import com.scoutli.interaction.service.CommentService;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@Path("/api/comments")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class CommentResource {

    @Inject
    CommentService commentService;

    @POST
    public Response createComment(CreateCommentRequest request) {
        Comment comment = commentService.createComment(
            request.discoveryId,
            request.userId,
            request.content,
            request.userName
        );

        return Response.status(Response.Status.CREATED).entity(comment).build();
    }
}
```

**DTO:**
```java
package com.scoutli.interaction.dto;

public class CreateCommentRequest {
    public Long discoveryId;
    public Long userId;
    public String userName;
    public String content;
}
```

### Bước 8: Test

**Chạy Quarkus:**
```bash
cd microservices/scoutli-interaction-service
mvn quarkus:dev
```

**Test với curl:**
```bash
curl -X POST http://localhost:8080/api/comments \
  -H "Content-Type: application/json" \
  -d '{
    "discoveryId": 1,
    "userId": 2,
    "userName": "John Doe",
    "content": "Great place! Would love to visit again."
  }'
```

**Kết quả mong đợi trong console:**
```
✅ Comment saved (ID: 123) and event emitted to Kafka
📬 Received event from Kafka: CommentCreatedEvent{commentId=123, discoveryId=1, authorName='John Doe'}
✅ Notification created for user 1
```

## 🎓 Câu hỏi để hiểu sâu hơn

### Câu 1: Channel Mapping
**Q:** `@Channel("comment-created-out")` và `@Incoming("comment-created-in")` map với phần nào trong `application.properties`?

**A:**
- `@Channel("comment-created-out")` → `mp.messaging.outgoing.comment-created-out.*`
- `@Incoming("comment-created-in")` → `mp.messaging.incoming.comment-created-in.*`

Channel name phải khớp với prefix trong config.

### Câu 2: Multiple Instances
**Q:** Nếu chạy 2 instances của Quarkus app (port 8080 và 8081), khi gửi 1 comment, event sẽ được xử lý mấy lần?

**A:** **1 lần** - vì cả 2 instances đều có cùng `group.id=notification-service`. Kafka sẽ phân phối message cho 1 trong 2 instances (load balancing). Nếu muốn cả 2 đều xử lý, phải dùng khác `group.id`.

### Câu 3: Error Handling
**Q:** Nếu Notification Consumer bị lỗi (throw Exception), event có bị mất không?

**A:** **Không mất!** Kafka sẽ retry theo cấu hình. Có thể config:
```properties
# Retry strategy
mp.messaging.incoming.comment-created-in.failure-strategy=fail
mp.messaging.incoming.comment-created-in.max-retries=3
```

Strategies:
- `fail`: Dừng consumer (cần manual restart)
- `ignore`: Bỏ qua message lỗi, tiếp tục
- `dead-letter-queue`: Gửi message lỗi sang topic khác

## 🔧 Best Practices

### 1. Event Design
- **Immutable:** Events không nên thay đổi sau khi emit
- **Self-contained:** Event chứa đủ thông tin để consumer xử lý
- **Versioned:** Thêm version field nếu cần evolve schema

```java
public class CommentCreatedEvent {
    public String eventVersion = "1.0";  // Schema version
    // ... other fields
}
```

### 2. Error Handling
```java
@Incoming("comment-created-in")
@Blocking
@Transactional
public void handleEvent(CommentCreatedEvent event) {
    try {
        processEvent(event);
    } catch (Exception e) {
        LOG.error("Failed to process event: " + event, e);
        // Option 1: Rethrow để Kafka retry
        throw e;
        // Option 2: Log và skip (nếu event không critical)
        // return;
    }
}
```

### 3. Idempotency
Consumer phải idempotent (xử lý cùng event nhiều lần cho kết quả giống nhau):

```java
@Transactional
public void handleCommentCreated(CommentCreatedEvent event) {
    // Check xem notification đã tồn tại chưa
    boolean exists = Notification.count("relatedId = ?1 and relatedType = 'COMMENT'",
                                         event.commentId) > 0;
    if (exists) {
        LOG.info("Notification already exists, skipping");
        return;
    }

    // Tạo notification
    createNotification(event);
}
```

### 4. Dead Letter Queue
Xử lý messages failed sau nhiều lần retry:

```properties
mp.messaging.incoming.comment-created-in.failure-strategy=dead-letter-queue
mp.messaging.incoming.comment-created-in.dead-letter-queue.topic=comment-events-dlq
```

### 5. Monitoring
Log events để debug:
```java
@Incoming("comment-created-in")
public void handleEvent(CommentCreatedEvent event) {
    LOG.infof("Processing event: %s (offset: %d, partition: %d)",
              event,
              metadata.getOffset(),
              metadata.getPartition());
}
```

## 🧪 Testing

### Unit Test với In-Memory
```java
@QuarkusTest
@TestProfile(KafkaTestProfile.class)
class NotificationConsumerTest {

    @Inject
    @Channel("comment-created-in")
    Emitter<CommentCreatedEvent> emitter;

    @Test
    void testNotificationCreated() {
        // Emit test event
        CommentCreatedEvent event = new CommentCreatedEvent(...);
        emitter.send(event);

        // Wait và verify
        await().atMost(5, SECONDS).until(() ->
            Notification.count("userId = ?1", event.discoveryOwnerId) > 0
        );
    }
}
```

### Integration Test với Testcontainers
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-test-kafka-companion</artifactId>
    <scope>test</scope>
</dependency>
```

## 📚 More Use Cases cho Scoutli

### 1. Discovery Events
```java
// DiscoveryCreatedEvent
// DiscoveryUpdatedEvent
// DiscoveryDeletedEvent

// Consumers:
// - Search Indexer (sync to Elasticsearch)
// - Analytics Service
// - Reverse Geocoding Service
```

### 2. Rating Events
```java
// RatingCreatedEvent
// RatingUpdatedEvent

// Consumers:
// - Discovery Service (update average rating)
// - Analytics Service (track popular locations)
```

### 3. Moderation Events
```java
// ContentFlaggedEvent
// ContentApprovedEvent
// ContentRejectedEvent

// Consumers:
// - Notification Service
// - Audit Log Service
```

## 🔗 Resources

- [Quarkus Kafka Guide](https://quarkus.io/guides/kafka)
- [SmallRye Reactive Messaging](https://smallrye.io/smallrye-reactive-messaging/)
- [Kafka Documentation](https://kafka.apache.org/documentation/)
- [Event-Driven Architecture Patterns](https://martinfowler.com/articles/201701-event-driven.html)

## 🎯 Next Steps

1. ✅ Implement Comment → Notification flow
2. Add Discovery events (created, updated)
3. Add Rating events
4. Implement Analytics consumer
5. Add Dead Letter Queue handling
6. Monitor với Kafka UI hoặc Prometheus
7. Deploy Kafka cluster trên Kubernetes (Strimzi operator)
