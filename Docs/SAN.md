# Tính năng tủ SAN và mức hỗ trợ trên OpenStack Cinder

**Tủ khảo sát:** Dell Unity XT 880 · IBM FlashSystem 7300 · Hitachi VSP G700 · Hitachi VSP E590H
**Bản OpenStack:** Yoga (03/2022, Cinder 20) · 2023.1 Antelope (03/2023, Cinder 22) · 2024.1 Caracal (04/2024, Cinder 24)
**Cách làm:** đối chiếu tài liệu, chưa kiểm thử trên thiết bị thật. Đã rà lại toàn bộ nguồn ngày 09/10/2026.



## 1. Tóm tắt

1. **Chức năng bắt buộc của Cinder:** cả ba driver đều có đủ ở cả ba bản.
2. **Dell Unity và IBM FlashSystem 7300:** driver hỗ trợ gần hết chức năng tùy chọn (QoS, nhân bản, di chuyển volume do tủ thực hiện), ổn định qua ba bản.
3. **Hitachi:** bước thay đổi lớn là từ Yoga lên Antelope. Antelope thêm GAD, nén/dedup, di chuyển volume do tủ thực hiện, nhiều pool. QoS và nhân bản kèm failover thì cả ba bản đều chưa có.
5. **VSP E590H:** sổ tay của Hitachi xác nhận tủ có các tính năng trong bảng. Riêng tài liệu driver ở cả ba bản không ghi tên model này (từ Antelope có ghi E590), nên chưa có xác nhận driver hỗ trợ chính thức (mục 9).

---

| Ký hiệu | Nghĩa |
|:---:|---|
| ✔ | Có, có tài liệu xác nhận |
| ✖ | Không: tài liệu và mã nguồn driver không có chức năng này |
| ? | Chưa tìm thấy tài liệu xác nhận, chưa kết luận được |

---

## 2. Bảng so sánh tính năng của tủ và mức hỗ trợ của driver

Áp dụng cho cả ba bản Yoga, Antelope, Caracal, trừ các ô ghi "từ Antelope". Cột driver của Hitachi áp dụng chính thức cho VSP G700; VSP E590H chưa được tài liệu driver ghi tên (mục 9).

| Tính năng | Giải thích | Cinder yêu cầu | Tủ Dell Unity XT 880 | Tủ IBM FlashSystem 7300 | Tủ Hitachi VSP G700 | Tủ Hitachi VSP E590H | Driver Cinder Dell | Driver Cinder IBM | Driver Cinder  Hitachi |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Tạo, xóa ổ đĩa (volume) | Cấp phát và thu hồi ổ đĩa từ pool | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Gắn, gỡ ổ đĩa cho máy chủ | Cho máy chủ nhìn thấy và dùng ổ đĩa, hoặc thu lại | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Tăng dung lượng ổ đĩa | Ví dụ từ 100 GB lên 200 GB, không phải tạo lại | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Tạo, xóa snapshot | Bản chụp ổ đĩa tại một thời điểm | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Tạo ổ đĩa mới từ snapshot | Ổ đĩa mới từ một bản chụp | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Clone | Ổ đĩa mới độc lập từ ổ đĩa có sẵn | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Tăng dung lượng khi ổ đĩa đang được dùng | Không phải tắt máy ảo hay gỡ ổ đĩa ra | Tùy chọn | ✔ | ✔ | ✔ | ? | ✔ | ✔ | ✔ |
| Giới hạn tốc độ ổ đĩa (QoS) | Đặt trần IOPS hoặc MB/s cho từng ổ đĩa | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Nhân bản sang tủ dự phòng (replication) | Sao chép liên tục sang tủ thứ hai, kèm failover từ Cinder | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Snapshot đồng thời nhiều ổ đĩa (consistency group) | Gom nhiều ổ đĩa để snapshot cùng thời điểm | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Cấp phát mỏng (thin provisioning) | Cấp dung lượng logic lớn hơn vật lý | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Chuyển ổ đĩa sang pool khác (tủ tự chép) | Tủ tự chuyển dữ liệu sang pool khác (storage-assisted) | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ từ Antelope |
| Multi-attach | Dùng cho hệ thống chạy cụm nhiều máy | Tùy chọn | ✔ | ✔ | ✔ | ? | ✔ | ✔ | ✔ |
| Revert to snapshott | Đưa ổ đĩa về trạng thái của bản chụp | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Thick provisioning | Cấp đủ dung lượng vật lý ngay khi tạo | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Nén dữ liệu | Nén để giảm dung lượng vật lý | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ chỉ pool toàn SSD | ✔ | ✔ từ Antelope |
| Loại dữ liệu trùng (dedup) | Dữ liệu giống nhau chỉ lưu một bản | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✖ | ✖ | ✔ từ Antelope |
| Automated tiering | Tự chuyển dữ liệu nóng, lạnh giữa các tầng ổ | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Replication đồng bộ | Ghi xong ở cả hai tủ mới báo thành công | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Replication bất đồng bộ | Ghi ở tủ chính trước, chép sang tủ kia sau | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Replication theo nhóm volume | Nhân bản và chuyển sang tủ dự phòng theo cả nhóm | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Hai bản sao ở hai pool cùng tủ (mirror) | Một pool hỏng thì ổ đĩa vẫn còn bản ở pool kia | Tùy chọn | ? | ✔ | ✔ | ✔ | ✖ | ✔ | ✖ |

