# Keycloak Implementation Guide

## Goal
Implement Keycloak as the Identity and Access Management (IAM) server for the Scoutli microservices architecture. This will handle user authentication (OIDC) and authorization (RBAC).

## Prerequisites
- Docker & Docker Compose
- Java 17+
- Maven

## Step 1: Infrastructure Setup (Docker)
Add Keycloak to `microservices/docker-compose.yml`.

```yaml
  keycloak:
    image: quay.io/keycloak/keycloak:23.0.0
    container_name: scoutli-keycloak
    command: start-dev
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://scoutli-db:5432/scoutli
      KC_DB_USERNAME: scoutli
      KC_DB_PASSWORD: scoutli
      KC_DB_SCHEMA: keycloak
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8180:8080" # Port 8180 to avoid conflict with auth-service (8080)
    depends_on:
      - scoutli-db
    networks:
      - scoutli-net
```

> **Note**: We map Keycloak to port `8180` because `auth-service` is currently using `8080`.

## Step 2: Database Configuration
Ensure the `keycloak` schema exists in the shared Postgres database or create a separate database.
*   **Action**: Update `init-db` scripts or manually create schema.

## Step 3: Realm & Client Configuration
Configure Keycloak with the necessary Realm, Clients, and Roles.

### 3.1. Create Realm
- **Name**: `scoutli`

### 3.2. Create Clients
1.  **Backend Services** (`interaction-service`, `discovery-service`):
    - Client ID: `scoutli-backend`
    - Client Protocol: `openid-connect`
    - Access Type: `confidential` (Service Accounts enabled) or `bearer-only` (if only validating tokens).
2.  **Frontend/Postman**:
    - Client ID: `scoutli-frontend`
    - Access Type: `public`
    - Valid Redirect URIs: `http://localhost/*`

### 3.3. Create Roles
- `MEMBER`
- `ADMIN`

### 3.4. Create Users
- Create test users and assign roles.

## Step 4: Service Integration (Quarkus)
Update `application.properties` in each microservice (`interaction-service`, etc.) to validate tokens against Keycloak.

```properties
# OIDC Configuration
quarkus.oidc.auth-server-url=http://localhost:8180/realms/scoutli
quarkus.oidc.client-id=scoutli-backend
quarkus.oidc.credentials.secret=secret
quarkus.oidc.tls.verification=none # For local dev only
```

## Step 5: Verification
1.  **Get Token**: Use Postman to get an access token from Keycloak (`http://localhost:8180/realms/scoutli/protocol/openid-connect/token`).
2.  **Call API**: Send a request to a secured endpoint (e.g., `POST /api/discoveries/1/comments`) with the `Authorization: Bearer <token>` header.
3.  **Verify Access**:
    - Valid token + Correct Role -> **200/201 OK**
    - No token -> **401 Unauthorized**
    - Invalid Role -> **403 Forbidden**

## Step 6: Migration (Optional)
If `auth-service` was handling auth, plan the migration logic or deprecation.
