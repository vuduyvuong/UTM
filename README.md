# Khối 4 — Giám sát Không lưu 3D & Xử lý Xung đột Chiến thuật (Ports 3000 & 8000)

Module **Web 3D + Backend FastAPI** chịu trách nhiệm giám sát UAV thời gian thực trên bản đồ 3D khu vực Đà Nẵng (CesiumJS + Three.js), dự báo quỹ đạo 12s nét đứt (`- - - -`), phát hiện & phân tách xung đột 3D CPA, đồng bộ tiến độ/quyết định về **Khối 1 (`main` `:8080`)** và gửi lệnh điều khiển MAVLink trực tiếp sang **Khối 2 (`simulation` `:8001`)**.

---

## 1. Kiến trúc kết nối 3 nhánh

| Nhánh Git | Vai trò | Cổng (Port) |
| --- | --- | --- |
| `main` | **Khối 1 — SkyPermit UTM**: Đăng ký UAV, nộp & duyệt kế hoạch bay 4D, Geozone | `8080` |
| `simulation` | **Khối 2 — Bộ mô phỏng PX4 SITL**: Mô phỏng 6-DOF, MAVLink, phát telemetry 10Hz qua WebSocket | `8001` |
| `3D` (nhánh này) | **Khối 4 — Giám sát 3D & Chiến thuật**: Tiếp nhận kế hoạch bay (`:8000`), hiển thị 3D & điều khiển (`:3000`) | `8000` & `3000` |

---

## 2. Yêu cầu môi trường

- **Node.js 20 trở lên** (`node -v`, `npm -v`)
- **Python 3.10 trở lên** (`python --version`)
- **Git**

---

## 3. Cài đặt & Chạy riêng Khối 4

### Bước 1: Clone nhánh `3D` và cài thư viện

```powershell
git clone -b 3D --single-branch https://github.com/vuduyvuong/UTM.git utm-khoi4
cd utm-khoi4

# Cài thư viện Frontend
npm install

# Cài thư viện Backend FastAPI (hoặc dùng backend/requirements-dev.txt nếu muốn chạy pytest)
python -m pip install -r backend/requirements.txt
```

### Bước 2: Chạy hệ thống Khối 4 (2 Terminal)

- **Terminal 1 — Backend FastAPI (`:8000`)**:
  ```powershell
  python -m uvicorn backend.app.main:app --host 127.0.0.1 --port 8000
  ```
- **Terminal 2 — Frontend Web 3D (`:3000`)**:
  ```powershell
  npm run dev -- --host 127.0.0.1 --port 3000
  ```

### Bước 3: Truy cập

- **Giao diện Giám sát 3D:** http://127.0.0.1:3000 (hoặc http://localhost:3000)
- **Swagger Backend API:** http://127.0.0.1:8000/docs
- **Healthcheck Backend:** http://127.0.0.1:8000/api/tactical/health

---

## 4. Cấu hình kết nối mặc định

Khi chạy cục bộ cả 3 khối trên cùng máy, **không cần tạo `.env.local`** — Khối 4 mặc định khởi động ở chế độ **LIVE** với các địa chỉ:

- Telemetry WebSocket từ Khối 2: `ws://localhost:8001/ws/telemetry` (`VITE_WS_TELEMETRY_URL`)
- Handoff FlightPlan WebSocket từ Backend Khối 4: `ws://localhost:8000/ws/flight-plans` (`VITE_OPS_WS_URL`)
- Gửi báo cáo tiến độ/quyết định về Khối 1: `http://localhost:8080/api/integration/monitoring-updates` (`VITE_VUONG_OUTBOUND_URL`)

Nếu cần đổi host/port hoặc bật khóa xác thực `UTM_MONITORING_TOKEN`, sao chép file mẫu [.env.example](.env.example) thành `.env.local` và chỉnh sửa theo nhu cầu.

---

## 5. Chạy trọn bộ cả 3 nhánh trên 1 máy (Quickstart)

```powershell
# 1. Clone 3 nhánh
git clone -b main --single-branch https://github.com/vuduyvuong/UTM.git utm-khoi1
git clone -b simulation --single-branch https://github.com/vuduyvuong/UTM.git utm-khoi2
git clone -b 3D --single-branch https://github.com/vuduyvuong/UTM.git utm-khoi4

# 2. Cài đặt 1 lần
Copy-Item .\utm-khoi1\application-local.properties.example .\utm-khoi1\application-local.properties
python -m pip install -r .\utm-khoi2\requirements.txt
python -m pip install -r .\utm-khoi4\backend\requirements.txt
npm --prefix .\utm-khoi4 install

# 3. Chạy trên 4 Terminal:
# Terminal 1 (utm-khoi1): .\mvnw.cmd spring-boot:run
# Terminal 2 (utm-khoi2): python -m uvicorn khoi2_px4_simulator.main:app --host 0.0.0.0 --port 8001
# Terminal 3 (utm-khoi4): python -m uvicorn backend.app.main:app --host 127.0.0.1 --port 8000
# Terminal 4 (utm-khoi4): npm run dev -- --host 127.0.0.1 --port 3000
```

Mở http://localhost:8080 (Khối 1), http://localhost:8001 (Khối 2), và http://localhost:3000 (Khối 4).

---

## 6. Kiểm thử & Build sản xuất

```powershell
# 1. Cài thêm gói test backend (lần đầu)
python -m pip install -r backend/requirements-dev.txt

# 2. Chạy kiểm thử Backend (Pytest)
python -m pytest backend/tests/ -v

# 3. Chạy kiểm thử Frontend & tích hợp (Vitest)
npx vitest run

# 4. Đóng gói kiểm tra TypeScript + Vite Build
npm run build
```
