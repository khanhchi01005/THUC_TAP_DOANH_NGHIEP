# Tính năng tủ SAN và mức hỗ trợ trên OpenStack Cinder

**Tủ:** Dell Unity XT 880 · IBM FlashSystem 7300 · Hitachi VSP G700 · Hitachi VSP E590H
**OpenStack:** Yoga (03/2022) · 2023.1 Antelope (03/2023) · 2024.1 Caracal (04/2024)

---

## 1. Tóm tắt

1. **Unity và FS7300:** tích hợp với OpenStack đầy đủ và **không đổi** qua cả ba bản.
2. **Hitachi:** mọi khác biệt giữa ba bản đều nằm ở driver Hitachi. Caracal là bản đầu tiên có active-active (GAD), nén/dedup và migration do tủ thực hiện.
3. **Hitachi vẫn thiếu ở cả ba bản:** QoS và replication/DR qua Cinder. Hai việc này phải cấu hình trực tiếp trên tủ.
4. **E590H:** không được driver ghi tên ở cả ba bản (từ Antelope chỉ có E590 all-flash). Cần Hitachi xác nhận.

---

## 2. Cách đọc bảng

Bảng chính có hai nửa:

- **Trên tủ:** tủ có tính năng đó hay không (theo tài liệu hãng).
- **OpenStack:** driver Cinder của hãng có điều khiển được tính năng đó hay không. Unity và IBM mỗi hãng một cột vì giống nhau ở cả ba bản. Hitachi tách ba cột theo bản.

| Ký hiệu | Nghĩa |
|:---:|---|
| ✔ | Có |
| ✖ | Không |
| ◐ | Một phần, hoặc có điều kiện (xem ghi chú) |
| – | Tài liệu không nêu, hoặc chưa xác nhận được |
| ¹ ² ³ | Số ghi chú ở mục 4 |

---

## 3. Bảng chính

