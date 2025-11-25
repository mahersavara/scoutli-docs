# Hướng dẫn Triển khai & Nhật ký Dự án Scoutli (New Stack)

Tài liệu này tổng hợp quy trình triển khai hạ tầng AWS EKS, CI/CD với GitHub Actions, và cài đặt ArgoCD, cũng như các bài học kinh nghiệm từ quá trình thực hiện.

---

## 1. Guideline (Quy trình Triển khai Chuẩn)

### Giai đoạn 1: Chuẩn bị Môi trường (Local)

1.  **Cài đặt công cụ:**
    *   WSL2 (Ubuntu/Debian).
    *   Docker Desktop (Tích hợp WSL2).
    *   Terraform.
    *   AWS CLI.
    *   GitHub CLI (`gh`).
    *   `kubectl`.
    *   `helm`.

2.  **Cấu hình AWS CLI:**
    *   Tạo IAM User có quyền AdministratorAccess.
    *   Chạy `aws configure` và nhập Access Key ID, Secret Access Key, Region (`ap-southeast-1`).

### Giai đoạn 2: Khởi tạo Cấu trúc Multi-Repo

1.  **Tạo thư mục gốc:** `scoutli-project`.
2.  **Tạo các thư mục con (Repos):**
    *   `scoutli-infra`: Chứa code Terraform.
    *   `scoutli-backend`: Chứa code Quarkus.
    *   `scoutli-frontend`: Chứa code Angular.
    *   `scoutli-helm`: Chứa Helm Charts.
    *   `scoutli-gitops`: Chứa ArgoCD App manifests.
    *   `scoutli-docs`: Chứa tài liệu.
3.  **Đẩy lên GitHub:** Sử dụng `gh repo create` để tạo repo và push code.

### Giai đoạn 3: Dựng Hạ tầng AWS (Terraform)

**Bước 1: Tạo VPC (Mạng)**
*   Vào thư mục `scoutli-infra/vpc`.
*   Chạy `terraform init`.
*   Chạy `terraform apply -auto-approve`.
*   **Kết quả:** VPC, 2 Public Subnets, 2 Private Subnets, NAT Gateways.

**Bước 2: Tạo EKS Cluster (Kubernetes)**
*   Vào thư mục `scoutli-infra/eks`.
*   Cập nhật `terraform.tfvars` với `vpc_id` và `private_subnet_ids` từ Bước 1.
*   **Cấu hình Node:** `node_instance_type = "t3.small"` (hoặc lớn hơn).
*   Chạy `terraform init`.
*   Chạy `terraform apply -auto-approve`.
*   **Kết quả:** EKS Cluster, Node Group.

**Bước 3: Tạo ECR (Container Registry)**
*   Vào thư mục `scoutli-infra/ecr`.
*   Chạy `terraform init`.
*   Chạy `terraform apply -auto-approve`.
*   **Kết quả:** 2 ECR Repositories (backend, frontend).

### Giai đoạn 4: Thiết lập CI (GitHub Actions)

1.  **Backend (Quarkus):**
    *   Tạo `Dockerfile.native`.
    *   Tạo `.github/workflows/build.yml` (Build Native -> Push ECR).
    *   **Manual:** Thêm Secrets (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`) vào GitHub Repo.

2.  **Frontend (Angular):**
    *   Tạo `Dockerfile` (Multi-stage: Node -> Nginx).
    *   Tạo `.github/workflows/build.yml` (Build -> Push ECR).
    *   **Manual:** Thêm Secrets vào GitHub Repo.

### Giai đoạn 5: Cài đặt ArgoCD (Terraform Addons)

1.  **Cập nhật Kubeconfig:**
    *   Chạy lệnh: `aws eks update-kubeconfig --region ap-southeast-1 --name scoutli-cluster`.
2.  **Cấu hình Terraform:**
    *   Sử dụng `provider "helm"` phiên bản 2.x (`version = ">= 2.0, < 3.0"`).
    *   Sử dụng Data Source (`aws_eks_cluster`, `aws_eks_cluster_auth`) để lấy token xác thực.
    *   Truyền thông tin xác thực trực tiếp vào block `kubernetes {}` bên trong provider `helm`.
3.  **Chạy Terraform:**
    *   Vào thư mục `scoutli-infra/addons`.
    *   Chạy `terraform init`.
    *   Chạy `terraform apply -auto-approve`.

---

## 2. Các thao tác Thủ công (Manual Actions)

Người dùng cần nhập tay các lệnh sau trong quá trình thực hiện:

1.  **AWS CLI Config:**
    ```bash
    aws configure
    ```

2.  **Cập nhật Kubeconfig (Bắt buộc trước khi cài ArgoCD):**
    ```powershell
    aws eks update-kubeconfig --region ap-southeast-1 --name scoutli-cluster
    ```

3.  **Set biến môi trường KUBECONFIG (Trên Windows PowerShell):**
    ```powershell
    $env:KUBECONFIG = "$env:USERPROFILE\.kube\config"
    ```

4.  **Thêm Secrets vào GitHub Repo:**
    *   Vào `Settings -> Secrets and variables -> Actions` trên GitHub.
    *   Thêm `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`.

---

## 3. Troubleshooting (Nhật ký Lỗi & Khắc phục)

### Lỗi 1: `SSL: CERTIFICATE_VERIFY_FAILED` khi chạy AWS CLI
*   **Nguyên nhân:** Mạng công ty sử dụng Proxy/Firewall chặn SSL và thay thế bằng certificate riêng.
*   **Khắc phục:**
    *   Hỏi IT lấy file CA Bundle (`.pem`).
    *   Cấu hình AWS CLI: `aws configure set default.ca_bundle /path/to/cert.pem`.
    *   (Tạm thời - Không khuyến khích): Dùng cờ `--no-verify-ssl`.

### Lỗi 2: `Unsupported block type "kubernetes"` trong Terraform Helm Provider
*   **Nguyên nhân:** Provider Helm v3.0.0+ đã loại bỏ block `kubernetes {}` bên trong cấu hình provider.
*   **Khắc phục:**
    *   **Cách 1 (Thất bại):** Cố gắng dùng `config_path`.
    *   **Cách 2 (Thành công):** Ghim phiên bản provider về v2.x: `version = ">= 2.0, < 3.0"`. Xóa thư mục `.terraform` và chạy lại `terraform init`.

### Lỗi 3: `context deadline exceeded` khi cài ArgoCD
*   **Nguyên nhân:** Worker Node (`t3.micro`) quá yếu (2 vCPU, 1GB RAM) hoặc mạng chậm, không kịp pull image và khởi động ArgoCD trong thời gian mặc định (5 phút).
*   **Khắc phục:**
    *   Nâng cấp Worker Node lên **`t3.small`** (2 vCPU, 2GB RAM).
    *   Tăng `timeout` trong Terraform resource `helm_release` lên **900s (15 phút)**.

### Lỗi 4: `maven wrapper: No such file or directory` trong CI
*   **Nguyên nhân:** Repo `scoutli-backend` được khởi tạo rỗng, thiếu file `mvnw` và `.mvn`.
*   **Khắc phục:** Sửa GitHub Actions Workflow để sử dụng lệnh `mvn` (có sẵn trên runner) thay vì `./mvnw`.

### Lỗi 5: Branch name mismatch trong CI
*   **Nguyên nhân:** GitHub mặc định nhánh chính là `main` hoặc `master`, nhưng file workflow cấu hình sai tên nhánh trigger (`on: push: branches: [ "main" ]` trong khi repo là `master`).
*   **Khắc phục:** Sửa file workflow thành `branches: [ "master" ]`.
