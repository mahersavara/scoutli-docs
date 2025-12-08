# Kafka Implementation Guide - Step by Step

Hướng dẫn chi tiết để bạn tự tay implement Kafka vào project Scoutli từ đầu.

## 🎯 Mục tiêu học tập

Sau khi hoàn thành guide này, bạn sẽ:
- ✅ Hiểu cách Kafka hoạt động trong Quarkus
- ✅ Biết cách setup Kafka infrastructure
- ✅ Implement Producer để emit events
- ✅ Implement Consumer để xử lý events
- ✅ Test end-to-end flow
- ✅ Debug và troubleshoot Kafka issues

## 📋 Prerequisites

- [ ] Java 17+ installed
- [ ] Maven installed
- [ ] Docker & Docker Compose installed
- [ ] Đã đọc `KAFKA_QUARKUS_GUIDE.md` (lý thuyết cơ bản)

## 🗺️ Implementation Roadmap

```
Phase 1: Infrastructure Setup (30 phút)
  ├─ Step 1: Setup Kafka với Docker
  └─ Step 2: Verify Kafka is running

Phase 2: Simple Producer (45 phút)
  ├─ Step 3: Add dependencies
  ├─ Step 4: Create Event model
  ├─ Step 5: Configure Producer
  └─ Step 6: Implement & Test Producer

Phase 3: Simple Consumer (45 phút)
  ├─ Step 7: Configure Consumer
  ├─ Step 8: Implement Consumer
  └─ Step 9: Test Producer → Consumer flow

Phase 4: Real Use Case (1 giờ)
  ├─ Step 10: Implement Comment Notification
  └─ Step 11: End-to-end testing

Phase 5: Advanced Topics (bonus)
  ├─ Error handling
  ├─ Idempotency
  └─ Monitoring
```

---

## Phase 1: Infrastructure Setup

### Step 1: Setup Kafka với Docker Compose

**Mục tiêu:** Chạy Kafka + Zookeeper locally để test.

**Bạn cần làm gì:**

1. Mở file `microservices/docker-compose.yml`
2. Thêm 2 services: `zookeeper` và `kafka` vào file

**Hướng dẫn:**

```yaml
# Thêm vào phần services: (sau interaction-db)

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    container_name: scoutli-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    networks:
      - scoutli-net

  kafka:
    image: confluentinc/cp-kafka:latest
    container_name: scoutli-kafka
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
    networks:
      - scoutli-net
```

**Giải thích:**
- `zookeeper`: Kafka cần Zookeeper để quản lý cluster metadata
- `KAFKA_ADVERTISED_LISTENERS`: Địa chỉ mà clients sẽ connect tới
- `KAFKA_AUTO_CREATE_TOPICS_ENABLE`: Tự động tạo topics khi Producer gửi message
- Port `9092`: Kafka broker port (standard)

**Test:**
```bash
cd microservices
docker-compose up -d zookeeper kafka
docker-compose ps
# Phải thấy zookeeper và kafka đang running
```

**Checkpoint:** Nếu cả 2 containers đều running, bạn đã pass Step 1! ✅

---

### Step 2: Verify Kafka is Running

**Mục tiêu:** Đảm bảo Kafka hoạt động đúng.

**Bạn cần làm gì:**

Kiểm tra logs để đảm bảo Kafka đã start thành công:

```bash
# Xem logs của Kafka
docker-compose logs kafka

# Tìm dòng này trong logs:
# "INFO [KafkaServer id=1] started"
```

Done

**Test nâng cao (optional):**

Dùng Kafka CLI để test:

```bash
# Exec vào Kafka container
docker exec -it scoutli-kafka bash

# List topics (ban đầu sẽ trống)
kafka-topics --list --bootstrap-server localhost:9092

# Tạo test topic
kafka-topics --create --topic test --bootstrap-server localhost:9092

# List lại để xem topic vừa tạo
kafka-topics --list --bootstrap-server localhost:9092

# Exit
exit
```

**Checkpoint:** Nếu bạn thấy logs "started" hoặc có thể list topics, pass Step 2! ✅

---

## Phase 2: Simple Producer

