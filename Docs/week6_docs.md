# Báo cáo Kiểm thử Khôi phục Thảm họa (DR Test Report): One-way Journal 

## 1. Phạm vi và Môi trường

* **Chế độ:** Nhật ký một chiều (chế độ pool, chỉ truyền `tx-only` trên Cluster 1, chỉ nhận `rx-only` trên Cluster 2 ở trạng thái cơ sở).
* **Các kịch bản:** Sập cụm chính (Primary site down, bao gồm chu trình chuyển đổi dự phòng toàn diện và chuyển đổi ngược lại) và Phân vùng mạng (Network partition).
* **Cluster 1 (Chính):** `13.212.58.79` (`ceph-a1`), fsid `850bf3b8-b0ec-11f1-86b6-413027b51323`.
* **Cluster 2 (Phụ):** `13.250.115.83` (`ceph-b1`), fsid `d125dc36-b0f2-11f1-b04c-51dabfe0935c`.
* **Nút OpenStack/DevStack:** `18.141.184.213` (`ip-172-31-27-114`), Cinder backend `ceph` trên Cluster 1.
* **Ổ đĩa (Volume):** `journal-cinder-test-vol` (`5e99b66f-dbcf-43ce-848a-ec9be838ea71`), được tạo trước đó bằng lệnh `openstack volume create --image cirros --type ceph`.

---

## 2. Phương pháp

### Phương pháp kiểm tra tính toàn vẹn

Kernel RBD (krbd) không thể ánh xạ các image chế độ nhật ký (journal-mode), và `rbd-nbd` chưa được cài đặt trên nút này. Thay vào đó, sử dụng một đoạn mã Python nhỏ trên nút OpenStack để đọc và ghi trực tiếp thông qua `librbd` với các binding `rados`/`rbd`. Mỗi lần kiểm tra đều dùng phép so sánh mã SHA-256 với một tệp nguồn đã biết. Công cụ này yêu cầu truyền keyring một cách tường minh, điều đã khắc phục sau khi lần đọc đầu tiên từ Cluster 2 bị từ chối.

---

## 3. Kịch bản 1: Sập Cụm Chính (Primary Site Down)

* **Trạng thái cơ sở (Baseline):** Một khối dữ liệu ngẫu nhiên 32 MiB được ghi vào offset 0 thông qua đường dẫn Cinder. Checksum khớp trên Cluster 1, và sau khi đồng bộ cũng khớp trên Cluster 2.
* **Thời điểm xảy ra sự cố:** Dữ liệu 64 MiB được ghi tại offset 32 MiB, sau đó Cluster 1 bị dừng nhanh nhất có thể ngay sau khi ghi. Cluster 2 báo cáo độ trễ bằng 0 ngay trước khi dừng, nhưng checksum sau đó cho thấy một phần dữ liệu này chưa kịp nhân bản.

### Dòng thời gian

| Thời gian (UTC) | Sự kiện |
| --- | --- |
| `15:33:09` | Ghi dữ liệu 64 MiB đang dang dở (in-flight) |
| `15:33:13.557` | Cluster 1 bị dừng (`systemctl stop` trên target ceph) |
| `15:33:22` | Chặn (blocklist) tiến trình theo dõi cũ (stale watcher), sau đó chạy `rbd mirror image promote --force` trên Cluster 2 |
| `15:33:24.070` | Nâng quyền thành công (khoảng 10.5 giây sau khi dừng) |
| `15:33:29.9 – 15:33:30.7` | Chuyển tệp `ceph.conf` của OpenStack sang Cluster 2 |
| `15:33:34` | Xác thực các thao tác đọc thông qua Cinder thành công |
| `15:35:00` | Lần ghi đầu tiên qua Cinder thành công, sau khi đã khắc phục sự cố khóa (lock) mô tả bên dưới |

### Kết quả kiểm tra dữ liệu đọc lại

* **Baseline (32 MiB):** Toàn vẹn.
* **Dữ liệu dang dở, in-flight (64 MiB):** 14/16 khối toàn vẹn. 8 MiB cuối cùng bị mất.


### Kết quả tổng hợp

| Chỉ số | Giá trị | Ghi chú |
| --- | --- | --- |
| **RPO** | 8 MiB | Hai khối 4 MiB cuối cùng của dữ liệu dang dở chưa được nhân bản |
| **RTO (chỉ nâng quyền)** | Khoảng 10.5 giây | |
| **RTO (có thể ghi qua Cinder)** | Khoảng 1 phút 47 giây | Phần lớn thời gian dành cho việc xử lý sự cố khóa |
| **Trạng thái OpenStack** | Luôn `available` | Kể cả khi cụm chính sập |

---

## 4. Chuyển đổi ngược lại (Failback)

### Dòng thời gian

