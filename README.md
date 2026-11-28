# SkyPermit UTM — Khối 1: Đăng ký, Cấp phép Bay 4D & Quản lý Không phận (Port 8080)

Hệ thống UTM hoàn chỉnh gồm **3 nhánh độc lập** trên cùng repository và kết nối khép kín theo thời gian thực:

| Nhánh Git | Thành phần | Công nghệ | Cổng (Port) |
| --- | --- | --- | --- |
| `main` (nhánh này) | **Khối 1 — SkyPermit UTM**: Đăng ký UAV, nộp & duyệt kế hoạch bay 4D, Geozone, Nhật ký, Điều phối tích hợp | Java 21 + Spring Boot + H2 / SQL Server | `8080` |
| `simulation` | **Khối 2 — Bộ mô phỏng PX4 SITL**: Động lực học 6-DOF, MAVLink, SQLite, tự đồng bộ UAV từ Khối 1 & phát telemetry 10Hz | Python + FastAPI + Uvicorn + SQLite | `8001` |
| `3D` | **Khối 4 — Giám sát 3D & Chiến thuật**: Bản đồ 3D CesiumJS/Three.js Đà Nẵng, dự báo quỹ đạo 12s, cảnh báo & gửi lệnh MAVLink | React 19 + TypeScript + CesiumJS / Python FastAPI | `3000` & `8000` |

---

## 1. Yêu cầu môi trường

- **Git**
- **JDK 21 trở lên** (`java -version` nhận đúng Java 21+) — dùng cho nhánh `main` (đã có sẵn `mvnw.cmd` / `./mvnw`, không cần cài Maven riêng).
- **Python 3.10 trở lên** (`python --version`) — dùng cho nhánh `simulation` và backend nhánh `3D`.
- **Node.js 20 trở lên** (`node -v`, `npm -v`) — dùng cho frontend nhánh `3D`.

---

## 2. Chạy trọn bộ cả 3 nhánh trên máy (Quickstart 5 phút)

### Bước 1: Clone cả 3 nhánh vào 3 thư mục ngang hàng

```powershell
mkdir utm-system && cd utm-system

git clone -b main --single-branch https://github.com/vuduyvuong/UTM.git utm-khoi1
git clone -b simulation --single-branch https://github.com/vuduyvuong/UTM.git utm-khoi2
git clone -b 3D --single-branch https://github.com/vuduyvuong/UTM.git utm-khoi4
```

### Bước 2: Cài đặt thư viện cho từng khối (chỉ làm 1 lần đầu)

```powershell
# 1. Khối 1 (main): tạo file cấu hình local từ mẫu
Copy-Item .\utm-khoi1\application-local.properties.example .\utm-khoi1\application-local.properties

# 2. Khối 2 (simulation): cài thư viện Python
python -m pip install -r .\utm-khoi2\requirements.txt

# 3. Khối 4 (3D): cài thư viện Python backend & Node.js frontend
python -m pip install -r .\utm-khoi4\backend\requirements.txt
npm --prefix .\utm-khoi4 install
```

### Bước 3: Mở 4 cửa sổ Terminal để chạy toàn hệ thống

- **Terminal 1 — Khối 1 (SkyPermit UTM `:8080`)**:
  ```powershell
  cd utm-khoi1
  .\mvnw.cmd spring-boot:run
  ```
- **Terminal 2 — Khối 2 (PX4 Simulator `:8001`)**:
  ```powershell
  cd utm-khoi2
  python -m uvicorn khoi2_px4_simulator.main:app --host 0.0.0.0 --port 8001
  ```
- **Terminal 3 — Khối 4 Backend (FastAPI `:8000`)**:
  ```powershell
  cd utm-khoi4
  python -m uvicorn backend.app.main:app --host 127.0.0.1 --port 8000
  ```
- **Terminal 4 — Khối 4 Frontend (Web 3D `:3000`)**:
  ```powershell
  cd utm-khoi4
  npm run dev -- --host 127.0.0.1 --port 3000
  ```

*(Trên Linux/macOS, thay `Copy-Item` bằng `cp` và `.\mvnw.cmd` bằng `./mvnw`.)*

### Bước 4: Mở trình duyệt & kiểm thử luồng khép kín

| Giao diện | Địa chỉ |
| --- | --- |
| Cổng Đăng ký & Cấp phép Bay (Khối 1) | http://localhost:8080 |
| Swagger API Khối 1 | http://localhost:8080/swagger-ui.html |
| Trạm Mô phỏng PX4 / MAVLink 3D (Khối 2) | http://localhost:8001 |
| Trạm Giám sát Không lưu 3D (Khối 4) | http://localhost:3000 |
| API Chiến thuật & Handoff Khối 4 | http://127.0.0.1:8000/docs |