### Step 3: Add Kafka Dependencies

**Mục tiêu:** Thêm dependencies cần thiết cho Kafka.

**Bạn cần làm gì:**

Chọn 1 service để bắt đầu. Mình recommend `scoutli-interaction-service` (service xử lý Comments).

1. Mở `microservices/scoutli-interaction-service/pom.xml`
2. Thêm dependency sau vào `<dependencies>`:

```xml
<!-- Kafka Reactive Messaging -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-reactive-messaging-kafka</artifactId>
</dependency>
```

**Giải thích:**
- `quarkus-smallrye-reactive-messaging-kafka`: Extension chính để dùng Kafka
- Nó bao gồm: Kafka client, Reactive Messaging API, serializers

**Test:**
```bash
cd microservices/scoutli-interaction-service
mvn clean compile
# Phải compile thành công, không có error về dependency
```

**Checkpoint:** Nếu compile success, pass Step 3! ✅

---

### Step 4: Create Event Model

**Mục tiêu:** Tạo class đại diện cho event sẽ gửi qua Kafka.

**Bạn cần làm gì:**

Tạo package và class mới:

**File:** `microservices/scoutli-interaction-service/src/main/java/com/scoutli/event/CommentCreatedEvent.java`

```java
package com.scoutli.event;

import java.time.LocalDateTime;

/**
 * Event được emit khi có comment mới được tạo.
 * Event này sẽ được gửi qua Kafka để các services khác xử lý.
 */
public class CommentCreatedEvent {

    // TODO: Thêm các fields cần thiết
    // Gợi ý: commentId, discoveryId, authorId, content, timestamp...

    public Long commentId;
    public Long discoveryId;
    public String authorEmail;
    public String content;
    public LocalDateTime timestamp;

    // TODO: Tạo constructor mặc định (bắt buộc cho JSON deserialization)
    public CommentCreatedEvent() {
    }

    // TODO: Tạo constructor với params để dễ khởi tạo
    public CommentCreatedEvent(Long commentId, Long discoveryId,
                               String authorEmail, String content) {
        this.commentId = commentId;
        this.discoveryId = discoveryId;
        this.authorEmail = authorEmail;
        this.content = content;
        this.timestamp = LocalDateTime.now();
    }

    // TODO: Override toString() để dễ debug
    @Override
    public String toString() {
        return "CommentCreatedEvent{" +
                "commentId=" + commentId +
                ", discoveryId=" + discoveryId +
                ", authorEmail='" + authorEmail + '\'' +
                '}';
    }
}
```

**Giải thích:**
- Event là POJO (Plain Old Java Object)
- Phải có **constructor mặc định** (Jackson cần để deserialize)
- Fields public để Jackson dễ dàng serialize/deserialize
- `toString()` giúp debug khi log

**Câu hỏi tự kiểm tra:**
1. Tại sao cần constructor mặc định?
   - *Trả lời: Jackson deserializer cần nó để tạo object từ JSON*

2. Event này chứa thông tin gì?
   - *Trả lời: Tất cả info cần thiết để Consumer xử lý mà không cần query DB*

**Checkpoint:** File compile được và có constructor mặc định? Pass Step 4! ✅

---

### Step 5: Configure Producer

**Mục tiêu:** Config để service có thể gửi events vào Kafka.

**Bạn cần làm gì:**

Mở file `microservices/scoutli-interaction-service/src/main/resources/application.properties`

Thêm config sau (vào cuối file, sau phần common configs):

```properties
# ==================== KAFKA CONFIGURATION ====================

# Kafka broker address
kafka.bootstrap.servers=localhost:9092

# ==================== PRODUCER: Comment Events ====================
# Channel name: comment-created-out (tên này sẽ dùng trong code)
mp.messaging.outgoing.comment-created-out.connector=smallrye-kafka
mp.messaging.outgoing.comment-created-out.topic=comment-events
mp.messaging.outgoing.comment-created-out.value.serializer=io.quarkus.kafka.client.serialization.ObjectMapperSerializer
```

**Giải thích từng dòng:**

