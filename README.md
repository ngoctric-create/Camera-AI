# Ứng Dụng Tích Hợp Camera AI Chụp Biển Số Xe Vào Phân Hệ Giám Sát Chuyến Xe Buýt (EBMS / DNVT)

> Hệ thống tích hợp giữa **Traffic AI Analytics Platform (Oryza / Sohatech)** và **Phân hệ Quản lý Thu thập Sản lượng - Xác nhận Chuyến Xe Buýt (EBMS / dnvt.ebms.vn)** dành cho Doanh nghiệp Vận tải (FUTABUS, Saigonbus, Samco) và Trung tâm Quản lý Giao thông công cộng TP.HCM.

---

## 1. Giới Thiệu & Bài Toán Nghiệp Vụ

### Thực trạng trước khi tích hợp
- Trong phân hệ giám sát chuyến xe (`dnvt.ebms.vn`), việc xác nhận chuyến (giờ bắt đầu xuất bến, giờ kết thúc cập bến, biển số xe thực hiện) trước đây chủ yếu dựa vào thiết bị định vị GPS hoặc xác nhận thủ công của điều độ viên.
- **Tồn tại nhiều nhược điểm:**
  - Thiết bị GPS thường mất sóng khi vào bến, hầm đậu xe hoặc bị can thiệp tắt thiết bị.
  - Khó kiểm soát việc xe có thực sự đón/trả khách và xuất/cập bến đúng giờ hay không.
  - Khi nghiệm thu sản lượng chuyến có trợ giá nhà nước, thiếu **bằng chứng hình ảnh khách quan** chống gian lận.

### Giải pháp tích hợp Camera AI ANPR
- Tận dụng hệ thống Camera AI nhận dạng biển số tự động (ANPR) đã lắp đặt tại các bến xe buýt (Bến xe buýt Chợ Lớn, Điểm đầu cuối Nguyễn Siêu, Ga Ba Son,...).
- Hệ thống gồm các cụm camera giám sát tại:
  - **Cổng Đi (Xuất bến):** Camera 106, 107 (góc chụp tiếp cận đầu xe, đọc biển số xe xuất phát).
  - **Cổng Về (Cập bến):** Camera 104, 105, 236, 237 (góc chụp rời khỏi / đuôi xe, đọc biển số xe hoàn thành chuyến).
- **Quy trình hoạt động tự động:**
  1. Khi xe buýt lăn bánh qua cổng bến, Camera AI chụp ảnh toàn cảnh + crop biển số, nhận dạng biển số với độ chính xác trên 98%.
  2. Sự kiện được đẩy theo thời gian thực (qua WebSocket / REST API Gateway) về phân hệ đối soát.
  3. **Thuật toán Matching Engine** tự động ghép biển số và khung giờ với biểu đồ chạy xe kế hoạch của Tuyến (ví dụ Tuyến [01] Ga Ba Son - Bến xe buýt Chợ Lớn).
  4. Tự động tính chênh lệch xuất bến/về bến (phút:giây), tự động phát hiện vi phạm (về trễ > 15 phút, chạy sai xe kế hoạch, bỏ chuyến).
  5. Đính kèm bằng chứng ảnh chụp (Visual Proof) vào từng chuyến xe để điều độ viên và cán bộ giám sát kiểm tra và xác nhận chỉ với 1 cú click.

---

## 2. Kiến Trúc Kỹ Thuật

