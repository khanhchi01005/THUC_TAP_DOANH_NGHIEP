# Tính năng tủ SAN và mức hỗ trợ trên OpenStack Cinder

**Tủ khảo sát:** Dell Unity XT 880 · IBM FlashSystem 7300 · Hitachi VSP G700 · Hitachi VSP E590H
**Bản OpenStack:** Yoga (03/2022, Cinder 20) · 2023.1 Antelope (03/2023, Cinder 22) · 2024.1 Caracal (04/2024, Cinder 24)
**Cách làm:** đối chiếu tài liệu, chưa kiểm thử trên thiết bị thật. Đã rà lại toàn bộ nguồn ngày 09/10/2026.

---

## Điều chỉnh so với bản trước

Bản trước ghi driver Hitachi có active-active (GAD), nén/dedup và di chuyển volume do tủ thực hiện **từ Caracal**. Kiểm tra lại với release notes chính thức của Cinder cho thấy các tính năng này được thêm **từ 2023.1 Antelope** (Cinder 22.0.0). Bản trước dựa vào trang tài liệu driver, mà trang này đến Caracal mới được cập nhật. Các kết luận liên quan trong bản này đã sửa theo release notes.

---

## 1. Tóm tắt

1. **Chức năng bắt buộc của Cinder:** cả ba driver đều có đủ ở cả ba bản.
2. **Dell Unity và IBM FlashSystem 7300:** driver hỗ trợ gần hết chức năng tùy chọn (QoS, nhân bản, di chuyển volume do tủ thực hiện), ổn định qua ba bản.
3. **Hitachi:** bước thay đổi lớn là từ Yoga lên Antelope. Antelope thêm GAD, nén/dedup, di chuyển volume do tủ thực hiện, nhiều pool. QoS và nhân bản kèm failover thì cả ba bản đều chưa có.
4. **Antelope so với Caracal:** với ba driver này không có tính năng mới, chỉ có sửa lỗi.
5. **VSP E590H:** sổ tay của Hitachi xác nhận tủ có các tính năng trong bảng. Riêng tài liệu driver ở cả ba bản không ghi tên model này (từ Antelope có ghi E590), nên chưa có xác nhận driver hỗ trợ chính thức (mục 9).

---


## 2. Cách làm và cách đọc bảng

**Cách đọc**

- Bốn cột **Tủ**: tủ có tính năng đó hay không.
- Ba cột **Driver**: dùng được tính năng đó từ OpenStack hay không.
- Cột **Cinder yêu cầu**: "Bắt buộc" là Cinder quy định mọi driver phải có; "Tùy chọn" là mỗi hãng tự quyết định có làm trong driver hay không.

| Ký hiệu | Nghĩa |
|:---:|---|
| ✔ | Có, có tài liệu xác nhận |
| ✖ | Không: tài liệu và mã nguồn driver không có chức năng này |
| ? | Chưa tìm thấy tài liệu xác nhận, chưa kết luận được |
| ¹ ² ³ | Số ghi chú|

---

## Bảng so sánh tính năng của tủ và mức hỗ trợ của driver

Áp dụng cho cả ba bản Yoga, Antelope, Caracal, trừ các ô ghi "từ Antelope". Cột driver của Hitachi áp dụng chính thức cho VSP G700; VSP E590H chưa được tài liệu driver ghi tên (mục 9).

