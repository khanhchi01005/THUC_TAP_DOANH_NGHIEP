# Tính năng tủ SAN và mức hỗ trợ trên OpenStack Cinder

**Tủ:** Dell Unity XT 880 · IBM FlashSystem 7300 · Hitachi VSP G700 · Hitachi VSP E590H
**OpenStack:** Yoga (03/2022) · 2023.1 Antelope (03/2023) · 2024.1 Caracal (04/2024)

---

## Khái niệm cơ bản

### Tủ SAN

**Tủ SAN** (storage array) là thiết bị lưu trữ tập trung: gom nhiều ổ cứng vào một nơi rồi cấp dung lượng cho các máy chủ qua mạng, thay cho việc mỗi máy chủ dùng ổ cứng riêng. **SAN** (Storage Area Network) là mạng riêng nối máy chủ với tủ.

| Thuật ngữ | Nghĩa |
|---|---|
| Pool | Một nhóm ổ cứng gộp lại thành một kho dung lượng chung |
| LUN (volume) | Một phần dung lượng cắt từ pool, cấp cho máy chủ. Với máy chủ, LUN hiện ra như một ổ cứng thông thường |
| Host | Máy chủ được cấp LUN |
| FC, iSCSI | Hai giao thức kết nối máy chủ với tủ: FC dùng cáp quang và switch riêng, iSCSI chạy trên mạng Ethernet |
| Firmware | Phần mềm chạy bên trong tủ, quyết định tủ làm được những gì |
| License | Giấy phép mở khóa tính năng trên tủ, một số phải mua thêm |

### OpenStack, Cinder và driver

**OpenStack** là phần mềm dựng đám mây riêng, cho phép tạo máy ảo, ổ đĩa, mạng theo yêu cầu. **Cinder** là dịch vụ của OpenStack chuyên cấp ổ đĩa cho máy ảo. Cinder không tự lưu dữ liệu mà điều phối một hệ thống lưu trữ phía sau, ở đây là tủ SAN.

**Driver** là phần mềm trung gian giúp Cinder điều khiển được tủ SAN. Mỗi hãng tủ có cách ra lệnh riêng, nên hãng viết driver để dịch yêu cầu của Cinder thành lệnh mà tủ của họ hiểu.

- **Ai viết:** hãng tủ viết và đóng góp vào mã nguồn OpenStack.
- **Nằm ở đâu:** có sẵn trong Cinder, không cài riêng.
- **Phiên bản:** mỗi bản OpenStack kèm một phiên bản driver; bản mới có thể được hãng bổ sung tính năng.

### Nguyên tắc xuyên suốt báo cáo

Một tính năng của tủ chỉ dùng được qua OpenStack khi đủ hai điều kiện:

1. **Tủ có tính năng đó** (firmware hỗ trợ và có license).
2. **Driver của hãng hỗ trợ tính năng đó** ở bản OpenStack đang dùng.

Nếu tủ có mà driver chưa hỗ trợ, tính năng vẫn dùng được nhưng phải cấu hình trực tiếp trên tủ; OpenStack không điều khiển được. Bảng chính ở mục 3 chia hai nửa "Trên tủ" và "Qua OpenStack" theo đúng hai điều kiện này.

---

## 1. Tóm tắt

1. **Unity và FS7300:** tích hợp với OpenStack đầy đủ và **không đổi** qua cả ba bản.
2. **Hitachi:** mọi khác biệt giữa ba bản đều nằm ở driver Hitachi. Caracal là bản đầu tiên có active-active (GAD), nén/dedup và migration do tủ thực hiện.
3. **Hitachi vẫn thiếu ở cả ba bản:** QoS và replication/DR qua Cinder. Hai việc này phải cấu hình trực tiếp trên tủ.
4. **E590H:** tài liệu driver ở cả ba bản không ghi tên E590H; từ Antelope có ghi E590, là bản all-flash cùng dòng và cùng hệ điều hành. Về kỹ thuật khả năng cao chạy được, nhưng chưa có xác nhận hỗ trợ chính thức. Chi tiết ở mục 6.

---

## 2. Cách đọc bảng

Bảng chính có hai nửa:

- **Trên tủ:** tủ có tính năng đó hay không (theo tài liệu hãng).
- **Qua OpenStack:** driver Cinder của hãng có điều khiển được tính năng đó hay không. Mỗi hãng một cột, áp dụng cho cả ba bản Yoga, Antelope, Caracal.

Driver Unity và driver IBM giống nhau ở cả ba bản. Driver Hitachi phần lớn cũng giống; những dòng chỉ có từ một bản nào đó được ghi rõ **"✔ từ Antelope"** hoặc **"✔ từ Caracal"** (các bản trước đó không có). Tóm tắt theo bản ở cuối mục 5.

| Ký hiệu | Nghĩa |
|:---:|---|
| ✔ | Có |
| ✖ | Không |
| ✔ từ Antelope | Driver Hitachi có từ bản 2023.1 Antelope trở đi; Yoga không có |
| ✔ từ Caracal | Driver Hitachi có từ bản 2024.1 Caracal trở đi; Yoga và Antelope không có |
| ◐ | Một phần, hoặc có điều kiện (xem ghi chú) |
| – | Tài liệu không nêu, hoặc chưa xác nhận được |
| ¹ ² ³ | Số ghi chú ở mục 4 |

