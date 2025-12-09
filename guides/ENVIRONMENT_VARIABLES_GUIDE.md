# Environment Variables Guide

## Why `.env` Files?

`.env` files allow you to:
- ✅ Keep secrets out of Git
- ✅ Customize config per developer
- ✅ Change values without editing code
- ✅ Match production environment structure

## File Structure

```
scoutli_java/
├── .gitignore                                    # Ignores .env files
├── microservices/
│   ├── .env.keycloak                            # Keycloak service env vars
│   ├── scoutli-interaction-service/
│   │   ├── .env                                 # Your local secrets (DO NOT COMMIT!)
│   │   └── .env.example                         # Template (commit this!)
│   ├── scoutli-discovery-service/
│   │   ├── .env
│   │   └── .env.example
│   └── scoutli-auth-service/
│       ├── .env
│       └── .env.example
```

## How to Use

### 1. For New Developers

```bash
# Copy the example file
cd microservices/scoutli-interaction-service
cp .env.example .env

# Edit .env with your local values
nano .env
```

### 2. In application.properties

```properties
# Use ${VARIABLE_NAME:default-value} syntax
quarkus.datasource.jdbc.url=jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5434}/${DB_NAME:scoutli_interaction}
quarkus.datasource.username=${DB_USERNAME:scoutli}
quarkus.datasource.password=${DB_PASSWORD:scoutli}

quarkus.oidc.auth-server-url=${KEYCLOAK_URL:http://localhost:8180}/realms/scoutli
quarkus.oidc.client-id=${KEYCLOAK_CLIENT_ID:scoutli-backend}
quarkus.oidc.credentials.secret=${KEYCLOAK_CLIENT_SECRET:secret}
```

### 3. In Docker Compose

```yaml
# docker-compose.yml
services:
  keycloak:
    env_file:
      - .env.keycloak  # Load from file
    environment:
      KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN:-admin}  # Or override here
```

### 4. Running the Service

```bash
# Quarkus automatically reads .env file in the project root
mvn quarkus:dev

# Or explicitly set
export $(cat .env | xargs) && mvn quarkus:dev
```

## Files Created

| File | Purpose | Commit? |
|------|---------|---------|
| `.env` | Your actual secrets | ❌ NO (in .gitignore) |
| `.env.example` | Template for team | ✅ YES |
| `.env.keycloak` | Keycloak Docker config | ⚠️ Optional (dev only) |

## Best Practices

### ✅ DO:
- Keep `.env` out of Git (in `.gitignore`)
- Commit `.env.example` as a template
- Use meaningful variable names
- Document required variables
- Use defaults in `application.properties`

### ❌ DON'T:
- Never commit `.env` to Git
- Don't hardcode production secrets
- Don't share `.env` in Slack/email
- Don't reuse same secrets across envs

## Production vs Development

| Environment | How to Set Variables |
|-------------|---------------------|
| **Local Dev** | `.env` file |
| **Docker Compose** | `.env` or `env_file:` |
| **Kubernetes** | ConfigMap + Secrets |
| **AWS ECS** | Task Definition env vars |
| **Heroku** | `heroku config:set` |

## Example Workflow

```bash
# 1. Clone repo
git clone https://github.com/your-org/scoutli_java.git

# 2. Copy environment template
cd microservices/scoutli-interaction-service
cp .env.example .env

# 3. Edit with your values
nano .env

# 4. Start containers
docker-compose up -d

# 5. Run service (reads .env automatically)
mvn quarkus:dev
```

## Security Notes

- 🔒 `.env` is in `.gitignore` - **verify before committing!**
- 🔒 Never share `.env` files in public channels
- 🔒 Rotate secrets if accidentally committed
- 🔒 Use different secrets for dev/uat/prod

## Your Senior is Right! ✅

This is exactly how modern applications manage environment-specific configuration. Companies like Netflix, Spotify, and Amazon all use this pattern.