---

## Lưu ý khi đọc bảng


- **Dấu "?" không có nghĩa là "không có"**, chỉ là chưa tìm thấy tài liệu ghi rõ. Bảng còn 3 ô "?".
- **Dấu ✖ ở cột Driver** nghĩa là muốn dùng tính năng đó thì phải cấu hình trực tiếp trên tủ.
- **Tên gọi riêng của từng hãng** cho mỗi tính năng xem ở Phụ lục A; **license** xem ở Phụ lục B.


## 3. So sánh ba hãng

Phạm vi: tính năng có trên tủ và mức hỗ trợ của driver. Không đánh giá giá, hiệu năng, chất lượng hỗ trợ.

| Tiêu chí | Dell Unity XT 880 | IBM FlashSystem 7300 | Hitachi VSP G700, VSP E590H |
|---|---|---|---|
| Chức năng bắt buộc của Cinder | Đủ | Đủ | Đủ |
| QoS | Có, driver hỗ trợ | Có, driver hỗ trợ | Có, driver chưa hỗ trợ |
| Nhân bản sang tủ dự phòng | Có, driver hỗ trợ | Có, driver hỗ trợ | Có (gói Advanced), driver chưa hỗ trợ |
| Active-active | Phải mua thêm thiết bị, driver không hỗ trợ | Có, driver hỗ trợ | Có (gói Advanced), driver hỗ trợ từ Antelope |
| Nén, dedup | Có; driver hỗ trợ nén, không có dedup | Có; driver hỗ trợ nén, không có dedup | Có; driver hỗ trợ cả hai từ Antelope |
| Thick provisioning, tiering | Có, driver hỗ trợ | Có, driver hỗ trợ | Có, driver không hỗ trợ |
| License | Gần như trọn gói | Trọn gói, trừ mã hóa và ảo hóa tủ ngoài | Chia gói cơ bản và gói Advanced |
| Tích hợp OpenStack | Đầy đủ, ổn định | Đầy đủ nhất | Hạn chế hơn; đủ dùng từ Antelope, thiếu QoS và nhân bản |

**Nhận định:** IBM có bộ tính năng và mức tích hợp OpenStack đầy đủ nhất. Dell Unity trọn gói, tích hợp ổn định, không có active-active sẵn. Hitachi mạnh trên tủ; qua OpenStack thì từ Antelope đã có phần lớn tính năng, trừ QoS và nhân bản.

---

## 4. Riêng về Hitachi VSP E590H

**E590 và E590H khác nhau thế nào**

| | VSP E590 | VSP E590H |
|---|---|---|
| Loại | All-flash | Hybrid (lai) |
| Ổ đĩa | Chỉ SSD | SSD kết hợp HDD, tối đa 480 ổ HDD 3.5 inch |
| Hệ điều hành tủ | SVOS RF | SVOS RF |
| Cổng kết nối máy chủ | FC, iSCSI | FC, iSCSI |

**Tài liệu driver ghi gì** (đã kiểm tra bảng "Supported storages" của từng bản)

| Bản OpenStack | Dòng E được ghi tên | Có "E590H" |
|---|---|:---:|
| Yoga | E990 | Không |
| 2023.1 Antelope | E590, E790, E990, E1090, E1090H | Không |
| 2024.1 Caracal | E590, E790, E990, E1090, E1090H | Không |
| Bản mới nhất (10/2026) | E590, E790, E990, E1090, E1090H | Không |

Đã tra thêm tài liệu của Hitachi (trang tương thích driver OpenStack, hai hướng dẫn cài driver cho Red Hat OpenStack): không nguồn nào khẳng định driver hỗ trợ E590H, cũng không nguồn nào khẳng định không.

**Tính năng trên tủ E590H.** Bộ sổ tay SVOS RF 9.8 của Hitachi ghi rõ áp dụng cho "E390H, E590H, E790H, E1090H" cùng với G700, và ghi các model lai và model toàn SSD có thao tác về cơ bản giống nhau. Các ô ✔ ở cột Tủ Hitachi VSP E590H dựa trên bộ sổ tay này.

**Kết luận về E590H**

