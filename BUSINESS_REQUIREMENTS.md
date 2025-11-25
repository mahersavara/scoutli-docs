# Yêu cầu Nghiệp vụ - Dự án Scoutli

Tài liệu này tổng hợp các yêu cầu về mặt nghiệp vụ và tính năng cho ứng dụng Scoutli, làm cơ sở để xây dựng lại dự án trên stack công nghệ mới.

---

### 1. Khái niệm cốt lõi
- Xây dựng một ứng dụng web tên là "Scoutli", hoạt động như một "Trung tâm Khám phá Địa phương".
- Mục tiêu: Cho phép người dùng chia sẻ, khám phá, và đánh giá các địa điểm thú vị ("hidden gems") như quán cà phê, điểm du lịch, không gian công cộng, v.v.

---

### 2. Quản lý Người dùng (Authentication & Authorization)

- **Xác thực:**
  - Người dùng có thể **Đăng ký** tài khoản mới (yêu cầu email, password).
  - Người dùng có thể **Đăng nhập** vào tài khoản đã có.
  - Người dùng có thể **Đăng xuất** khỏi tài khoản.

- **Phân quyền:**
  - Hệ thống có 3 vai trò (roles): `member`, `moderator`, `admin`.
  - **Member (Thành viên):**
    - Là vai trò mặc định cho người dùng mới đăng ký.
    - **Quyền:** Tạo, xem, sửa, xóa các `Discovery` do chính mình tạo. Có thể tạo bình luận và xếp hạng. Chỉ có thể xóa bình luận của chính mình.
  - **Moderator (Điều hành viên):**
    - **Quyền:** Có tất cả quyền của `member`, cộng thêm quyền **xóa bất kỳ bình luận (comment) nào** của người dùng khác.
  - **Admin (Quản trị viên):**
    - **Quyền:** Toàn quyền trên hệ thống. Có thể xem, sửa, xóa **bất kỳ** `Discovery`, `Comment`, hoặc `Rating` nào.

---

### 3. Đối tượng `Discovery` (Địa điểm khám phá)

- **Thuộc tính:**
  - `name`: Tên địa điểm (String, bắt buộc).
  - `description`: Mô tả chi tiết (Text).
  - `street_address`: Địa chỉ đường (String).
  - `district`: Quận/Huyện (String).
  - `city`: Thành phố (String).
  - `country`: Quốc gia (String).
  - `latitude`: Vĩ độ (Float).
  - `longitude`: Kinh độ (Float).
  - `user_id`: Khóa ngoại liên kết đến người tạo ra nó.

- **Nghiệp vụ liên quan:**
  - **Tự động Geocoding:** Khi người dùng tạo hoặc cập nhật một `Discovery`, hệ thống phải tự động chuyển đổi các trường địa chỉ (`street_address`, `city`, `country`) thành tọa độ `latitude` và `longitude` và lưu vào cơ sở dữ liệu.
  - **Gắn thẻ (Tagging):** Người dùng có thể gán nhiều thẻ (tags) cho một `Discovery` thông qua một trường nhập văn bản, các thẻ phân cách nhau bằng dấu phẩy.

---

### 4. Các tính năng tương tác

- **Bình luận (Comments):**
  - Một `Discovery` có thể có nhiều `Comment`.
  - Một `User` có thể có nhiều `Comment`.
  - Chỉ người dùng đã đăng nhập mới có thể tạo `Comment`.

- **Xếp hạng (Ratings):**
  - Người dùng đã đăng nhập có thể cho điểm (xếp hạng) một `Discovery` với thang điểm từ 1 đến 5.
  - Một `User` chỉ có thể có **một** `Rating` cho một `Discovery` cụ thể. Nếu người dùng xếp hạng lại, điểm số cũ sẽ được cập nhật.
  - Hệ thống cần hiển thị điểm xếp hạng trung bình của mỗi `Discovery`.

---

### 5. Yêu cầu Giao diện & Trải nghiệm Người dùng (UI/UX)

- **Trang chủ / Danh sách (`/discoveries`):**
  - Hiển thị danh sách các `Discovery` dưới dạng các "card" thông tin.
  - Phía trên danh sách là một bản đồ lớn hiển thị marker cho **tất cả** các `Discovery`.
  - **Chức năng "Find Near Me":**
    - Có một nút bấm cho phép người dùng chia sẻ vị trí hiện tại của họ.
    - Sau khi có vị trí, hệ thống lọc lại danh sách `Discovery` trong một bán kính có thể chọn (ví dụ: 1km, 5km, 10km).
    - Bản đồ được cập nhật, zoom vào vị trí của người dùng và hiển thị các `Discovery` lân cận.
  - **Lọc theo Tag:** Khi người dùng nhấn vào một thẻ tag bất kỳ, trang sẽ tải lại và chỉ hiển thị các `Discovery` có chứa tag đó.

- **Trang chi tiết (`/discoveries/:id`):**
  - Hiển thị đầy đủ thông tin của một `Discovery`.
  - Hiển thị một bản đồ nhỏ với marker chỉ định vị trí của `Discovery` đó.
  - Hiển thị danh sách các bình luận đã có.
  - Hiển thị form cho phép người dùng thêm bình luận mới.
  - Hiển thị điểm xếp hạng trung bình và form cho phép người dùng xếp hạng.
