# Báo cáo so sánh tính năng tủ đĩa SAN và OpenStack Cinder

## 1. Tóm tắt

OpenStack Cinder bao phủ gần hết thao tác vòng đời volume của tủ SAN, nhưng thiếu phần lớn tính năng bảo vệ dữ liệu theo chính sách và DR nhiều site; các tính năng đó vẫn phải vận hành trực tiếp trên tủ.

- **Mức cơ bản:** trong 21 tính năng SAN, 13 có tương ứng đầy đủ trên OpenStack, 3 tương ứng một phần, 2 nằm ở tầng khác (Keystone) và 3 không có vì tủ tự xử lý.
- **Mức nâng cao:** trong 27 tính năng SAN, 6 tương ứng đầy đủ, 9 một phần, 2 thuộc tầng hoặc dịch vụ khác và 10 không có.
- **Chỉ OpenStack có:** 13 tính năng, gồm 8 mức cơ bản; hai trong số đó là hàm bắt buộc của driver (Create Image from Volume, Volume Migration qua host).
- **Theo driver bản Yoga:** năm dòng tủ đạt 8 trên 9 tính năng tùy chọn; Hitachi VSP và IBM DS8000 thấp nhất với 5 trên 9.

## 2. Phạm vi và phương pháp

Mười dòng tủ được khảo sát: Dell PowerMax, Dell PowerStore, Dell Unity, HPE 3PAR/Primera, Hitachi VSP, Huawei OceanStor Dorado, IBM FlashSystem/SVC (Spectrum Virtualize), IBM DS8000, NetApp ONTAP (AFF/ASA), Pure Storage FlashArray.

| Mức | Phía tủ SAN | Phía OpenStack Cinder |
| --- | --- | --- |
| Cơ bản | Có sẵn trên mọi dòng tủ, thường không cần license riêng | Hàm bắt buộc của driver (Required Driver Functions) hoặc chức năng lõi của Cinder |
| Nâng cao | Phụ thuộc dòng tủ, firmware hoặc license | Tính năng tùy chọn trong support matrix, hoặc cần dịch vụ, cấu hình bổ sung |

## 3. So sánh tính năng cơ bản

Bảng gồm 21 tính năng cơ bản của tủ SAN (B1–B21) và 8 tính năng cơ bản chỉ OpenStack có (O1–O8); O1 và O2 là hàm bắt buộc của driver. Cột "Tủ SAN" ghi số dòng tủ có tính năng trên mười dòng khảo sát ở mục 2, nêu rõ dòng ngoại lệ, rồi tới tên gọi theo đúng thứ tự mười dòng khi tên khác nhau giữa các hãng.

