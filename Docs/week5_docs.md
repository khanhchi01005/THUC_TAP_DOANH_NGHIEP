# GIẢI PHÁP DR CEPH 

# Hạ tầng 
- Ceph cluster 1: gồm 2 OSD, mỗi OSD 30GB 
- Ceph cluster 2: gồm 2 OSD, mỗi OSD 30GB
- Máy ảo cài Openstack: Tạo cinder volume kết nối tới Ceph cluster 1 
- Ceph cluster 1 và cluster 2 được cấu hình rbd mirror với cluster 1 là primary 

# Kịch bản Failover 
- Thực hiện Primary site down 
- Hiện tượng Network Partition giữa 2 Ceph cluster 

# One-way Journal 
# Scenario 1: Primary site failure 
- **Môi trường thử nghiệm**
| Thành phần | Chi tiết |
|---|---|
| **Volume thử nghiệm** | `volumes/volume-907d059a-8501-48cc-8672-97bcc9ed8c30` (`failover-test-vol`) — Volume Cinder chạy trên nền Cirros-OS thực tế, sử dụng RBD journaling mirroring. |
| **Cluster 1 (Site A)** | `ceph-a1` (`13.212.20.167`) + `ceph-a2` (`172.31.46.199`, nội bộ), gồm 2 MON, 2 OSD, Ceph v20.2.4. |
| **Cluster 2 (Site B)** | `ceph-b1` (`18.141.205.135`) + `ceph-b2` (`172.31.18.188`), có 1 tiến trình `rbd-mirror` chạy trên `ceph-b2`. |
| **Cấu hình Mirror** | One-way, `site-a → site-b`; Site A là **Primary**, Site B là **Secondary (rx-only)**; sử dụng **RBD journal-based mirroring (asynchronous)**. |

- **Thực hiện**: Duy trì tải ghi liên tục trên Site A trong khi replication bất đồng bộ sang Site B đang có độ trễ, sau đó chủ động ngắt Site A để mô phỏng sự cố mất toàn bộ site. Tiếp theo, thực hiện force-promote Site B thành primary và kiểm tra khả năng khôi phục ghi thực tế, từ đó đánh giá RPO do replication lag và RTO của quá trình failover.
 
**Quy trình kiểm thử**
1. **Bước 1 — Tạo tải ghi liên tục trên Site A**

   Chạy `rbd bench` để tạo luồng ghi ngẫu nhiên 4 KiB liên tục vào volume:

   ```bash
   rbd bench --io-type write --io-size 4096 --io-total 2G --io-pattern rand \
     volumes/volume-907d059a-8501-48cc-8672-97bcc9ed8c30
   ```

   Benchmark được chạy trong `cephadm shell` trên `ceph-a1`. Với tốc độ ghi khoảng `11–15 MiB/s`, quá trình ghi kéo dài đủ lâu để replication có thời gian phát sinh độ trễ.

2. **Bước 2 — Kiểm tra replication lag**

   Trong khi Site A vẫn đang ghi, kiểm tra trạng thái mirror từ Cluster 2:

   ```bash
   rbd mirror image status volumes/volume-907d059a-8501-48cc-8672-97bcc9ed8c30
   ```

   Tại `03:47:17Z`, kết quả cho thấy:

   ```text
   entries_behind_primary: 71793
   ETA: 261 seconds
   ```

   Điều này xác nhận Site B đang **chậm hơn Site A trong quá trình replay journal**.

3. **Bước 3 — Ghi nhận trạng thái replication ngay trước khi xảy ra sự cố**

   Tại `03:47:43Z`, kiểm tra lại vị trí journal:

   ```json
   {
     "non_primary_position": {
       "entry_tid": 18824
     },
     "primary_position": {
       "entry_tid": 155846
     }
   }
   ```

   Khoảng cách giữa hai vị trí là:

   ```text
   155846 - 18824 = 137022 entries
   ```

   Đây là trạng thái replication lag được ghi nhận ngay trước thời điểm Site A bị ngắt.

4. **Bước 4 — Mô phỏng Site A bị mất hoàn toàn**

   Ghi nhận thời điểm xảy ra sự cố:

   ```bash
   date -u +"%Y-%m-%dT%H:%M:%S.%3NZ"
   # 2026-09-23T03:47:59.735Z
   ```

   Sau đó dừng toàn bộ Ceph services được quản lý bởi cephadm trên `ceph-a1`:

   ```bash
   sudo systemctl stop ceph-850bf3b8-b0ec-11f1-86b6-413027b51323.target
   ```

   Target này quản lý các daemon như MON, MGR, OSD và các service liên quan trên host. Việc dừng target mô phỏng việc `ceph-a1` mất hoàn toàn.

   Do Cluster 1 chỉ có 2 MON, việc dừng `mon.ceph-a1` khiến cluster mất MON quorum. Vì vậy, dù `ceph-a2` vẫn còn hoạt động, Cluster 1 không còn khả năng phục vụ I/O bình thường cho client.

