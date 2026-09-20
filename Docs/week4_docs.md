# One-way vs Two-way Mirror 

![one-way](image-11.png)
- **One-way RBD Mirroring** là cơ chế sao chép dữ liệu một chiều từ RBD image thuộc Primary Ceph cluster sang Non-primary image thuộc Secondary Ceph cluster. Client chỉ thực hiện thao tác ghi trên image Primary, trong khi image Secondary được duy trì như một bản sao dự phòng và không cho phép client ghi trực tiếp. Thành phần rbd-mirror được triển khai tại Secondary cluster, chịu trách nhiệm kết nối đến Primary cluster, thu thập các thay đổi của image và áp dụng chúng lên bản sao tại Secondary. Mô hình này hỗ trợ triển khai nhiều Secondary cluster để phục vụ các kịch bản Disaster Recovery và tăng khả năng bảo vệ dữ liệu.

![two_way](image-12.png)
- **Two-way RBD Mirroring** là cơ chế đồng bộ dữ liệu giữa hai Ceph cluster, cho phép chuyển đổi hướng replication dựa trên vai trò Primary của RBD image. Khác với One-way Mirroring, cả hai cluster đều phải triển khai rbd-mirror daemon để hỗ trợ việc promote và demote image trên từng cluster. Khi image tại Site A là Primary, dữ liệu được đồng bộ từ A sang B. Khi xảy ra failover và image tại Site B được promote thành Primary, các thay đổi mới có thể được thực hiện tại Site B và đồng bộ ngược về Site A. Cơ chế này hỗ trợ xây dựng hệ thống Disaster Recovery với khả năng failover và failback giữa hai site, đồng thời yêu cầu kiểm soát chặt chẽ quyền ghi để tránh tình trạng split-brain
# Benchmark Results
# Kiến trúc 
    - 2 site, mỗi site 2 VM
    - 1VM: 2vCPU, 4GiB Memory, OS Disk 20GB, OSD 30GB/VM 

# Phương pháp Benchmark:
    - Công cụ : FIO 
    - Block size: 4 KiB
    - iodepth: 16
    - numjobs: 4
    - Thời gian: 60 giây
    - Các workload: Random read, random write 
    - Các chỉ số đo: IOPS, throughput, latency, CPU, I/O wait, replication lag.
## 1. One-way Mirroring

### Normal Write
| Chỉ số đo | Giá trị |
| :--- | :--- |
| **IOPS Trung bình** | 1,221 |
| **Throughput (Băng thông)** | 4,613 KiB/s |
| **Latency Trung bình** | 58.75 ms |
| **Latency Tail (p95)** | 144.00 ms |
| **Latency Tail (p99)** | 222.00 ms |
| **CPU FIO Client (usr / sys)** | 0.20% / 0.09% |
| **CPU Compute/Busy Node** | 59.81% |
| **CPU I/O Wait (%iowait)** | 16.51% |
| **Replication Lag** | 0s (Idle / In Sync) |

![Time series metrics - One-way Normal Write](./img/plot_nosnap_write.png)
*Hình: Chuỗi thời gian (time series) các metric kịch bản Normal Write.*

---

### Normal Read
| Chỉ số đo | Giá trị |
| :--- | :--- |
| **IOPS Trung bình** | 28.9k |
| **Throughput (Băng thông)** | 113 MiB/s |
| **Latency Trung bình** | 2.12 ms |
| **Latency Tail (p95)** | 3.62 ms |
| **Latency Tail (p99)** | 5.08 ms |
| **CPU FIO Client (usr / sys)** | 2.16% / 1.40% |
| **CPU Compute/Busy Node** | 94.04% |
| **CPU I/O Wait (%iowait)** | 0.28% |
| **Replication Lag** | 0s (Idle / In Sync) |

![Time series metrics - One-way Normal Read](./img/plot_nosnap_read.png)
*Hình: Chuỗi thời gian (time series) các metric kịch bản Normal Read.*

---
#  RBD Mirror One-Way Jounal 
- Kiểm thử cơ chế đồng bộ dữ liệu một chiều từ Site A sang Site B bằng phương pháp Journal-based 
    - Site A: Primary — cho phép ghi dữ liệu
    - Site B: Non-primary — chỉ nhận dữ liệu được đồng bộ
    - Hướng đồng bộ: Site A → Site B