| # | Tính năng | Mô tả | Tủ SAN (trên 10 dòng khảo sát) | OpenStack Cinder | Ghi chú |
| --- | --- | --- | --- | --- | --- |
| B1 | Storage pool / RAID | Gom ổ đĩa thành pool có bảo vệ RAID để cấp dung lượng cho LUN | 10/10 dòng; 8 dòng cho quản trị viên tự tạo pool, PowerStore và Pure tự quản một pool duy nhất. Tên gọi: PowerMax: SRP; Unity: Storage Pool; 3PAR: CPG; VSP: DP Pool; Dorado: Storage Pool; Spectrum Virtualize: Storage Pool; DS8000: Extent Pool; ONTAP: Aggregate | Có: backend và pool trong scheduler | Tạo pool, RAID vẫn làm trên tủ; Cinder chỉ khai báo và chọn pool |
| B2 | Tạo / xóa LUN | Cấp phát và thu hồi ổ logic cho host | 10/10 dòng. Tên gọi: PowerMax: Volume; PowerStore: Volume; Unity: LUN; 3PAR: Virtual Volume; VSP: LDEV; Dorado: LUN; Spectrum Virtualize: Volume (VDisk); DS8000: Volume; ONTAP: LUN; Pure: Volume | Có: Create Volume, Delete Volume (bắt buộc) | Driver gọi API tủ |
| B3 | Mở rộng LUN online | Tăng dung lượng LUN khi đang dùng | 10/10 dòng | Có: Extend Volume (bắt buộc); Extend an Attached Volume (tùy chọn) | Mở rộng khi volume đang gắn vào VM là tùy chọn, cả mười driver khảo sát đều có |
| B4 | LUN mapping / masking | Gán LUN cho host theo WWPN, IQN; giới hạn host nào thấy LUN nào | 10/10 dòng. Tên gọi: PowerMax: Masking View; PowerStore: Host mapping; Unity: Host access; 3PAR: VLUN; VSP: Host Group và LU path; Dorado: Mapping View; Spectrum Virtualize: Host mapping; DS8000: Volume Group và Host connection; ONTAP: igroup và LUN map; Pure: Host connection | Có: Attach Volume, Detach Volume (bắt buộc); Multi-Attach (tùy chọn) | Gán một LUN cho nhiều host là cơ bản trên tủ nhưng là tùy chọn trên Cinder |
| B5 | Giao thức FC / iSCSI | Cổng front-end cho host truy cập block | FC: 10/10 dòng. iSCSI: 9/10 dòng, không có trên DS8000 | Có: loại driver FC hoặc iSCSI, os-brick connector | Chọn qua `volume_driver` của backend |
| B6 | Multipath / ALUA | Nhiều đường I/O tới cùng LUN, tự chuyển khi lỗi | 10/10 dòng | Có: multipath của os-brick và Nova (cấu hình) | Bật `volume_use_multipath` và multipathd trên compute |
| B7 | Thin provisioning | Cấp dung lượng logic lớn hơn vật lý | 10/10 dòng; là kiểu cấp phát duy nhất hoặc mặc định trên PowerMax, PowerStore, Dorado, Pure; DS8000 dùng volume kiểu ESE | Có: Thin Provisioning (tùy chọn) | Cơ bản trên tủ nhưng là tùy chọn trên Cinder; driver DS8000 chưa có |
| B8 | Snapshot | Bản chụp LUN tại một thời điểm | 10/10 dòng. Tên gọi: PowerMax: SnapVX; PowerStore: Snapshot; Unity: Snapshot; 3PAR: Virtual Copy; VSP: Thin Image; Dorado: HyperSnap; Spectrum Virtualize: FlashCopy; DS8000: FlashCopy; ONTAP: Snapshot; Pure: Snapshot | Có: Create Snapshot, Delete Snapshot (bắt buộc) | Snapshot Cinder là snapshot trên tủ |
| B9 | Clone / LUN từ snapshot | Tạo LUN ghi được từ snapshot hoặc LUN khác | 10/10 dòng. Tên gọi: PowerMax: SnapVX linked target; PowerStore: Thin Clone; Unity: Thin Clone; 3PAR: Physical Copy; VSP: ShadowImage; Dorado: HyperClone; Spectrum Virtualize: FlashCopy clone; DS8000: FlashCopy; ONTAP: FlexClone; Pure: Volume copy | Có: Create Volume from Snapshot, Create Volume from Volume (bắt buộc) | Driver dùng cơ chế clone của tủ |
| B10 | Controller dự phòng, nâng cấp không gián đoạn | Failover controller tự động, nâng firmware không dừng dịch vụ | 10/10 dòng | Không có | Tủ tự xử lý, trong suốt với Cinder |
| B11 | Giám sát hiệu năng và cảnh báo | Theo dõi IOPS, latency, dung lượng; cảnh báo SNMP, email, call-home | 10/10 dòng. Công cụ: PowerMax: Unisphere; PowerStore: PowerStore Manager; Unity: Unisphere; 3PAR: SSMC; VSP: Ops Center; Dorado: DeviceManager; Spectrum Virtualize: GUI và Storage Insights; DS8000: DS GUI và Storage Insights; ONTAP: System Manager; Pure: Purity GUI và Pure1 | Một phần: chỉ báo cáo dung lượng và capability của backend | Hiệu năng và cảnh báo phần cứng vẫn xem trên tủ |
| B12 | GUI / CLI / REST API | Giao diện vận hành và API tự động hóa | 10/10 dòng; VSP cần cài thêm Configuration Manager REST API | Có: Cinder API, CLI, Horizon | Cinder là lớp API thống nhất cho nhiều tủ |
| B13 | Phân quyền người dùng | Tài khoản, vai trò, tích hợp LDAP/AD | 10/10 dòng | Có ở tầng khác: Keystone RBAC và policy của Cinder | Driver dùng một tài khoản dịch vụ trên tủ |
| B14 | SCSI UNMAP | Host báo vùng đã xóa để tủ giải phóng dung lượng | 10/10 dòng, áp dụng cho LUN thin | Có: discard (cấu hình) | Bật ở cả Cinder và Nova, VM dùng virtio-scsi |
| B15 | Hot spare và rebuild | Tự dựng lại dữ liệu khi hỏng ổ | 10/10 dòng | Không có | Tủ tự xử lý |
| B16 | Mở rộng dung lượng tủ online | Thêm ổ, khay đĩa, mở rộng pool khi đang chạy | 10/10 dòng | Một phần: tự cập nhật dung lượng pool | Pool tạo thêm phải khai báo vào cấu hình backend |
| B17 | Cấu hình cổng và xác thực CHAP | Đặt IP, VLAN cho cổng; xác thực initiator iSCSI | Cấu hình cổng: 10/10 dòng. CHAP: 9/10 dòng, không áp dụng cho DS8000 vì chỉ có FC | Có: tùy chọn CHAP và danh sách cổng target của driver (cấu hình) | IP, VLAN của cổng vẫn đặt trên tủ |
| B18 | Bảo vệ cache ghi khi mất điện | Pin hoặc tụ giữ điện để đẩy cache xuống ổ | 10/10 dòng | Không có | Thuộc phần cứng tủ |
| B19 | Nhật ký sự kiện và audit log | Ghi sự kiện và thao tác quản trị; syslog, NTP | 10/10 dòng | Có ở tầng khác: log Cinder, notification, audit middleware | Log tủ chỉ thấy tài khoản dịch vụ; cần đối chiếu hai nguồn khi truy vết |
| B20 | SCSI-3 Persistent Reservation | Giữ chỗ và fencing trên LUN dùng chung cho cluster host | 10/10 dòng | Một phần: qua Multi-Attach và đĩa kiểu LUN passthrough của Nova | Cinder không quản lý reservation |
| B21 | Zoning trên SAN switch / director | Giới hạn cổng host nào nói chuyện được với cổng tủ nào trên fabric FC | 0/10 dòng: không phải tính năng của tủ mà của SAN switch (Brocade, Cisco MDS); cần cho kết nối FC của cả 10 dòng | Có: Fibre Channel Zone Manager (cấu hình) | Cinder tự tạo, xóa zone khi attach, detach |
| O1 | Create Image from Volume | Đẩy nội dung volume lên Glance thành image | Không có (0/10 dòng) | Có (bắt buộc) | Tủ không có khái niệm image và kho image |
| O2 | Volume Migration qua host | Copy dữ liệu qua host Cinder, di chuyển được giữa hai backend khác hãng | Không có (0/10 dòng) | Có (bắt buộc) | Tủ chỉ di chuyển nội bộ hoặc giữa các tủ cùng dòng |
| O3 | Volume type và extra specs | Định nghĩa lớp dịch vụ lưu trữ để người dùng chọn | Không có (0/10 dòng) | Có | Mỗi tủ chỉ có chính sách riêng của nó |
| O4 | Multi-backend và scheduler | Một Cinder quản nhiều tủ khác hãng, tự chọn backend theo dung lượng và capability | Không có (0/10 dòng) | Có | Tủ không điều phối giữa các hãng |
| O5 | Quota theo project | Giới hạn số volume, snapshot, GB cho từng project | Không có (0/10 dòng) | Có | Tủ không biết project của OpenStack |
| O6 | Volume transfer | Chuyển quyền sở hữu volume giữa hai project | Không có (0/10 dòng) | Có | Tủ không có chủ sở hữu theo tenant |
| O7 | Availability zone | Gắn backend vào AZ để đặt volume cùng vùng với VM | Không có (0/10 dòng) | Có | Khái niệm của cloud |
| O8 | Boot from volume, tạo volume từ image | Tạo volume khởi động cho VM từ image Glance | Không có (0/10 dòng) | Có | Phụ thuộc Glance và Nova |