5. **Bước 5 — Xác nhận Site A đã ngừng hoạt động**

   Kiểm tra trạng thái systemd target:

   ```bash
   sudo systemctl is-active ceph-850bf3b8-...target
   ```

   Kết quả:

   ```text
   inactive
   ```

   Tiến trình benchmark đang chạy trên `ceph-a1` cũng dừng theo. Log ghi nhận khoảng `242.912` operations đã được client xác nhận trước khi tiến trình bị gián đoạn.

6. **Bước 6 — Force-promote Site B thành Primary**

   Tại Cluster 2, ghi nhận thời điểm bắt đầu failover:

   ```bash
   date -u +"%Y-%m-%dT%H:%M:%S.%3NZ"
   # 2026-09-23T03:49:25.584Z

   rbd mirror image promote --force \
     volumes/volume-907d059a-8501-48cc-8672-97bcc9ed8c30
   ```

   Kết quả:

   ```text
   Image promoted to primary
   ```

   Ghi nhận thời điểm hoàn tất:

   ```bash
   date -u +"%Y-%m-%dT%H:%M:%S.%3NZ"
   # 2026-09-23T03:49:27.830Z
   ```

   Sử dụng `--force` vì Site A đã mất hoàn toàn và không thể liên lạc để thực hiện quá trình demote thông thường.

7. **Bước 7 — Kiểm tra khả năng ghi sau khi promote**

   Mặc dù image trên Site B đã được đánh dấu là `primary`, lần ghi đầu tiên vẫn thất bại:

   ```text
   failed to acquire exclusive lock:
   (30) Read-only file system
   ```

   Các trạng thái cấp cluster như `ceph -s`, pool quota, `full_ratio` và mức sử dụng OSD đều không cho thấy vấn đề.

   Kiểm tra `rbd status` cho thấy vẫn còn một watcher cũ trên image. Watcher này thuộc tiến trình:

   ```text
   rbd-mirror.ceph-b2.ercngi
   ```

   Tiến trình mirror chưa làm mới hoàn toàn trạng thái sau khi image được promote, khiến việc ghi chưa thể thực hiện.

8. **Bước 8 — Restart rbd-mirror daemon**

   Restart daemon để buộc tiến trình đọc lại trạng thái mirror mới:

   ```bash
   ceph orch daemon restart rbd-mirror.ceph-b2.ercngi --force
   ```

   Sau khi daemon khởi động lại, trạng thái của image được cập nhật tương ứng với vai trò Primary mới trên Site B.

9. **Bước 9 — Kiểm tra ghi lại sau khi xử lý**

   Thử ghi lại vào image:

   ```bash
   date -u +"%Y-%m-%dT%H:%M:%S.%3NZ"
   # 2026-09-23T03:51:44.367Z

   rbd bench --io-type write --io-size 4096 --io-total 4M ...
   ```

   Kết quả:

   ```text
   ops/sec: 8533.28
   bytes/sec: 33 MiB/s
   ```

   Lần ghi này thành công, xác nhận Site B đã có thể tiếp tục phục vụ write sau failover.

10. **Bước 10 — Xác nhận dữ liệu đã được ghi**

    Kiểm tra lại thông tin image:

    ```bash
    rbd info volumes/volume-907d059a-8501-48cc-8672-97bcc9ed8c30
    ```

    `modify_timestamp` được cập nhật tại `03:52:14Z`, xác nhận image đã tiếp nhận thay đổi mới sau failover.

11. **Bước 11 — Khôi phục Site A**

    Sau khi hoàn tất kiểm tra failover, khởi động lại các Ceph services trên Site A:

    ```bash
    systemctl start ceph-850bf3b8-b0ec-11f1-86b6-413027b51323.target
    ```

    Thời điểm khởi động:

    ```text
    03:56:51Z
    ```

    Sau khi khôi phục, Cluster 1 trở lại trạng thái:

    ```text
    HEALTH_OK
    97/97 PG active+clean
    ```