- Triển khai:
    - Tạo một RBD image mới dành riêng cho bài kiểm thử Journal trên Site A.
    - Cấu hình kết nối và thiết lập quan hệ peer giữa hai Ceph cluster.
    - Cấu hình pool rbd ở chế độ image mirroring.
    - Bật Journal-based Mirroring cho image trên Site A.
    - Triển khai rbd-mirror daemon và kiểm tra trạng thái đồng bộ.
    - Chạy FIO trên image Primary tại Site A trong 60 giây
### Journal Write
| Chỉ số đo | Giá trị |
| :--- | :--- |
| **IOPS Trung bình** | 47 |
| **Throughput (Băng thông)** | 81.1 KiB/s |
| **Latency Trung bình** | 3,668.53 ms |
| **Latency Tail (p95)** | 8,423.00 ms |
| **Latency Tail (p99)** | 13,892.00 ms |
| **CPU FIO Client (usr / sys)** | 0.01% / 0.00% |
| **CPU Compute/Busy Node** | 24.73% |
| **CPU I/O Wait (%iowait)** | 32.18% |
| **Replication Lag** | 0 entries behind (In Sync) |

![Time series metrics - One-way Journal Write](./img/plot_journal_write.png)
*Hình: Chuỗi thời gian (time series) các metric kịch bản Journal Write.*

---

### Journal Read
| Chỉ số đo | Giá trị |
| :--- | :--- |
| **IOPS Trung bình** | 150 |
| **Throughput (Băng thông)** | 334 KiB/s |
| **Latency Trung bình** | 1,023.39 ms |
| **Latency Tail (p95)** | 6,140.46 ms |
| **Latency Tail (p99)** | 8,220.84 ms |
| **CPU FIO Client (usr / sys)** | 0.01% / 0.01% |
| **CPU Compute/Busy Node** | 23.32% |
| **CPU I/O Wait (%iowait)** | 21.12% |
| **Replication Lag** | 0 entries behind (In Sync) |

![Time series metrics - One-way Journal Read](./img/plot_journal_read.png)
*Hình: Chuỗi thời gian (time series) các metric kịch bản Journal Read.*

---
#  RBD Mirror One-Way Snapshot
- Triển khai 
    - Tạo một RBD image mới dành riêng cho bài kiểm thử Snapshot trên Site A
    - Bật Snapshot-based Mirroring cho image trên Site A.
    - Thực hiện Snapshot sau 10s
    - Chạy FIO trên image Primary tại Site A trong 60 giây.

### Snap Write
| Chỉ số đo | Giá trị |
| :--- | :--- |
| **IOPS Trung bình** | 1,138 |
| **Throughput (Băng thông)** | 4,375 KiB/s |
| **Latency Trung bình** | 75.00 ms |
| **Latency Tail (p95)** | 163.00 ms |
| **Latency Tail (p99)** | 617.00 ms |
| **CPU FIO Client (usr / sys)** | 0.17% / 0.06% |
| **CPU Compute/Busy Node** | 54.57% |
| **CPU I/O Wait (%iowait)** | 14.43% |
| **Replication Lag** | 0s (Idle / In Sync) |

![Time series metrics - One-way Snap Write](./img/plot_snap_write.png)
*Hình: Chuỗi thời gian (time series) các metric kịch bản Snap Write.*

---

### Snap Read
| Chỉ số đo | Giá trị |
| :--- | :--- |
| **IOPS Trung bình** | 23.7k |
| **Throughput (Băng thông)** | 92.6 MiB/s |
| **Latency Trung bình** | 2.28 ms |
| **Latency Tail (p95)** | 3.95 ms |
| **Latency Tail (p99)** | 6.39 ms |
| **CPU FIO Client (usr / sys)** | 2.11% / 1.34% |
| **CPU Compute/Busy Node** | 93.74% |
| **CPU I/O Wait (%iowait)** | 0.32% |
| **Replication Lag** | 0s (Idle / In Sync) |

![Time series metrics - One-way Snap Read](./img/plot_snap_read.png)
*Hình: Chuỗi thời gian (time series) các metric kịch bản Snap Read.*

---

## 2. Two-way Mirroring

