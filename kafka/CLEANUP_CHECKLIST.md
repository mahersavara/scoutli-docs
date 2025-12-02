# Kafka Code Cleanup Checklist

Files bạn cần clean up trước khi bắt đầu implement lại Kafka từ đầu.

## 📋 Files to Clean

### 1. AuthService (scoutli-auth-service)

**File:** `microservices/scoutli-auth-service/src/main/java/com/scoutli/service/AuthService.java`

**Lines to remove:** 11-12, 24-26, 38-40

```java
// ❌ REMOVE these imports:
import org.eclipse.microprofile.reactive.messaging.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;

// ❌ REMOVE this field:
@Inject
@Channel("users-out")
Emitter<UserRegisteredEvent> userEventEmitter;

// ❌ REMOVE these lines in register() method:
UserRegisteredEvent event = new UserRegisteredEvent(request.email, simulatedKeycloakUserId, simulatedRole);
userEventEmitter.send(event);
log.info("UserRegisteredEvent sent to Kafka for user: {}", request.email);
```

**After cleanup, register() method should only:**
- Log registration attempt
- Return success/failure boolean
- NO Kafka code

---

### 2. DiscoveryService (scoutli-discovery-service)

**File:** `microservices/scoutli-discovery-service/src/main/java/com/scoutli/service/DiscoveryService.java`

**Lines to remove:** 17-19, 42-44, 98-99, 109-120

```java
// ❌ REMOVE these imports:
import org.eclipse.microprofile.reactive.messaging.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;
import org.eclipse.microprofile.reactive.messaging.Incoming;

// ❌ REMOVE this field:
@Inject
@Channel("discoveries-out")
Emitter<DiscoveryCreatedEvent> discoveryEventEmitter;

// ❌ REMOVE this line in createDiscovery():
discoveryEventEmitter.send(DiscoveryCreatedEvent.fromDiscovery(discovery));

// ❌ REMOVE entire consumer method:
@Incoming("discovery-events-in")
public Uni<Void> processDiscoveryEvents(DiscoveryCreatedEvent event) {
    // ... entire method
}
```

---

### 3. CommentService (scoutli-interaction-service)

**File:** `microservices/scoutli-interaction-service/src/main/java/com/scoutli/service/CommentService.java`

**Check:** Does this file have any Kafka code?
- If yes: Remove similar to above
- If no: ✅ Already clean

---

### 4. Event Classes

**Option A: Keep for reference** (Recommended)
- Keep files as examples
- You'll recreate similar structures

**Option B: Delete completely**

Files:
- `microservices/scoutli-auth-service/src/main/java/com/scoutli/event/UserRegisteredEvent.java`
- `microservices/scoutli-discovery-service/src/main/java/com/scoutli/event/DiscoveryCreatedEvent.java`

If keeping: **No action needed**

If deleting:
```bash
rm microservices/scoutli-auth-service/src/main/java/com/scoutli/event/UserRegisteredEvent.java
rm microservices/scoutli-discovery-service/src/main/java/com/scoutli/event/DiscoveryCreatedEvent.java
```

---

### 5. Application Properties

**Files to clean:**
- `microservices/scoutli-interaction-service/src/main/resources/application.properties`

**Lines to remove:** Line 13-14, 33-34, 51-52

```properties
# ❌ REMOVE these lines:

# Line 13-14 (Production Kafka config)
mp.messaging.connector.smallrye-kafka.bootstrap.servers=<your-prod-kafka-brokers>:9092

# Line 33-34 (Dev Kafka config)
%dev.mp.messaging.connector.smallrye-kafka.bootstrap.servers=localhost:9092

# Line 51-52 (UAT Kafka config)
%uat.mp.messaging.connector.smallrye-kafka.bootstrap.servers=<your-uat-kafka-brokers>:9092
```

**Note:** Nếu có application.properties trong auth-service hoặc discovery-service có Kafka config, remove tương tự.

---

### 6. Dependencies (Optional)

**Files:** `pom.xml` trong mỗi service

**Option A: Keep dependencies** (Recommended)
- Giữ lại dependency `quarkus-smallrye-reactive-messaging-kafka`
- Bạn sẽ cần nó khi implement

**Option B: Remove temporarily**

```xml
<!-- ❌ REMOVE (or comment out): -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-reactive-messaging-kafka</artifactId>
</dependency>
```

Mình recommend **Option A** (keep it).

---

## 🧹 Cleanup Commands

### Quick Cleanup Script

Create file `cleanup-kafka.sh`:

```bash
#!/bin/bash

echo "🧹 Cleaning up Kafka code..."

# Backup trước khi cleanup (just in case)
echo "📦 Creating backup..."
tar -czf kafka-code-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
    microservices/*/src/main/java/com/scoutli/service/*Service.java \
    microservices/*/src/main/java/com/scoutli/event/

echo "✅ Backup created!"

# Note: Bạn sẽ tự manually remove code theo checklist trên
# Không nên dùng sed/awk vì có thể sai
echo "📝 Please manually remove code according to CLEANUP_CHECKLIST.md"
echo "   - AuthService.java"
echo "   - DiscoveryService.java"
echo "   - CommentService.java (if needed)"
echo "   - application.properties files"

echo "✅ Done! Ready to implement from scratch."
```

### Manual Cleanup Steps

1. **Backup code** (just in case):
```bash
cd microservices
git stash push -m "backup kafka code before cleanup"
# Or:
git checkout -b kafka-cleanup
```

2. **Open each file listed above**

3. **Remove/comment out** Kafka-related code

4. **Compile to verify** no errors:
```bash
cd microservices/scoutli-auth-service
mvn clean compile

cd microservices/scoutli-discovery-service
mvn clean compile

cd microservices/scoutli-interaction-service
mvn clean compile
```

5. **Commit cleanup**:
```bash
git add .
git commit -m "chore: remove incomplete Kafka implementation for clean rebuild"
```

---

## ✅ Verification

After cleanup, verify:

### ✅ No Kafka imports
```bash
# Should return empty or only in Event classes
grep -r "@Channel\|@Incoming\|Emitter" microservices/*/src/main/java/com/scoutli/service/
```

### ✅ Services compile successfully
```bash
cd microservices/scoutli-interaction-service
mvn clean compile
# Should succeed with no errors
```

### ✅ No Kafka config in application.properties
```bash
grep -r "kafka\|mp.messaging" microservices/*/src/main/resources/application.properties
# Should return empty or only commented lines
```

### ✅ Dependencies still present (if kept)
```bash
grep -A 3 "quarkus-smallrye-reactive-messaging-kafka" microservices/*/pom.xml
# Should show dependency in pom.xml files
```

---

## 🚀 Next Steps

After cleanup is complete:

1. ✅ Read `KAFKA_IMPLEMENTATION_GUIDE.md` from Phase 1
2. ✅ Start with **Step 1: Setup Kafka with Docker**
3. ✅ Follow the guide step-by-step
4. ✅ Test after each checkpoint
5. ✅ Ask questions if stuck!

---

## 📝 Notes

**Why clean up?**
- Học tốt hơn khi tự tay implement từ đầu
- Hiểu rõ mỗi bước làm gì và tại sao
- Tránh confusion từ code cũ chưa hoàn chỉnh

**What if I make mistakes?**
- Git stash/branch đã backup code cũ
- Có thể revert bất cứ lúc nào
- Implementation guide có troubleshooting section

**How long will this take?**
- Cleanup: 15-30 phút
- Implementation (follow guide): 3-4 giờ
- Totally worth it for learning! 💪

---

Good luck với việc implement! 🎉