| Tính năng | Mô tả | Unity 880 | FS7300 | G700 | E590H | Driver Unity (cả 3 bản) | Driver IBM (cả 3 bản) | Driver Hitachi, bản Yoga | Driver Hitachi, bản Antelope | Driver Hitachi, bản Caracal |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| | | **Trên tủ** | | | | **OpenStack** | | | | |
| **A. Cơ bản** | | | | | | | | | | |
| Tạo/xóa LUN, map/unmap host | Cấp phát volume từ pool và gán cho host | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Mở rộng LUN (kể cả đang attach) | Tăng dung lượng volume, không cần tạo lại | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ ¹ | ✔ ¹ | ✔ ¹ | ✔ ¹ |
| Giao thức FC, iSCSI | Kết nối host tới tủ qua Fibre Channel hoặc iSCSI | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Thin provisioning | Cấp dung lượng logic lớn hơn vật lý, chỉ chiếm chỗ khi ghi | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Thick provisioning | Cấp phát đủ dung lượng vật lý ngay khi tạo | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | – | – | – |
| Snapshot | Bản chụp volume tại một thời điểm | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Clone, tạo volume từ snapshot | Tạo volume mới độc lập từ volume hoặc snapshot | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Khôi phục LUN về snapshot | Đưa volume về đúng trạng thái của snapshot | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Chia sẻ LUN cho nhiều host | Một volume gán đồng thời cho nhiều host (cluster) | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| **B. Hiệu năng và tiết kiệm dung lượng** | | | | | | | | | | |
| NVMe phía host | Giao thức NVMe qua FC/Ethernet, độ trễ thấp hơn SCSI | ✖ | ✔ | ✖ | – ² | ✖ | ✖ | ✖ | ✖ | ✖ |
| QoS giới hạn IOPS/băng thông | Đặt trần IOPS hoặc MB/s cho từng volume | ✔ | ✔ | ✔ | ✔ ³ | ✔ | ✔ | ✖ | ✖ | ✖ |
| Nén dữ liệu | Nén inline để giảm dung lượng vật lý | ✔ | ✔ | ✔ ⁴ | ✔ | ✔ ⁵ | ✔ | ✖ | ✖ | ✔ ⁶ |
| Dedup | Loại bỏ các block dữ liệu trùng lặp | ✔ | ✔ | ✔ ⁴ | ✔ | – | – | ✖ | ✖ | ✔ ⁶ |
| Auto-tiering | Tự chuyển dữ liệu nóng/lạnh giữa các tầng đĩa | ✔ ⁷ | ✔ | ✔ | – ² | ✔ | ✔ | ◐ ⁸ | ◐ ⁸ | ◐ ⁸ |
| SSD cache | Dùng SSD tăng tốc cho pool đĩa quay | ✔ ⁷ | ✖ | ◐ | – ² | ✖ | ✖ | ✖ | ✖ | ✖ |
| **C. Bảo vệ dữ liệu và DR** | | | | | | | | | | |
| Consistency group, snapshot nhóm | Snapshot nhất quán cho một nhóm volume | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Replication đồng bộ | Nhân bản sang tủ khác, RPO = 0, khoảng cách ngắn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ | ✖ | ✖ |
| Replication bất đồng bộ | Nhân bản sang site xa, RPO tính bằng giây/phút | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ | ✖ | ✖ |
| Replication theo consistency group | Nhân bản cả nhóm volume, giữ nhất quán | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ | ✖ | ✖ |
| Failover/failback sang site DR | Chuyển vai trò chính sang tủ DR và chuyển ngược lại | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ | ✖ | ✖ |
| Active-active metro cluster | Một volume đọc/ghi đồng thời ở 2 tủ, failover tự động | ◐ ⁹ | ✔ | ✔ | ✔ | ✖ | ✔ | ✖ | ◐ ¹⁰ | ✔ ¹¹ |
| Mirror volume giữa 2 pool | Hai bản sao của volume ở 2 pool trong cùng tủ | – | ✔ | ✔ | ✔ | ✖ | ✔ | ✖ | ✖ | ✖ |
| Replication 3 site | Bản đồng bộ gần kết hợp bản bất đồng bộ xa | – | ✔ | ✔ | ✔ | ✖ | ✖ | ✖ | ✖ | ✖ |
| Snapshot bất biến chống ransomware | Snapshot không thể sửa/xóa trước hạn lưu giữ | ✖ | ✔ | ◐ ¹² | ◐ ¹² | ✖ | ✖ | ✖ | ✖ | ✖ |
| Backup volume không gián đoạn | Backup từ snapshot, không phải dừng volume | ✔ | ✔ | ✔ | ✔ | ✔ | – | ✔ | ✔ | ✔ |
| Mã hóa dữ liệu trên tủ | Mã hóa dữ liệu lưu trên đĩa (data-at-rest) | ✔ | ✔ | ✔ | ✔ | ◐ ¹³ | ◐ ¹³ | ◐ ¹³ | ◐ ¹³ | ◐ ¹³ |
| **D. Di chuyển và quản lý volume** | | | | | | | | | | |
| Di chuyển LUN giữa pool do tủ thực hiện | Tủ tự chuyển dữ liệu sang pool khác khi host vẫn I/O | ✔ | ✔ | ✔ | ✔ ³ | ✔ | ✔ | ✖ | ✖ | ✔ ¹⁴ |
| Đổi thuộc tính LUN online (retype) | Đổi thin/thick, nén, tier của volume đang dùng | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ◐ ¹⁵ | ◐ ¹⁵ | ◐ ¹⁵ |
| Import LUN có sẵn (manage/unmanage) | Đưa LUN đã tồn tại trên tủ vào Cinder, không copy dữ liệu | ✔ | ✔ | ✔ | ✔ | – | ✔ | ✔ | ✔ | ✔ |
| Ảo hóa tủ ngoài | Dùng LUN của tủ hãng khác làm dung lượng cho tủ này | ✖ | ✔ | ✔ | ✔ | ✖ | ◐ ¹³ | ◐ ¹³ | ◐ ¹³ | ◐ ¹³ |
| **E. Vận hành phía OpenStack** | | | | | | | | | | |
| Model được driver ghi tên | Tủ có trong danh sách hỗ trợ của tài liệu driver | | | | | ✔ | ✔ | G700 ✔ · E590H ✖ | G700 ✔ · E590H ✖ ¹⁶ | G700 ✔ · E590H ✖ ¹⁶ |
| Nhiều pool trên một backend | Một backend Cinder quản lý nhiều pool của tủ | | | | | ✔ | ✔ | ✖ | ✔ | ✔ |
| FC auto-zoning | Tự tạo/xóa zone trên SAN switch khi attach/detach | | | | | ✔ | – | – | ✔ | ✔ |
| Port scheduler (chia WWN lên các cổng) | Đăng ký WWN của host lần lượt lên các cổng tủ | | | | | – | – | ✖ | ✔ | ✔ |
| Chọn cổng tủ theo volume type | Chỉ định cổng tủ dùng cho từng loại volume | | | | | – | – | ✖ | ✖ | ✔ |
| Cinder active/active HA | Chạy nhiều cinder-volume song song cho một backend | | | | | ✖ | ✖ | ✖ | ✖ | ✖ |