### Normal Write
| Chỉ số đo | Site A | Site B |
| :--- | :--- | :--- |
| **IOPS Trung bình** | 1,456 | 1,555 |
| **Throughput (Băng thông)** | 5,727 KiB/s | 6,084 KiB/s |
| **Latency Trung bình** | 58.08 ms | 54.84 ms |
| **Latency Tail (p95)** | 140.00 ms | 140.00 ms |
| **Latency Tail (p99)** | 213.00 ms | 215.00 ms |
| **CPU FIO Client (usr / sys)** | 0.21% / 0.08% | 0.20% / 0.08% |
| **CPU Compute/Busy Node** | 70.58% | 57.67% |
| **CPU I/O Wait (%iowait)** | 12.08% | 16.84% |
| **Replication Lag** | 0s (Idle / In Sync) | 0s (Idle / In Sync) |

| Biểu đồ Site A | Biểu đồ Site B |
| :---: | :---: |
| ![Site A](./img/plot_nosnap_write_2way-snap-a.png) | ![Site B](./img/plot_nosnap_write_2way-snap-b.png) |
| *Time series metrics: Site A* | *Time series metrics: Site B* |

---

### Normal Read
| Chỉ số đo | Site A | Site B |
| :--- | :--- | :--- |
| **IOPS Trung bình** | 6,013 | 5,890 |
| **Throughput (Băng thông)** | 23.4 MiB/s | 22.9 MiB/s |
| **Latency Trung bình** | 2.46 ms | 2.40 ms |
| **Latency Tail (p95)** | 3.82 ms | 3.75 ms |
| **Latency Tail (p99)** | 9.37 ms | 12.26 ms |
| **CPU FIO Client (usr / sys)** | 1.94% / 1.21% | 1.98% / 1.19% |
| **CPU Compute/Busy Node** | 86.04% | 84.66% |
| **CPU I/O Wait (%iowait)** | 0.74% | 0.68% |
| **Replication Lag** | 0s (Idle / In Sync) | 0s (Idle / In Sync) |

| Biểu đồ Site A | Biểu đồ Site B |
| :---: | :---: |
| ![Site A](./img/plot_nosnap_read_2way-snap-a.png) | ![Site B](./img/plot_nosnap_read_2way-snap-b.png) |
| *Time series metrics: Site A* | *Time series metrics: Site B* |

---
# RBD Mirroring Two-Way Journal 
- Kiểm thử cơ chế đồng bộ dữ liệu hai chiều từ Site A sang Site B bằng phương pháp Journal-based 
    - image-site-a: Primary tại Site A → đồng bộ sang Site B.
    - image-site-b: Primary tại Site B → đồng bộ sang Site A.
    - Hướng đồng bộ: Site A <-> Site B
    - Hai image hoạt động độc lập, cho phép kiểm thử đồng bộ theo cả hai hướng đồng thời.

- Triển khai:
    - Tạo hai RBD image riêng biệt, mỗi site một image dành cho bài kiểm thử Journal.
    - Cấu hình kết nối và thiết lập quan hệ peer giữa hai Ceph cluster.
    - Cấu hình pool rbd ở chế độ image mirroring.
    - Bật Journal-based Mirroring cho cả hai image.
    - Thiết lập vai trò Primary:    
        - image-site-a trên Site A.
        - image-site-b trên Site B
    - Triển khai rbd-mirror daemon trên cả hai site.
    - Chạy FIO trên image Primary tại Site A trong 60 giây
### Journal Write
| Chỉ số đo | Site A | Site B |
| :--- | :--- | :--- |
| **IOPS Trung bình** | 40 | 149 |
| **Throughput (Băng thông)** | 67.6 KiB/s | 449 KiB/s |
| **Latency Trung bình** | 4,358.31 ms | 1,277.61 ms |
| **Latency Tail (p95)** | 8,658.00 ms | 3,540.00 ms |
| **Latency Tail (p99)** | 17,113.00 ms | 5,671.00 ms |
| **CPU FIO Client (usr / sys)** | 0.00% / 0.01% | 0.00% / 0.00% |
| **CPU Compute/Busy Node** | 26.10% | 33.54% |
| **CPU I/O Wait (%iowait)** | 44.62% | 42.03% |
| **Replication Lag** | 0 entries behind (In Sync) | 0 entries behind (In Sync) |

