# Giải pháp DR Ceph

## Hạ tầng

- Ceph cluster 1: gồm 2 OSD, mỗi OSD 30 GB.
- Ceph cluster 2: gồm 2 OSD, mỗi OSD 30 GB.
- Máy ảo cài OpenStack: tạo Cinder volume kết nối tới Ceph cluster 1.
- Ceph cluster 1 và cluster 2 được cấu hình RBD mirror, với cluster 1 là primary.

## Kịch bản failover

- Thực hiện primary site down.
- Mô phỏng network partition giữa 2 Ceph cluster.

## Kết luận chung
- Qua các scenario kiểm thử, Journal và Snapshot Mirroring đều đáp ứng được khả năng replication và recovery, nhưng có sự khác biệt rõ về cách đánh đổi giữa RPO, hiệu năng và vận hành
- **Journal Mode** có ưu điểm về RPO thấp và replication liên tục, phù hợp với các workload có yêu cầu nghiêm ngặt về dữ liệu. Tuy nhiên, benchmark cho thấy Journal Mode gây overhead I/O lớn, làm giảm đáng kể write throughput và tăng write latency.
- **Snapshot Mode**  duy trì hiệu năng I/O gần như baseline, loại bỏ overhead ghi liên tục và cung cấp RPO có thể dự báo theo lịch trình. Đây là lựa chọn tối ưu cho phần lớn các volume OpenStack/Cinder (VM boot, data disks, workload ghi nhiều)
- Với môi trường OpenStack + Cinder + Ceph RBD, có thể sử dụng Snapshot Mode cho các volume thông thường và workload ưu tiên hiệu năng, trong khi Journal Mode dành cho các volume đặc biệt có yêu cầu RPO rất thấp. Cách tiếp cận này cho phép lựa chọn mirroring mode theo đặc tính và yêu cầu của từng workload thay vì áp dụng một cấu hình duy nhất cho toàn bộ hệ thống.
- Kết quả kiểm thử cho thấy việc lựa chọn mirroring mode không chỉ phụ thuộc vào khả năng failover, mà cần cân nhắc đồng thời RPO, RTO, hiệu năng I/O và độ phức tạp vận hành

### Quy trình thực hiện chi tiết 
## One-way journal

### Scenario 1: Primary site failure

#### Môi trường thử nghiệm

| Thành phần             | Chi tiết                                                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Volume thử nghiệm**  | `volumes/volume-907d059a-8501-48cc-8672-97bcc9ed8c30` (`failover-test-vol`) — Volume Cinder chạy trên nền Cirros-OS thực tế, sử dụng RBD journaling mirroring. |
| **Cluster 1 (Site A)** | `ceph-a1` (`13.212.20.167`) + `ceph-a2` (`172.31.46.199`, nội bộ), gồm 2 MON, 2 OSD, Ceph v20.2.4.                                                             |
| **Cluster 2 (Site B)** | `ceph-b1` (`18.141.205.135`) + `ceph-b2` (`172.31.18.188`), có 1 tiến trình `rbd-mirror` chạy trên `ceph-b2`.                                                  |
| **Cấu hình Mirror**    | One-way, `site-a → site-b`; Site A là **Primary**, Site B là **Secondary (rx-only)**; sử dụng **RBD journal-based mirroring (asynchronous)**.                  |

#### Thực hiện

Duy trì tải ghi liên tục trên Site A trong khi replication bất đồng bộ sang Site B đang có độ trễ, sau đó chủ động ngắt Site A để mô phỏng sự cố mất toàn bộ site. Tiếp theo, thực hiện force-promote Site B thành primary và kiểm tra khả năng khôi phục ghi thực tế, từ đó đánh giá RPO do replication lag và RTO của quá trình failover.

#### Quy trình kiểm thử

1. **Bước 1 — Tạo tải ghi liên tục trên Site A**

   Chạy rbd bench ghi ngẫu nhiên 4 KiB liên tục vào volume trên Site A với tốc độ khoảng 11–15 MiB/s, quá trình ghi kéo dài đủ lâu để replication có thời gian phát sinh độ trễ.

   ```bash
   rbd bench --io-type write --io-size 4096 --io-total 2G --io-pattern rand \
     volumes/volume-907d059a-8501-48cc-8672-97bcc9ed8c30
   ```

2. **Bước 2 — Kiểm tra replication lag**

   TTheo dõi replication lag: Trong quá trình ghi, Site B bị chậm hơn Site A. Tại 03:47:17Z, ghi nhận 71.793 entries behind, và ngay trước sự cố khoảng cách journal đạt 137.022 entries.

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