| Thời gian (UTC) | Sự kiện |
| --- | --- |
| `15:35:10` | Cluster 1 khởi động lại, đạt trạng thái `HEALTH_OK` lúc `15:35:35` |
| `~15:36` | Hạ cấp (demote) bản sao chính cũ trên Cluster 1. Không có tiến trình theo dõi nào nên không cần blocklist |
| `~15:36` | Cả hai đầu ngang hàng (peers) được cấu hình thành `rx-tx` và yêu cầu đồng bộ lại (resync) trên Cluster 1 |
| `~15:37` | Độ trễ nhân bản trên Cluster 1 về 0 |
| `15:37:45` | Thực hiện hạ cấp có kiểm soát (graceful demote) trên Cluster 2 |
| `15:37:47` | Lần nâng quyền đầu tiên trên Cluster 1 bị từ chối: lệnh hạ cấp chưa lan truyền kịp |
| `15:38:08` | Nâng quyền trên Cluster 1 thành công sau khi thử lại |
| `15:38:16` | Khôi phục tệp `ceph.conf` của OpenStack về Cluster 1 |
| `15:38:21` | Xác thực checksum thông qua Cinder |

### Xác thực sau khi failback

Baseline và dấu mốc được ghi trong thời gian sự cố đều khớp. Dữ liệu dang dở vẫn giữ nguyên 14/16 khối toàn vẹn, không có thêm dữ liệu nào bị mất trong quá trình failback.

### Kết quả tổng hợp

| Chỉ số | Giá trị |
| --- | --- |
| **Thời gian failback** (từ lúc khởi động Cluster 1 đến khi khôi phục Cinder) | Khoảng 3 phút 06 giây |
| **Tổng thời gian** (từ lúc sập cụm đến khi failback hoàn tất) | Khoảng 5 phút 08 giây |

---

## 5. Kịch bản 2: Phân vùng Mạng (Network Partition)

Không cần chuyển đổi dự phòng; bài kiểm tra này đánh giá hành vi nhân bản và tính toàn vẹn dữ liệu.

### Dòng thời gian

| Thời gian (UTC) | Sự kiện |
| --- | --- |
| `15:39:00` | Máy chủ chạy daemon của Cluster 2 bị đưa vào danh sách chặn trên Cluster 1 |
| `15:39:03.7 – 15:39:07.7` | Ghi 64 MiB qua Cinder trong khoảng 4 giây, OpenStack vẫn hoạt động |
| `15:39:11` | Cluster 2 báo cáo trạng thái `up+error` |
| `15:39:20.9` | Gỡ bỏ blocklist |
| `15:40:00` | Bản sao chuyển sang trạng thái `up+replaying`, khoảng 39 giây sau khi gỡ blocklist |
| `15:40:21.96` | Hàng đợi (backlog) gồm 4.108 mục đã được xử lý về 0, khoảng 61 giây sau khi gỡ blocklist |

### Kết quả tổng hợp

| Chỉ số | Giá trị | Ghi chú |
| --- | --- | --- |
| **RPO** | 0 | Dữ liệu ghi trong thời gian phân vùng đã đến nơi an toàn. SHA-256 khớp với nguồn và baseline vẫn khớp trên bản sao |
| **RTO** | Khoảng 39 giây / 61 giây | 39 giây để kết nối lại, 61 giây để bắt kịp hoàn toàn |
| **Tác động I/O cụm chính** | Không có | |
| **Sức khỏe Cluster 1** | `HEALTH_OK` | Duy trì suốt quá trình |

---

## 6. Tổng kết theo Vòng đời DR

| Giai đoạn | Trạng thái | Ghi chú |
| --- | --- | --- |
| **Nhân bản (Replication)** | Hoàn tất | Nhật ký một chiều, đã xác thực đồng bộ và checksum |
| **Failover, tầng Ceph** | Hoàn tất | Nâng quyền bắt buộc (`--force`), khoảng 10.5 giây |
| **Failover, tầng OpenStack** | Hoàn tất | Đọc và ghi thông qua đường dẫn Cinder |
| **Failback** | Hoàn tất | Hạ cấp và nâng quyền có kiểm soát, đã xác thực dữ liệu |
| **Kiểm thử** | Hoàn tất cả 2 kịch bản | Đã ghi nhận các giá trị checksum, RPO và RTO |
| **Tính liên tục cấp Guest** | Chưa thực hiện | Chưa khởi động máy ảo (instance), các mạng đã bị xóa trước đó |
| **Giám sát & Cảnh báo** | Chưa thực hiện | Sự cố không hiển thị qua lệnh `ceph -s` và `openstack volume show` |

---

## 7. Các mục tồn đọng và lưu ý

1. **Định danh `client.cinder` trên Cluster 2:** hiện vẫn tồn tại. Xóa bằng lệnh `ceph auth rm client.cinder` nếu không muốn giữ lại.
2. **Tệp tạm còn lại trên nút OpenStack:** `/etc/ceph/ceph.cluster2.conf`, `/etc/ceph/ceph.cluster1.conf.bak`, và `/tmp/payload*.bin`.
3. **Điểm chưa rõ:** Lệnh `rbd journal info` trả về lỗi `ENOENT` trên image nhân bản nhật ký trong khi trạng thái nhân bản vẫn khỏe mạnh (chưa đào sâu tìm hiểu điểm này).
4. **Thống kê Cinder cũ:** Sau khi đổi tệp cấu hình, lệnh `openstack volume backend pool list` vẫn hiển thị Cluster 1. Hãy xác thực quá trình failover bằng cách đọc trực tiếp qua RBD thay vì dùng lệnh đó.

---