12. **Bước 12 — Phát hiện nguy cơ split-brain**

    Sau khi Site A trở lại, kiểm tra lại trạng thái mirror của image cho thấy Site A vẫn có:

    ```text
    mirroring primary: true
    ```

    Điều này xảy ra vì Site A bị mất kết nối trong thời gian failover nên chưa nhận được lệnh demote.

    Do đó, tại thời điểm kiểm tra:

    ```text
    Site A → PRIMARY
    Site B → PRIMARY
    ```

    Đây là trạng thái **split-brain**. Chưa thực hiện bước hòa giải trong Scenario này; cần demote bản sao cũ trên Site A và thực hiện resync từ Site B trước khi cho phép Site A tiếp tục nhận ghi.

- **Kết quả**

| Chỉ số | Giá trị | Ghi chú |
|---|---:|---|
| **RPO** | ≥ 137.022 entries (~561 MB) | Mức tối thiểu dựa trên replication lag ngay trước sự cố; thực tế có thể cao hơn do client vẫn tiếp tục ghi trước khi Site A dừng. |
| **RTO – Promote** | 88 giây | Từ `03:47:59.735Z` đến `03:49:27.830Z`. |
| **RTO – Khôi phục ghi thực tế** | ~3 phút 45 giây | Tính đến khi write thành công sau khi xử lý trạng thái `rbd-mirror`. |
| **Thời gian Site A gián đoạn** | ~9 phút | Từ `03:47:59Z` đến `03:56:51Z`. |
| **Tốc độ ghi trước sự cố** | ~11–15 MiB/s, ~2.600–3.800 IOPS | Random write 4 KiB, benchmark 2 GiB. |
| **Tốc độ ghi sau failover** | 33 MiB/s | Mẫu test 4 MiB, chỉ dùng để xác nhận khả năng ghi, không dùng để so sánh hiệu năng. |
| **Client operations trước khi dừng** | 242.912 ops | Giá trị từ log `rbd bench`; không đối chiếu trực tiếp với journal TID. |
| **Trạng thái Cluster 1 sau phục hồi** | `HEALTH_OK`, 97/97 PG `active+clean` | Cluster 1 đã khôi phục trạng thái hoạt động bình thường. |
| **Rủi ro tồn đọng** | **Split-brain** | Cả hai site đều báo `mirroring primary: true`; cần demote Site A và resync từ Site B trước khi cho phép Site A ghi lại. |

- **Kết luận**
    - Replication lag dẫn đến RPO khác 0: Với journaling mirror theo cơ chế asynchronous, dữ liệu chưa kịp replicate sang Site B có thể bị mất khi Site A gặp sự cố. Trong thử nghiệm, replication lag ngay trước sự cố tương ứng tối thiểu khoảng 561 MB dữ liệu chưa được đồng bộ.
    - **RPO:** ≥ 137.022 entries (~561 MB) chưa được replicate tại thời điểm Site A gặp sự cố, do replication bất đồng bộ.
    - **RTO:** ~3 phút 45 giây để Site B thực sự ghi được dữ liệu sau failover; thời gian promote riêng là 88 giây.
    - **Failover:** Promote thành công chưa đồng nghĩa dịch vụ đã được khôi phục: Sau khi promote --force, image đã trở thành primary nhưng chưa thể ghi ngay do trạng thái của rbd-mirror chưa được làm mới. Vì vậy, RTO cần được tính đến thời điểm write thực tế thành công, thay vì chỉ tính đến thời điểm promote
    - **Rủi ro:** Khi Site A quay lại, có thể xảy ra **split-brain**; cần demote Site A và resync từ Site B trước khi cho phép ghi lại.

# Scenario 2: Network partion 
- **Môi trường thực nghiệm:** 
    - Ceph Cluster 1 (Primary): ceph-a1 (18.143.172.221, private 172.31.41.64) + ceph-a2 (172.31.46.199, chỉ có private IP)
    - Ceph Cluster 2 (Secondary): ceph-b1 (18.140.57.71, private 172.31.27.91) + ceph-b2 (172.31.18.188, chỉ có private IP)
    - Volume thử nghiệm: journal-cinder-test-vol — được tạo mới hoàn toàn cho bài test này, không dùng lại từ các kịch bản trước.

- **Thực hiện**: Tạo volume qua OpenStack Cinder → xác nhận tự động mirror sang Cluster 2 và đồng bộ thành công.
Giả lập network partition → Cluster 1 vẫn ghi bình thường, Cluster 2 chuyển down+error, trong khi OpenStack vẫn báo available.
Gỡ partition → mirror tự động phục hồi, replay backlog từ 12.801 về 0 trong khoảng 2 phút.

