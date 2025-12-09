# Keycloak Implementation Guide

## Goal
Implement Keycloak as the Identity and Access Management (IAM) server for the Scoutli microservices architecture. This will handle user authentication (OIDC) and authorization (RBAC).

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│            scoutli-db (PostgreSQL :5433)            │
├─────────────────────────────────────────────────────┤
│ ┌──────┐ ┌─────────┐ ┌───────────┐ ┌────────────┐  │
│ │ auth │ │discovery│ │interaction│ │  keycloak  │  │
│ │schema│ │ schema  │ │  schema   │ │   schema   │  │
│ └──────┘ └─────────┘ └───────────┘ └────────────┘  │
└─────────────────────────────────────────────────────┘
```

> **Note**: We use a **single shared PostgreSQL database** (`scoutli-db`) with separate schemas for each service, including Keycloak.

## Prerequisites
- Docker & Docker Compose
- Java 17+
- Maven

## Step 1: Database Schema Setup

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

-- Grant privileges
GRANT ALL PRIVILEGES ON SCHEMA keycloak TO keycloak;
GRANT ALL PRIVILEGES ON DATABASE scoutli TO keycloak;
```

## Step 2: Infrastructure Setup (Docker)

Add Keycloak to `microservices/docker-compose.yml`.

```yaml
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
  healthcheck:
    test: ["CMD-SHELL", "exec 3<>/dev/tcp/127.0.0.1/8080; echo -e 'GET /health/ready HTTP/1.1\r\nhost: http://localhost\r\nConnection: close\r\n\r\n' >&3; cat <&3 | grep -q '200 OK'"]
    interval: 30s
    timeout: 10s
    retries: 3
```

> **Note**: We map Keycloak to port `8180` because `auth-service` uses `8081`.

## Step 3: Realm & Client Configuration (GitOps)

We follow a GitOps approach by storing the Realm configuration in `scoutli-gitops`.

### 3.1. Realm Configuration File
**File**: `scoutli-gitops/configs/keycloak/scoutli-realm.json`

This file contains the definition for:
- Realm: `scoutli`
- Clients: `scoutli-backend`, `scoutli-frontend`
- Roles: `MEMBER`, `ADMIN`
- Users: `test-user`, `admin-user`

### 3.2. Docker Compose Mount
The `docker-compose.yml` is configured to mount this file for automatic import on startup:
```yaml
volumes:
  - ../scoutli-gitops/configs/keycloak/scoutli-realm.json:/opt/keycloak/data/import/realm.json
command: start-dev --import-realm
```

### 3.3. Clients
| Client | Type | Secret | Purpose |
|--------|------|--------|---------|
| `scoutli-backend` | Confidential | `secret` | Microservices token validation |
| `scoutli-frontend` | Public | N/A | Angular SPA authentication |

### 3.4. Roles
- `MEMBER` - Basic user access
- `ADMIN` - Full administrative access

### 3.5. Test Users
| Username | Password | Roles |
|----------|----------|-------|
| test-user | password | MEMBER |
| admin-user | admin | ADMIN, MEMBER |

## Step 4: Service Integration (Quarkus)

Update `application.properties` in each microservice to validate tokens against Keycloak.

```properties
# OIDC Configuration
quarkus.oidc.auth-server-url=${KEYCLOAK_URL:http://localhost:8180}/realms/scoutli
quarkus.oidc.client-id=${KEYCLOAK_CLIENT_ID:scoutli-backend}
quarkus.oidc.credentials.secret=${KEYCLOAK_CLIENT_SECRET:secret}
quarkus.oidc.application-type=service
quarkus.oidc.token.issuer=${KEYCLOAK_URL:http://localhost:8180}/realms/scoutli

# Security
quarkus.security.enabled=true

# Dev profile (local development)
%dev.quarkus.oidc.tls.verification=none
```

## Step 5: Starting Services

```bash
cd microservices

# Start database first
docker-compose up -d scoutli-db

# Wait for database initialization
sleep 10

# Start Keycloak
docker-compose up -d keycloak
```

## Step 6: Verification

### 6.1 Get Token (PowerShell)
```powershell
$body = @{
    grant_type = "password"
    client_id = "scoutli-frontend"
    username = "test-user"
    password = "password"
}
$response = Invoke-WebRequest -Uri "http://localhost:8180/realms/scoutli/protocol/openid-connect/token" -Method POST -Body $body -UseBasicParsing
$token = ($response.Content | ConvertFrom-Json).access_token
```

### 6.2 Call Protected API
```powershell
$headers = @{ Authorization = "Bearer $token" }
Invoke-WebRequest -Uri "http://localhost:8083/api/discoveries/1/comments" -Headers $headers
```

### 6.3 Expected Results
| Scenario | Result |
|----------|--------|
| Valid token + Correct Role | **200/201 OK** |
| No token | **401 Unauthorized** |
| Invalid Role | **403 Forbidden** |

## Step 7: Admin Console

Access the Keycloak Admin Console:
- **URL**: http://localhost:8180/admin
- **Username**: `admin`
- **Password**: `admin`

## Quick Reference

| Service | URL |
|---------|-----|
| Keycloak Admin | http://localhost:8180/admin |
| Token Endpoint | http://localhost:8180/realms/scoutli/protocol/openid-connect/token |
| OIDC Discovery | http://localhost:8180/realms/scoutli/.well-known/openid-configuration |