```properties
# Địa chỉ Kafka broker (localhost vì chạy Docker local)
kafka.bootstrap.servers=localhost:9092

# Config cho outgoing channel (Producer)
# Format: mp.messaging.outgoing.<channel-name>.<property>

mp.messaging.outgoing.comment-created-out.connector=smallrye-kafka
# Chỉ định dùng Kafka connector

mp.messaging.outgoing.comment-created-out.topic=comment-events
# Tên topic trong Kafka (sẽ tự động tạo nếu chưa có)

mp.messaging.outgoing.comment-created-out.value.serializer=...
# Serializer: chuyển Java object → JSON để gửi qua Kafka
```

**Naming convention:**
- Channel name: `comment-created-out` (tên logic trong code)
- Topic name: `comment-events` (tên thật trong Kafka)
- `-out` suffix: Convention cho outgoing channels (Producer)

**Checkpoint:** Config đã thêm vào application.properties? Pass Step 5! ✅

---

### Step 6: Implement Producer in CommentService

**Mục tiêu:** Viết code để emit event khi tạo comment.

**Bạn cần làm gì:**

Mở file `microservices/scoutli-interaction-service/src/main/java/com/scoutli/service/CommentService.java`

**TODO 1: Inject Emitter**

Thêm vào đầu class (sau các @Inject khác):

```java
import org.eclipse.microprofile.reactive.messaging.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;
import com.scoutli.event.CommentCreatedEvent;

// ... existing code ...

@Inject
@Channel("comment-created-out")  // Tên channel phải khớp với config
Emitter<CommentCreatedEvent> eventEmitter;
```

**Giải thích:**
- `@Channel("comment-created-out")`: Link với config đã tạo ở Step 5
- `Emitter<CommentCreatedEvent>`: Interface để gửi events
- Quarkus sẽ tự động inject Emitter đã config sẵn

**TODO 2: Emit Event sau khi lưu Comment**

Tìm method `createComment()`, sau dòng `commentRepository.persist(comment);`, thêm:

```java
@Transactional
public CommentDTO createComment(Long discoveryId,
                                 CommentDTO.CreateRequest request,
                                 String userEmail) {
    // ... existing code: validate, create comment, persist ...

    commentRepository.persist(comment);

    // TODO: Emit event vào Kafka
    CommentCreatedEvent event = new CommentCreatedEvent(
        comment.getId(),
        discoveryId,
        userEmail,
        request.content
    );

    eventEmitter.send(event);

    log.info("✅ Comment created (ID: {}) and event emitted to Kafka", comment.getId());

    return toDTO(comment);
}
```

**Giải thích flow:**
1. Lưu comment vào DB (transaction)
2. Tạo event object với data từ comment
3. Gửi event vào Kafka qua Emitter (non-blocking)
4. Return response về FE ngay (không chờ event được xử lý)

**Câu hỏi tự kiểm tra:**

1. Event được emit khi nào?
   - *Sau khi comment đã lưu vào DB thành công*

2. Nếu emit event bị lỗi, comment đã lưu có bị rollback không?
   - *Không, vì emit là non-blocking và ngoài transaction*

3. FE có cần chờ event được xử lý không?
   - *Không, response trả về ngay sau khi lưu DB*

**Test:**

1. Start Kafka (nếu chưa chạy):
```bash
cd microservices
docker-compose up -d
```

2. Start interaction-service:
```bash
cd microservices/scoutli-interaction-service
mvn quarkus:dev
```

3. Tạo comment qua API (adjust theo API của bạn):
```bash
curl -X POST http://localhost:8083/api/comments \
  -H "Content-Type: application/json" \
  -d '{
    "discoveryId": 1,
    "content": "Great place to visit!"
  }'
```

4. Check logs, phải thấy:
```
✅ Comment created (ID: 1) and event emitted to Kafka
```

5. Verify event trong Kafka:
```bash
docker exec -it scoutli-kafka bash

# Consumer để xem messages trong topic
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic comment-events --from-beginning

# Phải thấy JSON của event
```

**Checkpoint:** Nếu thấy event trong Kafka topic, pass Step 6! ✅🎉

---

## Phase 3: Simple Consumer

### Step 7: Configure Consumer

