# So sánh Deployment và StatefulSet trong Kubernetes

Tài liệu này tóm tắt sự khác nhau giữa **Deployment** và **StatefulSet**, kèm bảng so sánh trực quan để chọn đúng loại cho từng ứng dụng.

---

## 1. Khái niệm

### Deployment

- Dùng cho **stateless app**.
- Mỗi pod là **instance giống hệt nhau**, không cần danh tính cố định.
- Không quan tâm pod nào chạy ở đâu; chỉ cần đủ số lượng replica.
- Thích hợp cho:
  - Web API, backend, frontend, worker queue stateless…

### StatefulSet

- Dùng cho **stateful app** (có dữ liệu / state gắn với từng instance).
- Mỗi pod có **danh tính ổn định**: `app-0`, `app-1`, `app-2`…
- Mỗi replica thường có **PVC riêng** (`data-app-0`, `data-app-1`…).
- Thích hợp cho:
  - Database (MySQL, PostgreSQL, MongoDB…)
  - Kafka, Zookeeper, Redis cluster…

---

## 2. Bảng so sánh Deployment vs StatefulSet

| Thuộc tính                          | Deployment                                           | StatefulSet                                                   |
|-------------------------------------|------------------------------------------------------|----------------------------------------------------------------|
| Mục đích chính                      | Stateless app                                        | Stateful app                                                   |
| Danh tính pod                       | Không ổn định (pod name random, thay đổi)           | Ổn định: `name-0`, `name-1`, …                                |
| Thứ tự tạo / xóa pod                | Không đảm bảo                                       | Có thứ tự (tạo `0 → 1 → 2`, xóa `2 → 1 → 0`)                  |
| Quản lý volume                      | Thường dùng chung PVC hoặc không dùng PVC           | Mỗi pod 1 PVC riêng (qua `volumeClaimTemplates`)              |
| Khả năng gắn mỗi pod với 1 volume riêng | Không có sẵn, phải tự cấu hình                      | Hỗ trợ sẵn (pod-0 ↔ PVC-0, pod-1 ↔ PVC-1, …)                  |
| Use-case điển hình                  | Web/API, worker stateless, frontend                  | DB, message queue, dịch vụ cần identity cố định                |
| Tự động scale (HPA)                 | Rất phù hợp, dễ dùng                                | Kỹ thuật có thể, nhưng cần cẩn thận (cluster logic riêng)     |
| Restart pod                         | Pod mới có thể tên khác, không gắn với data cũ      | Pod mới giữ nguyên tên & PVC tương ứng                        |
| Headless Service cho DNS từng pod   | Thường không dùng                                   | Dùng nhiều (`svc-headless` + DNS `app-0.svc`, `app-1.svc`)    |
| Thay đổi số replica                 | Chỉ đơn giản là thêm/bớt pod giống nhau             | Thêm/bớt node trong cluster stateful, thường cần operator     |

---

### 3. Khi nào dùng cái nào?
Dùng Deployment khi:

App không cần lưu state trên local pod.
Dữ liệu chính nằm ở DB/Redis/S3 bên ngoài.
Cần scale ngang linh hoạt, autoscale theo HPA.
Dùng StatefulSet khi:

Mỗi instance cần danh tính riêng + volume riêng.
App là DB/queue/cluster cần ổn định node ID.
Cần đảm bảo restart/scaling không làm “lạc” dữ liệu gắn với từng pod.