## 4. So sánh tính năng nâng cao

Bảng gồm 27 tính năng nâng cao của tủ SAN (A1–A27) và 5 tính năng nâng cao chỉ OpenStack có (O9–O13). Mười tính năng SAN không có tương ứng trên OpenStack, tập trung ở nhóm bảo vệ dữ liệu và vận hành thông minh. Cột "Tủ SAN" ghi theo cùng khuôn với mục 3: số dòng tủ có tính năng trên mười dòng khảo sát, dòng ngoại lệ, rồi tên thương mại của tính năng trên từng dòng tủ.

| # | Tính năng | Mô tả | Tủ SAN (trên 10 dòng khảo sát) | OpenStack Cinder | Ghi chú |
| --- | --- | --- | --- | --- | --- |
| A1 | Replication đồng bộ | Ghi đồng thời sang tủ thứ hai, RPO = 0 | 10/10 dòng; PowerStore tùy phiên bản. Tên gọi: PowerMax: SRDF/S; PowerStore: Synchronous replication; Unity: Synchronous replication; 3PAR: Remote Copy Sync; VSP: TrueCopy; Dorado: HyperReplication/S; Spectrum Virtualize: Metro Mirror; DS8000: Metro Mirror; ONTAP: SnapMirror Synchronous; Pure: ActiveCluster | Có: Volume Replication (tùy chọn) | Chế độ sync chọn qua extra spec riêng từng driver; driver Hitachi VSP bản Yoga chưa có |
| A2 | Replication bất đồng bộ | Sao chép theo chu kỳ hoặc journal sang site xa | 10/10 dòng. Tên gọi: PowerMax: SRDF/A; PowerStore: Asynchronous replication; Unity: Asynchronous replication; 3PAR: Remote Copy Periodic; VSP: Universal Replicator; Dorado: HyperReplication/A; Spectrum Virtualize: Global Mirror; DS8000: Global Mirror; ONTAP: SnapMirror; Pure: Async replication, ActiveDR | Có: Volume Replication, failover-host, failback (tùy chọn) | Failover là thao tác admin cho cả backend, không tự động |
| A3 | Active-active metro | Một LUN đọc ghi đồng thời ở hai site, failover tự động | 9/10 dòng; không có sẵn trên Unity. Tên gọi: PowerMax: SRDF/Metro; PowerStore: Metro Volume; 3PAR: Peer Persistence; VSP: Global-Active Device; Dorado: HyperMetro; Spectrum Virtualize: HyperSwap; DS8000: HyperSwap (phụ thuộc hệ điều hành host); ONTAP: SnapMirror active sync, MetroCluster; Pure: ActiveCluster | Một phần: extra spec riêng của driver | Có ở driver Huawei, PowerMax, 3PAR, Pure, IBM; không có tính năng chung |
| A4 | DR ba trung tâm | Kết hợp metro hoặc sync với async tới site thứ ba | 8/10 dòng; không có trên PowerStore, Unity. Tên gọi: PowerMax: SRDF/Star; 3PAR: Synchronous Long Distance; VSP: GAD + Universal Replicator; Dorado: HyperMetro + HyperReplication; Spectrum Virtualize: 3-site replication; DS8000: Metro/Global Mirror; ONTAP: MetroCluster + SnapMirror; Pure: ActiveCluster + async | Không có | Cinder Yoga chỉ mô hình hóa một cặp nguồn, đích |
| A5 | Consistency group | Snapshot hoặc replicate nhiều LUN nhất quán cùng thời điểm | 10/10 dòng. Tên gọi: PowerMax: Storage Group; PowerStore: Volume Group; Unity: Consistency Group; 3PAR: VV Set; VSP: Consistency Group (CTG); Dorado: Protection Group; Spectrum Virtualize: Consistency Group; DS8000: Consistency Group; ONTAP: Consistency Group; Pure: Protection Group | Có: Consistency Groups (tùy chọn) | Dùng generic volume group và group snapshot |
| A6 | Revert từ snapshot | Đưa LUN về trạng thái của một snapshot | 10/10 dòng, chọn được snapshot bất kỳ. Tên gọi: PowerMax: SnapVX restore; PowerStore: Restore; Unity: Restore; 3PAR: Promote; VSP: Thin Image restore; Dorado: HyperSnap rollback; Spectrum Virtualize: FlashCopy restore; DS8000: FlashCopy reverse restore; ONTAP: SnapRestore; Pure: Restore volume | Một phần: Revert to Snapshot (tùy chọn) | Cinder chỉ revert về snapshot mới nhất; driver Huawei chưa có |
| A7 | Lịch snapshot và retention | Tự tạo, xóa snapshot theo chính sách | 8/10 dòng có sẵn trên tủ; VSP và DS8000 cần phần mềm quản lý của hãng (Ops Center Protector, Copy Services Manager) | Không có | Phải dùng công cụ ngoài gọi API |
| A8 | Snapshot bất biến chống ransomware | Snapshot không thể xóa trước hạn, kể cả bởi admin | 9/10 dòng; không có cho block trên Unity. Tên gọi: PowerMax: Secure Snaps; PowerStore: Secure Snapshot; 3PAR: Virtual Lock; VSP: Thin Image với Data Retention Utility; Dorado: Secure Snapshot; Spectrum Virtualize: Safeguarded Copy; DS8000: Safeguarded Copy; ONTAP: Tamperproof Snapshot (SnapLock); Pure: SafeMode | Không có | Cấu hình trực tiếp trên tủ |
| A9 | QoS trên tủ | Giới hạn hoặc bảo đảm IOPS, băng thông theo LUN | 10/10 dòng. Tên gọi: PowerMax: Host I/O Limits; PowerStore: QoS policy; Unity: Host I/O Limits; 3PAR: Priority Optimization; VSP: Server Priority Manager; Dorado: SmartQoS; Spectrum Virtualize: I/O throttling; DS8000: I/O Priority Manager; ONTAP: QoS policy group, Adaptive QoS; Pure: QoS limit | Có: QoS (tùy chọn) | Driver PowerStore, Hitachi VSP, DS8000 bản Yoga chưa có |
| A10 | Auto-tiering | Tự chuyển block nóng, lạnh giữa các lớp ổ | 6/10 dòng; không áp dụng cho PowerMax, PowerStore, Dorado, Pure vì là tủ all-flash một lớp. Tên gọi: Unity: FAST VP; 3PAR: Adaptive Optimization; VSP: Dynamic Tiering; Spectrum Virtualize: Easy Tier; DS8000: Easy Tier; ONTAP: FabricPool | Một phần: extra spec riêng của driver | Chọn chính sách qua volume type nếu driver hỗ trợ |
| A11 | Dedup và compression | Giảm dung lượng inline hoặc hậu xử lý | 9/10 dòng có cả hai; DS8000 không có dedup. Luôn bật trên PowerStore, Dorado, Pure; bật theo LUN hoặc pool trên PowerMax, Unity, 3PAR, VSP, Spectrum Virtualize, ONTAP | Một phần: extra spec riêng của driver | Tỉ lệ giảm dung lượng không báo về Cinder |
| A12 | Mã hóa dữ liệu lưu trữ | Mã hóa toàn bộ dữ liệu trên ổ | 10/10 dòng. Tên gọi: PowerMax, PowerStore, Unity: D@RE; 3PAR, VSP, Dorado, Spectrum Virtualize, DS8000: ổ tự mã hóa hoặc mã hóa ở controller với key manager; ONTAP: NSE, NVE; Pure: luôn bật | Không có | Tủ tự xử lý; Cinder có cơ chế mã hóa riêng ở host, xem O10 |
| A13 | Di chuyển LUN trong tủ | Chuyển LUN online giữa pool hoặc mức dịch vụ | 9/10 dòng; không áp dụng cho Pure vì chỉ có một pool. Tên gọi: PowerMax: đổi Service Level hoặc SRP; PowerStore: di chuyển giữa appliance; Unity: LUN Move; 3PAR: Dynamic Optimization; VSP: Volume Migration; Dorado: SmartMigration; Spectrum Virtualize: migrate volume; DS8000: Dynamic Volume Relocation; ONTAP: vol move, LUN move | Có: Volume Migration (Storage Assisted), Retype (tùy chọn) | Năm trong mười driver khảo sát có |
| A14 | Ảo hóa tủ ngoài và import | Dùng LUN tủ khác làm backend hoặc nhập dữ liệu từ tủ cũ | Ảo hóa tủ ngoài: 3/10 dòng, gồm VSP: Universal Volume Manager; Dorado: SmartVirtualization; Spectrum Virtualize: External virtualization. Import dữ liệu: 8/10 dòng, không có sẵn trên DS8000, Pure; thêm PowerMax: Non-Disruptive Migration; PowerStore: Native import; Unity: import từ VNX; 3PAR: Online Import; ONTAP: Foreign LUN Import | Một phần: Manage / Unmanage | Cinder chỉ nhận quản lý LUN có sẵn |
| A15 | Multi-tenancy trên tủ | Chia tủ thành các miền quản trị tách biệt | 7/10 dòng; không có cho block trên PowerMax, PowerStore, Unity. Tên gọi: 3PAR: Virtual Domains; VSP: Resource Group, VSM; Dorado: vStore; Spectrum Virtualize: Ownership group; DS8000: Resource group; ONTAP: SVM; Pure: Realm (tùy phiên bản) | Có ở tầng khác: project và quota | Trên tủ mọi volume thuộc cùng một tài khoản dịch vụ |
| A16 | NVMe over Fabrics | Truy cập block bằng NVMe/FC, NVMe/TCP, NVMe/RoCE | 8/10 dòng, tùy model và firmware; không có trên Unity, DS8000; với 3PAR chỉ có từ thế hệ Primera | Một phần: NVMe-oF connector của os-brick | Driver mười dòng tủ trong matrix Yoga chỉ ghi FC và iSCSI |
| A17 | VMware VVol | Cấp đĩa theo từng VM qua chính sách vSphere | 9/10 dòng qua VASA provider; không có trên DS8000 | Không có | Tính năng riêng của vSphere |
| A18 | Continuous data protection | Snapshot mật độ rất cao, quay về gần như mọi thời điểm | 1/10 dòng có tính năng CDP riêng: Dorado: HyperCDP. Chín dòng còn lại chỉ đạt gần tương đương bằng lịch snapshot dày (A7) | Không có | Snapshot CDP trên tủ không hiện trong Cinder |
| A19 | Snapshot nhất quán ứng dụng | Đóng băng I/O của database trước khi snapshot | 10/10 dòng, qua phần mềm đi kèm chứ không nằm trong firmware tủ. Tên gọi: PowerMax, PowerStore, Unity: AppSync; 3PAR: RMC; VSP: Ops Center Protector; Dorado: BCManager; Spectrum Virtualize, DS8000: Copy Services Manager; ONTAP: SnapCenter; Pure: plugin cho từng ứng dụng | Một phần: quiesce của Nova qua qemu-guest-agent | Chỉ đóng băng filesystem, không phối hợp với database |
| A20 | SSD cache và phân vùng cache | SSD làm cache đọc cho pool ổ quay | 3/10 dòng, chỉ trên cấu hình hybrid. Tên gọi: Unity: FAST Cache; 3PAR: Adaptive Flash Cache; ONTAP: Flash Pool. Không áp dụng cho bảy dòng còn lại | Một phần: extra spec riêng của driver | Tùy driver; với Huawei chỉ áp dụng cho dòng OceanStor hybrid, không phải Dorado |
| A21 | Phát hiện ransomware trên tủ | Cảnh báo bất thường về entropy, tỉ lệ nén, mẫu I/O | 4/10 dòng phát hiện trên dữ liệu, tùy phiên bản: Dorado, Spectrum Virtualize, ONTAP (Autonomous Ransomware Protection), Pure (qua Pure1). PowerMax, PowerStore, Unity có cảnh báo qua CloudIQ. Không có sẵn trên 3PAR, VSP, DS8000 | Không có | Volume mã hóa LUKS ở host làm giảm khả năng phát hiện của tủ |
| A22 | AIOps và phân tích dự báo | Dự báo dung lượng, hiệu năng, lỗi ổ | 10/10 dòng. Tên gọi: PowerMax, PowerStore, Unity: CloudIQ; 3PAR: InfoSight; VSP: Ops Center Clear Sight; Dorado: DME IQ; Spectrum Virtualize, DS8000: Storage Insights; ONTAP: Active IQ; Pure: Pure1 | Không có | Cinder không thu thập telemetry của tủ |
| A23 | Sao lưu snapshot ra cloud | Đẩy snapshot sang S3 hoặc cloud để lưu dài hạn | 4/10 dòng có sẵn trên tủ. Tên gọi: PowerMax: Cloud Mobility; Spectrum Virtualize: Transparent Cloud Tiering; ONTAP: SnapMirror Cloud; Pure: CloudSnap. Sáu dòng còn lại cần phần mềm hoặc thiết bị sao lưu ngoài | Một phần: Volume Backup ra Swift, S3 | Cùng mục đích, khác cơ chế; xem O9 |
| A24 | Scale-out và nâng cấp controller tại chỗ | Thêm controller, engine, node vào cùng hệ thống | 7/10 dòng thêm được controller hoặc node: PowerMax: engine; PowerStore: appliance trong cluster; 3PAR: cặp node; VSP: node (dòng 5000); Dorado: controller enclosure; Spectrum Virtualize: I/O group; ONTAP: cluster node. Unity, DS8000, Pure chỉ mở rộng theo chiều dọc; Pure thay thế hệ controller tại chỗ (Evergreen) | Không có | Tủ tự xử lý; cổng target mới cần thêm vào cấu hình driver |
| A25 | Unified storage | Cung cấp thêm NFS, SMB bên cạnh block | 8/10 dòng; không có trên Spectrum Virtualize, DS8000. Sẵn trong tủ: PowerStore, Unity, Dorado, ONTAP, Pure (FlashArray File); qua module hoặc tùy chọn NAS: PowerMax, 3PAR (File Persona), VSP | Có ở dịch vụ khác: Manila | Không thuộc Cinder |
| A26 | Tích hợp container | Kubernetes cấp volume qua CSI | 10/10 dòng. Tên gọi: PowerMax, PowerStore, Unity: Dell CSI driver; 3PAR: HPE CSI Driver; VSP: Hitachi Storage Plug-in for Containers; Dorado: Huawei CSI; Spectrum Virtualize, DS8000: IBM Block CSI; ONTAP: Trident; Pure: Portworx | Có: Cinder CSI plugin | Tủ nào có driver Cinder cũng dùng được |
| A27 | Xóa dữ liệu an toàn | Ghi đè hoặc hủy khóa để dữ liệu không khôi phục được | 10/10 dòng ở mức hủy khóa mã hóa (crypto-erase) khi bật A12. Ghi đè theo LUN: 2/10 dòng, gồm VSP: Volume Shredder; Dorado: Data destruction | Không có | Tùy chọn ghi đè khi xóa của Cinder chỉ áp dụng cho LVM |
| O9 | Volume Backup | Sao lưu volume ra kho độc lập với tủ: Swift, S3, Ceph, NFS; có incremental | Không có (0/10 dòng) | Có: dịch vụ cinder-backup | Tủ chỉ có snapshot và replication sang tủ cùng dòng |
| O10 | Volume Encryption | Mã hóa LUKS ở compute host, khóa riêng từng volume lưu ở Barbican | Không có (0/10 dòng) | Có: encrypted volume type | Mã hóa của tủ dùng khóa chung và chỉ bảo vệ dữ liệu trên ổ |
| O11 | Front-end QoS | Giới hạn IOPS, băng thông tại hypervisor | Không có (0/10 dòng) | Có: QoS specs với consumer = front-end | Dùng được với driver chưa có QoS trên tủ |
| O12 | Retype giữa các backend | Đổi lớp dịch vụ, kèm chuyển dữ liệu sang tủ khác hãng | Không có (0/10 dòng) | Có | Tủ không chuyển dữ liệu sang tủ hãng khác |
| O13 | Active/Active HA của cinder-volume | Nhiều tiến trình cinder-volume cùng phục vụ một backend | Không có (0/10 dòng) | Có (tùy chọn) | Là dự phòng của lớp điều khiển Cinder; chỉ driver Pure có |