**Mục tiêu:** Config để service có thể đọc events từ Kafka.

**Bạn cần làm gì:**

Mở lại `application.properties`, thêm config consumer:

```properties
# ==================== CONSUMER: Comment Events ====================
# Channel name: comment-created-in (convention: -in suffix cho consumer)
mp.messaging.incoming.comment-created-in.connector=smallrye-kafka
mp.messaging.incoming.comment-created-in.topic=comment-events
mp.messaging.incoming.comment-created-in.value.deserializer=io.quarkus.kafka.client.serialization.ObjectMapperDeserializer

# Chỉ định class type cho deserializer
mp.messaging.incoming.comment-created-in.value.deserializer.type=com.scoutli.event.CommentCreatedEvent

# Consumer group ID (quan trọng!)
mp.messaging.incoming.comment-created-in.group.id=interaction-service

# Đọc từ đầu topic khi consumer mới join
mp.messaging.incoming.comment-created-in.auto.offset.reset=earliest
```

**Giải thích:**

```properties
# Deserializer: chuyển JSON từ Kafka → Java object
value.deserializer=...ObjectMapperDeserializer

# Phải nói cho deserializer biết convert sang class nào
value.deserializer.type=com.scoutli.event.CommentCreatedEvent

# Consumer Group ID: Cực kỳ quan trọng!
group.id=interaction-service
# - Các consumers cùng group.id sẽ chia nhau xử lý messages (load balancing)
# - Mỗi message chỉ được 1 consumer trong group xử lý
# - Kafka track offset theo group.id

# auto.offset.reset=earliest
# - earliest: Đọc từ đầu topic khi consumer mới
# - latest: Chỉ đọc messages mới từ lúc consumer start
```

**Câu hỏi quan trọng:**

1. Nếu 2 instances cùng `group.id=interaction-service`, khi có 1 event, mấy instance xử lý?
   - *Trả lời: Chỉ 1 instance (Kafka load balance)*

2. Nếu 2 instances khác `group.id`, khi có 1 event, mấy instance xử lý?
   - *Trả lời: CẢ 2 instances (mỗi group nhận 1 copy)*

**Checkpoint:** Config đã thêm? Pass Step 7! ✅

---

### Step 8: Implement Consumer

**Mục tiêu:** Viết code để xử lý events từ Kafka.

**Bạn cần làm gì:**

Tạo class mới để handle events:

**File:** `microservices/scoutli-interaction-service/src/main/java/com/scoutli/consumer/CommentEventConsumer.java`

```java
package com.scoutli.consumer;

import com.scoutli.event.CommentCreatedEvent;
import io.smallrye.reactive.messaging.annotations.Blocking;
import org.eclipse.microprofile.reactive.messaging.Incoming;
import jakarta.enterprise.context.ApplicationScoped;
import lombok.extern.slf4j.Slf4j;

@ApplicationScoped
@Slf4j
public class CommentEventConsumer {

    /**
     * Consumer xử lý CommentCreatedEvent từ Kafka.
     *
     * @Incoming: Chỉ định channel name (phải khớp config)
     * @Blocking: Xử lý synchronously (không async)
     */
    @Incoming("comment-created-in")
    @Blocking
    public void handleCommentCreated(CommentCreatedEvent event) {

        log.info("📬 Received event from Kafka: {}", event);

        // TODO: Implement business logic
        // VD: Tạo notification, gửi email, update cache, etc.

        // Bây giờ chỉ log ra để test
        log.info("   - Comment ID: {}", event.commentId);
        log.info("   - Discovery ID: {}", event.discoveryId);
        log.info("   - Author: {}", event.authorEmail);
        log.info("   - Content: {}", event.content);
        log.info("   - Timestamp: {}", event.timestamp);

        log.info("✅ Event processed successfully!");
    }
}
```

**Giải thích:**

**Annotations:**
- `@ApplicationScoped`: Bean sẽ tồn tại suốt lifecycle của app
- `@Incoming("comment-created-in")`: Link với channel config ở Step 7
- `@Blocking`: Method xử lý synchronously, không return CompletionStage

