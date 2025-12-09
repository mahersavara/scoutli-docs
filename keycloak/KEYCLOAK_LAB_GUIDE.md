# Keycloak Integration Lab Guide

## Complete Step-by-Step Implementation with Troubleshooting

---

## 📋 Overview

This guide documents the complete process of integrating Keycloak as an Identity and Access Management (IAM) server into the Scoutli microservices architecture.

**Duration**: ~2 hours  
**Difficulty**: Intermediate  
**Prerequisites**: Docker, Java 17+, Maven, Basic understanding of OAuth2/OIDC

---

## 🎯 What We Built

```
┌─────────────────────────────────────────────────────────────┐
│                   SCOUTLI ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────────┐      ┌─────────────────────────────┐     │
│   │   Frontend   │      │      Keycloak (IAM)         │     │
│   │  Angular     │◄────►│   http://localhost:8180     │     │
│   │  Port: 4200  │      │   Realm: scoutli            │     │
│   └──────────────┘      └─────────────────────────────┘     │
│          │                         │                         │
│          │ JWT Token              │ Token Validation        │
│          ▼                         ▼                         │
│   ┌──────────────────────────────────────────────────┐      │
│   │              Microservices                        │      │
│   │  ┌────────────┐ ┌─────────────┐ ┌─────────────┐  │      │
│   │  │auth-service│ │discovery-svc│ │interaction  │  │      │
│   │  │  Port 8081 │ │  Port 8082  │ │  Port 8083  │  │      │
│   │  └────────────┘ └─────────────┘ └─────────────┘  │      │
│   └──────────────────────────────────────────────────┘      │
│                              │                               │
│                              ▼                               │
│   ┌──────────────────────────────────────────────────┐      │
│   │        SHARED PostgreSQL Database (scoutli-db)   │      │
│   │  ┌──────┐ ┌─────────┐ ┌───────────┐ ┌────────┐  │      │
│   │  │ auth │ │discovery│ │interaction│ │keycloak│  │      │
│   │  │schema│ │ schema  │ │  schema   │ │ schema │  │      │
│   │  └──────┘ └─────────┘ └───────────┘ └────────┘  │      │
│   └──────────────────────────────────────────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

> **Note**: We use a **single shared database** with separate schemas for each service. This simplifies infrastructure and reduces resource usage.

---

## 📚 Table of Contents

1. [Infrastructure Setup](#step-1-infrastructure-setup)
2. [Database Schema Setup](#step-11-database-schema-for-keycloak)
3. [Realm Configuration](#step-2-realm-configuration)
4. [Environment-Based Configuration](#step-3-environment-based-configuration)
5. [Microservices Integration](#step-4-microservices-integration)
6. [Testing & Verification](#step-5-testing--verification)
7. [Bugs & Solutions](#bugs--solutions-encountered)
8. [Quick Reference](#quick-reference)

---

## Step 1: Infrastructure Setup

### 1.1 Database Schema for Keycloak

**File**: `microservices/init-db/init-schemas.sql`

```sql
-- Create schemas for each microservice (including Keycloak)
CREATE SCHEMA IF NOT EXISTS auth;
CREATE SCHEMA IF NOT EXISTS discovery;
CREATE SCHEMA IF NOT EXISTS interaction;
CREATE SCHEMA IF NOT EXISTS keycloak;

-- Create keycloak user for Keycloak service
DO $$
BEGIN
    IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = 'keycloak') THEN
        CREATE USER keycloak WITH PASSWORD 'keycloak';
    END IF;
END
$$;

-- Grant privileges to scoutli user
GRANT ALL PRIVILEGES ON SCHEMA auth TO scoutli;
GRANT ALL PRIVILEGES ON SCHEMA discovery TO scoutli;
GRANT ALL PRIVILEGES ON SCHEMA interaction TO scoutli;

-- Grant privileges on keycloak schema to keycloak user
GRANT ALL PRIVILEGES ON SCHEMA keycloak TO keycloak;
GRANT ALL PRIVILEGES ON DATABASE scoutli TO keycloak;