---

## 4. Ghi chú

| # | Nội dung |
|:---:|---|
| ¹ | Driver IBM và Hitachi không extend được volume đang có snapshot. |
| ² | E590H: bảng thông số chỉ ghi cổng host FC và iSCSI; Dynamic Tiering không có trong danh sách tính năng đi kèm. Chưa xác nhận được, cần hỏi Hitachi. |
| ³ | Có theo SVOS RF dùng chung cho cả dòng VSP, nhưng tài liệu riêng của E590H không ghi rõ. |
| ⁴ | G700: nén/dedup (capacity saving) chỉ áp dụng trên ổ flash nội bộ. |
| ⁵ | Driver Unity: chỉ tạo được volume nén trên pool all-flash. |
| ⁶ | Driver Hitachi Caracal: nén và dedup bật cùng nhau qua `hbsd:capacity_saving=deduplication_compression`. Cần bật trước trên pool và có license dedup/compression. |
| ⁷ | Unity: FAST VP và FAST Cache chỉ có trên model hybrid (880, không phải 880F). |
| ⁸ | Tiering của Hitachi chạy theo pool trên tủ; Cinder không đặt được chính sách tier cho từng volume. |
| ⁹ | Unity không có active-active gốc cho block; cần mua thêm metro node hoặc VPLEX. |
| ¹⁰ | Tài liệu Caracal viết GAD có "từ bản 2023.1", nhưng tài liệu driver bản 2023.1 không mô tả GAD và không có tùy chọn `hitachi_mirror_*`. Coi Caracal là bản chắc chắn có. |
| ¹¹ | GAD qua Cinder (Caracal): dùng `hbsd:topology=active_active_mirror_volume`. Giới hạn: không dùng ALUA; volume GAD không bật được nén/dedup, không migrate bằng tủ, không manage/unmanage; G700 cần firmware 88-03-21 trở lên. |
| ¹² | Hitachi có Data Retention Utility trong SVOS RF; chưa xác nhận được giải pháp snapshot bất biến hoàn chỉnh. |
| ¹³ | Trong suốt với Cinder: bật trên tủ thì volume được hưởng, nhưng Cinder không điều khiển. Riêng mã hóa, Cinder có cơ chế mã hóa volume riêng (LUKS). |
| ¹⁴ | Chỉ migrate trong cùng một tủ. Tài liệu driver ghi có, nhưng ma trận hỗ trợ của Cinder vẫn ghi "missing" cho Hitachi; nên thử trước trên môi trường test. |
| ¹⁵ | Driver Hitachi không có retype riêng; đổi loại volume thông qua migrate. |
| ¹⁶ | Từ Antelope, driver ghi tên E590 (all-flash), E790, E1090, E1090H; E590H vẫn không có tên. |

---

## 5. So sánh ba bản OpenStack

### Giống nhau

- Driver Unity và IBM: danh sách thao tác, extra spec, tùy chọn cấu hình giống nhau ở cả ba bản (chỉ đổi tên trong tài liệu).
- Ma trận hỗ trợ chính thức của Cinder cho cả ba driver giống hệt nhau ở cả ba bản.
- Driver Hitachi: không có QoS, không có replication kiểu Cinder ở cả ba bản.
- Không driver nào hỗ trợ NVMe phía host, snapshot bất biến, replication 3 site, Cinder active/active HA.

### Khác nhau (chỉ ở driver Hitachi)

| Điểm khác | Yoga | Antelope | Caracal |
|---|:---:|:---:|:---:|
| Model dòng E được ghi tên | Chỉ E990 | + E590, E790, E1090, E1090H | Như Antelope |
| Nhiều pool trên một backend | ✖ | ✔ | ✔ |
| FC auto-zoning | ✖ | ✔ | ✔ |
| Port scheduler | ✖ | ✔ | ✔ |
| Chọn cổng theo volume type | ✖ | ✖ | ✔ |
| Active-active (GAD) | ✖ | ◐ | ✔ |
| Nén và dedup | ✖ | ✖ | ✔ |
| Migration do tủ thực hiện | ✖ | ✖ | ✔ |

