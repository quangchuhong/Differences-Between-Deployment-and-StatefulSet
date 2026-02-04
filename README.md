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
 
### stateless app.

- Pod không có danh tính cố định (pod-abc, pod-def… xoá là tạo pod mới tên khác).
- Không đảm bảo thứ tự start/stop pod.
- Scale up/down đơn giản: chỉ cần số replica.
- Thường kết hợp với:
  - Service kiểu ClusterIP/LoadBalancer
  - PVC dùng chung cho nhiều pod chỉ khi app tự xử lý được (ít gặp).
    
**Đặc điểm chính**: 

- Không lưu state quan trọng trên local disk của pod/container:
  - Log, cache tạm thì được; dữ liệu lâu dài thì không.
- Không phụ thuộc session trong memory của pod:
  - Session nên lưu ở Redis, DB, JWT… thay vì chỉ trong RAM của 1 instance.
- Dễ scale ngang:
  - Tăng từ 2 → 10 replica chỉ là thêm nhiều pod giống hệt, không cần setup gì đặc biệt.
 
**Ví dụ stateless app**

- Web API, backend REST/GraphQL:
  - Xử lý request, đọc/ghi dữ liệu qua DB/PostgreSQL/MySQL, trả response.
- Frontend (React/Vue/Angular) serve file tĩnh.
- Worker xử lý hàng đợi (nếu mỗi job tự chứa đủ thông tin và kết quả ghi về DB/S3,…).

### stateful app (có dữ liệu, cần stable identity).

- Mỗi pod có tên cố định theo index:
  - app-0, app-1, app-2…
- Mỗi replica gắn với PVC riêng:
  - data-app-0, data-app-1…
- Khi restart/scaling:
  - Pod app-0 luôn gắn với đúng PVC data-app-0.
- Hỗ trợ:
  - Thứ tự start: từ 0 → n-1
  - Thứ tự stop: ngược lại
- Dùng cho:
  - Database (MySQL, PostgreSQL, MongoDB…)
  - Kafka, Zookeeper, Redis cluster…

**Đặc điểm chính**: 

1. Có dữ liệu gắn với từng instance

- Mỗi instance (node/pod) có dữ liệu riêng trên disk:
  - DB files (MySQL, PostgreSQL, MongoDB…)
  - Log, queue segments (Kafka)
  - Metadata, snapshot, index (Elasticsearch…)
- Dữ liệu này không thể mất khi pod chết.
  
2. Cần identity (danh tính) cố định

- Các instance được phân vai: node-0, node-1, node-2…
- Cluster/clients biết rõ “node nào là ai”.
- Khi restart, node-0 phải quay lại đúng volume của node-0.
  
3. Không thể scale ngang “vô tội vạ” như stateless

- Thêm 1 replica = thêm 1 node vào cluster → cần:
  - Cấu hình join/leave,
  - Rebalance data,
  - Thường cần operator hoặc manual.
    
4. Cần storage persistent

  - Dùng PVC, PV (EBS, local SSD, …) gắn cố định với từng pod.
  - Thường triển khai bằng StatefulSet trong Kubernetes.
    
Ví dụ điển hình stateful app:

- MySQL/PostgreSQL/MongoDB
- Redis (khi dùng persistence, cluster/sentinel)
- Kafka, ZooKeeper
- Elasticsearch, Cassandra, etcd
- 
Ngược lại, stateless app là API/web/worker chỉ xử lý request và lưu state ở nơi khác (DB, Redis, S3…), pod chết hay tạo mới đều không ảnh hưởng state.

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

- Dùng Deployment khi:

  - App không cần lưu state trên local pod.
  - Dữ liệu chính nằm ở DB/Redis/S3 bên ngoài.
  - Cần scale ngang linh hoạt, autoscale theo HPA.
    
- Dùng StatefulSet khi:

  - Mỗi instance cần danh tính riêng + volume riêng.
  - App là DB/queue/cluster cần ổn định node ID.
  - Cần đảm bảo restart/scaling không làm “lạc” dữ liệu gắn với từng pod.
