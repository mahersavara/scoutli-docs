# Hướng dẫn Triển khai & Nhật ký Dự án Scoutli (New Stack)

Tài liệu này tổng hợp quy trình triển khai hạ tầng AWS EKS, CI/CD với GitHub Actions, và cài đặt ArgoCD, cũng như các bài học kinh nghiệm từ quá trình thực hiện.

---

## 1. Guideline Chi tiết (Step-by-Step Guide)

### Giai đoạn 1: Chuẩn bị Môi trường (Local)

**Bước 1.1: Cài đặt công cụ trên Windows**
*   **WSL2:** Mở PowerShell (Admin) -> chạy `wsl --install`. Khởi động lại máy.
*   **Docker Desktop:** Tải và cài đặt. Vào Settings -> Resources -> WSL Integration -> Bật Ubuntu.
*   **Terraform:** Tải file zip, giải nén vào `C:\terraform`, thêm vào PATH.
    *   Kiểm tra: `terraform -version`
*   **AWS CLI:** Tải và cài đặt file MSI.
    *   Kiểm tra: `aws --version`
*   **GitHub CLI (`gh`):** Tải file zip, giải nén, thêm vào PATH.
    *   Kiểm tra: `gh --version`

**Bước 1.2: Cấu hình AWS CLI**
1.  Tạo IAM User trên AWS Console (quyền `AdministratorAccess`).
2.  Tạo Access Key cho user đó.
3.  Mở PowerShell và chạy:
    ```powershell
    aws configure
    ```
4.  Nhập thông tin:
    *   `AWS Access Key ID`: [Key của bạn]
    *   `AWS Secret Access Key`: [Secret của bạn]
    *   `Default region name`: `ap-southeast-1` (Singapore)
    *   `Default output format`: `json`

---

### Giai đoạn 2: Khởi tạo Cấu trúc Multi-Repo

**Bước 2.1: Tạo cấu trúc thư mục**
1.  Mở PowerShell, điều hướng đến nơi bạn muốn lưu dự án.
2.  Tạo thư mục gốc và các thư mục con:
    ```powershell
    mkdir scoutli-project
    cd scoutli-project
    mkdir scoutli-infra scoutli-backend scoutli-frontend scoutli-helm scoutli-gitops scoutli-docs
    ```

**Bước 2.2: Khởi tạo Git và đẩy lên GitHub (Tự động hóa)**
1.  Đăng nhập GitHub CLI:
    ```powershell
    gh auth login
    ```
2.  Chạy script sau trong PowerShell để tạo 6 repo cùng lúc:
    ```powershell
    $repos = @("scoutli-infra", "scoutli-backend", "scoutli-frontend", "scoutli-helm", "scoutli-gitops", "scoutli-docs")
    foreach ($repo in $repos) {
        Set-Location $repo
        git init
        git add .
        git commit -m "Initial commit"
        gh repo create $repo --public --source=. --remote=origin --push -y
        Set-Location ..
    }
    ```

---

### Giai đoạn 3: Dựng Hạ tầng AWS (Terraform)

**Bước 3.1: Tạo VPC (Mạng)**
1.  Vào thư mục VPC:
    ```bash
    cd scoutli-infra/vpc
    ```
2.  Khởi tạo và chạy Terraform:
    ```bash
    terraform init
    terraform apply -auto-approve
    ```
3.  **Kết quả mong đợi:** Terraform báo `Apply complete!`. Ghi lại `vpc_id` và `private_subnet_ids` từ phần Outputs.

**Bước 3.2: Tạo EKS Cluster (Kubernetes)**
1.  Vào thư mục EKS:
    ```bash
    cd ../eks
    ```
2.  Cập nhật file `terraform.tfvars` với ID lấy được từ Bước 3.1:
    ```hcl
    vpc_id = "vpc-xxxxxx"
    private_subnet_ids = ["subnet-xxxx", "subnet-yyyy"]
    node_instance_type = "t3.small" # Khuyến nghị t3.small cho ArgoCD
    ```
3.  Chạy Terraform:
    ```bash
    terraform init
    terraform apply -auto-approve
    ```
4.  **Kết quả mong đợi:** Sau khoảng 15-20 phút, Terraform báo thành công.

**Bước 3.3: Tạo ECR (Container Registry)**
1.  Vào thư mục ECR:
    ```bash
    cd ../ecr
    ```
2.  Chạy Terraform:
    ```bash
    terraform init
    terraform apply -auto-approve
    ```
3.  **Kết quả mong đợi:** Terraform tạo 2 repositories `scoutli-backend` và `scoutli-frontend`.

---

### Giai đoạn 4: Thiết lập CI (GitHub Actions)

