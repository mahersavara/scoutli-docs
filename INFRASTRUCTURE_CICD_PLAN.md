# Kế hoạch Triển khai Hạ tầng & CI/CD

Tài liệu này đặc tả kế hoạch để đóng gói, triển khai, và tự động hóa ứng dụng Scoutli bằng Docker, Kubernetes, Helm, GitHub Actions, và ArgoCD.

---

### 1. Containerization (Đóng gói ứng dụng với Docker)

Mỗi thành phần của ứng dụng (Backend và Frontend) sẽ được đóng gói thành một Docker image riêng biệt.

- **Backend Image (Quarkus):**
  - Sử dụng công cụ build tích hợp của Quarkus để tạo ra một `Dockerfile`.
  - **Chiến lược:** Sẽ ưu tiên build ra **native executable** bằng GraalVM.
  - `Dockerfile` sẽ được tối ưu, chỉ chứa native executable và các thư viện cần thiết, giúp image có kích thước nhỏ, khởi động nhanh và bảo mật hơn.

- **Frontend Image (Angular):**
  - Sử dụng **multi-stage Docker build** để tối ưu hóa image.
  - **Giai đoạn 1 (Build Stage):**
    - Sử dụng một base image `node:lts` (Long-Term Support).
    - Sao chép mã nguồn Angular vào.
    - Chạy `npm install` để cài đặt dependencies.
    - Chạy `npm run build --prod` để build ứng dụng Angular ra các file HTML, CSS, JS tĩnh trong thư mục `/dist`.
  - **Giai đoạn 2 (Runtime Stage):**
    - Sử dụng một base image web server rất nhẹ như `nginx:alpine`.
    - Sao chép các file tĩnh đã được build từ Giai đoạn 1 vào thư mục phục vụ của Nginx.
    - Cấu hình Nginx để phục vụ ứng dụng SPA (Single-Page Application), trong đó tất cả các request không phải là file tĩnh sẽ được chuyển hướng về `index.html`.

---

### 2. Triển khai lên Kubernetes (K8s)

Kiến trúc trên K8s sẽ bao gồm các tài nguyên sau:

- **`Deployment`:**
  - `scoutli-backend-deployment`: Quản lý các Pod chứa container backend Quarkus.
  - `scoutli-frontend-deployment`: Quản lý các Pod chứa container frontend Nginx.

- **`Service`:**
  - `scoutli-backend-service`: Tạo một địa chỉ IP nội bộ ổn định cho các Pod backend. Kiểu `ClusterIP`.
  - `scoutli-frontend-service`: Tạo một địa chỉ IP nội bộ ổn định cho các Pod frontend. Kiểu `ClusterIP`.

- **`Ingress`:**
  - Một tài nguyên `Ingress` duy nhất để quản lý traffic từ bên ngoài.
  - **Quy tắc điều hướng (Routing Rules):**
    - Mọi request đến `your-domain.com/api/*` sẽ được chuyển đến `scoutli-backend-service`.
    - Mọi request khác đến `your-domain.com/*` sẽ được chuyển đến `scoutli-frontend-service`.

- **`ConfigMap` & `Secret`:**
  - `scoutli-db-secret`: Lưu trữ username, password, và URL kết nối tới database PostgreSQL.
  - `scoutli-backend-configmap`: Lưu trữ các cấu hình không nhạy cảm cho backend (ví dụ: API key của dịch vụ Geocoding).

---

### 3. Quản lý bằng HelmCharts

- Toàn bộ các file YAML của K8s ở trên sẽ được đóng gói vào một **Helm Chart** duy nhất tên là `scoutli`.
- Cấu trúc chart sẽ bao gồm các template cho từng loại tài nguyên.
- File `values.yaml` sẽ cho phép tùy chỉnh các cấu hình khi triển khai, ví dụ:
  ```yaml
  backend:
    replicaCount: 2
    image:
      repository: your-registry/scoutli-backend
      tag: "latest"
  frontend:
    replicaCount: 1
    image:
      repository: your-registry/scoutli-frontend
      tag: "latest"
  ingress:
    domain: scoutli.example.com
  ```

---

### 4. Quy trình CI/CD (GitHub Actions & ArgoCD)

Quy trình sẽ tuân thủ theo mô hình GitOps.

- **CI - GitHub Actions (Tích hợp liên tục):**
  - **Trigger:** Khi có commit mới được đẩy lên nhánh `main`.
  - **Các bước (Jobs):**
    1.  **Test:** Chạy unit test cho cả backend và frontend.
    2.  **Build:**
        - Build native executable cho Quarkus.
        - Build các file tĩnh cho Angular.
    3.  **Package:**
        - Build Docker image cho backend và frontend.
        - Gắn tag cho image bằng mã commit SHA (ví dụ: `sha-a1b2c3d`).
    4.  **Publish:** Đẩy 2 Docker image đã được tag lên một Container Registry (ví dụ: GitHub Container Registry).
    5.  **Update GitOps Repo:**
        - Tự động checkout một kho Git thứ hai (GitOps Repo).
        - Cập nhật file `values.yaml` của Helm Chart trong repo đó, thay đổi giá trị `image.tag` thành mã commit SHA mới.
        - Commit và đẩy thay đổi lên GitOps Repo.

- **CD - ArgoCD (Triển khai liên tục):**
  - **Cài đặt:** ArgoCD được cài đặt trên cluster K8s.
  - **Cấu hình:** Tạo một `Application` trong ArgoCD để nó "theo dõi" (sync) Helm Chart trong **GitOps Repo**.
  - **Hoạt động:**
    1.  ArgoCD liên tục kiểm tra GitOps Repo.
    2.  Khi phát hiện `image.tag` đã thay đổi (do GitHub Actions vừa cập nhật), ArgoCD sẽ nhận thấy trạng thái trên cluster ("live state") khác với trạng thái trong Git ("desired state").
    3.  ArgoCD sẽ tự động thực hiện `helm upgrade`, áp dụng các thay đổi vào cluster.
    4.  Kubernetes sẽ thực hiện một **rolling update**, từ từ thay thế các Pod cũ bằng các Pod mới chạy image phiên bản mới, đảm bảo ứng dụng không bị gián đoạn.