-- Set default privileges
ALTER DEFAULT PRIVILEGES IN SCHEMA auth GRANT ALL ON TABLES TO scoutli;
ALTER DEFAULT PRIVILEGES IN SCHEMA discovery GRANT ALL ON TABLES TO scoutli;
ALTER DEFAULT PRIVILEGES IN SCHEMA interaction GRANT ALL ON TABLES TO scoutli;
ALTER DEFAULT PRIVILEGES IN SCHEMA keycloak GRANT ALL ON TABLES TO keycloak;
```

### 1.2 Add Keycloak to Docker Compose

**File**: `microservices/docker-compose.yml`

```yaml
# Shared PostgreSQL - one database, multiple schemas
scoutli-db:
  image: postgres:15
  container_name: scoutli-db
  environment:
    POSTGRES_DB: scoutli
    POSTGRES_USER: scoutli
    POSTGRES_PASSWORD: scoutli
  ports:
    - "5433:5432"
  volumes:
    - scoutli_db_data:/var/lib/postgresql/data
    - ./init-db:/docker-entrypoint-initdb.d  # Auto-runs init-schemas.sql
  networks:
    - scoutli-net

# Keycloak uses scoutli-db with 'keycloak' schema (NO separate database)
keycloak:
  image: quay.io/keycloak/keycloak:24.0.1
  container_name: scoutli-keycloak
  command: start-dev --import-realm
  environment:
    KC_DB: postgres
    KC_DB_URL: jdbc:postgresql://scoutli-db:5432/scoutli?currentSchema=keycloak
    KC_DB_USERNAME: keycloak
    KC_DB_PASSWORD: keycloak
    KEYCLOAK_ADMIN: admin
    KEYCLOAK_ADMIN_PASSWORD: admin
    KC_HEALTH_ENABLED: "true"
    KC_METRICS_ENABLED: "true"
  ports:
    - "8180:8080"
  volumes:
    - ../scoutli-gitops/configs/keycloak/scoutli-realm.json:/opt/keycloak/data/import/realm.json
  depends_on:
    - scoutli-db
  networks:
    - scoutli-net
```

### 1.3 Key Decisions Made

| Decision | Why |
|----------|-----|
| **Shared database with schemas** | Simplified infrastructure, reduced resources |
| **Keycloak schema** | Isolated data within same PostgreSQL instance |
| **Port 8180** | Avoid conflict with services on 8080/8081/8082/8083 |
| **GitOps config location** | Realm config in `scoutli-gitops` for Infrastructure as Code |
| **`--import-realm`** | Automatic realm setup on first startup |

---

## Step 2: Realm Configuration

### 2.1 Create Realm JSON

**File**: `scoutli-gitops/configs/keycloak/scoutli-realm.json`

```json
{
    "realm": "scoutli",
    "enabled": true,
    "sslRequired": "none",
    "roles": {
        "realm": [
            {"name": "MEMBER", "description": "Regular member role"},
            {"name": "ADMIN", "description": "Administrator role"}
        ]
    },
    "clients": [
        {
            "clientId": "scoutli-backend",
            "enabled": true,
            "clientAuthenticatorType": "client-secret",
            "secret": "secret",
            "directAccessGrantsEnabled": true,
            "redirectUris": ["http://localhost:8081/*", "http://localhost:8082/*", "http://localhost:8083/*"],
            "webOrigins": ["*"],
            "publicClient": false,
            "serviceAccountsEnabled": true,
            "protocol": "openid-connect",
            "fullScopeAllowed": true
        },
        {
            "clientId": "scoutli-frontend",
            "enabled": true,
            "publicClient": true,
            "directAccessGrantsEnabled": true,
            "redirectUris": ["http://localhost/*"],
            "webOrigins": ["*"],
            "protocol": "openid-connect",
            "fullScopeAllowed": true
        }
    ],
    "users": [
        {
            "username": "test-user",
            "enabled": true,
            "email": "test@example.com",
            "firstName": "Test",
            "lastName": "User",
            "credentials": [{"type": "password", "value": "password", "temporary": false}],
            "realmRoles": ["MEMBER"]
        },
        {
            "username": "admin-user",
            "enabled": true,
            "email": "admin@example.com",
            "firstName": "Admin",
            "lastName": "User",
            "credentials": [{"type": "password", "value": "admin", "temporary": false}],
            "realmRoles": ["ADMIN", "MEMBER"]
        }
    ]
}
```

### 2.2 Understanding the Configuration

| Element | Purpose |
|---------|---------|
| **Realm** | Isolated security domain for Scoutli |
| **Roles** | MEMBER (basic), ADMIN (full access) |
| **scoutli-backend** | Confidential client for microservices |
| **scoutli-frontend** | Public client for Angular app |
| **directAccessGrantsEnabled** | Allows password grant for testing |
| **Test Users** | Pre-created for development |

---

## Step 3: Environment-Based Configuration

### 3.1 File Structure (Per Service)

```
scoutli-interaction-service/
├── .env                              # Local secrets (GITIGNORED)
├── .env.example                      # Template (COMMIT THIS)
└── src/main/resources/
    ├── application.properties        # Shared config
    ├── application-dev.properties    # Local development
    ├── application-uat.properties    # UAT environment
    └── application-prod.properties   # Production