**Bước 4.1: Cấu hình GitHub Secrets**
1.  Vào GitHub -> Repository `scoutli-backend` -> Settings -> Secrets -> Actions.
2.  Thêm 3 secrets bắt buộc:
    *   `AWS_ACCESS_KEY_ID`
    *   `AWS_SECRET_ACCESS_KEY`
    *   `AWS_REGION` (`ap-southeast-1`)
3.  Làm tương tự cho repo `scoutli-frontend`.

**Bước 4.2: Kích hoạt Workflow**
*   Code workflow `.github/workflows/build.yml` đã có sẵn trong repo.
*   Chỉ cần push một commit bất kỳ vào nhánh `master` để kích hoạt CI.
*   Vào tab **Actions** trên GitHub để xem quá trình Build và Push Docker Image lên ECR.

---

### Giai đoạn 5: Cài đặt ArgoCD (Terraform Addons)

**Bước 5.1: Kết nối kubectl với EKS**
1.  Chạy lệnh sau trên máy local để lấy file cấu hình:
    ```powershell
    aws eks update-kubeconfig --region ap-southeast-1 --name scoutli-cluster
    ```
2.  Kiểm tra kết nối:
    ```powershell
    kubectl get nodes
    ```
    (Phải hiện ra danh sách các worker nodes).

**Bước 5.2: Cài đặt ArgoCD**
1.  Thiết lập biến môi trường `KUBECONFIG` (Bắt buộc trên Windows):
    ```powershell
    $env:KUBECONFIG = "$env:USERPROFILE\.kube\config"
    ```
2.  Vào thư mục addons và chạy Terraform:
    ```bash
    cd scoutli-infra/addons
    terraform init
    terraform apply -auto-approve
    ```
    *(Lưu ý: `main.tf` phải dùng provider helm v2.x và KHÔNG có block `provider "helm"` để tự động dùng môi trường).*

**Bước 5.3: Lấy thông tin đăng nhập ArgoCD**
1.  Lấy mật khẩu Admin ban đầu:
    ```powershell
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```
2.  Lấy địa chỉ truy cập (DNS của Load Balancer):
    ```powershell
    kubectl -n argocd get svc argocd-server -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"
    ```
3.  Truy cập trình duyệt với địa chỉ vừa lấy (chấp nhận cảnh báo bảo mật SSL). Đăng nhập với user `admin` và mật khẩu vừa lấy.

---

## 2. Các thao tác Thủ công (Manual Actions)

Người dùng cần nhập tay các lệnh sau trong quá trình thực hiện:

1.  **AWS CLI Config:** `aws configure`
2.  **Cập nhật Kubeconfig:** `aws eks update-kubeconfig ...`
3.  **Set biến môi trường KUBECONFIG:** `$env:KUBECONFIG = ...`
4.  **Thêm Secrets vào GitHub Repo:** Làm trên giao diện Web của GitHub.

---

## 3. Troubleshooting (Nhật ký Lỗi & Khắc phục)

### Lỗi 1: `SSL: CERTIFICATE_VERIFY_FAILED` khi chạy AWS CLI
*   **Nguyên nhân:** Mạng công ty sử dụng Proxy/Firewall chặn SSL và thay thế bằng certificate riêng.
*   **Khắc phục:** Cấu hình `ca_bundle` trong `~/.aws/config` hoặc dùng biến môi trường `AWS_CA_BUNDLE`.

### Lỗi 2: `Unsupported block type "kubernetes"` trong Terraform Helm Provider
*   **Nguyên nhân:** Provider Helm v3.0.0+ đã loại bỏ block `kubernetes {}`.
*   **Khắc phục:** 
    *   Ghim phiên bản provider về v2.x: `version = ">= 2.0, < 3.0"`.
    *   Hoặc tốt hơn: Để trống block `provider "helm" {}` và dùng biến môi trường `KUBECONFIG` để Terraform tự tìm cấu hình.

### Lỗi 3: `context deadline exceeded` khi cài ArgoCD
*   **Nguyên nhân:** Worker Node (`t3.micro`) quá yếu hoặc mạng chậm.
*   **Khắc phục:** 
    *   Nâng cấp Worker Node lên **`t3.small`**.
    *   Tăng `timeout` trong Terraform resource `helm_release` lên **900s**.

### Lỗi 4: `maven wrapper: No such file or directory` trong CI
*   **Nguyên nhân:** Repo backend thiếu file `mvnw`.
*   **Khắc phục:** Sửa GitHub Actions Workflow để sử dụng lệnh `mvn` có sẵn.

### Lỗi 5: Branch name mismatch trong CI
*   **Nguyên nhân:** GitHub mặc định nhánh `master`, workflow để `main`.
*   **Khắc phục:** Sửa file workflow thành `branches: [ "master" ]`.