#### Kết quả

| Chỉ số                                |                              Giá trị | Ghi chú                                                                                                                           |
| ------------------------------------- | -----------------------------------: | --------------------------------------------------------------------------------------------------------------------------------- |
| **RPO**                               |          ≥ 137.022 entries (~561 MB) | Mức tối thiểu dựa trên replication lag ngay trước sự cố; thực tế có thể cao hơn do client vẫn tiếp tục ghi trước khi Site A dừng. |
| **RTO – Promote**                     |                              88 giây | Từ `03:47:59.735Z` đến `03:49:27.830Z`.                                                                                           |
| **RTO – Khôi phục ghi thực tế**       |                      ~3 phút 45 giây | Tính đến khi write thành công sau khi xử lý trạng thái `rbd-mirror`.                                                              |
| **Thời gian Site A gián đoạn**        |                              ~9 phút | Từ `03:47:59Z` đến `03:56:51Z`.                                                                                                   |
| **Tốc độ ghi trước sự cố**            |      ~11–15 MiB/s, ~2.600–3.800 IOPS | Random write 4 KiB, benchmark 2 GiB.                                                                                              |
| **Tốc độ ghi sau failover**           |                             33 MiB/s | Mẫu test 4 MiB, chỉ dùng để xác nhận khả năng ghi, không dùng để so sánh hiệu năng.                                               |
| **Client operations trước khi dừng**  |                          242.912 ops | Giá trị từ log `rbd bench`; không đối chiếu trực tiếp với journal TID.                                                            |
| **Trạng thái Cluster 1 sau phục hồi** | `HEALTH_OK`, 97/97 PG `active+clean` | Cluster 1 đã khôi phục trạng thái hoạt động bình thường.                                                                          |
| **Rủi ro tồn đọng**                   |                      **Split-brain** | Cả hai site đều báo `mirroring primary: true`; cần demote Site A và resync từ Site B trước khi cho phép Site A ghi lại.           |

#### Kết luận

- Replication lag dẫn đến RPO khác 0: Với journaling mirror theo cơ chế asynchronous, dữ liệu chưa kịp replicate sang Site B có thể bị mất khi Site A gặp sự cố. Trong thử nghiệm, replication lag ngay trước sự cố tương ứng tối thiểu khoảng 561 MB dữ liệu chưa được đồng bộ.
- **RPO:** ≥ 137.022 entries (~561 MB) chưa được replicate tại thời điểm Site A gặp sự cố, do replication bất đồng bộ.
- **RTO:** ~3 phút 45 giây để Site B thực sự ghi được dữ liệu sau failover; thời gian promote riêng là 88 giây.
- **Failover:** Promote thành công chưa đồng nghĩa dịch vụ đã được khôi phục: Sau khi promote `--force`, image đã trở thành primary nhưng chưa thể ghi ngay do trạng thái của `rbd-mirror` chưa được làm mới. Vì vậy, RTO cần được tính đến thời điểm write thực tế thành công, thay vì chỉ tính đến thời điểm promote.
- **Rủi ro:** Khi Site A quay lại, có thể xảy ra **split-brain**; cần demote Site A và resync từ Site B trước khi cho phép ghi lại.

### Scenario 2: Network partition

#### Môi trường thực nghiệm

- Ceph Cluster 1 (Primary): ceph-a1 (18.143.172.221, private 172.31.41.64) + ceph-a2 (172.31.46.199, chỉ có private IP).
- Ceph Cluster 2 (Secondary): ceph-b1 (18.140.57.71, private 172.31.27.91) + ceph-b2 (172.31.18.188, chỉ có private IP).
- Volume thử nghiệm: `journal-cinder-test-vol` — được tạo mới hoàn toàn cho bài test này, không dùng lại từ các kịch bản trước.

#### Thực hiện

Tạo volume qua OpenStack Cinder, xác nhận tự động mirror sang Cluster 2 và đồng bộ thành công.

Giả lập network partition: Cluster 1 vẫn ghi bình thường, Cluster 2 chuyển `down+error`, trong khi OpenStack vẫn báo `available`.

Gỡ partition: mirror tự động phục hồi, replay backlog từ 12.801 về 0 trong khoảng 2 phút.

#### Quy trình

1. **Tạo volume thông qua API OpenStack thực tế:**

   ```bash
   openstack volume create --size 2 --image cirros --type ceph journal-cinder-test-vol
   ```

   Chờ cho đến khi trạng thái chuyển sang `available`. Kết quả: Volume ID `5e99b66f-dbcf-43ce-848a-ec9be838ea71`, chứa dữ liệu OS Cirros thật, đi qua Cinder RBD driver để vào Cluster 1.

