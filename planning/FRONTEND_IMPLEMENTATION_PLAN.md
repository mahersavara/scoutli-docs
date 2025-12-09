# Kế hoạch Triển khai Frontend - Angular

Tài liệu này chi tiết hóa cách xây dựng giao diện người dùng đã định nghĩa trong đặc tả kỹ thuật bằng cách sử dụng Angular và các thư viện liên quan.

---

### 1. Thiết lập Dự án & Cấu trúc

- **Khởi tạo dự án:** Sử dụng Angular CLI: `ng new scoutli-frontend --style=scss --routing`.
- **Cấu trúc thư mục:**
  - `src/app/components`: Chứa các component tái sử dụng (Navbar, Footer, Map...).
  - `src/app/features`: Chứa các module cho từng tính năng lớn (Discovery, User...).
  - `src/app/core/services`: Chứa các service để gọi API (AuthService, DiscoveryService...).
  - `src/app/core/interceptors`: Chứa các interceptor (ví dụ: tự động gắn JWT token vào header).
  - `src/app/core/guards`: Chứa các route guard (ví dụ: AuthGuard để bảo vệ route).

---

### 2. Các Thư viện chính

- **UI Components:**
  - **Angular Material** hoặc **NG-ZORRO**: Cung cấp một bộ component UI phong phú (card, button, form, table...).
- **Bản đồ:**
  - `leaflet`: Thư viện bản đồ.
  - `@asymmetrik/ngx-leaflet`: Wrapper để tích hợp Leaflet vào Angular một cách dễ dàng.
- **Xác thực:**
  - `@auth0/angular-jwt`: Thư viện giúp tự động đính kèm JWT vào các request API và kiểm tra token hết hạn.

---

### 3. Triển khai Luồng Xác thực (Authentication Flow)

1.  **Login:**
    - `LoginComponent` gọi `AuthService.login(email, password)`.
    - `AuthService` gửi request `POST /api/users/login`.
    - Khi nhận được token, `AuthService` lưu nó vào `localStorage` của trình duyệt.
2.  **Tự động gắn Token:**
    - Tạo một `JwtInterceptor` (HTTP Interceptor).
    - Interceptor này sẽ tự động lấy token từ `localStorage` và gắn nó vào header `Authorization` của **mọi** request gửi đến backend.
3.  **Bảo vệ Route:**
    - Tạo một `AuthGuard`.
    - Gắn `AuthGuard` vào các route cần đăng nhập (ví dụ: `/discoveries/new`).
    - `AuthGuard` sẽ kiểm tra xem token có hợp lệ không trước khi cho phép người dùng truy cập route.

---

### 4. Triển khai các Tính năng chính

- **Hiển thị Bản đồ (`DiscoveryMapComponent`):**
  - Component này sẽ nhận một mảng các địa điểm (markers) qua một `@Input()`.
  - Sử dụng `ngx-leaflet` để vẽ bản đồ và các marker.
  - Có các phương thức để tự động zoom (`fitBounds`) hoặc đặt tâm (`setView`).

- **Trang chủ (`DiscoveryListComponent`):**
  - `ngOnInit()`: Gọi `DiscoveryService.getDiscoveries()` để lấy danh sách các địa điểm.
  - Dữ liệu trả về được gán cho `DiscoveryMapComponent` và được dùng để render các `DiscoveryCardComponent` bằng vòng lặp `*ngFor`.
  - **"Find Near Me":**
    - Lấy tọa độ từ trình duyệt bằng `navigator.geolocation`.
    - Gọi `DiscoveryService.getDiscoveries(lat, lon, radius)`.
    - Cập nhật lại danh sách và bản đồ với dữ liệu mới.

- **Quản lý Trạng thái (State Management):**
  - **Bắt đầu:** Sử dụng các `BehaviorSubject` trong các service (ví dụ: `AuthService` có một `currentUser$` observable) để chia sẻ trạng thái giữa các component.
  - **Nâng cao (nếu cần):** Khi ứng dụng phức tạp hơn, có thể cân nhắc sử dụng các thư viện quản lý trạng thái chuyên dụng như **NgRx** hoặc **Akita**.
