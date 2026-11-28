# Khối 2 — Bộ Mô phỏng PX4 SITL 6-DOF / MAVLink & SQLite (Port 8001)

Module mô phỏng chuyến bay UAV khu vực Đà Nẵng (6-DOF PX4 SITL + MAVLink + SQLite + Dashboard 3D), kết nối vòng kín trực tiếp với **Khối 1 (`main` `:8080`)** và **Khối 4 (`3D` `:3000` & `:8000`)**.

| Nhánh Git | Vai trò | Cổng (Port) |
| --- | --- | --- |
| `main` | **Khối 1 — SkyPermit UTM**: Đăng ký UAV, nộp & duyệt kế hoạch bay 4D, Geozone | `8080` |
| `simulation` (nhánh này) | **Khối 2 — Bộ mô phỏng PX4 SITL**: Tự đồng bộ UAV từ Khối 1, chạy vật lý 6-DOF, phát telemetry 10Hz | `8001` |
| `3D` | **Khối 4 — Giám sát 3D & Chiến thuật**: Tiếp nhận kế hoạch bay (`:8000`), hiển thị 3D & điều khiển MAVLink (`:3000`) | `8000` & `3000` |

---

## 1. Yêu cầu môi trường

- **Python 3.10 trở lên** (`python --version`)
- **Git**
- *(Tùy chọn)* **Docker** hoặc máy ảo **Ubuntu** (để chạy kèm container / môi trường PX4/Gazebo SDF).

---

## 2. Cài đặt & Chạy riêng Khối 2 trên Python

### Bước 1: Clone nhánh `simulation` và cài thư viện

```powershell
git clone -b simulation --single-branch https://github.com/vuduyvuong/UTM.git utm-khoi2
cd utm-khoi2
python -m pip install -r requirements.txt
```

Danh sách thư viện trong [requirements.txt](requirements.txt) gồm `fastapi`, `uvicorn[standard]`, `pydantic`, `pytest`, `httpx`.

### Bước 2: Khởi động server mô phỏng (Port 8001)

```powershell
python -m uvicorn khoi2_px4_simulator.main:app --host 0.0.0.0 --port 8001
```

Mở trình duyệt tại http://localhost:8001 để xem giao diện 3D Đà Nẵng và điều khiển UAV mô phỏng.

---

## 3. Biến môi trường & Endpoint chính

| Biến môi trường | Mặc định | Mô tả |
| --- | --- | --- |
| `PORT` | `8001` | Cổng chạy máy chủ FastAPI của Khối 2 |
| `VUONG_REGISTERED_UAVS_URL` | `http://127.0.0.1:8080/api/integration/registered-uavs` | Endpoint tự động đồng bộ danh sách UAV đã đăng ký từ Khối 1 mỗi 5 giây |

| Endpoint / WebSocket | Chức năng |
| --- | --- |
| `GET /` | Giao diện Dashboard 3D & điều khiển PX4/MAVLink của Khối 2 |
| `WS /ws/telemetry` | Luồng phát telemetry 10Hz gửi trực tiếp sang Khối 4 (`:3000`) |
| `POST /api/integration/flight-plans` | Tiếp nhận kế hoạch bay đã duyệt / cập nhật trạng thái từ Khối 1 (`:8080`) |
| `POST /api/commands` | Nhận lệnh điều khiển MAVLink trực tiếp (`TAKEOFF`, `RESUME_MISSION`, `HOLD_POSITION`, `RTL`, `LAND`, `CHANGE_ALTITUDE`, `CHANGE_SPEED`, `CHANGE_HEADING`) |
| `POST /api/sync-vuong` | Đồng bộ thủ công ngay danh sách UAV đã đăng ký từ Khối 1 |

---

## 4. Chạy bằng Docker / Ubuntu VM (Tùy chọn)

- **Docker** (dùng [Dockerfile.khoi2](khoi2_px4_simulator/Dockerfile.khoi2)):
  ```bash
  docker build -f khoi2_px4_simulator/Dockerfile.khoi2 -t utm-khoi2-ubuntu .
  docker run -d --name utm-khoi2-vm -p 8001:8001 utm-khoi2-ubuntu
  ```
- **Máy ảo Ubuntu** (dùng [run_in_ubuntu_vm.sh](khoi2_px4_simulator/run_in_ubuntu_vm.sh)):
  ```bash
  bash khoi2_px4_simulator/run_in_ubuntu_vm.sh
  ```

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

## 6. Chạy kiểm thử tự động

Chạy bộ test [test_khoi2_px4_simulator.py](backend/tests/test_khoi2_px4_simulator.py):

```powershell
python -m pytest backend/tests/test_khoi2_px4_simulator.py -v
```