2. **Xác nhận tính năng tự động mirror ở phía Ceph (trên Cluster 1):**

   ```bash
   rbd info volumes/volume-5e99b66f-dbcf-43ce-848a-ec9be838ea71
   ```

   Kết quả xác nhận `mirroring state: enabled`, `mirroring mode: journal`, `mirroring primary: true` mà không cần chạy thủ công lệnh bật mirror cho từng image.

3. **Xác nhận đồng bộ sang Cluster 2:** Trạng thái đạt `up+replaying`, `entries_behind_primary: 0` sau khoảng 10 giây.

4. **Áp dụng phân vùng mạng:** Chặn (blocklist) host daemon secondary của Cluster 2 trên Cluster 1 (primary):

   ```bash
   ceph osd blocklist add 172.31.18.188
   ```

   Hướng chặn ngược lại với bài test raw-RBD trước đó do Cinder chỉ có thể ghi vào Cluster 1.

5. **Tạo tải ghi trên primary:** Dùng `rbd bench` trực tiếp lên image Ceph. Quá trình hoàn tất bình thường ở tốc độ ~14 MiB/s, chứng minh phân vùng mạng không ảnh hưởng gì đến hiệu năng ghi ở phía primary.

6. **Kiểm tra đồng thời cả hai lớp trong lúc sự cố:**

   - Tầng Ceph (trên Cluster 2): trạng thái `down+error`, mô tả `replay completed with error: (108) Cannot send after transport endpoint shutdown` (phát hiện sau ~5 giây).

   - Tầng OpenStack (trên node DevStack):

     ```bash
     openstack volume show journal-cinder-test-vol -f value -c status
     ```

     Lệnh trả về `available`. Trạng thái Cinder không có bất kỳ tầm nhìn nào vào mối quan hệ mirror; nó chỉ phản ánh sức khỏe của Cluster 1, vốn không hề bị ảnh hưởng.

7. **Xác nhận sức khỏe tổng thể của cụm:** Cả hai cụm không có cảnh báo nào liên quan đến mirror.

8. **Gỡ bỏ blocklist và theo dõi tự phục hồi:** Gỡ blocklist vào lúc `16:22:19.012Z`. Trạng thái tự động khôi phục về `up+replaying` sau khoảng 37 giây mà không cần khởi động lại daemon thủ công.

9. **Xác nhận lượng backlog đã được xử lý hết:** `entries_behind_primary` giảm dần từ 12.801 về 0 trong tổng thời gian ~2 phút.

10. **Kiểm tra lại tầng OpenStack:** Trạng thái vẫn là `available` xuyên suốt từ đầu đến cuối bài test.

#### Kết quả

| Metric                              | Kết quả                                     |
| ----------------------------------- | ------------------------------------------- |
| **Thời gian phát hiện (Ceph)**      | ~5 giây                                     |
| **Thời gian phát hiện (OpenStack)** | Không có — zero visibility                  |
| **Ảnh hưởng I/O ở Primary**         | Không có — duy trì ~14 MiB/s                |
| **Ảnh hưởng sức khỏe toàn cụm**     | Không có trên cả hai cụm                    |
| **RPO**                             | 0 byte                                      |
| **RTO – Khôi phục kết nối**         | ~37 giây, tự động, không cần restart daemon |
| **RTO – Đồng bộ hoàn tất**          | ~2 phút                                     |

#### Kết luận

- OpenStack vẫn báo volume `available` và không cảnh báo khi RBD mirror bị gián đoạn, vì OpenStack không có tầm nhìn vào trạng thái của site DR.
- Giám sát phải đặt ở tầng Ceph: cần theo dõi RBD mirror pool/image status và tích hợp cảnh báo tại Ceph; không thể chỉ dựa vào trạng thái sức khỏe của OpenStack để phát hiện lỗi replication.
- **RPO:** Trong bài test, RPO = 0 byte — các write đã được xác nhận tại primary không bị mất; chúng chỉ chưa kịp replication sang secondary trong thời gian partition.
- **RTO:** Mirror tự động khôi phục kết nối sau ~37 giây và hoàn tất đồng bộ backlog sau ~2 phút, không cần restart daemon thủ công.

## One-way Snapshot

### Scenario 1: Primary site failure

#### Môi trường thực nghiệm

