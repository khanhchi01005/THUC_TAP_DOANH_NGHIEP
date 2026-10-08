# Tính năng tủ SAN và mức hỗ trợ trên OpenStack Yoga (Cinder)

Phạm vi: Dell Unity XT 880, IBM FlashSystem 7300, Hitachi VSP G700, Hitachi VSP E590H, đối chiếu với driver Cinder bản Yoga.

## Ký hiệu

- ✔ có, đã đối chiếu với tài liệu hãng
- ✔* có theo SVOS RF dùng chung cho cả dòng VSP, nhưng tài liệu riêng của model không ghi rõ
- ✖ không
- ◐ một phần, cần sản phẩm mua thêm, hoặc trong suốt với Cinder
- – tài liệu không nêu hoặc chưa xác nhận được

## Kết quả đối chiếu với tài liệu hãng


Về license:

- **Dell Unity XT:** phần mềm all-inclusive. Mã hóa (D@RE) là tùy chọn chọn lúc đặt hàng. FAST Cache và FAST VP chỉ có trên model hybrid (880, không phải 880F). metro node, VPLEX, RecoverPoint Advanced, AppSync Advanced, PowerPath mua riêng.
- **IBM FS7300:** mọi tính năng đi kèm, trừ ảo hóa tủ ngoài (tính theo dung lượng) và mã hóa (feature code riêng). Product guide có ghi HyperSwap cần license remote mirroring, nên xác nhận lại với IBM khi đặt hàng.
- **Hitachi G700:** gói Foundation gồm SVOS RF, Universal Volume Manager, Local Replication (Thin Image, ShadowImage), Data Mobility (Dynamic Tiering, active flash). Gói Advanced thêm Remote Replication (TrueCopy, Universal Replicator) và global-active device. Mã hóa cần license riêng cùng phần cứng back-end mã hóa.
- **Hitachi E590H:** gói Base gồm Adaptive Data Reduction, Storage Virtualization, In-System Replication, Non-disruptive Migration. Gói Advanced thêm Remote Replication và global-active device.

Chưa kiểm tra được: license thực tế đã mua trên từng tủ. Xem ở Unisphere (Unity), `lslicense` (IBM), màn hình License trong Storage Navigator (Hitachi).

## Lưu ý 

- **VSP E590H không có trong danh sách hỗ trợ của driver Hitachi bản Yoga.** Trong dòng E, bản Yoga chỉ liệt kê E990. "Hitachi" trong cột OpenStack chỉ áp dụng chính thức cho G700.
- **FS7300 dùng driver IBM Spectrum Virtualize (Storwize)**, phủ họ FlashSystem 5xxx, 7xxx, 9xxx.
- **Unity 880 dùng driver Dell EMC Unity**, yêu cầu Unity OE 4.1.X trở lên.

## Bảng so sánh