- Phía tủ: sổ tay Hitachi xác nhận E590H có các tính năng trong bảng.
- Phía driver: tài liệu driver không ghi tên E590H ở bản nào; E590 cùng dòng, cùng hệ điều hành SVOS RF được ghi tên từ Antelope.
- Chưa có tài liệu nào xác nhận driver hỗ trợ chính thức E590H, nên báo cáo không kết luận E590H dùng được hay không với driver.
- Triển khai khi chưa có xác nhận thì hãng có thể từ chối hỗ trợ khi có sự cố. Cần hỏi Hitachi trước.

---


## Phụ lục A. Tên gọi tính năng theo hãng

| Tính năng | Dell Unity XT | IBM FlashSystem | Hitachi VSP |
|---|---|---|---|
| Thin provisioning | Thin LUN | Thin volume | Dynamic Provisioning |
| Snapshot | Unified Snapshots | FlashCopy | Thin Image |
| Clone | Thin Clone | FlashCopy (có chép nền) | ShadowImage, Thin Image |
| QoS | Quality of Service | Throttling | QoS controls |
| Nén, dedup | Inline Data Reduction | Data Reduction Pool, nén trên FCM | Adaptive Data Reduction |
| Auto-tiering | FAST VP | Easy Tier | Dynamic Tiering |
| Nhân bản đồng bộ | Native Sync Replication | Metro Mirror | TrueCopy |
| Nhân bản bất đồng bộ | Native Async Replication | Global Mirror | Universal Replicator |
| Active-active | metro node, VPLEX (mua thêm) | HyperSwap | global-active device (GAD) |

## Phụ lục B. License phía tủ

| Tủ | Kèm sẵn | Phải mua thêm hoặc tùy chọn |
|---|---|---|
| Dell Unity XT 880 | Thin, data reduction, QoS, snapshot, thin clone, nhân bản đồng bộ và bất đồng bộ, FAST VP và FAST Cache | metro node, VPLEX, RecoverPoint Advanced, PowerPath, mã hóa |
| IBM FlashSystem 7300 | Mọi tính năng cho ổ lắp trong tủ, trừ các mục bên phải | Ảo hóa tủ ngoài, mã hóa. HyperSwap: tài liệu ghi cần license remote mirroring |
| Hitachi VSP G700 | Gói Foundation: Universal Volume Manager, Thin Image, ShadowImage, Dynamic Tiering | Gói Advanced: TrueCopy, Universal Replicator, GAD. Mã hóa: license và phần cứng riêng |
| Hitachi VSP E590H | Gói Base: Adaptive Data Reduction, Storage Virtualization, In-System Replication, Non-disruptive Migration | Gói Advanced: nhân bản từ xa, GAD. Dynamic Tiering: cần hỏi Hitachi |

---

## Nguồn