| Thành phần            | Cấu hình                    |
| --------------------- | --------------------------- |
| **Storage backend**   | Ceph RBD                    |
| **OpenStack service** | Cinder                      |
| **Volume**            | `snapshot-cinder-test-vol2` |
| **Cinder volume ID**  | `volume-0dc0cc12-...`       |
| **Ceph pool**         | `volumes`                   |
| **Replication**       | One-way, Site 1 → Site 2    |
| **Mirroring mode**    | Snapshot                    |
| **Snapshot schedule** | **Mỗi 2 phút**              |
| **Site 1**            | Primary                     |
| **Site 2**            | Secondary                   |
| **Volume size**       | 2 GiB                       |
| **Failover**          | Force promote Site 2        |

#### Thực hiện

- Snapshot mirroring với chu kỳ 2 phút được thiết lập cho Cinder volume trên Ceph.
- Mô phỏng Site 1 failure cho thấy 25 MiB dữ liệu sau snapshot cuối chưa được replicate sang Site 2
- Site 2 được force-promote thành Primary và tiếp tục ghi thành công sau khi xử lý stale watcher

#### Quy trình

1. **Bước 1 — Đồng bộ snapshot ban đầu**

   Tại `17:02:00`, snapshot được tạo theo lịch 2 phút/lần và đồng bộ hoàn toàn sang Site 2.

   Tại thời điểm này, hai bản sao đã đồng bộ.

2. **Bước 2 — Tạo dữ liệu sau snapshot**

   Tại `17:02:39.989Z`, ghi thêm 25 MiB vào volume.

   Dữ liệu này phát sinh sau snapshot `17:02:00` và trước snapshot tiếp theo, nên chưa được replicate sang Site 2.

3. **Bước 3 — Mô phỏng Site 1 failure**

   Tại `17:02:49.760Z`, dừng toàn bộ Ceph services trên Site 1:

   ```bash
   systemctl stop ceph-<fsid>.target
   ```

   Site 1 không còn khả năng phục vụ volume.

4. **Bước 4 — Kiểm tra dữ liệu trên Site 2**

   Kiểm tra cho thấy snapshot cuối cùng được đồng bộ trên Site 2 vẫn là snapshot tại `17:02:00`.

   Do đó, 25 MiB dữ liệu ghi sau snapshot cuối cùng chưa được replicate.

5. **Bước 5 — Kiểm tra OpenStack/Cinder**

   OpenStack vẫn hiển thị volume ở trạng thái:

   ```text
   available
   ```

   mặc dù Ceph primary tại Site 1 đã hoàn toàn mất kết nối.

   Điều này cho thấy trạng thái volume ở tầng Cinder không tự động phản ánh việc backend Ceph primary đã gặp sự cố.

6. **Bước 6 — Xử lý stale watcher**

   Trước khi promote, kiểm tra `rbd status` và phát hiện watcher cũ:

   ```text
   client.144317
   ```

   Watcher được blocklist trước khi thực hiện promote nhằm tránh việc lệnh promote bị treo.

7. **Bước 7 — Failover sang Site 2**

   Thực hiện force-promote image trên Site 2.

   Quá trình promote thành công trong khoảng 6,7 giây, chuyển image thành Primary.

8. **Bước 8 — Kiểm tra khả năng ghi**

   Ngay sau khi promote, thực hiện ghi thử vào volume.

   Kết quả ghi thành công ngay lập tức, không cần restart `rbd-mirror`.

   Điều này xác nhận Site 2 có thể tiếp tục phục vụ write sau khi Site 1 gặp sự cố

#### Kết quả

| Chỉ số                       | Kết quả                                                             |
| ---------------------------- | ------------------------------------------------------------------- |
| **RPO**                      | **25 MiB** — dữ liệu ghi sau snapshot cuối cùng chưa được replicate |
| **RPO time window**          | ~50 giây, từ snapshot `17:02:00` đến failure `17:02:49.760Z`        |
| **Snapshot schedule**        | **2 phút/lần**                                                      |
| **RTO đến khi promote**      | ~52 giây từ thời điểm failure đến khi hoàn tất promote              |
| **RTO đến khi ghi lại được** | ~64 giây                                                            |
| **OpenStack/Cinder**         | Vẫn hiển thị `available` trong thời gian Site 1 bị lỗi              |
| **Post-promote**             | Site 2 trở thành Primary và ghi thành công                          |
| **Journaling feature**       | Đã loại bỏ hoàn toàn                                                |
| **Restart mirror daemon**    | Không cần thiết                                                     |

#### Kết luận

- Hạn chế chính là RPO phụ thuộc vào chu kỳ snapshot; với lịch 2 phút, dữ liệu phát sinh giữa hai snapshot có nguy cơ chưa được bảo vệ tại Site 2 khi Site 1 đột ngột mất.

