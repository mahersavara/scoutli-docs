# Định nghĩa và Hướng dẫn các Công nghệ trong Workflow (v2)

Provision as Code

Tài liệu này giải thích định nghĩa, vai trò và cách sử dụng cơ bản của từng công cụ trong kiến trúc hạ tầng và CI/CD của dự án.

---

### 1. Terraform

- **Định nghĩa:** Một công cụ "Infrastructure as Code" (IaC) để định nghĩa và cung cấp hạ tầng dưới dạng code.
- **Vai trò trong dự án:** Viết "bản thiết kế" cho toàn bộ hạ tầng trên AWS (VPC, EKS...). 
- **Ví dụ Cách sử dụng (Workflow cơ bản):**
  1.  `terraform init`: Khởi tạo thư mục làm việc, tải về các plugin cần thiết (provider cho AWS).
  2.  `terraform plan`: Xem trước những thay đổi sẽ được áp dụng. Lệnh này an toàn, không tạo ra thay đổi thực tế.
  3.  `terraform apply`: Áp dụng các thay đổi, tạo/cập nhật hạ tầng trên AWS.

---

### 2. AWS EKS (Elastic Kubernetes Service)

- **Định nghĩa:** Dịch vụ Kubernetes được quản lý bởi Amazon.
- **Vai trò trong dự án:** Là "ngôi nhà" nơi ứng dụng của chúng ta sẽ chạy.
- **Ví dụ Cách sử dụng (Định nghĩa bằng Terraform):** Chúng ta không thao tác trực tiếp với EKS mà sẽ định nghĩa nó trong file `.tf`.
  ```terraform
  # Ví dụ một đoạn trong file main.tf để tạo EKS
  resource "aws_eks_cluster" "scoutli_cluster" {
    name     = "scoutli-cluster"
    role_arn = aws_iam_role.eks_cluster_role.arn # Vai trò IAM cho EKS

    vpc_config {
      subnet_ids = [ "id-subnet-1", "id-subnet-2" ] # Sử dụng các subnet đã tạo ở bước trước
    }
  }
  ```

---

### 3. Docker, Kaniko và AWS ECR

- **Docker:**
  - **Định nghĩa:** Nền tảng để tạo và chạy "container".
  - **Vai trò:** Đóng gói ứng dụng (Quarkus, Angular) thành các **Docker image**.
  - **Ví dụ Cách sử dụng:**
    - `docker build -t scoutli-backend:v1 .`: Build image từ Dockerfile và đặt tên là `scoutli-backend` phiên bản `v1`.
    - `docker run -p 8080:8080 scoutli-backend:v1`: Chạy image vừa build trên máy local để test.

- **AWS ECR (Elastic Container Registry):**
  - **Định nghĩa:** Kho chứa Docker image riêng tư của AWS.
  - **Vai trò:** Lưu trữ các image của ứng dụng một cách an toàn.
  - **Ví dụ Cách sử dụng (Đẩy image lên ECR):**
    ```bash
    # 1. Lấy lệnh đăng nhập từ AWS CLI
    aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com

    # 2. Gắn tag cho image theo định dạng của ECR
    docker tag scoutli-backend:v1 <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/scoutli-backend:v1

    # 3. Đẩy image lên ECR
    docker push <ACCOUNT_ID>.dkr.ecr.ap-southeast-1.amazonaws.com/scoutli-backend:v1
    ```

- **Kaniko:**
  - **Định nghĩa:** Công cụ build Docker image bên trong một container khác, không cần Docker daemon.
  - **Vai trò:** Build image một cách an toàn trong môi trường CI/CD.
  - **Ví dụ Cách sử dụng (Trong file `.gitlab-ci.yml`):**
    ```yaml
    build_backend_image:
      stage: build
      image: gcr.io/kaniko-project/executor:debug # Sử dụng image của Kaniko
      script:
        - echo "{"credsStore":"ecr-login"}" > /kaniko/.docker/config.json # Cấu hình đăng nhập ECR
        - /kaniko/executor
            --context $CI_PROJECT_DIR/backend
            --dockerfile $CI_PROJECT_DIR/backend/Dockerfile
            --destination $ECR_REGISTRY/scoutli-backend:$CI_COMMIT_SHA
    ```

---

### 4. Helm và ArgoCD

- **Helm:**
  - **Định nghĩa:** Trình quản lý gói cho Kubernetes.
  - **Vai trò:** Đóng gói toàn bộ cấu hình K8s của ứng dụng vào một **Chart**.
  - **Ví dụ Cách sử dụng:**
    - `helm install scoutli-release ./charts/scoutli`: Cài đặt ứng dụng từ chart trong thư mục `./charts/scoutli`.
    - `helm upgrade scoutli-release ./charts/scoutli --set backend.image.tag=v2`: Nâng cấp ứng dụng lên phiên bản `v2`.
    - `helm list`: Xem các ứng dụng đã được cài đặt bằng Helm.

- **ArgoCD:**
  - **Định nghĩa:** Công cụ Continuous Delivery theo triết lý GitOps.
  - **Vai trò:** Tự động đồng bộ hóa trạng thái của ứng dụng trên EKS với cấu hình trong Git.
  - **Ví dụ Cách sử dụng (Định nghĩa một `Application` trong file YAML):**
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: scoutli-production
    spec:
      project: default
      source:
        repoURL: 'https://github.com/your-org/scoutli-gitops.git' # Link tới repo chứa cấu hình
        path: 'apps/scoutli' # Đường dẫn tới Helm chart trong repo
        targetRevision: HEAD
      destination:
        server: 'https://kubernetes.default.svc'
        namespace: 'production'
      syncPolicy:
        automated: { prune: true, selfHeal: true } # Tự động đồng bộ
    ```

---

### 5. Ingress Controller và Cert-Manager

- **Ingress Controller:**
  - **Định nghĩa:** Cổng kiểm soát giao thông vào cluster K8s.
  - **Vai trò:** Điều hướng traffic từ domain `scoutli.example.com` vào đúng service backend hoặc frontend.
- **Cert-Manager:**
  - **Định nghĩa:** Add-on tự động quản lý chứng chỉ SSL/TLS.
  - **Vai trò:** Tự động lấy chứng chỉ HTTPS cho domain của chúng ta.
- **Ví dụ Cách sử dụng (Định nghĩa `Ingress` trong file YAML):**
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: scoutli-ingress
    annotations:
      # Yêu cầu cert-manager tạo chứng chỉ cho domain này
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
  spec:
    tls:
    - hosts:
      - scoutli.example.com
      secretName: scoutli-tls-secret # Tên của Secret chứa chứng chỉ sẽ được tạo ra
    rules:
    - host: scoutli.example.com
      http:
        paths:
        - path: /api
          pathType: Prefix
          backend:
            service:
              name: scoutli-backend-service
              port: { number: 8080 }
        - path: /
          pathType: Prefix
          backend:
            service:
              name: scoutli-frontend-service
              port: { number: 80 }
  ```