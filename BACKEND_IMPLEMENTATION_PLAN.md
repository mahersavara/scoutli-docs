# Kế hoạch Triển khai Backend - Quarkus

Tài liệu này chi tiết hóa cách xây dựng các API đã định nghĩa trong đặc tả kỹ thuật bằng cách sử dụng Java, Quarkus, và các thư viện liên quan.

---

### 1. Thiết lập Dự án & Các Extension Cần thiết

- **Khởi tạo dự án:** Sử dụng Quarkus CLI hoặc trang [code.quarkus.io](https://code.quarkus.io) để tạo dự án Maven/Gradle.
- **Các extension quan trọng cần thêm:**
  - `quarkus-resteasy-reactive-jackson`: Để xây dựng RESTful API với JSON.
  - `quarkus-hibernate-orm-panache`: Để làm việc với cơ sở dữ liệu một cách đơn giản hóa.
  - `quarkus-jdbc-postgresql`: Driver để kết nối tới PostgreSQL.
  - `quarkus-hibernate-validator`: Để kiểm tra dữ liệu đầu vào (validation).
  - `quarkus-smallrye-jwt`: Để tạo và xác thực JSON Web Tokens (JWT) cho việc bảo mật.
  - `quarkus-arc`: Framework Dependency Injection (tích hợp sẵn).
  - `quarkus-flyway` hoặc `quarkus-liquibase`: Để quản lý các phiên bản thay đổi của schema cơ sở dữ liệu (tương tự `db/migrate` của Rails).

---

### 2. Triển khai Mô hình Dữ liệu (JPA/Panache Entities)

- Mỗi bảng trong cơ sở dữ liệu sẽ được ánh xạ tới một class Java, gọi là **Entity**.
- Chúng ta sẽ sử dụng **PanacheEntity** để tự động có các trường `id` và các phương thức truy vấn tiện lợi (`find`, `list`, `delete`...).
- **Ví dụ (Discovery.java):**
  ```java
  import io.quarkus.hibernate.orm.panache.PanacheEntity;
  import javax.persistence.*;
  import java.util.List;

  @Entity
  @Table(name = "discoveries")
  public class Discovery extends PanacheEntity {
      @Column(nullable = false)
      public String name;
      public String description;
      // ... các trường khác

      @ManyToOne(fetch = FetchType.LAZY)
      @JoinColumn(name = "user_id", nullable = false)
      public User user;

      @ManyToMany(cascade = CascadeType.ALL)
      @JoinTable(name = "discovery_tags",
          joinColumns = @JoinColumn(name = "discovery_id"),
          inverseJoinColumns = @JoinColumn(name = "tag_id"))
      public List<Tag> tags;
  }
  ```

---

### 3. Triển khai API (JAX-RS Resources)

- Mỗi nhóm endpoint sẽ được triển khai trong một class Java gọi là **Resource**.
- Sử dụng các annotation của JAX-RS để định nghĩa đường dẫn và phương thức HTTP.
- **Ví dụ (DiscoveryResource.java):**
  ```java
  import javax.ws.rs.*;
  import javax.ws.rs.core.MediaType;
  import java.util.List;

  @Path("/api/discoveries")
  @Produces(MediaType.APPLICATION_JSON)
  @Consumes(MediaType.APPLICATION_JSON)
  public class DiscoveryResource {

      @GET
      public List<Discovery> list(@QueryParam("tag") String tag) {
          if (tag != null) {
              return Discovery.find("...query by tag...", tag).list();
          }
          return Discovery.listAll();
      }

      @GET
      @Path("/{id}")
      public Discovery get(Long id) {
          return Discovery.findById(id);
      }

      @POST
      @RolesAllowed({"MEMBER", "ADMIN"}) // Bảo mật endpoint
      public Response create(Discovery discovery) {
          // ... logic to save discovery
          return Response.status(201).entity(discovery).build();
      }
  }
  ```

---

### 4. Triển khai các Nghiệp vụ Phức tạp

- **Bảo mật (JWT):**
  - Sử dụng `quarkus-smallrye-jwt`.
  - Endpoint `/api/users/login` sẽ kiểm tra username/password, sau đó dùng thư viện `smallrye-jwt` để tạo ra một chuỗi token.
  - Các endpoint cần bảo vệ sẽ được đánh dấu với annotation `@Authenticated` hoặc `@RolesAllowed({"ADMIN"})`.

- **Geocoding:**
  - Không có thư viện tương đương Geocoder gem. Chúng ta sẽ phải **tạo một `GeocodingService`**.
  - Service này sẽ gọi đến một API Geocoding bên ngoài (ví dụ: OpenStreetMap Nominatim, Google Maps API).
  - `DiscoveryResource` sẽ `@Inject` `GeocodingService` và gọi nó trước khi lưu một `Discovery` mới.

- **Phân quyền:**
  - Logic kiểm tra quyền (ví dụ: user có phải chủ sở hữu của discovery không) sẽ được viết trong các class Resource, trước khi thực hiện các hành động update hoặc delete.