| Tính năng | Giải thích | Cinder yêu cầu | Tủ Dell Unity XT 880 | Tủ IBM FlashSystem 7300 | Tủ Hitachi VSP G700 | Tủ Hitachi VSP E590H | Driver Cinder Dell | Driver Cinder IBM | Driver Cinder Hitachi |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Tạo, xóa ổ đĩa (volume) | Cấp phát và thu hồi ổ đĩa từ pool | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Gắn, gỡ ổ đĩa cho máy chủ | Cho máy chủ nhìn thấy và dùng ổ đĩa, hoặc thu lại | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Tăng dung lượng ổ đĩa | Ví dụ từ 100 GB lên 200 GB, không phải tạo lại | Bắt buộc | ✔ ² | ✔ | ✔ | ✔ | ✔ | ✔ ¹ | ✔ ¹ |
| Tạo, xóa snapshot | Bản chụp ổ đĩa tại một thời điểm | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Tạo ổ đĩa mới từ snapshot | Ổ đĩa mới từ một bản chụp | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Clone | Ổ đĩa mới độc lập từ ổ đĩa có sẵn | Bắt buộc | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Tăng dung lượng khi ổ đĩa đang được dùng | Không phải tắt máy ảo hay gỡ ổ đĩa ra | Tùy chọn | ✔ ² | ✔ | ✔ | ? ³ | ✔ | ✔ ¹ | ✔ ¹ |
| Giới hạn tốc độ ổ đĩa (QoS) | Đặt trần IOPS hoặc MB/s cho từng ổ đĩa | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ ⁴ |
| Nhân bản sang tủ dự phòng (replication) | Sao chép liên tục sang tủ thứ hai, kèm failover từ Cinder | Tùy chọn | ✔ | ✔ | ✔ ¹⁶ | ✔ ¹⁶ | ✔ ¹⁶ | ✔ ¹⁶ | ✖ |
| Snapshot đồng thời nhiều ổ đĩa (consistency group) | Gom nhiều ổ đĩa để snapshot cùng thời điểm | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ ⁵ | ✔ |
| Cấp phát mỏng (thin provisioning) | Cấp dung lượng logic lớn hơn vật lý | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Chuyển ổ đĩa sang pool khác (tủ tự chép) | Tủ tự chuyển dữ liệu sang pool khác (storage-assisted) | Tùy chọn | ✔ ⁷ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ từ Antelope ⁶ |
| Multi-attach | Dùng cho hệ thống chạy cụm nhiều máy | Tùy chọn | ✔ | ✔ | ✔ | ? ³ | ✔ | ✔ | ✔ |
| Revert to snapshot | Đưa ổ đĩa về trạng thái của bản chụp | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Thick provisioning | Cấp đủ dung lượng vật lý ngay khi tạo | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ ⁸ |
| Nén dữ liệu | Nén để giảm dung lượng vật lý | Tùy chọn | ✔ ⁹ | ✔ | ✔ ¹⁰ | ✔ | ✔ chỉ pool toàn SSD ⁹ | ✔ | ✔ từ Antelope ¹² |
| Loại dữ liệu trùng (dedup) | Dữ liệu giống nhau chỉ lưu một bản | Tùy chọn | ✔ | ✔ | ✔ ¹⁰ | ✔ | ✖ ¹¹ | ✖ ¹¹ | ✔ từ Antelope ¹² |
| Automated tiering | Tự chuyển dữ liệu nóng, lạnh giữa các tầng ổ | Tùy chọn | ✔ ¹³ | ✔ | ✔ | ✔ ¹⁴ | ✔ | ✔ | ✖ ¹⁵ |
| Replication đồng bộ | Ghi xong ở cả hai tủ mới báo thành công | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Replication bất đồng bộ | Ghi ở tủ chính trước, chép sang tủ kia sau | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Replication theo nhóm volume | Nhân bản và chuyển sang tủ dự phòng theo cả nhóm | Tùy chọn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Hai bản sao ở hai pool cùng tủ (mirror) | Một pool hỏng thì ổ đĩa vẫn còn bản ở pool kia | Tùy chọn | ? | ✔ | ✔ ¹⁷ | ✔ ¹⁷ | ✖ | ✔ | ✖ |

---

## Ghi chú