### Nên chọn bản nào

| Tình huống | Nhận định |
|---|---|
| Chỉ dùng Unity và FS7300 | Ba bản như nhau về tích hợp tủ; chọn theo lý do khác. |
| Dùng G700, cần active-active hoặc nén/dedup qua OpenStack | Cần Caracal. |
| Dùng G700, chỉ cần nhiều pool và auto-zoning | Antelope là đủ. |
| Dùng E590H | Chưa bản nào ghi tên chính thức; cần Hitachi xác nhận trước. |
| Cần QoS hoặc DR của Hitachi qua OpenStack | Chưa bản nào trong ba bản đáp ứng. |

---

## Phụ lục A. Mô tả tính năng và tên gọi theo hãng

| Tính năng | Mô tả | Dell Unity | IBM FS7300 | Hitachi |
|---|---|---|---|---|
| Thin provisioning | Cấp dung lượng logic lớn hơn vật lý, chỉ chiếm chỗ khi ghi | Thin LUN | Thin volume | Dynamic Provisioning |
| Snapshot | Bản chụp volume tại một thời điểm, tiết kiệm dung lượng | Unified Snapshots | FlashCopy | Thin Image |
| Clone | Volume mới độc lập tạo từ volume/snapshot | Thin Clone | FlashCopy (có background copy) | ShadowImage, Thin Image |
| QoS | Đặt trần IOPS hoặc MB/s cho volume/host | Quality of Service (Host I/O Limits) | Throttling | QoS controls trong SVOS RF |
| Nén, dedup | Giảm dung lượng vật lý bằng nén và loại block trùng | Inline Data Reduction | Data Reduction Pool, nén phần cứng trên FCM | Adaptive Data Reduction (capacity saving) |
| Auto-tiering | Tự chuyển dữ liệu nóng/lạnh giữa các tầng đĩa | FAST VP | Easy Tier | Dynamic Tiering |
| SSD cache | Dùng SSD tăng tốc cho pool đĩa quay | FAST Cache | Không có | active flash |
| Replication đồng bộ | Nhân bản sang tủ khác, RPO = 0 | Native Sync Replication | Metro Mirror | TrueCopy |
| Replication bất đồng bộ | Nhân bản sang site xa, RPO giây/phút | Native Async Replication | Global Mirror, GMCV, policy-based replication | Universal Replicator |
| Active-active metro | Một volume đọc/ghi đồng thời ở 2 tủ, failover tự động | metro node, VPLEX (mua thêm) | HyperSwap | global-active device (GAD) |
| Mirror giữa 2 pool | Hai bản sao của volume trong cùng một tủ | Không có | Volume Mirroring | ShadowImage |
| Replication 3 site | Bản đồng bộ gần kết hợp bản bất đồng bộ xa | Không nêu | 3-site replication | 3DC (GAD + UR, TC + UR) |
| Snapshot bất biến | Snapshot không thể sửa/xóa trước hạn | Không có cho block | Safeguarded Copy | Data Retention Utility |
| Mã hóa | Mã hóa dữ liệu trên đĩa | D@RE | AES-XTS 256 | AES-256-XTS (cần back-end mã hóa) |
| Ảo hóa tủ ngoài | Dùng LUN của tủ hãng khác làm dung lượng | Không có | External virtualization | Universal Volume Manager |
| Consistency group | Gom nhiều volume để snapshot/replicate nhất quán | Consistency Group | Consistency group, volume group | Consistency group |

## Phụ lục B. License phía tủ