| Tính năng | Mô tả | Cấp | Dell Unity 880 | IBM FS7300 | Hitachi G700 | Hitachi E590H | License phía tủ | OpenStack Yoga (Cinder) |
|---|---|---|---|---|---|---|---|---|
| Tạo/xóa LUN, map/unmap host | Cấp phát volume từ pool và gán cho host qua host group/LUN masking | Cơ bản | ✔ | ✔ | ✔ | ✔ | Kèm sẵn cả 4 | ✔ cả 3 driver |
| Mở rộng LUN | Tăng dung lượng volume, không cần tạo lại | Cơ bản | ✔ | ✔ | ✔ | ✔ | Kèm sẵn cả 4 | ✔ cả 3, kể cả khi đang attach; IBM và Hitachi không extend được volume đang có snapshot |
| Giao thức FC, iSCSI | Kết nối host tới tủ qua Fibre Channel hoặc iSCSI | Cơ bản | ✔ FC 32Gb, iSCSI 10/25Gb | ✔ FC 32Gb, iSCSI 10/25/100Gb | ✔ | ✔ FC 32Gb, iSCSI 10/25Gb | Tùy card I/O | ✔ cả 3 |
| NVMe phía host | Giao thức NVMe qua FC/Ethernet cho độ trễ thấp hơn SCSI | Nâng cao | ✖ | ✔ NVMe/FC, NVMe RDMA, NVMe/TCP (từ 8.6) | ✖ | – chưa xác nhận (spec ghi FC, iSCSI) | IBM: kèm sẵn, tùy card | ✖ cả 3 driver chỉ có FC/iSCSI |
| Thin provisioning | Cấp dung lượng logic lớn hơn vật lý, chỉ chiếm chỗ khi ghi dữ liệu | Cơ bản | ✔ | ✔ | ✔ Dynamic Provisioning | ✔ Dynamic Provisioning | Kèm sẵn cả 4 | ✔ cả 3 |
| Thick provisioning | Cấp phát đủ dung lượng vật lý ngay khi tạo volume | Cơ bản | ✔ | ✔ | ✔ | ✔ | Kèm sẵn cả 4 | Unity ✔ · IBM ✔ (rsize=-1) · Hitachi – |
| Snapshot | Bản chụp volume tại một thời điểm, tiết kiệm dung lượng | Cơ bản | ✔ | ✔ FlashCopy | ✔ Thin Image | ✔ Thin Image | Unity, IBM: kèm sẵn · Hitachi: gói Foundation/Base | ✔ cả 3 |
| Clone, tạo volume từ snapshot | Tạo volume mới độc lập từ volume hoặc snapshot có sẵn | Cơ bản | ✔ Thin Clone | ✔ FlashCopy | ✔ ShadowImage/Thin Image | ✔ ShadowImage/Thin Image | Như trên | ✔ cả 3 |
| Khôi phục LUN về snapshot | Đưa volume về đúng trạng thái của snapshot | Cơ bản | ✔ | ✔ | ✔ | ✔ | Như trên | ✔ cả 3 |
| Multipath, ALUA | Nhiều đường I/O tới cùng một LUN để dự phòng và cân tải | Cơ bản | ✔ | ✔ | ✔ | ✔ | Kèm sẵn; PowerPath (Dell), HDLM (Hitachi) là tùy chọn | ✔ (cấu hình ở Nova/os-brick) |
| Chia sẻ LUN cho nhiều host | Một volume gán đồng thời cho nhiều host (cluster) | Cơ bản | ✔ | ✔ | ✔ | ✔ | Kèm sẵn cả 4 | ✔ multi-attach cả 3 |
| Consistency group và snapshot nhóm | Gom nhiều volume để snapshot nhất quán cùng thời điểm | Nâng cao | ✔ | ✔ | ✔ | ✔ | Theo license snapshot | ✔ cả 3 |
| QoS giới hạn IOPS/băng thông | Đặt trần IOPS hoặc MB/s cho từng volume/host | Nâng cao | ✔ Quality of Service (block, file, vVols) | ✔ Throttling | ✔ QoS controls trong SVOS RF | ✔* | Kèm sẵn | Unity ✔ (maxIOPS, maxBWS) · IBM ✔ (IOThrottling) · Hitachi ✖ |
| Nén dữ liệu | Nén inline để giảm dung lượng vật lý sử dụng | Nâng cao | ✔ Inline Data Reduction | ✔ DRP và nén phần cứng trên FCM | ✔ Capacity saving (chỉ trên flash nội bộ) | ✔ Adaptive Data Reduction | Kèm sẵn cả 4 | Unity ✔ (chỉ pool all-flash) · IBM ✔ · Hitachi ✖ |
| Dedup | Loại bỏ các block dữ liệu trùng lặp | Nâng cao | ✔ | ✔ DRP | ✔ (chỉ trên flash nội bộ) | ✔ | Kèm sẵn cả 4 | – cả 3 (bật ở pool trên tủ thì trong suốt) |
| Auto-tiering | Tự chuyển dữ liệu nóng/lạnh giữa các tầng đĩa (SSD/SAS/NL-SAS) | Nâng cao | ✔ FAST VP (chỉ model hybrid) | ✔ Easy Tier | ✔ Dynamic Tiering | – chưa xác nhận | Unity, IBM: kèm sẵn · G700: gói Foundation (Data Mobility) | Unity ✔ (storagetype:tiering) · IBM ✔ (easytier) · Hitachi ◐ theo pool |
| SSD cache | Dùng SSD làm cache hoặc đẩy nhanh dữ liệu nóng lên flash | Nâng cao | ✔ FAST Cache (chỉ hybrid, tối đa 6 TB) | ✖ | ◐ active flash (một chế độ của Dynamic Tiering) | – chưa xác nhận | Unity: kèm sẵn · G700: gói Foundation | ✖ không điều khiển từ Cinder |
| Replication đồng bộ | Nhân bản sang tủ khác, RPO = 0, khoảng cách ngắn | Nâng cao | ✔ Native Sync Replication | ✔ Metro Mirror | ✔ TrueCopy | ✔ TrueCopy | Unity, IBM: kèm sẵn · Hitachi: gói Advanced | Unity ✔ · IBM ✔ · Hitachi ✖ |
| Replication bất đồng bộ | Nhân bản sang site xa, RPO tính bằng giây/phút | Nâng cao | ✔ Native Async Replication | ✔ Global Mirror, policy-based replication | ✔ Universal Replicator | ✔ Universal Replicator | Như trên | Unity ✔ · IBM ✔ · Hitachi ✖ |
| Replication theo consistency group | Nhân bản cả nhóm volume, giữ nhất quán giữa chúng | Nâng cao | ✔ | ✔ | ✔ | ✔ | Như trên | Unity ✔ · IBM ✔ · Hitachi ✖ |
| Failover/failback sang site DR | Chuyển vai trò chính sang tủ DR và chuyển ngược lại | Nâng cao | ✔ | ✔ | ✔ | ✔ | Như trên | Unity ✔ · IBM ✔ · Hitachi ✖ |
| Active-active metro cluster | Một volume đọc/ghi đồng thời ở 2 site, failover tự động | Nâng cao | ◐ cần metro node hoặc VPLEX | ✔ HyperSwap (tối đa 300 km) | ✔ global-active device | ✔ global-active device (tối đa 500 km) | Unity: mua riêng · IBM: kèm sẵn (xem ghi chú) · Hitachi: gói Advanced | IBM ✔ (volume_topology=hyperswap) · Unity ✖ · Hitachi ✖ |
| Mirror volume giữa 2 pool | Giữ 2 bản sao của volume ở 2 pool trong cùng một tủ | Nâng cao | – | ✔ Volume Mirroring | ✔ ShadowImage | ✔ ShadowImage | IBM: kèm sẵn · Hitachi: gói Foundation/Base | IBM ✔ (mirror_pool) · còn lại ✖ |
| Di chuyển LUN giữa pool không gián đoạn | Tủ tự chuyển dữ liệu sang pool khác khi host vẫn I/O | Nâng cao | ✔ | ✔ | ✔ Tiered Storage Manager | ✔* | Unity, IBM: kèm sẵn · G700: gói Foundation | Unity ✔ · IBM ✔ · Hitachi ✖ (chỉ host-assisted) |
| Đổi thuộc tính LUN online (retype) | Đổi thin/thick, nén, tier... của volume đang dùng | Nâng cao | ✔ | ✔ | ✔ | ✔ | Kèm sẵn | Unity ✔ · IBM ✔ · Hitachi ◐ qua migrate |
| Import LUN có sẵn (manage/unmanage) | Đưa LUN đã tồn tại trên tủ vào hệ thống quản lý, không copy dữ liệu | Nâng cao | ✔ | ✔ | ✔ | ✔ | Không cần license | IBM ✔ · Hitachi ✔ · Unity – |
| Ảo hóa tủ ngoài | Dùng LUN của tủ hãng khác làm dung lượng cho tủ này | Nâng cao | ✖ (chỉ có SAN Copy Pull để migrate) | ✔ hơn 500 loại tủ | ✔ Universal Volume Manager | ✔ Storage Virtualization | IBM: mua riêng theo dung lượng · Hitachi: gói Foundation/Base | ◐ trong suốt, Cinder không quản lý |
| Migrate dữ liệu từ tủ khác | Chuyển dữ liệu từ tủ cũ sang không gián đoạn host | Nâng cao | ◐ native từ VNX, SAN Copy Pull từ tủ hãng khác | ✔ | ✔ Nondisruptive migration | ✔ Non-disruptive Migration | Unity, E590H: kèm sẵn · IBM: miễn phí 90 ngày · G700: tùy chọn | ✖ ngoài phạm vi Cinder |
| Mã hóa dữ liệu lưu trữ | Mã hóa dữ liệu trên đĩa (data-at-rest) | Nâng cao | ✔ D@RE, có KMIP | ✔ AES-XTS 256, USB key hoặc key server | ✔ AES-256-XTS | ✔ | Unity: tùy chọn lúc đặt hàng · IBM: feature code riêng · G700: license và phần cứng riêng | ◐ trong suốt; Cinder có mã hóa volume riêng (LUKS) |
| Snapshot bất biến chống ransomware | Snapshot không thể sửa/xóa trước hạn lưu giữ | Nâng cao | ✖ (chỉ có File-Level Retention cho NAS) | ✔ Safeguarded Copy | ◐ Data Retention Utility | ◐* | IBM: kèm sẵn · Hitachi: kèm SVOS RF | ✖ cả 3 |
| Replication 3 site | Kết hợp bản đồng bộ gần và bản bất đồng bộ xa | Nâng cao | – | ✔ | ✔ 3DC (GAD + UR, TC + UR) | ✔ 3DC | IBM: kèm sẵn · Hitachi: gói Advanced | ✖ (Cinder chỉ 1 replication target mỗi backend) |
| FC auto-zoning | Tự tạo/xóa zone trên SAN switch khi attach/detach | Nâng cao | – | – | – | – | Không áp dụng | ✔ qua FC Zone Manager của Cinder |
| Backup volume không gián đoạn | Backup từ snapshot, không phải dừng hay clone volume | Nâng cao | ✔ | ✔ | ✔ | ✔ | Theo license snapshot | Unity ✔ · Hitachi ✔ · IBM – |
| Cinder active/active HA | Chạy nhiều cinder-volume song song cho cùng một backend | Nâng cao (phía OpenStack) | – | – | – | – | Không áp dụng | ✖ cả 3 driver |