| # | Nội dung |
|:---:|---|
| ¹ | Driver IBM và Hitachi: ổ đĩa đang có snapshot thì không tăng dung lượng được, phải xóa snapshot trước. |
| ² | Tủ Dell: không tăng được dung lượng ổ đĩa nằm trong nhóm đang nhân bản đồng bộ. |
| ³ | Tủ Hitachi VSP E590H: sổ tay có mục tăng dung lượng ổ đĩa và gán ổ đĩa cho nhóm máy chủ, nhưng chưa tìm thấy câu ghi rõ làm được khi máy chủ đang dùng, hoặc một ổ đĩa cho nhiều máy cùng lúc. |
| ⁴ | Tủ Hitachi có QoS (Server Priority Manager), nhưng mã nguồn driver Hitachi khai báo không hỗ trợ QoS; phải cấu hình trên tủ. |
| ⁵ | Driver IBM từ Antelope có thêm kiểu nhóm mới (volume group), cần firmware 8.5.1.0 trở lên. |
| ⁶ | Driver Hitachi: chỉ chuyển trong cùng một tủ. Release notes và mã nguồn ghi có từ Antelope, nhưng Cinder Support Matrix chưa cập nhật mục này, nên cần kiểm thử trước khi dùng. |
| ⁷ | Tủ Dell: sau khi chuyển xong, các snapshot của ổ đĩa bị xóa. |
| ⁸ | Driver Hitachi chỉ tạo ổ đĩa thin (mã nguồn khai báo không hỗ trợ thick). |
| ⁹ | Driver Dell: chỉ tạo được ổ đĩa nén trên pool toàn SSD. Tủ Dell: từ phiên bản OE 5.2 nén được cả trên pool lai nếu tầng SSD chiếm từ 10% dung lượng pool. |
| ¹⁰ | Tủ Hitachi VSP G700: tài liệu firmware 88-01 ghi nén/dedup chỉ áp dụng cho ổ SSD lắp trong tủ; tài liệu SVOS RF 9.8 ghi dùng được với mọi loại ổ. Cần kiểm tra theo firmware thực tế. |
| ¹¹ | Driver Dell và IBM không có tùy chọn dedup; driver Dell chỉ có tùy chọn "ổ đĩa nén". |
| ¹² | Driver Hitachi: nén và dedup bật cùng nhau qua `hbsd:capacity_saving=deduplication_compression`, không tách riêng; cần license dedup/compression và phải bật trước trên pool. |
| ¹³ | Tủ Dell: FAST VP và FAST Cache chỉ có trên model lai (880, không phải 880F), cần license FAST VP. |
| ¹⁴ | Tủ Hitachi VSP E590H: sổ tay có mô tả Dynamic Tiering, nhưng datasheet gói phần mềm ghi gói này chỉ kèm sẵn cho dòng F/G, nên cần hỏi Hitachi về license. |
| ¹⁵ | Driver Hitachi không có tùy chọn phân tầng: phân tầng do tủ tự chạy theo pool, Cinder không đặt được chính sách cho từng ổ đĩa. |
| ¹⁶ | Nhân bản cần hai tủ cùng hãng. Driver Dell và IBM: mỗi kết nối chỉ khai báo được một tủ dự phòng. Tủ Hitachi cần gói license Advanced. |
| ¹⁷ | Tủ Hitachi dùng ShadowImage, cách hoạt động không hoàn toàn giống Volume Mirroring của IBM. Chỉ driver IBM có tùy chọn này. |

---

## 2. So sánh ba hãng

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

## 3. Khác nhau giữa ba bản OpenStack
- Unity và FS7300: ba bản Yoga, Antelope, Caracal hỗ trợ như nhau.
- Hitachi: bản càng mới càng đầy đủ. Caracal là bản đầu tiên điều khiển được active-active, nén/dedup và di chuyển ổ đĩa do tủ thực hiện.
- Hitachi ở cả ba bản: QoS và nhân bản vẫn phải cấu hình trực tiếp trên tủ.

---

## 4. 6. Riêng về Hitachi VSP E590H

### E590 và E590H khác nhau thế nào

| | VSP E590 | VSP E590H |
|---|---|---|
| Loại | All-flash | Hybrid (lai) |
| Ổ đĩa | Chỉ SSD: NVMe và SAS SSD | SSD kết hợp HDD, lắp qua các khay mở rộng |
| HDD tối đa | Không có | 480 ổ 3.5 inch SAS |
| Hệ điều hành tủ | SVOS RF | SVOS RF |
| Cổng kết nối máy chủ | FC, iSCSI | FC, iSCSI |
| Phù hợp | Ứng dụng cần độ trễ thấp | Dung lượng lớn, chi phí thấp hơn cho dữ liệu ít truy xuất |