---

## 3. Bảng chính

| Tính năng | Mô tả | Unity 880 | FS7300 | G700 | E590H | Driver Unity | Driver IBM | Driver Hitachi |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| | | **Trên tủ** | | | | **Qua OpenStack** | | |
| **A. Cơ bản** | | | | | | | | |
| Tạo/xóa LUN, map/unmap host | Cấp phát volume từ pool và gán cho host | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Mở rộng LUN (kể cả đang attach) | Tăng dung lượng volume, không cần tạo lại | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ ¹ | ✔ ¹ |
| Giao thức FC, iSCSI | Kết nối host tới tủ qua Fibre Channel hoặc iSCSI | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Thin provisioning | Cấp dung lượng logic lớn hơn vật lý, chỉ chiếm chỗ khi ghi | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Thick provisioning | Cấp phát đủ dung lượng vật lý ngay khi tạo | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | – |
| Snapshot | Bản chụp volume tại một thời điểm | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Clone, tạo volume từ snapshot | Tạo volume mới độc lập từ volume hoặc snapshot | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Khôi phục LUN về snapshot | Đưa volume về đúng trạng thái của snapshot | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Chia sẻ LUN cho nhiều host | Một volume gán đồng thời cho nhiều host (cluster) | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| **B. Hiệu năng và tiết kiệm dung lượng** | | | | | | | | |
| NVMe phía host | Giao thức NVMe qua FC/Ethernet, độ trễ thấp hơn SCSI | ✖ | ✔ | ✖ | – ² | ✖ | ✖ | ✖ |
| QoS giới hạn IOPS/băng thông | Đặt trần IOPS hoặc MB/s cho từng volume | ✔ | ✔ | ✔ | ✔ ³ | ✔ | ✔ | ✖ |
| Nén dữ liệu | Nén inline để giảm dung lượng vật lý | ✔ | ✔ | ✔ ⁴ | ✔ | ✔ ⁵ | ✔ | ✔ từ Caracal ⁶ |
| Dedup | Loại bỏ các block dữ liệu trùng lặp | ✔ | ✔ | ✔ ⁴ | ✔ | – | – | ✔ từ Caracal ⁶ |
| Auto-tiering | Tự chuyển dữ liệu nóng/lạnh giữa các tầng đĩa | ✔ ⁷ | ✔ | ✔ | – ² | ✔ | ✔ | ◐ ⁸ |
| **C. Bảo vệ dữ liệu và DR** | | | | | | | | |
| Consistency group, snapshot nhóm | Snapshot nhất quán cho một nhóm volume | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Replication đồng bộ | Nhân bản sang tủ khác, RPO = 0, khoảng cách ngắn | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Replication bất đồng bộ | Nhân bản sang site xa, RPO tính bằng giây/phút | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Replication theo consistency group | Nhân bản cả nhóm volume, giữ nhất quán | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Failover/failback sang site DR | Chuyển vai trò chính sang tủ DR và chuyển ngược lại | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Active-active metro cluster | Một volume đọc/ghi đồng thời ở 2 tủ, failover tự động | ◐ ⁹ | ✔ | ✔ | ✔ | ✖ | ✔ | ✔ từ Caracal ¹⁰ ¹¹ |
| Mirror volume giữa 2 pool | Hai bản sao của volume ở 2 pool trong cùng tủ | – | ✔ | ✔ | ✔ | ✖ | ✔ | ✖ |
| Replication 3 site | Bản đồng bộ gần kết hợp bản bất đồng bộ xa | – | ✔ | ✔ | ✔ | ✖ | ✖ | ✖ |
| Snapshot bất biến chống ransomware | Snapshot không thể sửa/xóa trước hạn lưu giữ | ✖ | ✔ | ◐ ¹² | ◐ ¹² | ✖ | ✖ | ✖ |
| Backup volume không gián đoạn | Backup từ snapshot, không phải dừng volume | ✔ | ✔ | ✔ | ✔ | ✔ | – | ✔ |
| Mã hóa dữ liệu trên tủ | Mã hóa dữ liệu lưu trên đĩa (data-at-rest) | ✔ | ✔ | ✔ | ✔ | ◐ ¹³ | ◐ ¹³ | ◐ ¹³ |
| **D. Di chuyển và quản lý volume** | | | | | | | | |
| Di chuyển LUN giữa pool do tủ thực hiện | Tủ tự chuyển dữ liệu sang pool khác khi host vẫn I/O | ✔ | ✔ | ✔ | ✔ ³ | ✔ | ✔ | ✔ từ Caracal ¹⁴ |
| Đổi thuộc tính LUN online (retype) | Đổi thin/thick, nén, tier của volume đang dùng | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ◐ ¹⁵ |
| Import LUN có sẵn (manage/unmanage) | Đưa LUN đã tồn tại trên tủ vào Cinder, không copy dữ liệu | ✔ | ✔ | ✔ | ✔ | – | ✔ | ✔ |
| Ảo hóa tủ ngoài | Dùng LUN của tủ hãng khác làm dung lượng cho tủ này | ✖ | ✔ | ✔ | ✔ | ✖ | ◐ ¹³ | ◐ ¹³ |
| **E. Vận hành phía OpenStack** | | | | | | | | |
| Model được driver ghi tên | Tủ có trong danh sách hỗ trợ của tài liệu driver | | | | | ✔ | ✔ | G700 ✔ · E590H ✖ ¹⁶ |
| Nhiều pool trên một backend | Một backend Cinder quản lý nhiều pool của tủ | | | | | ✔ | ✔ | ✔ từ Antelope |
| FC auto-zoning | Tự tạo/xóa zone trên SAN switch khi attach/detach | | | | | ✔ | – | ✔ từ Antelope |
| Cinder active/active HA | Chạy nhiều cinder-volume song song cho một backend | | | | | ✖ | ✖ | ✖ |

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
| ¹⁶ | Từ Antelope, driver ghi tên E590 (all-flash), E790, E1090, E1090H; E590H vẫn không có tên. "✖" ở đây nghĩa là không được ghi tên, không có nghĩa là đã xác định không chạy được. Xem mục 6. |