**Method signature:**
```java
public void handleCommentCreated(CommentCreatedEvent event)
```
- Parameter type: `CommentCreatedEvent` - Quarkus tự động deserialize JSON → object
- Return type: `void` - Kafka sẽ auto-commit offset sau khi method return (không throw exception)

**Processing:**
- Đơn giản log thông tin event
- Trong thực tế sẽ có business logic: tạo notification, gửi email, etc.

**Alternative signatures (nâng cao):**

```java
// Async processing
@Incoming("comment-created-in")
public CompletionStage<Void> handle(CommentCreatedEvent event) {
    return CompletableFuture.runAsync(() -> {
        // async processing
    });
}

// With metadata
@Incoming("comment-created-in")
public void handle(Message<CommentCreatedEvent> message) {
    CommentCreatedEvent event = message.getPayload();
    long offset = message.getMetadata(IncomingKafkaRecordMetadata.class)
                         .get().getOffset();
    // manual ack
    message.ack();
}
```

**Checkpoint:** Consumer class đã tạo? Pass Step 8! ✅

---

### Step 9: Test Producer → Consumer Flow

**Mục tiêu:** Verify toàn bộ flow hoạt động end-to-end.

**Bạn cần làm gì:**

**1. Start infrastructure:**
```bash
cd microservices
docker-compose up -d
```

**2. Start service (dev mode):**
```bash
cd microservices/scoutli-interaction-service
mvn quarkus:dev
```

**3. Check logs khi start:**
Tìm dòng confirm Kafka connected:
```
INFO  [io.sma.rea.mes.kafka] SRMCK18258: Kafka version: 3.x.x
INFO  [io.sma.rea.mes.kafka] SRMCK18259: Kafka commit-id: xxx
```

**4. Tạo comment để trigger event:**
```bash
curl -X POST http://localhost:8083/api/comments \
  -H "Content-Type: application/json" \
  -d '{
    "discoveryId": 1,
    "content": "Testing Kafka flow!"
  }'
```

**5. Observe logs:**

Bạn phải thấy 2 dòng logs:

```
✅ Comment created (ID: 1) and event emitted to Kafka   <- Producer log
📬 Received event from Kafka: CommentCreatedEvent{...}  <- Consumer log
   - Comment ID: 1
   - Discovery ID: 1
   - Author: user@example.com
   - Content: Testing Kafka flow!
✅ Event processed successfully!
```

**6. Verify trong Kafka (optional):**
```bash
docker exec -it scoutli-kafka bash

# Check topic exists
kafka-topics --list --bootstrap-server localhost:9092
# Should see: comment-events

# Check messages
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic comment-events --from-beginning

# Check consumer group
kafka-consumer-groups --bootstrap-server localhost:9092 --list
# Should see: interaction-service

# Check consumer group details
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group interaction-service
```

**Troubleshooting:**

**Problem:** Consumer không nhận event
- Check `group.id` có đúng không
- Check topic name có khớp giữa producer và consumer không
- Restart service để consumer reconnect

**Problem:** Deserialize error
- Check `value.deserializer.type` có đúng full class name không
- Check Event class có constructor mặc định không

**Problem:** Connection refused
- Check Kafka đang chạy: `docker-compose ps`
- Check port 9092 có available không: `netstat -an | grep 9092`

**Checkpoint:** Nếu thấy cả Producer và Consumer logs, pass Step 9! ✅🎉

---

## Phase 4: Real Use Case - Comment Notification

### Step 10: Implement Notification Feature

**Mục tiêu:** Xây dựng tính năng thực tế: Khi có comment mới, gửi notification cho owner của Discovery.

**Bạn cần làm gì:**

**TODO 1: Tạo Notification Entity (nếu chưa có)**

Check xem đã có `Notification` entity chưa. Nếu chưa, tạo mới:

**File:** `microservices/scoutli-interaction-service/src/main/java/com/scoutli/domain/entity/Notification.java`