| Tủ | Kèm sẵn | Phải mua thêm hoặc tùy chọn |
|---|---|---|
| Dell Unity XT 880 | Thin, data reduction, QoS, snapshot, thin clone, replication đồng bộ và bất đồng bộ, FAST VP và FAST Cache (hybrid) | metro node, VPLEX, RecoverPoint Advanced, AppSync Advanced, PowerPath; mã hóa D@RE chọn lúc đặt hàng |
| IBM FS7300 | Mọi tính năng trừ hai mục bên phải | Ảo hóa tủ ngoài (theo dung lượng), mã hóa (feature code). Cần hỏi IBM về license remote mirroring cho HyperSwap |
| Hitachi G700 | Gói Foundation: SVOS RF, Universal Volume Manager, Local Replication (Thin Image, ShadowImage), Data Mobility (Dynamic Tiering) | Gói Advanced: TrueCopy, Universal Replicator, GAD. Mã hóa: license và phần cứng riêng |
| Hitachi E590H | Gói Base: Adaptive Data Reduction, Storage Virtualization, In-System Replication, Non-disruptive Migration | Gói Advanced: remote replication, GAD |

License thực tế trên từng tủ cần kiểm tra trực tiếp: Unisphere (Unity), lệnh `lslicense` (IBM), màn hình License trong Storage Navigator (Hitachi).

## Phụ lục C. Driver Cinder tương ứng

| Tủ | Driver Cinder | Yêu cầu |
|---|---|---|
| Dell Unity 880 | Dell Unity driver (`cinder.volume.drivers.dell_emc.unity.Driver`) | Unity OE 4.1.X trở lên, storops 1.2.3 trở lên |
| IBM FS7300 | IBM Storage Virtualize driver (`StorwizeSVCFCDriver`, `StorwizeSVCISCSIDriver`) | Phủ họ FlashSystem 5xxx, 7xxx, 9xxx |
| Hitachi G700, E590H | Hitachi VSP driver (`HBSDFCDriver`, `HBSDISCSIDriver`) | G700: firmware 88-01-04 trở lên. License SVOS, Dynamic Provisioning, Thin Image |

Bảng dựa trên tài liệu upstream của OpenStack. Driver do Hitachi phát hành riêng có thể hỗ trợ rộng hơn.

---

## Nguồn

**OpenStack Yoga**
- [Cinder Driver Support Matrix](https://docs.openstack.org/cinder/yoga/reference/support-matrix.html)
- [Dell EMC Unity driver](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/dell-emc-unity-driver.html)
- [IBM Spectrum Virtualize driver](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/ibm-storwize-svc-driver.html)
- [Hitachi block storage driver](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/hitachi-vsp-driver.html)

**OpenStack 2023.1 Antelope**
- [Cinder support-matrix.ini](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2023.1/doc/source/reference/support-matrix.ini)
- [Dell Unity driver](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/dell-emc-unity-driver.html)
- [IBM Storage Virtualize driver](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/ibm-storwize-svc-driver.html)
- [Hitachi block storage driver](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/hitachi-vsp-driver.html)

**OpenStack 2024.1 Caracal**
- [Cinder support-matrix.ini](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/doc/source/reference/support-matrix.ini)
- [Dell Unity driver](https://docs.openstack.org/cinder/2024.1/configuration/block-storage/drivers/dell-emc-unity-driver.html)
- [IBM Storage Virtualize driver](https://docs.openstack.org/cinder/2024.1/configuration/block-storage/drivers/ibm-storwize-svc-driver.html)
- [Hitachi block storage driver](https://docs.openstack.org/cinder/2024.1/configuration/block-storage/drivers/hitachi-vsp-driver.html)

**Tài liệu hãng**
- [Dell Unity XT Series Specification Sheet](https://www.delltechnologies.com/asset/en-my/products/storage/technical-support/h17713_dell_emc_unity_xt_series_ss.pdf)
- [Dell Unity XT HFA Top Reasons (metro node)](https://www.delltechnologies.com/asset/he-il/products/storage/briefs-summaries/dell-unity-xt-hfa-top-reasons.pdf)
- [IBM Storage FlashSystem 7300 Product Guide](https://redbooks.ibm.com/redpapers/pdfs/redp5668.pdf)
- [Hitachi: Base and Advanced Software Packages for VSP Midrange Storage](https://www.hitachivantara.com/en-us/pdf/datasheet/base-advanced-software-packages-for-vsp-midrange-storage-datasheet.pdf)
- [Hitachi VSP G/F series: Software components and features](https://knowledge.hitachivantara.com/Documents/Storage/VSP_G130_GF350_GF370_GF700_GF900/88-01-0x/About_Your_System/Product_Overview/Software_components_and_features)
- [Hitachi VSP E Series Family Matrix](https://www.hitachivantara.com/en-us/pdf/specifications/virtual-storage-platform-e-series-family-matrix.pdf)