---

## 5. So sánh ba hãng

Phạm vi so sánh: tính năng có trên tủ và mức OpenStack điều khiển được. Báo cáo không đánh giá giá, hiệu năng và chất lượng hỗ trợ.

| Tiêu chí | Dell Unity 880 | IBM FS7300 | Hitachi G700, E590H |
|---|---|---|---|
| Điểm mạnh nhất | Phần mềm trọn gói, ít phải mua thêm | Bộ tính năng đầy đủ nhất | Bảo vệ dữ liệu và dự phòng thảm họa mạnh trên tủ |
| Nhân bản sang tủ dự phòng | Có, OpenStack điều khiển được | Có, OpenStack điều khiển được | Có (gói Advanced), phải cấu hình trên tủ |
| Active-active | Phải mua thêm thiết bị, OpenStack không điều khiển | Có sẵn, OpenStack điều khiển được | Có (gói Advanced), OpenStack điều khiển được từ Caracal |
| Nén, dedup | Có, OpenStack điều khiển được nén | Có, OpenStack điều khiển được nén | Có, OpenStack điều khiển được từ Caracal |
| Giới hạn tốc độ (QoS) | Có, OpenStack điều khiển được | Có, OpenStack điều khiển được | Có, phải cấu hình trên tủ |
| Snapshot chống ransomware | Không có cho ổ đĩa | Có, cấu hình trên tủ | Chưa xác nhận đầy đủ |
| License | Gần như trọn gói | Trọn gói, trừ mã hóa và ảo hóa tủ ngoài | Chia gói cơ bản và gói Advanced |
| Tích hợp OpenStack | Đầy đủ, ổn định | Đầy đủ nhất | Hạn chế, tốt dần theo từng bản |
| Điểm yếu chính | Không có active-active sẵn | Một số tính năng phải mua thêm | Driver OpenStack thiếu QoS và nhân bản |

**Nhận định:** IBM có bộ tính năng và mức tích hợp OpenStack đầy đủ nhất. Dell Unity đơn giản, trọn gói, tích hợp ổn định nhưng thiếu active-active sẵn có. Hitachi mạnh trên tủ nhưng quản lý qua OpenStack còn hạn chế.

### Về ba bản OpenStack

- **Unity và FS7300:** ba bản Yoga, Antelope, Caracal hỗ trợ như nhau.
- **Hitachi:** bản càng mới càng đầy đủ. Caracal là bản đầu tiên điều khiển được active-active, nén/dedup và di chuyển ổ đĩa do tủ thực hiện.
- **Hitachi ở cả ba bản:** QoS và nhân bản vẫn phải cấu hình trực tiếp trên tủ.

---

## 6. Riêng về Hitachi VSP E590H

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
- [Hitachi VSP E590/E790 Hardware Reference: Introduction](https://docs.hitachivantara.com/r/en-us/mk-97hm85050/latest/introduction)
- [Hitachi VSP E590/E790 Hardware Reference: Storage system specifications](https://docs.hitachivantara.com/r/en-us/mk-97hm85050/latest/introduction/storage-system-specifications)

**Tra cứu thêm về E590H**
- [Hitachi block storage driver, bản mới nhất](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/hitachi-vsp-driver.html)
- [Hitachi Product Compatibility Guide: Block Storage Driver for OpenStack](https://compatibility.hitachivantara.com/products/openstack)
- [Hitachi Block Storage Driver for Red Hat OpenStack Platform Install Guide](https://docs.hitachivantara.com/api/khub/documents/N_FAyhMyqh1QnS_yzDQA9w/content)
- [Hitachi Block Storage Driver for Red Hat OpenStack Services on OpenShift](https://docs.hitachivantara.com/api/khub/documents/cZnLwtM7OoHeNxr_LUmIKw/content)