```java
package com.scoutli.domain.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import java.time.LocalDateTime;

@Entity
@Table(name = "notifications")
@Getter
@Setter
public class Notification {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String userEmail;  // User sẽ nhận notification

    @Column(nullable = false)
    private String type;  // "COMMENT", "RATING", etc.

    @Column(nullable = false, length = 500)
    private String message;

    private Long relatedId;  // ID của comment, rating, etc.

    private String relatedType;  // "COMMENT", "RATING"

    @Column(nullable = false)
    private Boolean isRead = false;

    @Column(nullable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
}
```

**TODO 2: Create Flyway Migration**

**File:** `microservices/scoutli-interaction-service/src/main/resources/db/migration/V2.0.0__Create_Notifications_Table.sql`

```sql
CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    user_email VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    message VARCHAR(500) NOT NULL,
    related_id BIGINT,
    related_type VARCHAR(50),
    is_read BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_email (user_email),
    INDEX idx_created_at (created_at)
);
```

**TODO 3: Enhance Event to include Discovery Owner**

Vấn đề: Consumer cần biết owner của Discovery để tạo notification, nhưng event hiện tại không có thông tin này.

**Giải pháp:** Thêm field `discoveryOwnerEmail` vào `CommentCreatedEvent`.

Update `CommentCreatedEvent.java`:

```java
public class CommentCreatedEvent {
    public Long commentId;
    public Long discoveryId;
    public String authorEmail;
    public String content;
    public String discoveryOwnerEmail;  // ← Thêm field này
    public LocalDateTime timestamp;

    public CommentCreatedEvent() {
    }

    public CommentCreatedEvent(Long commentId, Long discoveryId,
                               String authorEmail, String content,
                               String discoveryOwnerEmail) {  // ← Update constructor
        this.commentId = commentId;
        this.discoveryId = discoveryId;
        this.authorEmail = authorEmail;
        this.content = content;
        this.discoveryOwnerEmail = discoveryOwnerEmail;
        this.timestamp = LocalDateTime.now();
    }
}
```

**TODO 4: Update Producer để lấy Discovery Owner**

Trong `CommentService.createComment()`, trước khi emit event:

```java
@Transactional
public CommentDTO createComment(Long discoveryId,
                                 CommentDTO.CreateRequest request,
                                 String userEmail) {
    // ... existing validation ...

    // TODO: Fetch Discovery để lấy owner email
    // Cách 1: Call Discovery Service qua REST Client
    // Cách 2: Lưu discovery owner trong Comment table (denormalization)
    // Cách 3: Consumer query Discovery Service

    // Giả sử bạn có DiscoveryServiceClient
    String discoveryOwnerEmail = discoveryServiceClient.getDiscoveryOwner(discoveryId);

    commentRepository.persist(comment);

    // Emit event với owner info
    CommentCreatedEvent event = new CommentCreatedEvent(
        comment.getId(),
        discoveryId,
        userEmail,
        request.content,
        discoveryOwnerEmail  // ← Owner email
    );

    eventEmitter.send(event);

    return toDTO(comment);
}
```

**Câu hỏi tư duy:**
- Nếu Discovery Service bị down lúc lấy owner, phải làm sao?
  - Option A: Không emit event (comment được tạo nhưng không có notification)
  - Option B: Emit event nhưng không có owner, Consumer sẽ query sau
  - Option C: Store owner email trong Comment table luôn (trade-off: data duplication)

Bạn chọn approach nào và tại sao?

**TODO 5: Implement Notification Logic trong Consumer**

Update `CommentEventConsumer.handleCommentCreated()`:

```java
@ApplicationScoped
@Slf4j
public class CommentEventConsumer {

    @Inject
    EntityManager em;

    @Incoming("comment-created-in")
    @Blocking
    @Transactional  // ← Thêm transaction
    public void handleCommentCreated(CommentCreatedEvent event) {

        log.info("📬 Received CommentCreatedEvent: {}", event);

        // TODO: Validate - không tạo notification nếu author = owner
        if (event.authorEmail.equals(event.discoveryOwnerEmail)) {
            log.info("Skipping notification: author is the owner");
            return;
        }

        // TODO: Check idempotency - đã tạo notification cho comment này chưa?
        long existingCount = (Long) em.createQuery(
            "SELECT COUNT(n) FROM Notification n WHERE n.relatedId = :commentId AND n.relatedType = 'COMMENT'"
        ).setParameter("commentId", event.commentId).getSingleResult();

        if (existingCount > 0) {
            log.info("Notification already exists for comment {}, skipping", event.commentId);
            return;
        }

        // TODO: Tạo notification
        Notification notification = new Notification();
        notification.setUserEmail(event.discoveryOwnerEmail);
        notification.setType("COMMENT");
        notification.setMessage(
            String.format("%s đã comment vào Discovery của bạn: \"%s\"",
                event.authorEmail,
                truncate(event.content, 50))
        );
        notification.setRelatedId(event.commentId);
        notification.setRelatedType("COMMENT");
        notification.setIsRead(false);
        notification.setCreatedAt(LocalDateTime.now());

        em.persist(notification);

        log.info("✅ Notification created for user {}", event.discoveryOwnerEmail);
    }

    private String truncate(String str, int maxLength) {
        if (str.length() <= maxLength) return str;
        return str.substring(0, maxLength) + "...";
    }
}
```

**Giải thích:**

1. **Idempotency check:** Đảm bảo không tạo duplicate notifications nếu event được replay
2. **Business logic validation:** Skip nếu author = owner
3. **Message formatting:** Tạo message dễ đọc cho user

**Checkpoint:** Notification được tạo trong DB sau khi comment? Pass Step 10! ✅

---

### Step 11: End-to-End Testing

**Mục tiêu:** Test toàn bộ flow từ API call → Event → Notification.

**Test Scenarios:**

**Scenario 1: Happy Path**

```bash
# 1. Tạo Discovery (giả sử owner: alice@example.com)
# 2. User khác (bob@example.com) comment vào Discovery

curl -X POST http://localhost:8083/api/comments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <bob-token>" \
  -d '{
    "discoveryId": 1,
    "content": "Amazing place!"
  }'

# 3. Check DB
SELECT * FROM notifications WHERE user_email = 'alice@example.com';
# Should have 1 notification: "bob@example.com đã comment vào Discovery của bạn..."
```

**Scenario 2: Owner tự comment (không tạo notification)**

```bash
curl -X POST http://localhost:8083/api/comments \
  -H "Authorization: Bearer <alice-token>" \
  -d '{
    "discoveryId": 1,
    "content": "Thanks everyone!"
  }'

# Check logs: "Skipping notification: author is the owner"
# Check DB: không có notification mới
```

**Scenario 3: Idempotency (replay event)**

```bash
# 1. Stop service
# 2. Kafka vẫn giữ event (vì retention policy)
# 3. Restart service
# 4. Consumer sẽ đọc lại event từ last committed offset

# Check logs: "Notification already exists for comment X, skipping"
# Check DB: vẫn chỉ có 1 notification (không duplicate)
```

**Scenario 4: Error Handling**

```bash
# Simulate error: Comment có invalid discoveryOwnerEmail

# Event sẽ retry theo config
# Nếu retry hết vẫn fail → Message bị stuck

# Solution: Implement error handling trong consumer
```

**Performance Testing:**

```bash
# Gửi 100 comments cùng lúc
for i in {1..100}; do
  curl -X POST http://localhost:8083/api/comments \
    -H "Content-Type: application/json" \
    -d "{\"discoveryId\": 1, \"content\": \"Comment $i\"}" &
done

# Check logs: Tất cả events được xử lý
# Check DB: 100 notifications được tạo
# Check timing: Mỗi event xử lý mất bao lâu?
```

**Monitoring:**

```bash
# Check consumer lag (events chưa xử lý)
docker exec scoutli-kafka \
  kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group interaction-service

# Output:
# GROUP             TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# interaction-svc   comment-events  0          150             150             0

# LAG = 0: Đã xử lý hết
# LAG > 0: Còn events chưa xử lý (consumer chậm)
```

**Checkpoint:** Tất cả scenarios pass? Pass Step 11! ✅🎉🎉🎉

---

## Phase 5: Advanced Topics (Bonus)

### Error Handling & Retry

**Problem:** Consumer throw exception khi xử lý event.

**Options:**