### Scenario 2: Network partition

#### Môi trường thử nghiệm

| Thành phần              | Cấu hình                    |
| ----------------------- | --------------------------- |
| **Mô hình replication** | One-way Snapshot Mirroring  |
| **Primary**             | Cluster 2 (Site 2)          |
| **Secondary**           | Cluster 1 (Site 1)          |
| **Storage backend**     | Ceph RBD                    |
| **OpenStack service**   | Cinder                      |
| **Volume kiểm thử**     | `snapshot-cinder-test-vol2` |
| **Kích thước volume**   | 2 GiB                       |
| **Snapshot schedule**   | 2 phút                      |

#### Thực hiện

1. **Bước 1 — Tạo Network Partition**

   Tại `17:10:32.969Z`, blocklist daemon của Cluster 1 trên Cluster 2 để mô phỏng mất kết nối giữa hai site.

   Hai cluster vẫn hoạt động, nhưng replication giữa chúng bị gián đoạn.

2. **Bước 2 — Kiểm tra I/O trên Primary**

   Trong thời gian partition, ghi 20 MiB vào volume trên Cluster 2 bằng `rbd bench`.

   Kết quả ghi hoàn thành bình thường với tốc độ khoảng 27 MiB/s, cho thấy network partition không làm gián đoạn I/O tại Primary.

3. **Bước 3 — Kiểm tra khả năng phát hiện lỗi**

   Thực hiện một snapshot thủ công để buộc mirror thử đồng bộ trong thời gian partition.

   Ceph chuyển sang trạng thái:

   ```text
   down+error
   failed to refresh remote image
   ```

   Lỗi được phát hiện sau khoảng 3 giây.

4. **Bước 4 — Kiểm tra OpenStack/Cinder**

   Kiểm tra volume bằng:

   ```bash
   openstack volume show snapshot-cinder-test-vol2
   ```

   Volume vẫn ở trạng thái:

   ```text
   available
   ```

   Điều này cho thấy OpenStack/Cinder không phản ánh trực tiếp trạng thái replication hoặc network partition ở tầng Ceph.

5. **Bước 5 — Khôi phục kết nối**

   Tại `17:11:26.189Z`, remove blocklist để khôi phục kết nối giữa hai cluster.

   Mirror tự động recovery và chuyển sang trạng thái:

   ```text
   up+replaying
   ```

   Sau khoảng 11 giây, quá trình recovery bắt đầu hoạt động bình thường.

6. **Bước 6 — Kiểm tra đồng bộ dữ liệu**

   Snapshot phát sinh trong thời gian partition được giữ lại và replicate sau khi kết nối được khôi phục.

   Hai cluster đạt trạng thái đồng bộ hoàn toàn sau khoảng 37 giây kể từ thời điểm unblock.

#### Kết quả

| Chỉ số                             | Kết quả                                                         |
| ---------------------------------- | --------------------------------------------------------------- |
| **Detection time tại Ceph**        | ~3 giây                                                         |
| **Detection tại OpenStack/Cinder** | Không phát hiện                                                 |
| **Ảnh hưởng Primary I/O**          | Không đáng kể; ghi 20 MiB thành công ở ~27 MiB/s                |
| **Ảnh hưởng cluster-wide**         | Không ghi nhận                                                  |
| **RPO**                            | **0 bytes** — dữ liệu được đồng bộ lại sau khi kết nối phục hồi |
| **RTO khôi phục replication**      | ~11 giây                                                        |
| **RTO đồng bộ hoàn toàn**          | ~37 giây                                                        |
| **Post-recovery**                  | Hai cluster hội tụ và đồng bộ hoàn toàn                         |

#### Kết luận

- Network partition không làm gián đoạn I/O tại Primary; chỉ làm quá trình replication tạm thời bị gián đoạn. Ceph phát hiện mất kết nối sau khoảng 3 giây và tự động tiếp tục replication khi network được khôi phục.
- Trong thử nghiệm, dữ liệu phát sinh trong thời gian partition không bị mất và được đồng bộ lại hoàn toàn, đạt RPO = 0 bytes. Thời gian để mirror bắt đầu recovery khoảng 11 giây, và khoảng 37 giây để hai cluster hoàn toàn hội tụ.
- Tương tự các scenario trước, OpenStack/Cinder vẫn hiển thị volume `available` trong thời gian replication bị gián đoạn, cho thấy trạng thái Cinder không phản ánh trực tiếp tình trạng replication giữa các Ceph site.