```mermaid
flowchart TD
    subgraph Bến Xe Buýt
        Cam106["Camera 106<br/>(Cổng Đi 1)"]
        Cam107["Camera 107<br/>(Cổng Đi 2)"]
        Cam104["Camera 104<br/>(Cổng Về 1)"]
        Cam105["Camera 105<br/>(Cổng Về 2)"]
        BoxAI["Oryza AI Box / Edge Server<br/>192.168.1.38"]
    end

    subgraph Nền Tảng Traffic AI (Sohatech)
        Keycloak["Keycloak SSO<br/>https://115.78.4.184:82"]
        Gateway["API Gateway / VMS<br/>https://115.78.4.184:4433/gateway"]
        WS["WebSocket Stream<br/>wss://115.78.4.184:81"]
    end

    subgraph Ứng Dụng Tích Hợp Giám Sát Chuyến
        Backend["Backend Service (Node.js)<br/>Port 5001"]
        Engine["Reconciliation & Matching Engine<br/>Đối soát kế hoạch vs Thực tế"]
        Frontend["EBMS Web Interface<br/>React + Tailwind CSS"]
    end

    Cam106 --> BoxAI
    Cam107 --> BoxAI
    Cam104 --> BoxAI
    Cam105 --> BoxAI

    BoxAI --> Gateway
    BoxAI --> WS

    Backend -->|Xác thực Token| Keycloak
    Backend -->|Lấy sự kiện LPR| Gateway
    Backend -->|Nhận diện Realtime| WS
    Backend --> Engine
    Engine --> Frontend
```

---

## 3. Các Tính Năng Chính Của Ứng Dụng

1. **Giao diện chuẩn nghiệp vụ EBMS (`dnvt.ebms.vn`):**
   - Giữ nguyên màu sắc, cấu trúc menu, bảng dữ liệu biểu đồ chạy xe của Trung tâm Quản lý Giao thông công cộng TP.HCM.
   - Quản lý đầy đủ 216 chuyến kế hoạch trong ngày của Tuyến 01 (Ga Ba Son - Bến xe buýt Chợ Lớn).

2. **Tự động điền dữ liệu thực tế từ Camera AI:**
   - Cột **Xe thực hiện**: Tự động điền biển số xe chụp được.
   - Cột **Bắt đầu thực tế**: Ghi nhận từ Camera Cổng Đi (Camera 106, 107).
   - Cột **Kết thúc thực tế**: Ghi nhận từ Camera Cổng Về (Camera 104, 105).
   - Cột **Chênh lệch (phút:giây)**: Tự động tính độ lệch xuất bến và về bến theo chuẩn `(mm:ss)` nếu sớm/trễ.

3. **Cảnh báo vi phạm tự động:**
   - Cảnh báo đỏ đối với chuyến **về trễ > 15 phút** (theo quy định xử phạt / giải trình của Sở GTVT).
   - Cảnh báo xe chạy sai phân công.
   - Bảng riêng quản lý **Chuyến không theo kế hoạch** (xe tăng cường, bảo dưỡng, quay đầu).

4. **Bằng chứng số từ Camera (Visual Proof Modal):**
   - Click vào nút **AI Khớp** trên từng dòng để mở xem:
     - Ảnh chụp toàn cảnh xe lúc xuất bến và cập bến.
     - Ảnh crop cận cảnh biển số xe.
     - Tên camera, vị trí cổng, độ chính xác nhận diện AI (%).
     - Dấu mốc thời gian (timestamp) đến từng giây.

5. **Giám sát Camera Trực Tiếp (Live Stream Grid):**
   - Khung xem trực tiếp 4 camera bến xe (Camera 104, 105, 106, 107).
   - Ticker luồng sự kiện nhận diện xe ra/vào bến thời gian thực.

6. **Xác nhận & Nghiệm thu sản lượng:**
   - Xác nhận từng chuyến hoặc xác nhận hàng loạt ("Xác nhận chuyến kế hoạch").
   - Xuất file báo cáo nghiệm thu sản lượng chuyến (CSV/Excel) chuẩn định dạng.

---

## 4. Hướng Dẫn Cài Đặt & Khởi Chạy

### Yêu cầu hệ thống
- Node.js >= 18.x
- npm >= 9.x

### Khởi chạy ứng dụng
```bash
# 1. Di chuyển vào thư mục dự án
cd "/Library/vibecode/camera AI"

# 2. Khởi chạy toàn bộ ứng dụng (Backend + Frontend tích hợp)
npm start
```

Ứng dụng sẽ hoạt động tại:
👉 **http://localhost:5001**