## 5. Mức hỗ trợ của driver Cinder Yoga theo dòng tủ

Dell PowerMax, Dell Unity, IBM Spectrum Virtualize, NetApp ONTAP và Pure Storage có 8 trên 9 tính năng tùy chọn; chỉ driver Pure Storage có Active/Active HA. Bảng lấy nguyên từ [support matrix Yoga](https://docs.openstack.org/cinder/yoga/reference/support-matrix.html); 11 hàm bắt buộc mặc định có ở mọi driver nên không liệt kê.

| Tính năng tùy chọn | PowerMax | PowerStore | Unity | HPE 3PAR | Hitachi VSP | Huawei Dorado V3/V6 | IBM Spectrum Virtualize | IBM DS8000 | NetApp ONTAP | Pure Storage |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Extend an Attached Volume | Có | Có | Có | Có | Có | Có | Có | Có | Có | Có |
| QoS | Có | Không | Có | Có | Không | Có | Có | Không | Có | Có |
| Volume Replication | Có | Có | Có | Có | Không | Có | Có | Có | Có | Có |
| Consistency Groups | Có | Có | Có | Có | Có | Có | Có | Có | Có | Có |
| Thin Provisioning | Có | Có | Có | Có | Có | Có | Có | Không | Có | Có |
| Volume Migration (Storage Assisted) | Có | Không | Có | Không | Không | Có | Có | Không | Có | Không |
| Multi-Attach | Có | Có | Có | Có | Có | Không | Có | Có | Có | Có |
| Revert to Snapshot | Có | Có | Có | Có | Có | Không | Có | Có | Có | Có |
| Active/Active HA | Không | Không | Không | Không | Không | Không | Không | Không | Không | Có |

"Không" nghĩa là driver trong bản Yoga chưa lộ tính năng đó ra Cinder, không có nghĩa tủ thiếu tính năng. Ví dụ Hitachi VSP có TrueCopy và Universal Replicator, Huawei Dorado cho gán một LUN vào nhiều host, nhưng driver Yoga chưa khai báo Volume Replication và Multi-Attach tương ứng.

## 6. Kết luận và đề xuất

Có thể giao toàn bộ thao tác cấp phát hằng ngày cho OpenStack, nhưng không thể bỏ công cụ quản trị của tủ.

- **Giao cho OpenStack:** tạo, xóa, mở rộng, attach, snapshot, clone, QoS, group snapshot và replication một cặp site, với điều kiện driver của dòng tủ có tính năng tương ứng ở mục 5.
- **Giữ trên tủ:** lịch snapshot, snapshot bất biến, CDP, DR ba trung tâm, phát hiện ransomware, giám sát hiệu năng và phần cứng.
- **Cân nhắc khi chọn tủ:** nếu cần QoS và replication điều khiển từ Cinder, driver Hitachi VSP và IBM DS8000 bản Yoga chưa đáp ứng; nếu cần Multi-Attach hoặc revert snapshot, driver Huawei Dorado bản Yoga chưa đáp ứng.
- **Bước tiếp theo:** chạy bộ kịch bản ở tab Ma trận và kịch bản test trên các dòng tủ có trong lab để xác nhận lại ma trận bằng kết quả thực tế.

