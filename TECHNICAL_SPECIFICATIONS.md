# Detailed Technical Specification - Scoutli Project (v2 - Microservices)

This document details the technical requirements and architecture for the Scoutli project, re-architected as a Microservices application on AWS EKS.

---

## 1. System Architecture

The system follows a **Microservices Architecture** deployed on **AWS EKS**.

### 1.1. Key Components
*   **Frontend:** Single Page Application (SPA) built with **Angular**.
*   **Backend:** Multiple microservices built with **Java (Quarkus)**.
    *   `scoutli-auth-service`: Manages Users, Authentication, JWT.
    *   `scoutli-discovery-service`: Manages Locations/Discoveries, Categories.
    *   `scoutli-interaction-service`: Manages Reviews, Ratings, Comments.
*   **Database:** **PostgreSQL** (AWS RDS) running version 16.6. Separate logical databases/schemas per service.
*   **Container Registry:** **AWS ECR** for storing Docker images.
*   **Infrastructure Orchestration:** **Kubernetes (AWS EKS)** version 1.34.
*   **Infrastructure as Code (IaC):** **Terraform** for provisioning VPC, EKS, RDS, ECR.
*   **CI/CD:**
    *   **CI:** **GitHub Actions** (Build & Push to ECR).
    *   **CD/GitOps:** **ArgoCD** (Syncs from `scoutli-gitops` repo to EKS).
*   **Ingress & SSL:**
    *   **NGINX Ingress Controller:** Single entry point via AWS Network Load Balancer (NLB).
    *   **Cert-Manager:** Automates Let's Encrypt SSL certificates.
    *   **Domains:**
        *   `www.journeywriter.space` -> Frontend
        *   `argocd.journeywriter.space` -> ArgoCD UI

---

## 2. Data Models (PostgreSQL)

### 2.1. Auth Service (`scoutli_auth` database)
- **Table `users`**
  - `id`: `BIGSERIAL` (Long), Primary Key.
  - `email`: `VARCHAR(255)`, Unique, Not Null.
  - `password`: `VARCHAR(255)`, Not Null.
  - `full_name`: `VARCHAR(255)`.
  - `roles`: `VARCHAR(255)` (Comma-separated roles).
  - `created_at`, `updated_at`: `TIMESTAMP`.

### 2.2. Discovery Service (`scoutli_discovery` database)
- **Table `locations`** (formerly `discoveries`)
  - `id`: `BIGSERIAL`, Primary Key.
  - `name`: `VARCHAR(255)`, Not Null.
  - `description`: `TEXT`.
  - `address`: `VARCHAR(255)`.
  - `category_id`: `BIGINT`.
  - `latitude`: `DOUBLE PRECISION`.
  - `longitude`: `DOUBLE PRECISION`.
  - `created_at`, `updated_at`: `TIMESTAMP`.

### 2.3. Interaction Service (`scoutli_interaction` database)
- **Table `reviews`** (To be implemented)
  - `id`, `content`, `rating`, `user_id`, `location_id`, timestamps.

---

## 3. API Specifications (RESTful)

Services communicate via REST. Auth service issues JWTs signed with RS256.

### 3.1. Auth Service (`/api/auth`)
- `POST /api/auth/register`: Register a new user.
  - Body: `{ "email": "...", "password": "...", "fullName": "..." }`
- `POST /api/auth/login`: Login and receive JWT.
  - Body: `{ "email": "...", "password": "..." }`
  - Response: `{ "token": "eyJ...", "userId": 123 }`

### 3.2. Discovery Service (`/api/locations`)
- `GET /api/locations`: List all locations.
- `GET /api/locations/{id}`: Get location details.
- `POST /api/locations`: Create a location.
  - Body: `{ "name": "...", "description": "...", "address": "...", ... }`

---

## 4. Infrastructure & Deployment Details

### 4.1. Network (AWS VPC)
- **VPC:** `10.0.0.0/16` in `ap-southeast-1`.
- **Subnets:**
  - **Public x3:** For Load Balancers (`kubernetes.io/role/elb=1`).
  - **Private x3:** For EKS Nodes & RDS (`kubernetes.io/role/internal-elb=1`).
- **Gateways:** Internet Gateway (IGW) for Public, NAT Gateways (x3) for Private outbound access.

### 4.2. Kubernetes (EKS)
- **Cluster Name:** `scoutli-cluster`.
- **Node Group:** `t3.small` instances (Auto Scaling 1-7 nodes).
- **Namespaces:**
  - `scoutli-app`: Application microservices.
  - `argocd`: CI/CD tooling.
  - `cert-manager`: SSL management.
  - `ingress-nginx`: Ingress Controller.

### 4.3. Security
- **SSL/TLS:** End-to-End HTTPS.
  - **External:** Let's Encrypt (Production) certificates via Cert-Manager ClusterIssuer.
  - **Internal:** Ingress talks to backends via HTTP or HTTPS (ArgoCD).
  - **HSTS:** Enabled on Ingress.
  - **Redirects:** Force SSL Redirect enabled.

### 4.4. GitOps (ArgoCD)
- **Repo:** `scoutli-gitops`.
- **Structure:** "App of Apps" pattern using `root-app.yaml`.
- **Apps Managed:**
  - `scoutli-auth`
  - `scoutli-discovery`
  - `scoutli-interaction` (Planned)
  - `scoutli-frontend`

---

## 5. Development Status (Snapshot)

- **Frontend:** Deployed and Accessible (`https://www.journeywriter.space`).
- **ArgoCD:** Deployed and Accessible (`https://argocd.journeywriter.space`).
- **Backend Auth:** Code implemented, CI build pending deployment.
- **Backend Discovery:** Initial migration/entity created.
- **Infrastructure:** Fully provisioned via Terraform (currently in destroyed state for cost saving).