| Biểu đồ Site A | Biểu đồ Site B |
| :---: | :---: |
| ![Site A](./img/plot_journal_write_2way-journal-a.png) | ![Site B](./img/plot_journal_write_2way-journal-b.png) |
| *Time series metrics: Site A* | *Time series metrics: Site B* |

---

### Journal Read
| Chỉ số đo | Site A | Site B |
| :--- | :--- | :--- |
| **IOPS Trung bình** | 107 | 212 |
| **Throughput (Băng thông)** | 216 KiB/s | 617 KiB/s |
| **Latency Trung bình** | 1,040.97 ms | 469.03 ms |
| **Latency Tail (p95)** | 5,939.14 ms | 2,634.02 ms |
| **Latency Tail (p99)** | 9,596.57 ms | 3,707.76 ms |
| **CPU FIO Client (usr / sys)** | 0.01% / 0.01% | 0.01% / 0.01% |
| **CPU Compute/Busy Node** | 28.64% | 32.22% |
| **CPU I/O Wait (%iowait)** | 39.86% | 6.22% |
| **Replication Lag** | 0 entries behind (In Sync) | 0 entries behind (In Sync) |

| Biểu đồ Site A | Biểu đồ Site B |
| :---: | :---: |
| ![Site A](./img/plot_journal_read_2way-journal-a.png) | ![Site B](./img/plot_journal_read_2way-journal-b.png) |
| *Time series metrics: Site A* | *Time series metrics: Site B* |

---
# RBD Mirroring Two-Way Snapshot
- Kiểm thử cơ chế đồng bộ dữ liệu hai chiều từ Site A sang Site B bằng phương pháp Snapshot-based 
    - image-site-a: Primary tại Site A → đồng bộ sang Site B.
    - image-site-b: Primary tại Site B → đồng bộ sang Site A.
    - Hướng đồng bộ: Site A <-> Site B
    - Hai image hoạt động độc lập, cho phép kiểm thử đồng bộ theo cả hai hướng đồng thời.

- Triển khai:
    - Tạo hai RBD image riêng biệt, mỗi site một image dành cho bài kiểm thử Snapshot.
    - Cấu hình kết nối và thiết lập quan hệ peer giữa hai Ceph cluster.
    - Cấu hình pool rbd ở chế độ image mirroring.
    - Bật Snapshot-based Mirroring cho cả hai image.
    - Thiết lập vai trò Primary:    
        - image-site-a trên Site A.
        - image-site-b trên Site B
    - Triển khai rbd-mirror daemon trên cả hai site.
    - Chạy FIO trên image Primary tại Site A trong 60 giây
### Snapshot Write
| Chỉ số đo | Site A | Site B |
| :--- | :--- | :--- |
| **IOPS Trung bình** | 1,102 | 1,144 |
| **Throughput (Băng thông)** | 4,289 KiB/s | 4,490 KiB/s |
| **Latency Trung bình** | 71.26 ms | 72.41 ms |
| **Latency Tail (p95)** | 161.00 ms | 155.00 ms |
| **Latency Tail (p99)** | 275.00 ms | 380.00 ms |
| **CPU FIO Client (usr / sys)** | 0.18% / 0.06% | 0.15% / 0.07% |
| **CPU Compute/Busy Node** | 68.01% | 53.60% |
| **CPU I/O Wait (%iowait)** | 10.61% | 13.83% |
| **Replication Lag** | 0s (Idle / In Sync) | Syncing (Lag: 0s, Duration: 0.0s) |

| Biểu đồ Site A | Biểu đồ Site B |
| :---: | :---: |
| ![Site A](./img/plot_snap_write_2way-snap-a.png) | ![Site B](./img/plot_snap_write_2way-snap-b.png) |
| *Time series metrics: Site A* | *Time series metrics: Site B* |

---