Hitachi mô tả hai model trong cùng một bộ tài liệu phần cứng. Khác biệt chính là loại ổ đĩa lắp được, không phải phần mềm điều khiển.

### Tài liệu driver OpenStack ghi gì

Đã kiểm tra lại trực tiếp bảng "Supported storages" trên trang tài liệu driver Hitachi của từng bản:

| Bản OpenStack | Dòng E được ghi tên | Có "E590H" |
|---|---|:---:|
| Yoga | E990 | Không |
| 2023.1 Antelope | E590, E790, E990, E1090, E1090H | Không |
| 2024.1 Caracal | E590, E790, E990, E1090, E1090H | Không |
| Bản mới nhất (kiểm tra thêm, 10/2026) | E590, E790, E990, E1090, E1090H | Không |

Tài liệu ghi tên không nhất quán giữa các model: có ghi cả "E1090, E1090H", nhưng chỉ ghi "E590, E790" mà không có bản H. Không rõ đây là do chưa kiểm thử hay do tài liệu chưa cập nhật.

### Đã tra thêm phía Hitachi

| Nguồn | Kết quả |
|---|---|
| Trang Product Compatibility Guide cho driver OpenStack | Không đọc được (trang tra cứu động, đang lỗi tìm kiếm) |
| Hướng dẫn cài driver cho Red Hat OpenStack Platform 16.2 và 17.1 (09/2024) | Không có danh sách model |
| Hướng dẫn cài driver cho Red Hat OpenStack Services on OpenShift 18.0 (06/2025) | Không có danh sách model |
| User guide driver riêng của Hitachi | Chỉ tìm được bản cũ (Train, Queens), trước thời E590H |

Không có nguồn nào khẳng định có hỗ trợ, cũng không có nguồn nào khẳng định không hỗ trợ.

### Đánh giá

Cần phân biệt hai mức:

| Mức | Nghĩa là | Tình trạng với E590H |
|---|---|---|
| Chạy được về kỹ thuật | Driver gửi lệnh và tủ thực hiện được | Khả năng cao là có. Driver làm việc với hệ điều hành của tủ qua REST API, không phụ thuộc loại ổ đĩa; E590 cùng hệ điều hành đã được ghi tên từ Antelope. Đây là suy luận, chưa kiểm chứng. |
| Được hỗ trợ chính thức | Hãng đã kiểm thử, ghi tên trong tài liệu, chịu trách nhiệm khi có lỗi | Chưa có tài liệu nào xác nhận. |

Rủi ro nếu triển khai khi chưa có xác nhận: gặp sự cố thì hãng có thể từ chối hỗ trợ vì cấu hình nằm ngoài danh sách.

### Đề xuất

1. Gửi câu hỏi cho Hitachi hoặc đối tác cung cấp tủ, xin trả lời bằng văn bản: "VSP E590H có được hỗ trợ bởi Hitachi Block Storage Driver trên OpenStack Yoga, 2023.1, 2024.1 không, và yêu cầu firmware bao nhiêu?"
2. Trong lúc chờ, coi Antelope là bản thấp nhất nên cân nhắc cho E590H, vì Yoga không ghi tên cả E590.
3. Nếu có sẵn tủ và môi trường thử nghiệm, test các thao tác cơ bản (tạo, gán, mở rộng, snapshot, clone) trước khi dùng thật.

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
| Hitachi G700, E590H | Hitachi VSP driver (`HBSDFCDriver`, `HBSDISCSIDriver`) | G700: firmware 88-01-04 trở lên. E590 (được ghi tên từ Antelope): firmware 93-03-22 trở lên; E590H chưa được ghi tên. License SVOS, Dynamic Provisioning, Thin Image |

Bảng dựa trên tài liệu upstream của OpenStack. Driver do Hitachi phát hành riêng có thể hỗ trợ rộng hơn.

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