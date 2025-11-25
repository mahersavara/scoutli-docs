# Kế hoạch Triển khai Ứng dụng Di động - Compose Multiplatform

Tài liệu này đặc tả kế hoạch xây dựng ứng dụng di động gốc (native) cho Android và iOS từ một mã nguồn chung (single codebase) bằng công nghệ Compose Multiplatform.

---

### 1. Lựa chọn Công nghệ & Kiến trúc

- **Công nghệ chính:** **Compose Multiplatform (KMP)**.
  - **Ngôn ngữ:** Kotlin.
  - **UI Framework:** Jetpack Compose.
  - **Lý do:** Cho phép chia sẻ tối đa code (UI và business logic) giữa Android và iOS, biên dịch ra ứng dụng gốc với hiệu năng cao, và có sự tương đồng về hệ sinh thái với backend Java/Quarkus.

- **Kiến trúc ứng dụng:** **MVVM (Model-View-ViewModel)**.
  - **Model:** Các lớp dữ liệu (data class) bằng Kotlin, được chia sẻ trong `commonMain`.
  - **View (Composable Screens):** Các hàm Composable chỉ chịu trách nhiệm hiển thị trạng thái (state) từ ViewModel và thông báo sự kiện người dùng.
  - **ViewModel:** Chứa state của UI, xử lý logic, và giao tiếp với tầng Data (Repository).

---

### 2. Cấu trúc Dự án

Dự án sẽ được cấu trúc theo mô hình chuẩn của Compose Multiplatform.

- **`shared` module:** Module trung tâm chứa code dùng chung.
  - `commonMain`: Chứa khoảng 90-95% code của ứng dụng.
    - **UI:** Các màn hình Compose (`DiscoveryListScreen`, `DiscoveryDetailScreen`...).
    - **ViewModels:** Logic cho từng màn hình.
    - **Data Layer:** `Repository` và các lời gọi API (sử dụng Ktor).
    - **Models:** Các data class (ví dụ: `Discovery`, `User`).
  - `androidMain`: Chứa code đặc thù cho Android (ví dụ: triển khai `MapView` cho Google Maps).
  - `iosMain`: Chứa code đặc thù cho iOS (ví dụ: triển khai `MapView` cho Apple MapKit).

- **`androidApp` module:** Project Android gốc, đóng vai trò là "vỏ" để chạy `shared` module.
- **`iosApp` module:** Project Xcode (iOS) gốc, đóng vai trò là "vỏ" để chạy `shared` module.

---

### 3. Các Thư viện và Công cụ chính

- **Networking:** **Ktor Client** để thực hiện các lời gọi API đến backend Quarkus.
- **JSON Serialization:** **kotlinx.serialization** để chuyển đổi dữ liệu JSON.
- **Quản lý Trạng thái (State Management):** Bắt đầu với `StateFlow` và `SharedFlow` của Kotlin Coroutines trong ViewModels. Có thể cân nhắc các thư viện như `MVI-Kotlin` khi ứng dụng phức tạp hơn.
- **Dependency Injection:** **Koin** hoặc **Kodein** là các lựa chọn phổ biến cho KMP.
- **Lưu trữ Cục bộ (Local Storage/Cache):** **SQLDelight** để tạo cơ sở dữ liệu SQL đa nền tảng, giúp lưu trữ dữ liệu offline.
- **Điều hướng (Navigation):** Sử dụng một thư viện điều hướng đa nền tảng cho Compose, ví dụ như **Voyager**.

---

### 4. Triển khai các Tính năng Phức tạp

- **Bản đồ (Maps):**
  - Tạo một Composable `MapView` dùng chung bằng cơ chế `expect/actual` của Kotlin.
  - **`actual` trên Android:** Sử dụng `AndroidView` Composable để bọc (wrap) `MapView` của Google Maps SDK.
  - **`actual` trên iOS:** Sử dụng `UIKitView` Composable để bọc `MKMapView` của Apple MapKit.
  - Logic về marker và camera sẽ được điều khiển từ ViewModel chung.

- **Định vị (Geolocation):**
  - Sử dụng thư viện `multiplatform-settings` để lưu trữ quyền truy cập vị trí.
  - Sử dụng cơ chế `expect/actual` để gọi API lấy vị trí của từng nền tảng (`LocationManager` trên Android và `CLLocationManager` trên iOS).

- **Xác thực (Authentication):**
  - Luồng đăng nhập/đăng ký sẽ gọi API và lưu trữ JWT token nhận được.
  - Token sẽ được lưu trữ an toàn bằng `multiplatform-settings` (có thể mã hóa).
  - Ktor Client sẽ được cấu hình để tự động đính kèm token này vào header của các request cần xác thực.