### Snapshot Read
| Chỉ số đo | Site A | Site B |
| :--- | :--- | :--- |
| **IOPS Trung bình** | 5,993 | 5,882 |
| **Throughput (Băng thông)** | 23.4 MiB/s | 22.9 MiB/s |
| **Latency Trung bình** | 2.56 ms | 2.55 ms |
| **Latency Tail (p95)** | 4.29 ms | 4.29 ms |
| **Latency Tail (p99)** | 10.42 ms | 11.21 ms |
| **CPU FIO Client (usr / sys)** | 1.88% / 1.22% | 1.89% / 1.18% |
| **CPU Compute/Busy Node** | 88.50% | 87.29% |
| **CPU I/O Wait (%iowait)** | 0.66% | 0.65% |
| **Replication Lag** | 0s (Idle / In Sync) | 0s (Idle / In Sync) |

| Biểu đồ Site A | Biểu đồ Site B |
| :---: | :---: |
| ![Site A](./img/plot_snap_read_2way-snap-a.png) | ![Site B](./img/plot_snap_read_2way-snap-b.png) |
| *Time series metrics: Site A* | *Time series metrics: Site B* |

# KỂT LUẬN 
- **So sánh Journal-based vs Snapshot-based Mirroring**
    - **Snapshot-based:** Hiệu năng đọc/ghi chỉ giảm 5-10% so với baseline, vì cơ chế này chỉ định kỳ tạo snapshot và đồng bộ phần delta thay vì log từng I/O. Phù hợp với các ứng dụng phổ thông, hệ thống yêu cầu hiệu năng ghi cao và chấp nhận RPO theo chu kỳ lịch trình snapshot.
    - **Journal-based:** IOPS ghi giảm tới ~96% so với baseline, latency tăng từ hàng chục ms lên hàng nghìn ms (p99 có lúc >17s ở kịch bản Two-way). Nguyên nhân là do cơ chế journal phải ghi log tuần tự (double write: ghi vào journal object rồi mới ghi vào data object), tạo ra I/O amplification lớn, thể hiện rõ qua %iowait rất cao (32–45%). Phù hợp với các hệ thống yêu cầu RPO ~ 0 
    - **Trade off:** Journal-based cho RPO gần như bằng 0 (đồng bộ gần thời gian thực) nhưng chi phí hiệu năng rất lớn; Snapshot-based cho hiệu năng gần với native nhưng RPO phụ thuộc vào chu kỳ snapshot (trong bài test là 10s), nghĩa là có thể mất dữ liệu trong khoảng thời gian giữa hai lần snapshot nếu xảy ra sự cố

- **So sánh One-way vs Two-way:**Với cùng một cơ chế (Journal hoặc Snapshot), Two-way không làm suy giảm hiệu năng đáng kể so với One-way ở từng site riêng lẻ — các chỉ số IOPS/latency/iowait của Site A và Site B trong kịch bản Two-way tương đương với kịch bản One-way tương ứng.
    - Two-way mirroring phù hợp cho kịch bản Active-Active hoặc yêu cầu failover/failback linh hoạt giữa hai site, nhưng cần kiểm soát chặt chẽ quyền ghi (tránh split-brain) và tính toán thêm tài nguyên CPU/network dự phòng cho cả hai chiều đồng bộ.

- **Ảnh hưởng đến tài nguyên hệ thống:**
    - CPU Compute/Busy Node ở các kịch bản Normal/Snapshot Read khá cao (86–94%) do đặc thù workload random read 4K với iodepth=16, numjobs=4 — đây là giới hạn của tài nguyên FIO client/test bench hơn là do cơ chế mirroring.
    - %iowait là chỉ số phân biệt rõ nhất giữa hai cơ chế: Journal-based luôn có iowait cao gấp 2–4 lần Snapshot-based, khẳng định nút thắt cổ chai nằm ở việc ghi journal tuần tự.

- **Ưu tiên Snapshot-based mirroring** cho các hệ thống production yêu cầu hiệu năng I/O cao, đặc biệt là ghi (write-intensive workload), chấp nhận RPO ở mức phút/giây tùy chu kỳ snapshot cấu hình
- **Ưu tiên Snapshot-based mirroring** cho các hệ thống production yêu cầu hiệu năng I/O cao, đặc biệt là ghi (write-intensive workload), chấp nhận RPO ở mức phút/giây tùy chu kỳ snapshot cấu hình
- Nếu triển khai Two-way mirroring, nên thử nghiệm thêm ở quy mô tải lớn hơn và nhiều image đồng thời để đánh giá chính xác overhead cộng dồn trước khi đưa vào production.