**Kịch bản demo nối thông 3 khối:**
1. Mở http://localhost:8080, đăng ký tài khoản `OPERATOR` mới, khai báo UAV (ví dụ `MULTIROTOR`, serial tùy chọn) rồi nộp kế hoạch bay 4D ở khu vực Đà Nẵng.
2. Đăng nhập tài khoản `ADMIN` (`admin` / `admin123@sky` nếu dùng [application-local.properties.example](application-local.properties.example)) để **Duyệt (APPROVED)** kế hoạch bay.
3. Khối 1 tự động gửi bản tin sang cả Khối 4 (`:8000`) và Khối 2 (`:8001`); đồng thời Khối 2 tự động đồng bộ danh sách UAV mới đăng ký từ Khối 1 mỗi 5 giây và cất cánh mô phỏng.
4. Mở http://localhost:3000 ở chế độ **LIVE**: thấy UAV bay thực tế kèm quỹ đạo dự báo nét đứt 12s, có thể gửi lệnh điều khiển MAVLink trực tiếp sang Khối 2 và đồng bộ tiến độ/quyết định ngược về Khối 1.

---

## 3. Chạy riêng Khối 1 (SkyPermit UTM)

### Cách A: Dùng H2 Database tích hợp sẵn (Khuyên dùng, chạy ngay)

Tạo `application-local.properties` từ file mẫu [application-local.properties.example](application-local.properties.example):

```powershell
Copy-Item application-local.properties.example application-local.properties
.\mvnw.cmd spring-boot:run
```

- Nếu chỉ muốn test riêng Khối 1 mà chưa bật Khối 2/Khối 4, sửa dòng `utm.integration.mode=mock` trong `application-local.properties`.
- Dữ liệu H2 được lưu tự động tại `./data/utm-v2-db` (đã được `.gitignore` loại khỏi Git).
- `application-local.properties` chứa cấu hình khởi tạo tài khoản mẫu `admin` / `admin123@sky` và `manager` / `manager123@sky`. Với môi trường thật, hãy đổi mật khẩu (tối thiểu 12 ký tự) hoặc xóa 4 dòng `utm.bootstrap.*` sau lần chạy đầu.

### Cách B: Dùng SQL Server + SSMS (Tùy chọn)

1. Mở SSMS, tạo database `utm`:
   ```sql
   CREATE DATABASE utm;
   GO
   USE utm;
   GO
   ```
2. Chạy một lần file [sqlserver-schema.sql](src/main/resources/db/sqlserver-schema.sql) trên database `utm`. (Nếu nâng cấp database cũ chưa có cột hồ sơ doanh nghiệp, xem thêm [sqlserver-business-registration.sql](src/main/resources/db/sqlserver-business-registration.sql).)
3. Trong `application-local.properties`, bỏ comment phần `spring.profiles.active=sqlserver`, `UTM_DB_URL`, `UTM_DB_USERNAME`, `UTM_DB_PASSWORD` và điền thông tin tài khoản SQL Authentication của bạn.
4. Chạy `.\mvnw.cmd spring-boot:run`.

---

## 4. Phân quyền & API chính

| Vai trò | Tài khoản mẫu | Quyền hạn chính |
| --- | --- | --- |
| `OPERATOR` | Tự đăng ký trên web | Quản lý UAV của mình, nộp/kích hoạt/hoàn tất/hủy kế hoạch bay, xem nhật ký riêng, xuất CSV, xem telemetry thuộc phạm vi `OWN_ONLY` |
| `MANAGER` | `manager` / `manager123@sky` | Quản lý Geozone tĩnh/động (`NFZ`, `RESTRICTED`, `TFR_UVR`), theo dõi toàn hệ thống (`ALL_SYSTEM`), xem nhật ký & tích hợp |
| `ADMIN` | `admin` / `admin123@sky` | Duyệt/từ chối/thu hồi kế hoạch bay, quản lý người dùng & vai trò, gửi lại bản tin tích hợp |

| Nhóm | Endpoint chính |
| --- | --- |
| Xác thực | `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/auth/me`, `POST /api/auth/logout` |
| Người dùng & Vai trò | `/api/users`, `/api/roles` |
| Phương tiện & Kế hoạch | `/api/drones`, `/api/flights`, `POST /api/flights/batch`, `PUT /api/flights/{id}/status`, `PUT /api/flights/{id}/revoke`, `GET /api/flights/export.csv` |
| Vùng cấm / Hạn chế bay | `/api/geozones` |
| Nhật ký & Bàn giao | `/api/logs`, `GET /api/logs/my`, `GET /api/flights/{id}/integration`, `GET /api/integrations`, `POST /api/integrations/{id}/retry` |
| Tích hợp Khối 2 & Khối 4 | `GET /api/integration/registered-uavs`, `POST /api/integration/monitoring-updates`, `POST /api/integration/monitoring-decisions`, `GET /api/integration/monitoring-feed` |

---

## 5. Chạy kiểm thử tự động

```powershell
.\mvnw.cmd test
```

*(Bộ test sử dụng H2 in-memory độc lập, không ảnh hưởng tới database đang chạy.)*