- **Quy trình**
Bước 1 — Tạo volume thông qua API OpenStack thực tế:
openstack volume create --size 2 --image cirros --type ceph journal-cinder-test-vol
Chờ cho đến khi trạng thái chuyển sang available. Kết quả: Volume ID 5e99b66f-dbcf-43ce-848a-ec9be838ea71, chứa dữ liệu OS Cirros thật, đi qua Cinder RBD driver để vào Cluster 1.

Bước 2 — Xác nhận tính năng tự động mirror ở phía Ceph (trên Cluster 1):
rbd info volumes/volume-5e99b66f-dbcf-43ce-848a-ec9be838ea71
Kết quả xác nhận mirroring state: enabled, mirroring mode: journal, mirroring primary: true mà không cần chạy thủ công lệnh bật mirror cho từng image.

Bước 3 — Xác nhận đồng bộ sang Cluster 2:
Trạng thái đạt up+replaying, entries_behind_primary: 0 sau khoảng 10 giây.

Bước 4 — Áp dụng phân vùng mạng. Chặn (blocklist) host daemon secondary của Cluster 2 trên Cluster 1 (primary):
ceph osd blocklist add 172.31.18.188 (Lưu ý: Hướng chặn ngược lại với bài test raw-RBD trước đó do Cinder chỉ có thể ghi vào Cluster 1).

Bước 5 — Tạo tải ghi trên primary (dùng rbd bench trực tiếp lên image Ceph):
Quá trình hoàn tất bình thường ở tốc độ ~14 MiB/s, chứng minh phân vùng mạng không ảnh hưởng gì đến hiệu năng ghi ở phía primary.

Bước 6 — Kiểm tra đồng thời cả hai lớp trong lúc sự cố:

Tầng Ceph (trên Cluster 2): Trạng thái down+error, mô tả replay completed with error: (108) Cannot send after transport endpoint shutdown (phát hiện sau ~5 giây).

Tầng OpenStack (trên node DevStack): openstack volume show journal-cinder-test-vol -f value -c status trả về available. Không có bất kỳ thay đổi nào. Trạng thái Cinder không có bất kỳ tầm nhìn nào vào mối quan hệ mirror; nó chỉ phản ánh sức khỏe của Cluster 1, vốn không hề bị ảnh hưởng.

Bước 7 — Xác nhận sức khỏe tổng thể của cụm: Cả hai cụm không có cảnh báo nào liên quan đến mirror.

Bước 8 & 9 — Gỡ bỏ blocklist và theo dõi tự phục hồi:
Gỡ blocklist vào lúc 16:22:19.012Z. Theo dõi trạng thái tự động khôi phục về up+replaying sau khoảng 37 giây mà không cần khởi động lại daemon thủ công.

Bước 10 — Xác nhận lượng backlog đã được xử lý hết:
entries_behind_primary giảm dần từ 12.801 về 0 trong tổng thời gian ~2 phút.

Bước 11 — Kiểm tra lại tầng OpenStack: Trạng thái vẫn là available xuyên suốt từ đầu đến cuối bài test.

- **Kết quả**
| Metric | Kết quả |
|---|---|
| **Thời gian phát hiện (Ceph)** | ~5 giây |
| **Thời gian phát hiện (OpenStack)** | Không có — zero visibility |
| **Ảnh hưởng I/O ở Primary** | Không có — duy trì ~14 MiB/s |
| **Ảnh hưởng sức khỏe toàn cụm** | Không có trên cả hai cụm |
| **RPO** | 0 byte |
| **RTO – Khôi phục kết nối** | ~37 giây, tự động, không cần restart daemon |
| **RTO – Đồng bộ hoàn tất** | ~2 phút |

- **Kết luận:**
- OpenStack vẫn báo volume available và không cảnh báo khi RBD mirror bị gián đoạn, vì OpenStack không có tầm nhìn vào trạng thái của site DR.
- Giám sát phải đặt ở tầng Ceph: Cần theo dõi rbd mirror pool/image status và tích hợp cảnh báo tại Ceph; không thể chỉ dựa vào trạng thái sức khỏe của OpenStack để phát hiện lỗi replication.
- RPO: Trong bài test, RPO = 0 byte — các write đã được xác nhận tại primary không bị mất; chúng chỉ chưa kịp replication sang secondary trong thời gian partition.
- RTO: Mirror tự động khôi phục kết nối sau ~37 giây và hoàn tất đồng bộ backlog sau ~2 phút, không cần restart daemon thủ công.