### Chế độ phát triển (Dành cho Lập trình viên)
```bash
# Terminal 1: Chạy Backend với hot-reload
npm run dev:backend

# Terminal 2: Chạy Frontend Vite Dev Server
npm run dev:frontend
# Truy cập: http://localhost:3000 (tự động proxy API sang port 5001)
```

---

## 5. Danh Sách Endpoint REST API

| Method | Endpoint | Mô tả |
|---|---|---|
| `GET` | `/api/trips` | Lấy danh sách chuyến theo kế hoạch (hỗ trợ lọc theo bến đầu, xe, doanh nghiệp, loại chuyến) |
| `GET` | `/api/trips/off-plan` | Lấy danh sách chuyến không theo kế hoạch ghi nhận qua camera |
| `GET` | `/api/trips/stats` | Thống kê sản lượng (tổng số chuyến, đã xác nhận, đã nghiệm thu, trễ > 15p, đã khớp AI) |
| `POST` | `/api/trips/:id/confirm` | Xác nhận chuyến hợp lệ theo ID |
| `POST` | `/api/trips/confirm-all` | Xác nhận hàng loạt chuyến kế hoạch có bằng chứng Camera AI |
| `POST` | `/api/reconcile` | Kích hoạt đối soát tự động với dữ liệu từ Traffic AI |
| `GET` | `/api/cameras` | Lấy danh sách camera tại bến (Camera 104, 105, 106, 107, 236, 237) |
| `GET` | `/api/live-events` | Lấy danh sách sự kiện nhận diện biển số xe mới nhất |
| `GET` | `/api/proxy-image` | Proxy ảnh chụp camera từ máy chủ AI để tránh lỗi SSL / CORS |
| `GET` | `/api/export-csv` | Xuất báo cáo nghiệm thu sản lượng chuyến ra file CSV |
| `GET` | `/api/status` | Kiểm tra trạng thái kết nối tới hệ thống Traffic AI |
| `GET` | `/api/admin/central-sync/status` | Trạng thái sao lưu trung tâm, số sự kiện chờ gửi, lỗi gần nhất |
| `POST` | `/api/admin/central-sync/config` | Lưu cấu hình URL Apps Script, mã bến, chu kỳ gửi (token tuỳ chọn) |
| `POST` | `/api/admin/central-sync/test` | Kiểm tra kết nối tới Apps Script / Google Sheet / Drive |
| `POST` | `/api/admin/central-sync/run` | Gửi ngay dữ liệu đang chờ lên trung tâm |
| `WS` | `/ws/events` | WebSocket stream đẩy sự kiện nhận diện biển số trực tiếp tới trình duyệt |

---

## 6. Sao Lưu Dự Phòng Lên Trung Tâm (Nhiều Bến)

Mỗi bến cài một bản riêng, dữ liệu tại bến là nguồn chính. Bản sao được gửi một chiều lên Google Sheets + Google Drive qua Apps Script; trung tâm không xem chéo realtime.

- Cấu hình tại giao diện: **Quản trị danh sách xe → tab Sao lưu trung tâm**.
- Chỉ cần URL Apps Script `/exec` và mã bến; **token không bắt buộc** (bật lại bằng `REQUIRE_TOKEN = true` nếu cần).
- Mỗi sự kiện có `event_uid` chống trùng; mất mạng thì giữ hàng đợi và tự gửi bù.
- Chỉ gửi **1 ảnh overview/sự kiện**, nén nhỏ; ảnh trên Drive tự xoá sau **90 ngày**.
- Hướng dẫn tạo Sheet/Apps Script/Web App: [google-apps-script/HUONG_DAN_TRIEN_KHAI.md](google-apps-script/HUONG_DAN_TRIEN_KHAI.md).
- Vận hành thật phải tắt dữ liệu mẫu và mô phỏng:
  - `EBMS_SEED_DATA=0`
  - `EBMS_SIMULATE_EVENTS=0`
  - Các launcher `CHAY_HE_THONG_WINDOWS.bat` và `CHAY_AN_KHONG_CUA_SO.vbs` đã đặt sẵn; chỉ dùng `CHAY_DEMO_WINDOWS.bat` khi kiểm thử.