**Quy định và danh mục chức năng của Cinder**
- [Cinder Support Matrix, bản Yoga](https://docs.openstack.org/cinder/yoga/reference/support-matrix.html)
- [Cinder support-matrix.ini, bản 2023.1](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2023.1/doc/source/reference/support-matrix.ini)
- [Cinder support-matrix.ini, bản 2024.1](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/doc/source/reference/support-matrix.ini)
- [Cinder: tài liệu cho người viết driver (tính năng tối thiểu)](https://docs.openstack.org/cinder/2024.1/contributor/drivers.html)

**Release notes của Cinder**
- [Yoga](https://docs.openstack.org/releasenotes/cinder/yoga.html) · [Zed](https://docs.openstack.org/releasenotes/cinder/zed.html) · [2023.1 Antelope](https://docs.openstack.org/releasenotes/cinder/2023.1.html) · [2023.2 Bobcat](https://docs.openstack.org/releasenotes/cinder/2023.2.html) · [2024.1 Caracal](https://docs.openstack.org/releasenotes/cinder/2024.1.html)

**Tài liệu driver**
- Dell Unity: [Yoga](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/dell-emc-unity-driver.html) · [2023.1](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/dell-emc-unity-driver.html) · [2024.1](https://docs.openstack.org/cinder/2024.1/configuration/block-storage/drivers/dell-emc-unity-driver.html)
- IBM Storage Virtualize: [Yoga](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/ibm-storwize-svc-driver.html) · [2023.1](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/ibm-storwize-svc-driver.html) · [2024.1](https://docs.openstack.org/cinder/2024.1/configuration/block-storage/drivers/ibm-storwize-svc-driver.html)
- Hitachi VSP: [Yoga](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/hitachi-vsp-driver.html) · [2023.1](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/hitachi-vsp-driver.html) · [2024.1](https://docs.openstack.org/cinder/2024.1/configuration/block-storage/drivers/hitachi-vsp-driver.html) · [mới nhất](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/hitachi-vsp-driver.html)

**Mã nguồn driver (tác giả, phiên bản, chức năng)**
- Dell Unity: [driver.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/dell_emc/unity/driver.py) · [adapter.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/dell_emc/unity/adapter.py) · [client.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/dell_emc/unity/client.py)
- IBM Storage Virtualize: [storwize_svc_fc.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/ibm/storwize_svc/storwize_svc_fc.py) · [storwize_svc_common.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/ibm/storwize_svc/storwize_svc_common.py)
- Hitachi VSP: [hbsd_fc.py bản Yoga](https://raw.githubusercontent.com/openstack/cinder/unmaintained/yoga/cinder/volume/drivers/hitachi/hbsd_fc.py) · [bản 2023.1](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2023.1/cinder/volume/drivers/hitachi/hbsd_fc.py) · [bản 2024.1](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/hitachi/hbsd_fc.py) · [hbsd_common.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/hitachi/hbsd_common.py) · [hbsd_rest.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/hitachi/hbsd_rest.py)

**Tài liệu hãng**
- [Dell Unity XT Series Specification Sheet](https://www.delltechnologies.com/asset/en-my/products/storage/technical-support/h17713_dell_emc_unity_xt_series_ss.pdf)
- [Dell Unity: Data Reduction Considerations and Best Practices](https://www.dell.com/support/kbdoc/en-us/000022795/unity-how-to-use-compression-considerations-and-best-practices)
- [Dell Unity: About LUN move sessions](https://www.dell.com/support/manuals/en-us/unity-480/unity_p_config_luns/about-lun-move-sessions?guid=guid-e8d2a258-d7fe-4ce1-ba66-afc8f36b61f5&lang=en-us)
- [Dell Unity: Changing LUN Provisioning Between Thin and Thick](https://www.dell.com/support/kbdoc/en-us/000021267/dell-emc-unity-is-it-possible-to-change-unity-luns-from-thin-to-thick-and-vice-versa-user-correctable)
- [Dell Unity: Consistency Groups with Replication Configured](https://www.dell.com/support/kbdoc/en-us/000202243/dell-unity-best-practices-to-manage-consistency-groups-with-replication-configured-dell-correctable)
- [Dell EMC Unity: Unisphere Overview (white paper H15085)](https://resources.r2ut.com/hubfs/dell-emc-unity-unisphere-overview.pdf)
- [Dell Unity XT HFA Top Reasons (metro node)](https://www.delltechnologies.com/asset/he-il/products/storage/briefs-summaries/dell-unity-xt-hfa-top-reasons.pdf)
- [IBM Storage FlashSystem 7300 Product Guide](https://redbooks.ibm.com/redpapers/pdfs/redp5668.pdf)
- [Hitachi VSP G/F series: Software components and features](https://knowledge.hitachivantara.com/Documents/Storage/VSP_G130_GF350_GF370_GF700_GF900/88-01-0x/About_Your_System/Product_Overview/Software_components_and_features)
- [Hitachi: Base and Advanced Software Packages for VSP Midrange Storage](https://www.hitachivantara.com/en-us/pdf/datasheet/base-advanced-software-packages-for-vsp-midrange-storage-datasheet.pdf)
- [Hitachi SVOS RF Provisioning Guide cho VSP Gx00/Fx00](https://knowledge.hitachivantara.com/@api/deki/files/50438/SVOS_RF_v8_3_Provisioning_Guide_VSP_Gx00_Fx00_MK-97HM85026-02.pdf?revision=1)
- Sổ tay Hitachi SVOS RF 9.8 (tiếng Nhật, ghi rõ áp dụng cho E590H và G700): [Document Map](https://itpfdoc.hitachi.co.jp/manuals/4049/40491JU04_SVOSRF98/40491JU04.pdf) · [System Construction Guide](https://itpfdoc.hitachi.co.jp/manuals/4049/40491JU09_SVOSRF985/40491JU09.pdf) · [Thin Image User Guide](https://itpfdoc.hitachi.co.jp/manuals/4049/40491JU13_SVOSRF986/40491JU13.pdf) · [Universal Replicator User Guide](https://itpfdoc.hitachi.co.jp/manuals/4046/40461JU15_SVOSRF96/40461JU15.pdf)
- [Hitachi Thin Image: Restoring Thin Image pairs](https://docs.hitachivantara.com/r/en-us/svos/9.8.6/mk-98rd9020/managing-thin-image-pairs/restoring-thin-image-pairs)
- [Hitachi VSP E Series Family Matrix](https://www.hitachivantara.com/en-us/pdf/specifications/virtual-storage-platform-e-series-family-matrix.pdf)
- [Hitachi VSP E590/E790 Hardware Reference](https://docs.hitachivantara.com/r/en-us/mk-97hm85050/latest/introduction)
- [Hitachi Product Compatibility Guide: Block Storage Driver for OpenStack](https://compatibility.hitachivantara.com/products/openstack)