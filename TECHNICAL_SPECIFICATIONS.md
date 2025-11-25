# Đặc tả Kỹ thuật Chi tiết - Dự án Scoutli (v2)

Tài liệu này trình bày chi tiết các yêu cầu về mặt kỹ thuật để xây dựng lại dự án Scoutli, bao gồm Mô hình Dữ liệu, API Backend, cấu trúc Frontend, và các yêu cầu phi chức năng.

---

### 1. Mô hình Dữ liệu (Data Models)
Cấu trúc cho các bảng trong cơ sở dữ liệu PostgreSQL.

- **Bảng `users`**
  - `id`: `BIGINT`, Primary Key, Auto-increment
  - `email`: `VARCHAR(255)`, Not Null, Unique
  - `password_hash`: `VARCHAR(255)`, Not Null
  - `role`: `VARCHAR(50)`, Not Null, Default: `'MEMBER'` (Giá trị: 'MEMBER', 'MODERATOR', 'ADMIN')

- **Bảng `discoveries`**
  - `id`: `BIGINT`, Primary Key, Auto-increment
  - `name`: `VARCHAR(255)`, Not Null
  - `description`: `TEXT`
  - `street_address`: `VARCHAR(255)`
  - `city`: `VARCHAR(255)`
  - `country`: `VARCHAR(255)`
  - `latitude`: `DOUBLE PRECISION`
  - `longitude`: `DOUBLE PRECISION`
  - `user_id`: `BIGINT`, Not Null, Foreign Key tới `users(id)`

- **Bảng `comments`**
  - `id`: `BIGINT`, Primary Key, Auto-increment
  - `content`: `TEXT`, Not Null
  - `created_at`: `TIMESTAMP`, Not Null
  - `user_id`: `BIGINT`, Not Null, Foreign Key tới `users(id)`
  - `discovery_id`: `BIGINT`, Not Null, Foreign Key tới `discoveries(id)`

- **Bảng `ratings`**
  - `id`: `BIGINT`, Primary Key, Auto-increment
  - `score`: `INTEGER`, Not Null (giá trị từ 1 đến 5)
  - `user_id`: `BIGINT`, Not Null, Foreign Key tới `users(id)`
  - `discovery_id`: `BIGINT`, Not Null, Foreign Key tới `discoveries(id)`
  - **Constraint:** `UNIQUE(user_id, discovery_id)`

- **Bảng `tags`**
  - `id`: `BIGINT`, Primary Key, Auto-increment
  - `name`: `VARCHAR(255)`, Not Null, Unique

- **Bảng `discovery_tags` (Bảng trung gian Many-to-Many)**
  - `discovery_id`: `BIGINT`, Foreign Key tới `discoveries(id)`
  - `tag_id`: `BIGINT`, Foreign Key tới `tags(id)`
  - **Constraint:** `PRIMARY KEY(discovery_id, tag_id)`

---

### 2. API Backend (RESTful API)
Giao tiếp giữa Backend (Quarkus) và Frontend (Angular) qua RESTful API, sử dụng JWT (JSON Web Tokens) để xác thực.

- **Authentication**
  - `POST /api/users/register`: Đăng ký user mới.
  - `POST /api/users/login`: Đăng nhập.

- **Discoveries**
  - `GET /api/discoveries`: Lấy danh sách discoveries (đã phân trang).
    - **Query Params:** `?tag=...`, `?latitude=...&longitude=...&radius=...`, `?page=0&size=20`
    - **Success Response:** `200 OK`, Body: `{ "content": [ { discovery_summary_1 }, ... ], "totalPages": 10, "totalElements": 198, ... }`
  - `POST /api/discoveries`: Tạo discovery mới (Yêu cầu token).
  - `GET /api/discoveries/{id}`: Lấy chi tiết một discovery.
    - **Success Response:** `200 OK`, Body: `{ "id": 1, "name": "...", "user": { "id": 1, "email": "..." }, "tags": [ { "name": "coffee" } ], ... }`
  - `PUT /api/discoveries/{id}`: Cập nhật discovery (Yêu cầu token, check quyền).
  - `DELETE /api/discoveries/{id}`: Xóa discovery (Yêu cầu token, check quyền).

- **Comments**
  - `GET /api/discoveries/{discoveryId}/comments`: Lấy danh sách comment (đã phân trang).
  - `POST /api/discoveries/{discoveryId}/comments`: Tạo comment mới (Yêu cầu token).
  - `DELETE /api/comments/{id}`: Xóa comment (Yêu cầu token, check quyền).

- **Ratings**
  - `POST /api/discoveries/{discoveryId}/ratings`: Tạo hoặc cập nhật rating (Yêu cầu token).

---

### 3. Thành phần Frontend (Angular Components)
- `NavbarComponent`, `FooterComponent`
- `DiscoveryListComponent`, `DiscoveryMapComponent`, `DiscoveryCardComponent`
- `DiscoveryDetailComponent`, `CommentSectionComponent`, `RatingComponent`
- `LoginComponent`, `RegisterComponent`
- `PaginatorComponent`: Component tái sử dụng để hiển thị thanh phân trang.

---

### 4. Yêu cầu Phi chức năng (Non-Functional Requirements)

- **Validation (Kiểm tra dữ liệu)**
  - `User`: `email` phải có định dạng hợp lệ. `password` phải có ít nhất 8 ký tự.
  - `Discovery`: `name` là trường bắt buộc, không được để trống.
  - `Comment`: `content` là trường bắt buộc.
  - `Rating`: `score` phải là số nguyên từ 1 đến 5.

- **Error Handling (Xử lý lỗi API)**
  - `400 Bad Request`: Trả về khi dữ liệu đầu vào không hợp lệ (validation failed). Response body nên chứa chi tiết lỗi cho từng trường.
  - `401 Unauthorized`: Trả về khi request thiếu hoặc có token JWT không hợp lệ.
  - `403 Forbidden`: Trả về khi người dùng không có quyền thực hiện hành động (ví dụ: xóa discovery của người khác khi không phải admin).
  - `404 Not Found`: Trả về khi không tìm thấy tài nguyên (ví dụ: `GET /api/discoveries/999` với id không tồn tại).

- **Pagination (Phân trang)**
  - Tất cả các API trả về danh sách (`GET /api/discoveries`, `GET /api/discoveries/{id}/comments`) phải hỗ trợ phân trang.
  - Sử dụng query params: `page` (số trang, bắt đầu từ 0) và `size` (số lượng item mỗi trang).
  - API response phải có cấu trúc chứa dữ liệu và thông tin phân trang:
    ```json
    {
      "content": [ ... ],
      "page": 0,
      "size": 20,
      "totalPages": 10,
      "totalElements": 198
    }
    ```