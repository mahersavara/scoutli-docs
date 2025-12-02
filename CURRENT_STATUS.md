# Current Project Status (as of Nov 28, 2025)

**Project Hierarchy:**
- AWS Cloud (ap-southeast-1)
  - VPC (scoutli-vpc)
    - EKS Cluster (scoutli-cluster)
      - Nodes (5x t3.small) - Managed by Auto Scaling Group
      - NGINX Ingress Controller (Namespace: ingress-nginx)
        - Service: LoadBalancer (NLB) - *Active Entry Point*
        - Ingress Rules:
          - `www.journeywriter.space` -> `scoutli-frontend` (HTTP -> HTTPS redirect)
          - `argocd.journeywriter.space` -> `argocd-server` (HTTP -> HTTPS redirect)
      - Cert-Manager (Namespace: cert-manager)
        - ClusterIssuer: `letsencrypt-prod`
        - Certificates: `frontend-tls-secret`, `argocd-tls-secret` (Status: Ready)
      - ArgoCD (Namespace: argocd)
        - Service: `argocd-server` (ClusterIP)
        - ConfigMap: `argocd-cm` (url: https://argocd.journeywriter.space)
        - Applications: `scoutli-frontend` (Healthy), `scoutli-auth` (Pending deployment)
      - Frontend (Namespace: scoutli-app)
        - Deployment: `scoutli-frontend`
        - Service: `scoutli-frontend-scoutli-microservice`
  - RDS (PostgreSQL 16.6)
    - Instance: `scoutli-rds-microservices`
    - Databases: `scoutli_auth`, `scoutli_discovery`, `scoutli_interaction`
  - ECR (Container Registry)
    - Repos: `scoutli-auth-service`, `scoutli-discovery-service`, `scoutli-interaction-service`, `scoutli-frontend`

**Detailed File System Overview:**

### Root Directory (`scoutli_java/`)
- `.gitignore`: Specifies files and directories to be ignored by Git version control.
- `CLAUDE.md`: Contains specific instructions and context for the AI assistant (Claude/Gemini) to understand the project conventions.
- `CLEANUP_CHECKLIST.md`: Checklist for manually cleaning up AWS resources.
- `CURRENT_STATUS.md`: This file. A comprehensive snapshot of the project's current state, architecture, and file structure.
- `KAFKA_IMPLEMENTATION_GUIDE.md`: Guide for implementing Kafka in the project.
- `KAFKA_QUARKUS_GUIDE.md`: Specific guide for using Kafka with Quarkus.
- `PRD.md`: Product Requirements Document. Outlines the business goals and features of the Scoutli application.
- `temp_values.yaml`: A temporary Helm values file generated during the session to apply configurations (like ClusterIP and labels) to ArgoCD during upgrade.
- `patch-argocd-cm.yaml`: A temporary Kubernetes patch file used to update the `argocd-cm` ConfigMap with the correct URL and rootpath.
- `install_ingress.ps1`: PowerShell script to automate the installation of the NGINX Ingress Controller using Helm.
- `remove-old-backend.ps1`: Utility PowerShell script to clean up resources from a previous monolithic backend deployment.
- `get-docker.sh`: Shell script to install Docker (likely for a Linux environment, though working on Windows).
- `feature_commit.md`: A template or guide for writing standardized commit messages for features.
- `current_work_flow.md`: Documentation describing the current development workflow.
- `current_work_flow.json`: JSON representation of the workflow state.
- `mermaid_erd.md`: Contains the Entity Relationship Diagram (ERD) code for the database schema, usable with Mermaid.js.
- `scoutli_cicd_quarkus.md`: Documentation outlining the CI/CD pipeline strategy for Quarkus microservices.
- `.claude/settings.local.json`: Local settings configuration for the AI assistant.

### Backend Microservices (`microservices/`)
- `docker-compose.yml`: Docker Compose configuration to spin up local development dependencies like PostgreSQL and Kafka.
- `scoutli_api_collection_full.json`: Exported API collection (Postman/Insomnia) for testing endpoints.

#### `scoutli-auth-service/` (Authentication Service)
- `.github/workflows/build.yml`: GitHub Actions workflow definition. Triggers on push to master, builds the Quarkus Native image using Docker, and pushes it to AWS ECR.
- `.gitignore`: Git ignore rules specific to this service.
- `pom.xml`: Maven Project Object Model. Defines dependencies (Quarkus, Hibernate Panache, Postgres JDBC, SmallRye JWT, Flyway), build plugins, and project version.
- `src/main/docker/Dockerfile.native`: Dockerfile used to build the production-ready Quarkus Native executable container image.
- `src/main/resources/application.properties`: Main configuration file. Defines DB connection (using env vars injected by K8s), HTTP port (8080), log levels, and Flyway settings.
- `src/main/resources/db/migration/V1.0.0__Initial_Schema.sql`: (Legacy) Initial schema script (likely empty or replaced).
- `src/main/resources/db/migration/V1.0.0__Create_User_Table.sql`: Flyway migration script that creates the `users` table in the database.
- `src/main/java/com/scoutli/auth/entity/User.java`: JPA Entity class representing the `users` table. Maps DB columns (id, email, password, roles) to Java fields.
- `src/main/java/com/scoutli/auth/dto/AuthRequest.java`: Data Transfer Object for handling Login and Register request bodies (JSON).
- `src/main/java/com/scoutli/auth/dto/AuthResponse.java`: Data Transfer Object for the API response, containing the JWT token and User ID.
- `src/main/java/com/scoutli/auth/service/TokenService.java`: Service class responsible for generating signed JWTs (using SmallRye JWT) for authenticated users.
- `src/main/java/com/scoutli/auth/resource/AuthResource.java`: JAX-RS Resource class. Defines the REST API endpoints (`/api/auth/register`, `/api/auth/login`) and handles incoming HTTP requests.
- **Legacy/Previous Implementation Files:**
    - `src/main/java/com/scoutli/api/controller/AuthController.java`: Older Controller class for handling auth requests.
    - `src/main/java/com/scoutli/api/dto/AuthDTO.java`: Older DTO for authentication data.
    - `src/main/java/com/scoutli/api/dto/UserDTO.java`: Older DTO for user data transfer.
    - `src/main/java/com/scoutli/domain/entity/User.java`: Older User Entity class.
    - `src/main/java/com/scoutli/domain/repository/UserRepository.java`: Repository interface for User entity.
    - `src/main/java/com/scoutli/event/UserRegisteredEvent.java`: Event class for domain events.
    - `src/main/java/com/scoutli/service/AuthService.java`: Older Service class containing business logic for authentication.

#### `scoutli-discovery-service/` (Discovery Service)
- `.github/workflows/build.yml`: GitHub Actions workflow for Discovery Service CI.
- `.gitignore`: Git ignore rules.
- `pom.xml`: Maven configuration. Dependencies include Quarkus, Hibernate, Postgres, Flyway.
- `src/main/docker/Dockerfile.native`: Dockerfile for Native image build.
- `src/main/resources/application.properties`: Quarkus configuration (DB connection, Flyway).
- `src/main/resources/db/migration/V1.0.0__Initial_Schema.sql`: (Legacy) Initial schema.
- `src/main/resources/db/migration/V1.0.0__Create_Locations_Table.sql`: Flyway migration script to create the `locations` table.
- `src/main/java/com/scoutli/discovery/entity/Location.java`: JPA Entity representing the `locations` table.
- `src/main/java/com/scoutli/discovery/resource/LocationResource.java`: JAX-RS Resource defining REST endpoints (`/api/locations`) for CRUD operations on locations.
- **Legacy/Previous Implementation Files:**
    - `src/main/java/com/scoutli/api/controller/DiscoveryController.java`: Older Controller for handling discovery/location requests.
    - `src/main/java/com/scoutli/api/dto/DiscoveryDTO.java`: Older DTO for transferring discovery data.
    - `src/main/java/com/scoutli/api/dto/UserDTO.java`: Older DTO for user info.
    - `src/main/java/com/scoutli/client/AuthServiceRestClient.java`: REST Client interface for communicating with the Auth Service.
    - `src/main/java/com/scoutli/domain/entity/Discovery.java`: Older Entity class for 'Discovery'.
    - `src/main/java/com/scoutli/domain/entity/Tag.java`: Entity class for 'Tag'.
    - `src/main/java/com/scoutli/domain/repository/DiscoveryRepository.java`: Repository for Discovery entities.
    - `src/main/java/com/scoutli/domain/repository/TagRepository.java`: Repository for Tag entities.
    - `src/main/java/com/scoutli/event/DiscoveryCreatedEvent.java`: Event class for when a discovery is created.
    - `src/main/java/com/scoutli/opensearch/model/DiscoveryDocument.java`: Model for indexing discoveries in OpenSearch.
    - `src/main/java/com/scoutli/opensearch/service/DiscoveryIndexerService.java`: Service to handle indexing data into OpenSearch.
    - `src/main/java/com/scoutli/service/DiscoveryService.java`: Service containing business logic for discoveries.
    - `src/main/java/com/scoutli/service/GeocodingService.java`: Service for geocoding addresses.

#### `scoutli-interaction-service/` (Interaction Service - Pending Implementation)
- `.github/workflows/build.yml`: CI workflow.
- `.gitignore`: Git ignore rules.
- `pom.xml`: Maven configuration.
- `src/main/docker/Dockerfile.native`: Dockerfile.
- `src/main/resources/application.properties`: Config file.
- `src/main/resources/db/migration/V1.0.0__Initial_Schema.sql`: Legacy schema.
- **Legacy/Previous Implementation Files:**
    - `src/main/java/com/scoutli/api/controller/CommentController.java`: Controller for handling comments.
    - `src/main/java/com/scoutli/api/controller/RatingController.java`: Controller for handling ratings.
    - `src/main/java/com/scoutli/api/dto/CommentDTO.java`: DTO for comment data.
    - `src/main/java/com/scoutli/api/dto/RatingDTO.java`: DTO for rating data.
    - `src/main/java/com/scoutli/api/dto/UserDTO.java`: DTO for user context.
    - `src/main/java/com/scoutli/client/AuthServiceRestClient.java`: Client to talk to Auth Service.
    - `src/main/java/com/scoutli/domain/entity/Comment.java`: Entity representing a Comment.
    - `src/main/java/com/scoutli/domain/entity/Rating.java`: Entity representing a Rating.
    - `src/main/java/com/scoutli/domain/repository/CommentRepository.java`: Repository for Comment entity.
    - `src/main/java/com/scoutli/domain/repository/RatingRepository.java`: Repository for Rating entity.
    - `src/main/java/com/scoutli/service/CommentService.java`: Business logic for comments.
    - `src/main/java/com/scoutli/service/RatingService.java`: Business logic for ratings.

### Frontend Application (`scoutli-frontend/`)
- `.github/workflows/build.yml`: GitHub Actions CI workflow.
- `.gitignore`: Git ignore rules.
- `Dockerfile`: Multi-stage Dockerfile.
- `angular.json`: Angular CLI configuration file.
- `package.json`: Defines Node.js dependencies.
- `package-lock.json`: Locked versions of npm dependencies.
- `tsconfig.json`: Main TypeScript config.
- `tsconfig.app.json`: TypeScript config for application code.
- `tsconfig.spec.json`: TypeScript config for tests.
- `README.md`: Frontend documentation.
- `public/favicon.ico`: Website favicon.
- `src/index.html`: Main HTML entry point.
- `src/main.ts`: Application entry point script.
- `src/styles.scss`: Global CSS/SCSS styles.
- `src/app/app.component.ts`: Root application component logic.
- `src/app/app.html`: Root component template.
- `src/app/app.scss`: Root component styles.
- `src/app/app.spec.ts`: Tests for root component.
- `src/app/app.ts`: (Possible legacy/alias file).
- `src/app/app.config.ts`: Application configuration (providers).
- `src/app/app.routes.ts`: Router configuration.
- `src/app/core/services/auth.service.ts`: Service for handling authentication logic.
- `src/app/core/services/auth.spec.ts`: Tests for auth service.
- `src/app/core/services/auth.ts`: (Possible legacy/alias file).
- `src/app/core/services/discovery.service.ts`: Service for fetching location data.
- `src/app/core/services/discovery.spec.ts`: Tests for discovery service.
- `src/app/core/services/discovery.ts`: (Possible legacy/alias file).
- `src/app/features/auth/login/login.component.ts`: Login component logic.
- `src/app/features/auth/login/login.html`: Login component template.
- `src/app/features/auth/login/login.scss`: Login component styles.
- `src/app/features/auth/login/login.spec.ts`: Login component tests.
- `src/app/features/auth/login/login.ts`: (Possible legacy/alias file).
- `src/app/features/auth/register/register.component.ts`: Register component logic.
- `src/app/features/auth/register/register.html`: Register component template.
- `src/app/features/auth/register/register.scss`: Register component styles.
- `src/app/features/auth/register/register.spec.ts`: Register component tests.
- `src/app/features/auth/register/register.ts`: (Possible legacy/alias file).
- `src/app/features/auth/state/auth.model.ts`: Auth state model.
- `src/app/features/auth/state/auth.query.ts`: Auth state query.
- `src/app/features/auth/state/auth.service.ts`: Auth state service.
- `src/app/features/auth/state/auth.store.ts`: Auth state store.
- `src/app/features/home/home.component.ts`: Home page component logic.
- `src/app/features/home/home.html`: Home page component template.
- `src/app/features/home/home.scss`: Home page component styles.
- `src/app/features/home/home.spec.ts`: Home page component tests.
- `src/app/features/home/home.ts`: (Possible legacy/alias file).
- `src/app/shared/components/footer/footer.html`: Footer template.
- `src/app/shared/components/footer/footer.scss`: Footer styles.
- `src/app/shared/components/footer/footer.spec.ts`: Footer tests.
- `src/app/shared/components/footer/footer.ts`: Footer logic.
- `src/app/shared/components/map/map.component.ts`: Map component logic.
- `src/app/shared/components/map/map.html`: Map component template.
- `src/app/shared/components/map/map.scss`: Map component styles.
- `src/app/shared/components/map/map.spec.ts`: Map component tests.
- `src/app/shared/components/map/map.ts`: (Possible legacy/alias file).
- `src/app/shared/components/navbar/navbar.component.ts`: Navbar component logic.
- `src/app/shared/components/navbar/navbar.html`: Navbar component template.
- `src/app/shared/components/navbar/navbar.scss`: Navbar component styles.
- `src/app/shared/components/navbar/navbar.spec.ts`: Navbar component tests.
- `src/app/shared/components/navbar/navbar.ts`: (Possible legacy/alias file).

### GitOps Configuration (`scoutli-gitops/`)
- `README.md`: GitOps repository documentation.
- `root-app.yaml`: The "App of Apps" ArgoCD Application manifest.
- `bootstrap/root-app.yaml`: Bootstrap copy of the root app.
- `bootstrap/cluster-issuer.yaml`: Cert-Manager ClusterIssuer resource defining Let's Encrypt Production.
- `bootstrap/db-init-job.yaml`: Kubernetes Job manifest to initialize the logical databases.
- `bootstrap/db-secret.yaml`: Template for the Kubernetes Secret containing database credentials.
- `apps/ingress-argocd.yaml`: Ingress resource for ArgoCD UI (TLS enabled, HSTS, Backend Protocol HTTPS).
- `apps/ingress-frontend.yaml`: Ingress resource for the Frontend (TLS enabled).
- `apps/scoutli-auth.yaml`: ArgoCD Application manifest for the Auth Service.
- `apps/scoutli-discovery.yaml`: ArgoCD Application manifest for the Discovery Service.
- `apps/scoutli-frontend.yaml`: ArgoCD Application manifest for the Frontend.
- `apps/scoutli-interaction.yaml`: ArgoCD Application manifest for the Interaction Service.
- `envs/dev/backend-values.yaml`: Environment-specific Helm values overrides for backend.
- `envs/dev/frontend-values.yaml`: Environment-specific Helm values overrides for frontend.

### Helm Charts (`scoutli-helm/`)
- `charts/scoutli-microservice/Chart.yaml`: Generic Helm chart metadata.
- `charts/scoutli-microservice/values.yaml`: Default values for the generic chart.
- `charts/scoutli-microservice/templates/deployment.yaml`: Deployment template.
- `charts/scoutli-microservice/templates/service.yaml`: Service template.
- `charts/scoutli-microservice/templates/ingress.yaml`: Ingress template.
- `charts/scoutli-microservice/templates/_helpers.tpl`: Helm template helpers.
- `scoutli-app/Chart.yaml`: App Helm chart metadata (older).
- `scoutli-app/values.yaml`: Default values for app chart.
- `scoutli-app/templates/deployment.yaml`: Deployment template.
- `scoutli-app/templates/service.yaml`: Service template.
- `scoutli-app/templates/ingress.yaml`: Ingress template.
- `scoutli-app/templates/_helpers.tpl`: Helm template helpers.

### Infrastructure (`scoutli-infra/`)
- `README.md`: Infrastructure documentation.
- `.gitignore`: Terraform ignore rules.
- `addons/main.tf`: Provisions ArgoCD and AWS Load Balancer Controller via Helm.
- `addons/variables.tf`: Input variables for addons.
- `addons/iam_policy.json`: IAM policy for LB Controller.
- `addons/.terraform.lock.hcl`: Provider lock file.
- `vpc/main.tf`: Defines AWS VPC, Public Subnets, Private Subnets, Internet Gateway, NAT Gateways, Route Tables.
- `vpc/outputs.tf`: Outputs VPC ID and Subnet IDs.
- `vpc/variables.tf`: Input variables (CIDR blocks, regions).
- `vpc/.terraform.lock.hcl`: Provider lock file.
- `eks/main.tf`: Defines EKS Cluster, Node Groups (EC2), IAM Roles for Cluster and Nodes.
- `eks/outputs.tf`: Outputs Cluster Endpoint and Cert Authority.
- `eks/variables.tf`: Input variables.
- `eks/.terraform.lock.hcl`: Provider lock file.
- `rds/main.tf`: Defines RDS Postgres Instance, Security Group, Subnet Group.
- `rds/outputs.tf`: Outputs DB Endpoint.
- `rds/variables.tf`: Input variables (DB name, credentials).
- `rds/.terraform.lock.hcl`: Provider lock file.
- `ecr/main.tf`: Defines ECR repositories for all services.
- `ecr/outputs.tf`: Outputs ECR Repository URLs.
- `ecr/variables.tf`: Input variables.
- `ecr/tfplan`: Terraform execution plan file.
- `ecr/.terraform.lock.hcl`: Provider lock file.
- `dns/main.tf`: Terraform module for DNS (Route53) - largely unused.
- `dns/outputs.tf`: Outputs for DNS.
- `dns/variables.tf`: Variables for DNS.
- `dns/.terraform.lock.hcl`: Provider lock file.

### Documentation (`scoutli-docs/`)
- `TECHNICAL_SPECIFICATIONS.md`: The main technical spec document, updated with the latest architecture.
- `BACKEND_IMPLEMENTATION_PLAN.md`: Detailed backend plan.
- `FRONTEND_IMPLEMENTATION_PLAN.md`: Detailed frontend plan.
- `INFRASTRUCTURE_CICD_PLAN.md`: Infrastructure and CICD plan.
- `MOBILE_APP_IMPLEMENTATION_PLAN.md`: Mobile app plan.
- `BUSINESS_REQUIREMENTS.md`: Business logic requirements.
- `PRD.md`: Product Requirements Document.
- `GUIDELINE_AND_RETROSPECTIVE.md`: Project guidelines.
- `TECHNOLOGY_STACK_ANALYSIS.md`: Analysis of tech choices.
- `TODO.md`: Task tracking.
- `TOOL_DEFINITIONS_AND_GUIDE.md`: Guide for tools used.
- `project_overview.md`: High-level overview.

### AI Prompts (`scoutli_promts/`)
- `README.md`: Description of prompts.
- `setup_tool_paths.md`: Prompt to setup environment paths.
- `sync_and_compare.md`: Prompt to sync git repositories.
- `summarize_project_spec.md`: Prompt used to generate this summary.
- `update_project_spec_from_chat.md`: Prompt to update specs from chat history.

### Tools (`tools/`)
- `helm.exe`: Helm CLI binary.
- `kubectl.exe`: Kubernetes CLI binary.
- `argocd.exe`: ArgoCD CLI binary.

---

**Project Situation Summary:**

The Scoutli project has successfully transitioned to a microservices architecture on AWS EKS.
*   **Infrastructure:** A production-grade EKS cluster (v1.34) is running with 5 worker nodes and a PostgreSQL RDS instance.
*   **Access & Security:** NGINX Ingress Controller is serving traffic via a single public Network Load Balancer (NLB). `cert-manager` is installed and has successfully issued valid Let's Encrypt certificates for both `www.journeywriter.space` and `argocd.journeywriter.space`.
*   **Frontend:** The Angular frontend is deployed and accessible at `https://www.journeywriter.space`.
*   **CI/CD:** GitHub Actions builds images (Frontend built). ArgoCD is accessible at `https://argocd.journeywriter.space` (login: admin) and manages deployments.
*   **Backend:**
    *   `scoutli-auth-service`: Code implemented (Entity, API, TokenService) and pushed. CI build pending/in-progress.
    *   `scoutli-discovery-service`: Skeleton and Location entity/migration created. Pending commit.

**Next Steps:**
1.  **Commit & Deploy Backend:** Commit `scoutli-discovery-service`. Ensure `scoutli-auth-service` image builds and deploys via ArgoCD.
2.  **Verify APIs:** Test the `/api/auth/register` and `/api/locations` endpoints.
3.  **Implement Interaction Service:** Create `scoutli-interaction-service` (Reviews/Ratings).
