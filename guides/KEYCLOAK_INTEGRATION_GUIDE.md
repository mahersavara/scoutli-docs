# 🔐 Hướng Dẫn Tích Hợp Keycloak - Step by Step

**Ngày:** Dec 8, 2025  
**Trạng thái hiện tại:** JWT local (plain text password)  
**Mục tiêu:** Chuyển sang OIDC + Keycloak

---

## **PHẦN 1: Setup Keycloak trong Docker Compose**

### Bước 1.1: Thêm Keycloak vào `docker-compose.yml`

**File:** `microservices/docker-compose.yml`

Thêm service này vào trước `auth-service`:

```yaml
  keycloak:
    image: quay.io/keycloak/keycloak:24.0.0
    container_name: scoutli-keycloak
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://scoutli-db:5432/scoutli?currentSchema=keycloak
      KC_DB_USERNAME: scoutli
      KC_DB_PASSWORD: scoutli
      KC_HOSTNAME: localhost
      KC_HOSTNAME_PORT: 8080
      KC_PROXY: edge
      KC_HTTP_ENABLED: true
    ports:
      - "8080:8080"
    depends_on:
      - scoutli-db
    networks:
      - scoutli-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Bước 1.2: Tạo schema cho Keycloak

**File:** `microservices/init-db/init-schemas.sql`

Thêm dòng này vào cuối file:

```sql
CREATE SCHEMA IF NOT EXISTS keycloak;
GRANT ALL PRIVILEGES ON SCHEMA keycloak TO scoutli;
```

### Bước 1.3: Chạy Docker Compose

```bash
cd microservices
docker compose up -d
```

### Bước 1.4: Kiểm tra Keycloak hoạt động

```bash
# Đợi 30-40 giây cho Keycloak khởi động
curl http://localhost:8080/health
```

Nếu thấy `{"status":"UP"}` → ✅ Keycloak ready!

---

## **PHẦN 2: Configure Keycloak**

### Bước 2.1: Đăng nhập Keycloak Admin Console

1. Mở: `http://localhost:8080/admin`
2. Username: `admin`
3. Password: `admin`

### Bước 2.2: Tạo Realm mới

1. Hover trên dropdown "Master" ở top-left
2. Click "Create Realm"
3. Name: `scoutli`
4. Click "Create"

### Bước 2.3: Tạo Client cho Auth Service

1. Menu trái → "Clients"
2. Click "Create client"
3. **Client ID:** `scoutli-auth-service`
4. Click "Next"
5. Bật: "Standard flow"
6. Click "Next"
7. **Valid redirect URIs:** 
   ```
   http://localhost:8080/*
   http://localhost:8083/*
   ```
8. **Web origins:** 
   ```
   http://localhost:8080
   http://localhost:8083
   ```
9. Click "Save"

### Bước 2.4: Lấy Client Secret

1. Click vào client `scoutli-auth-service`
2. Tab "Credentials"
3. Copy **Client Secret**

### Bước 2.5: Tạo User Test

1. Menu trái → "Users"
2. Click "Create new user"
3. Username: `testuser`
4. Email: `test@example.com`
5. First Name: `Test`
6. Last Name: `User`
7. Click "Create"
8. Tab "Credentials"
9. Click "Set password"
10. Password: `password123`
11. Bỏ tích "Temporary"
12. Click "Set password"

### Bước 2.6: Gán Role cho User

1. Tab "Role mapping"
2. Click "Assign role"
3. Chọn "MEMBER" (hoặc tạo role mới)
4. Click "Assign"

---

## **PHẦN 3: Cập Nhật Auth Service**

### Bước 3.1: Cập Nhật `application.properties`

**File:** `microservices/scoutli-auth-service/src/main/resources/application.properties`

Thêm vào cuối file:

```properties
# --- OIDC/Keycloak Configuration ---
quarkus.oidc.auth-server-url=http://localhost:8080/realms/scoutli
quarkus.oidc.client-id=scoutli-auth-service
quarkus.oidc.credentials.secret=YOUR_CLIENT_SECRET_HERE
quarkus.oidc.authentication.user-info-required=false
quarkus.oidc.token.issuer-required=false

%dev.quarkus.oidc.auth-server-url=http://localhost:8080/realms/scoutli
%dev.quarkus.oidc.client-id=scoutli-auth-service
%dev.quarkus.oidc.credentials.secret=YOUR_CLIENT_SECRET_HERE
```

**⚠️ Thay `YOUR_CLIENT_SECRET_HERE` bằng Client Secret từ Bước 2.4**

### Bước 3.2: Cập Nhật `AuthService.java`

**File:** `microservices/scoutli-auth-service/src/main/java/com/scoutli/service/AuthService.java`

Thay thế toàn bộ file bằng:

```java
package com.scoutli.service;

import com.scoutli.api.dto.AuthDTO;
import io.quarkus.oidc.client.OidcClient;
import io.quarkus.oidc.client.Tokens;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import lombok.extern.slf4j.Slf4j;

import java.util.HashMap;
import java.util.Map;

@ApplicationScoped
@Slf4j
public class AuthService {

    @Inject
    OidcClient oidcClient;

    /**
     * Keycloak Login Flow:
     * 1. Receive user credentials (email, password)
     * 2. Call Keycloak token endpoint with password grant
     * 3. Return access token to client
     */
    public String login(AuthDTO.LoginRequest request) {
        log.info("🔐 Attempting Keycloak login for user: {}", request.email);
        try {
            // Build password grant request
            Map<String, String> params = new HashMap<>();
            params.put("username", request.email);
            params.put("password", request.password);
            params.put("grant_type", "password");
            params.put("client_id", "scoutli-auth-service");

            // Call Keycloak token endpoint
            Tokens tokens = oidcClient.getTokens(params)
                    .onFailure()
                    .invoke(failure -> log.warn("❌ Keycloak login failed for user: {}. Error: {}", 
                            request.email, failure.getMessage()))
                    .await().indefinitely();

            if (tokens != null && tokens.getAccessToken() != null) {
                log.info("✅ Keycloak login successful for user: {}", request.email);
                log.debug("Token: {}", tokens.getAccessToken());
                return tokens.getAccessToken();
            }
        } catch (Exception e) {
            log.error("❌ Keycloak login failed for user: {}. Error: {}", request.email, e.getMessage(), e);
        }
        return null;
    }

    /**
     * Keycloak Registration:
     * Note: User registration should be handled through:
     * 1. Keycloak Admin REST API, or
     * 2. Keycloak registration page, or
     * 3. Frontend redirect to Keycloak registration
     */
    public boolean register(AuthDTO.RegisterRequest request) {
        log.info("📝 Registration request for: {}", request.email);
        log.warn("⚠️ User registration should be handled through Keycloak Admin API or registration page");
        // TODO: Implement Keycloak Admin API integration for user creation
        return false;
    }
}
```

### Bước 3.3: Cập Nhật `AuthController.java`

**File:** `microservices/scoutli-auth-service/src/main/java/com/scoutli/api/controller/AuthController.java`

Thay thế toàn bộ file bằng:

```java
package com.scoutli.api.controller;

import com.scoutli.api.dto.AuthDTO;
import com.scoutli.service.AuthService;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import lombok.extern.slf4j.Slf4j;

@Path("/api/auth")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Slf4j
public class AuthController {

    @Inject
    AuthService authService;

    /**
     * Login endpoint
     * POST /api/auth/login
     * Request: {"email": "test@example.com", "password": "password123"}
     * Response: {"token": "eyJhbGc..."}
     */
    @POST
    @Path("/login")
    public Response login(AuthDTO.LoginRequest request) {
        log.info("📥 Login request from: {}", request.email);
        
        if (request.email == null || request.email.isEmpty() || 
            request.password == null || request.password.isEmpty()) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity("Email and password are required")
                    .build();
        }

        String token = authService.login(request);
        
        if (token != null) {
            log.info("✅ Login successful for: {}", request.email);
            return Response.ok(new AuthDTO.AuthResponse(token))
                    .build();
        }

        log.warn("❌ Login failed for: {}", request.email);
        return Response.status(Response.Status.UNAUTHORIZED)
                .entity("Invalid credentials")
                .build();
    }

    /**
     * Register endpoint
     * POST /api/auth/register
     * Note: Redirect users to Keycloak registration page
     */
    @POST
    @Path("/register")
    public Response register(AuthDTO.RegisterRequest request) {
        log.info("📝 Registration request from: {}", request.email);
        
        return Response.status(Response.Status.NOT_IMPLEMENTED)
                .entity("Use Keycloak registration page at http://localhost:8080/realms/scoutli/account")
                .build();
    }

    /**
     * Health check endpoint
     */
    @GET
    @Path("/health")
    public Response health() {
        return Response.ok("{\"status\":\"UP\"}").build();
    }
}
```

---

## **PHẦN 4: Cập Nhật Interaction Service**

### Bước 4.1: Cập Nhật `application.properties`

**File:** `microservices/scoutli-interaction-service/src/main/resources/application.properties`

Thêm vào cuối file:

```properties
# --- OIDC/Keycloak Configuration ---
quarkus.oidc.auth-server-url=http://localhost:8080/realms/scoutli
quarkus.oidc.client-id=scoutli-interaction-service
quarkus.oidc.application-type=service

%dev.quarkus.oidc.auth-server-url=http://localhost:8080/realms/scoutli
%dev.quarkus.oidc.client-id=scoutli-interaction-service
%dev.quarkus.oidc.application-type=service
```

### Bước 4.2: Bỏ comment @RolesAllowed

**File:** `microservices/scoutli-interaction-service/src/main/java/com/scoutli/api/controller/CommentController.java`

Thay từ:

```java
@POST
// @RolesAllowed({ "MEMBER", "ADMIN" }) // Temporarily disabled for testing
public Response create(@PathParam("discoveryId") Long discoveryId, CommentDTO.CreateRequest request,
        @Context SecurityContext securityContext) {
```

Thành:

```java
@POST
@RolesAllowed({ "MEMBER", "ADMIN" })
public Response create(@PathParam("discoveryId") Long discoveryId, CommentDTO.CreateRequest request,
        @Context SecurityContext securityContext) {
```

### Bước 4.3: Xóa fallback hardcoded email

Từ:

```java
String email = securityContext.getUserPrincipal() != null 
    ? securityContext.getUserPrincipal().getName()
    : "test@example.com";
```

Thành:

```java
String email = securityContext.getUserPrincipal() != null 
    ? securityContext.getUserPrincipal().getName()
    : null;

if (email == null) {
    return Response.status(Response.Status.UNAUTHORIZED)
            .entity("Authentication required")
            .build();
}
```

---

## **PHẦN 5: Tạo Client cho Interaction Service trong Keycloak**

1. Keycloak Admin Console → Realm "scoutli" → Clients
2. Click "Create client"
3. **Client ID:** `scoutli-interaction-service`
4. Click "Next"
5. Bật: "Service accounts roles"
6. Click "Save"
7. Tab "Credentials" → Copy **Client Secret**
8. Cập nhật vào `application.properties`:

```properties
quarkus.oidc.credentials.secret=YOUR_INTERACTION_CLIENT_SECRET
```

---

## **PHẦN 6: Discovery Service (Tương tự Interaction Service)**

1. Tạo Client trong Keycloak: `scoutli-discovery-service`
2. Cập Nhật `application.properties`:

```properties
quarkus.oidc.auth-server-url=http://localhost:8080/realms/scoutli
quarkus.oidc.client-id=scoutli-discovery-service
quarkus.oidc.application-type=service
%dev.quarkus.oidc.auth-server-url=http://localhost:8080/realms/scoutli
%dev.quarkus.oidc.client-id=scoutli-discovery-service
%dev.quarkus.oidc.application-type=service
```

3. Bỏ comment `@RolesAllowed` nếu có

---

## **PHẦN 7: Testing**

### Test 7.1: Keycloak

```bash
# Kiểm tra Keycloak hoạt động
curl http://localhost:8080/health
```

### Test 7.2: Auth Service Login

```bash
# Login với Keycloak
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# Response sẽ là:
# {"token":"eyJhbGc..."}
```

### Test 7.3: Interaction Service with Token

```bash
# Lấy token từ bước trên
TOKEN="eyJhbGc..."

# Gửi request với token
curl -X POST http://localhost:8083/api/discoveries/1/comments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"content":"Test comment"}'

# Nếu không có token → 401 Unauthorized ✅
# Nếu token hết hạn → 403 Forbidden ✅
# Nếu valid → 201 Created ✅
```

---

## **PHẦN 8: Troubleshooting**

### Lỗi: "OIDC Server not available"

```bash
# Kiểm tra Keycloak đang chạy
docker ps | grep keycloak

# Logs
docker logs scoutli-keycloak

# Restart nếu cần
docker restart scoutli-keycloak
```

### Lỗi: "Invalid Client Secret"

- Kiểm tra `application.properties` có Client Secret đúng không
- So sánh với Keycloak Admin Console → Clients → Credentials

### Lỗi: "User not found"

- Kiểm tra user `testuser` đã tạo trong Keycloak chưa
- Kiểm tra realm là `scoutli` (không phải `master`)

### Lỗi: "Role not found"

- Kiểm tra user có role `MEMBER` không
- Nếu không có, thêm role trong Keycloak

---

## **PHẦN 9: Security Improvements (Optional)**

### 9.1: Enable HTTPS

```properties
KC_HTTPS_ENABLED=true
KC_HTTPS_PORT=8443
```

### 9.2: Setup Database Persistence

```yaml
keycloak:
  environment:
    KC_DB: postgres
    KC_DB_URL: jdbc:postgresql://scoutli-db:5432/scoutli?currentSchema=keycloak
```

### 9.3: Token Expiration

```properties
quarkus.oidc.token.refresh-token-time-skew=10S
quarkus.oidc.token.principal-claim=preferred_username
```

---

## **Checklist Hoàn Thành**

- [ ] ✅ Thêm Keycloak vào docker-compose.yml
- [ ] ✅ Khởi tạo schema keycloak
- [ ] ✅ Chạy `docker compose up -d`
- [ ] ✅ Đăng nhập Keycloak Admin (admin/admin)
- [ ] ✅ Tạo realm "scoutli"
- [ ] ✅ Tạo clients (auth, interaction, discovery)
- [ ] ✅ Tạo user "testuser" với password
- [ ] ✅ Gán role "MEMBER" cho testuser
- [ ] ✅ Cập Nhật auth-service code
- [ ] ✅ Cập Nhật interaction-service code
- [ ] ✅ Cập Nhật discovery-service code
- [ ] ✅ Test login endpoint
- [ ] ✅ Test protected endpoints
- [ ] ✅ Verify Keycloak tokens hoạt động

---

## **Các Lệnh Hữu Ích**

```bash
# Start all services
cd microservices
docker compose up -d

# View logs
docker logs scoutli-keycloak
docker logs scoutli-auth-service

# Rebuild auth service
mvn -f scoutli-auth-service/pom.xml clean quarkus:dev

# Remove all containers
docker compose down -v

# Check if Keycloak is ready
curl -s http://localhost:8080/health | jq
```

---

**Liên hệ hỗ trợ:** Nếu gặp vấn đề, kiểm tra logs: `docker logs scoutli-keycloak`