```

### 3.2 Shared Configuration

**File**: `application.properties`

```properties
# Application
quarkus.application.name=interaction-service
quarkus.http.port=${HTTP_PORT:8083}

# Database (uses environment variables)
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5433}/${DB_NAME:scoutli}?currentSchema=interaction
quarkus.datasource.username=${DB_USERNAME:scoutli}
quarkus.datasource.password=${DB_PASSWORD:scoutli}

# Keycloak/OIDC (uses environment variables)
quarkus.oidc.auth-server-url=${KEYCLOAK_URL:http://localhost:8180}/realms/scoutli
quarkus.oidc.client-id=${KEYCLOAK_CLIENT_ID:scoutli-backend}
quarkus.oidc.credentials.secret=${KEYCLOAK_CLIENT_SECRET:secret}
quarkus.oidc.application-type=service
quarkus.oidc.token.issuer=${KEYCLOAK_URL:http://localhost:8180}/realms/scoutli

# Security
quarkus.security.enabled=${SECURITY_ENABLED:true}

# Kafka
kafka.bootstrap.servers=${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
```

### 3.3 Environment File (.env)

```env
# Database (Shared scoutli-db)
DB_HOST=localhost
DB_PORT=5433
DB_NAME=scoutli
DB_USERNAME=scoutli
DB_PASSWORD=scoutli

# Keycloak
KEYCLOAK_URL=http://localhost:8180
KEYCLOAK_CLIENT_ID=scoutli-backend
KEYCLOAK_CLIENT_SECRET=secret

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Service
HTTP_PORT=8083
LOG_LEVEL=DEBUG
QUARKUS_PROFILE=dev
```

---

## Step 4: Microservices Integration

### 4.1 Add OIDC Dependency

**File**: `pom.xml`

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-oidc</artifactId>
</dependency>
```

### 4.2 Secure Endpoints (Optional)

**File**: `CommentController.java`

```java
import jakarta.annotation.security.RolesAllowed;

@Path("/api/discoveries/{discoveryId}/comments")
public class CommentController {
    
    @POST
    @RolesAllowed({"MEMBER", "ADMIN"})  // Requires authenticated user
    public Response create(...) {
        // Keycloak JWT automatically validated by Quarkus OIDC
    }
    
    @DELETE
    @RolesAllowed("ADMIN")  // Admin only
    public Response delete(...) {
        // ...
    }
}
```

---

## Step 5: Testing & Verification

### 5.1 Start All Services

```bash
cd microservices

# Start database first (creates schemas automatically)
docker-compose up -d scoutli-db

# Wait 10 seconds for DB to initialize
sleep 10

# Start Keycloak (uses same database)
docker-compose up -d keycloak
```

### 5.2 Verify Keycloak is Running

```bash
# Check container status
docker ps | grep keycloak

# Check logs
docker logs scoutli-keycloak --tail 20
```

### 5.3 Verify OIDC Discovery Endpoint

```bash
curl -s "http://localhost:8180/realms/scoutli/.well-known/openid-configuration" | jq
```

Expected: JSON with `issuer`, `token_endpoint`, etc.

### 5.4 Get JWT Token

```bash
export KEYCLOAK_URL=http://localhost:8180
export TOKEN=$(curl -s -X POST "$KEYCLOAK_URL/realms/scoutli/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=scoutli-frontend" \
  -d "username=test-user" \
  -d "password=password" | jq -r .access_token)

echo $TOKEN
```

### 5.5 Access Keycloak Admin Console

- **URL**: http://localhost:8180/admin
- **Username**: `admin`
- **Password**: `admin`

### 5.6 Test Microservices Integration

Once you have the `access_token` from Step 5.4, you can call the other services.

#### 1. Test Discovery Service (Protected Endpoint)
Try to create a discovery (Requires MEMBER or ADMIN role).

```bash
# Verify you have TOKEN exported from previous step
curl -X POST http://localhost:8082/api/discoveries \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Hidden Cafe",
    "description": "A quiet place",
    "city": "Hanoi",
    "country": "Vietnam",
    "latitude": 21.0,
    "longitude": 105.8
  }' | jq
```

#### 2. Test Interaction Service (Authentication Context)
Post a comment. The service extracts your email from the token.

```bash
# Discovery ID 1 typically created by above request
curl -X POST http://localhost:8083/api/discoveries/1/comments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Great spot!"
  }' | jq
```

#### 3. Test Auth Service (Internal User Data)
Verify the Auth Service can validate credentials (used by other services or frontend login).

```bash
curl -X POST http://localhost:8081/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "password"
  }' | jq
```

---

## 🐛 Bugs & Solutions Encountered

### Bug 1: Keycloak Port Wrong in Config

**Symptom**: Services couldn't connect to Keycloak
```
Connection refused: localhost:8080
```

**Root Cause**: Config had `localhost:8080` but Keycloak runs on `8180`

**Solution**:
```properties
# Wrong
%dev.quarkus.oidc.auth-server-url=http://localhost:8080/realms/scoutli

# Correct
%dev.quarkus.oidc.auth-server-url=http://localhost:8180/realms/scoutli
```

---

### Bug 2: `unauthorized_client` Error

**Symptom**: Token request failed
```json
{"error":"unauthorized_client","error_description":"..."}
```

**Root Cause**: Frontend client didn't have `directAccessGrantsEnabled`

**Solution**: Update `scoutli-realm.json`:
```json
{
    "clientId": "scoutli-frontend",
    "directAccessGrantsEnabled": true,  // ADD THIS
    ...
}
```

---

### Bug 3: Realm Config Not Updating

**Symptom**: Changes to `scoutli-realm.json` not reflected in Keycloak

**Root Cause**: Keycloak only imports realm on **first** startup with empty database

**Solution**: Delete database volume and recreate:
```bash
docker-compose down
docker volume rm microservices_scoutli_db_data
docker-compose up -d scoutli-db
sleep 10
docker-compose up -d keycloak
```

---

### Bug 4: Keycloak Can't Connect to Shared Database

**Symptom**: Keycloak fails to start with DB connection errors

**Root Cause**: Keycloak user/schema not created in shared database

**Solution**: Update `init-schemas.sql` to create keycloak user and schema:
```sql
CREATE SCHEMA IF NOT EXISTS keycloak;
CREATE USER keycloak WITH PASSWORD 'keycloak';
GRANT ALL PRIVILEGES ON SCHEMA keycloak TO keycloak;
```

---

### Bug 6: Keycloak Startup Failure on Existing Database

**Symptom**: Keycloak container exits immediately with "FATAL: password authentication failed for user 'keycloak'".

**Root Cause**: When adding Keycloak to an **existing** `scoutli-db` container that was created before the Keycloak configuration was added, the `init-schemas.sql` script (which creates the `keycloak` user and schema) does NOT run again automatically. PostgreSQL only runs initialization scripts on the very first startup when the data directory is empty.

**Solution**:
You must manually create the user and schema in the running database.

1. Exec into the database container:
   ```bash
   docker exec -it scoutli-db psql -U scoutli -d scoutli
   ```
   *(Or running the script from outside)*
   ```bash
   docker exec scoutli-db psql -U scoutli -d scoutli -f /docker-entrypoint-initdb.d/init-schemas.sql
   ```
2. Restart Keycloak:
   ```bash
   docker start scoutli-keycloak
   ```

---



## 📋 Quick Reference

### Database Architecture

```
┌─────────────────────────────────────────────────────┐
│            scoutli-db (PostgreSQL :5433)            │
├─────────────────────────────────────────────────────┤
│ SCHEMA: auth        │ User: scoutli                 │
│ SCHEMA: discovery   │ User: scoutli                 │
│ SCHEMA: interaction │ User: scoutli                 │
│ SCHEMA: keycloak    │ User: keycloak                │
└─────────────────────────────────────────────────────┘
```

### Service URLs

| Service | URL |
|---------|-----|
| Keycloak Admin | http://localhost:8180/admin |
| Keycloak Realm | http://localhost:8180/realms/scoutli |
| Auth Service | http://localhost:8081 |
| Discovery Service | http://localhost:8082 |
| Interaction Service | http://localhost:8083 |
| PostgreSQL | localhost:5433 |

### Test Credentials

| User | Password | Roles |
|------|----------|-------|
| test-user | password | MEMBER |
| admin-user | admin | ADMIN, MEMBER |
| admin (Keycloak) | admin | Admin Console |

### Client Configuration

| Client | Type | Secret |
|--------|------|--------|
| scoutli-backend | Confidential | `secret` |
| scoutli-frontend | Public | N/A |

### Quick Commands

```bash
# Start everything (database + keycloak)
docker-compose up -d scoutli-db && sleep 10 && docker-compose up -d keycloak

# View Keycloak logs
docker logs scoutli-keycloak -f

# Restart with fresh data (CAUTION: deletes all data!)
docker-compose down
docker volume rm microservices_scoutli_db_data
docker-compose up -d scoutli-db && sleep 10 && docker-compose up -d keycloak

# Get token (Bash)
curl -s -X POST "http://localhost:8180/realms/scoutli/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=scoutli-frontend" \
  -d "username=test-user" \
  -d "password=password" | jq -r .access_token
```

---

## ✅ Verification Checklist

- [ ] scoutli-db container running on port 5433
- [ ] Keycloak container running on port 8180
- [ ] `scoutli` realm exists
- [ ] Test users can log in
- [ ] JWT tokens are issued
- [ ] All schemas exist (auth, discovery, interaction, keycloak)
- [ ] Microservices validate tokens

---

## 📁 Files Created/Modified

| File | Action | Purpose |
|------|--------|---------|
| `init-db/init-schemas.sql` | Modified | Added keycloak schema + user |
| `docker-compose.yml` | Modified | Keycloak uses shared scoutli-db |
| `scoutli-realm.json` | Created | Realm configuration |
| `application.properties` | Modified | Added OIDC config (all services) |
| `application-dev.properties` | Created | Dev profile (all services) |
| `application-uat.properties` | Created | UAT profile (all services) |
| `application-prod.properties` | Created | Prod profile (all services) |
| `.env` | Created | Local environment variables |
| `.env.example` | Created | Template for team |
| `pom.xml` | Modified | Added quarkus-oidc dependency |

---

## 🎓 Key Learnings

1. **Shared database with schemas** - One PostgreSQL, multiple schemas for different services
2. **Keycloak schema isolation** - Keycloak data in `keycloak` schema with dedicated user
3. **GitOps for config** - Realm JSON in `scoutli-gitops` repo
4. **Environment variables** - Use `${VAR:default}` pattern
5. **Profile-based config** - Separate files for dev/uat/prod
6. **Auto-import realm** - Use `--import-realm` flag
7. **Fresh import needs empty DB** - Delete volume to re-import

---

**Congratulations!** 🎉 You have successfully integrated Keycloak with your microservices using a shared database!