**1. Fail Strategy:**
```properties
mp.messaging.incoming.comment-created-in.failure-strategy=fail
```
- Consumer dừng lại, cần manual restart
- Use case: Critical errors cần human intervention

**2. Ignore Strategy:**
```properties
mp.messaging.incoming.comment-created-in.failure-strategy=ignore
```
- Bỏ qua message lỗi, tiếp tục xử lý messages tiếp theo
- Use case: Non-critical events, có thể mất 1 vài messages

**3. Dead Letter Queue:**
```properties
mp.messaging.incoming.comment-created-in.failure-strategy=dead-letter-queue
mp.messaging.incoming.comment-created-in.dead-letter-queue.topic=comment-events-dlq
```
- Gửi message lỗi sang topic khác
- Use case: Cần reprocess failed messages sau

**Implementation:**

```java
@Incoming("comment-created-in")
@Blocking
@Transactional
public void handleCommentCreated(CommentCreatedEvent event) {
    try {
        // Business logic
        createNotification(event);

    } catch (ValidationException e) {
        // Không retry cho validation errors
        log.error("Validation error: {}", e.getMessage());
        // Không throw → message bị skip

    } catch (Exception e) {
        // Retry cho các errors khác
        log.error("Error processing event: {}", event, e);
        throw e;  // Kafka sẽ retry
    }
}
```

---

### Monitoring & Observability

**1. Metrics:**

Add dependency:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-micrometer-registry-prometheus</artifactId>
</dependency>
```

Metrics endpoint: `http://localhost:8083/q/metrics`

Key metrics:
- `kafka_consumer_records_consumed_total`
- `kafka_consumer_records_lag_max`
- `kafka_producer_record_send_total`

**2. Logging:**

```java
@Incoming("comment-created-in")
public void handle(Message<CommentCreatedEvent> message) {
    IncomingKafkaRecordMetadata metadata = message.getMetadata(IncomingKafkaRecordMetadata.class).get();

    log.info("Processing event from Kafka:");
    log.info("  Topic: {}", metadata.getTopic());
    log.info("  Partition: {}", metadata.getPartition());
    log.info("  Offset: {}", metadata.getOffset());
    log.info("  Timestamp: {}", metadata.getTimestamp());

    // Process...
}
```

**3. Health Checks:**

Quarkus tự động expose health checks:
- `http://localhost:8083/q/health/live`
- `http://localhost:8083/q/health/ready`

Kafka health được include tự động.

---

### Testing

**1. Unit Test với In-Memory:**

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-test-kafka-companion</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@QuarkusTest
class CommentEventConsumerTest {

    @Inject
    @Channel("comment-created-in")
    Emitter<CommentCreatedEvent> emitter;

    @Test
    void testNotificationCreated() {
        // Emit test event
        CommentCreatedEvent event = new CommentCreatedEvent(
            1L, 1L, "bob@test.com", "Test", "alice@test.com"
        );
        emitter.send(event);

        // Wait và verify
        await().atMost(5, SECONDS).until(() -> {
            List<Notification> notifications =
                Notification.list("userEmail", "alice@test.com");
            return !notifications.isEmpty();
        });
    }
}
```

---

## 🎓 Tổng kết

**Bạn đã học được:**
- ✅ Setup Kafka infrastructure với Docker
- ✅ Configure Producer và Consumer trong Quarkus
- ✅ Implement event-driven architecture
- ✅ Handle serialization/deserialization
- ✅ Implement idempotency
- ✅ Error handling strategies
- ✅ Testing và monitoring

**Next Steps:**
1. Apply pattern này cho Discovery events (DiscoveryCreated, DiscoveryUpdated)
2. Implement Rating events
3. Add more consumers: Analytics, Search Indexer
4. Deploy Kafka cluster lên Kubernetes (Strimzi)
5. Implement Schema Registry (Avro) cho enterprise use case

**Questions to reflect:**
1. Khi nào nên dùng Kafka thay vì REST API?
2. Trade-offs của event-driven architecture?
3. Làm sao đảm bảo data consistency giữa services?
4. Cách monitor và debug distributed events?

Chúc bạn học tốt! 🚀