## Nguồn

Tài liệu hãng:

- [Dell Unity XT Series Specification Sheet](https://www.delltechnologies.com/asset/en-my/products/storage/technical-support/h17713_dell_emc_unity_xt_series_ss.pdf)
- [Dell Unity XT HFA Top Reasons (metro node)](https://www.delltechnologies.com/asset/he-il/products/storage/briefs-summaries/dell-unity-xt-hfa-top-reasons.pdf)
- [IBM Storage FlashSystem 7300 Product Guide (Storage Virtualize 8.6)](https://redbooks.ibm.com/redpapers/pdfs/redp5668.pdf)
- [Hitachi: Base and Advanced Software Packages for VSP Midrange Storage](https://www.hitachivantara.com/en-us/pdf/datasheet/base-advanced-software-packages-for-vsp-midrange-storage-datasheet.pdf)
- [Hitachi VSP G/F350, G/F370, G/F700, G/F900: Software components and features](https://knowledge.hitachivantara.com/Documents/Storage/VSP_G130_GF350_GF370_GF700_GF900/88-01-0x/About_Your_System/Product_Overview/Software_components_and_features)
- [Hitachi VSP E Series Family Matrix](https://www.hitachivantara.com/en-us/pdf/specifications/virtual-storage-platform-e-series-family-matrix.pdf)
- [Hitachi VSP E590/E790 Hardware Reference: Introduction](https://knowledge.hitachivantara.com/Documents/Storage/VSP_E_Series/93-06-6x/VSP_E590_VSP_E790_Hardware_Reference/01_Introduction)

Tài liệu OpenStack:

- [Cinder Driver Support Matrix (Yoga)](https://docs.openstack.org/cinder/yoga/reference/support-matrix.html)
- [Dell EMC Unity driver (Yoga)](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/dell-emc-unity-driver.html)
- [IBM Spectrum Virtualize family volume driver (Yoga)](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/ibm-storwize-svc-driver.html)
- [Hitachi block storage driver (Yoga)](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/hitachi-vsp-driver.html)
- [Hitachi block storage driver (2023.1)](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/hitachi-vsp-driver.html)