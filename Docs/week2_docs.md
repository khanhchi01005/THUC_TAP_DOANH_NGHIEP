# Nghiên cứu: Ceph Fundamentals 

---

## 1. Khái niệm nền tảng về các thành phần trong Ceph 

- **Ceph MON:** Nhiệm vụ chính của Ceph MON là quản lý và duy trì trạng thái tổng thể của toàn bộ cluster, đảm bảo các thành phần khác biết được sơ đồ mạng và vị trí dữ liệu để hoạt động chính xác.Các chức năng chính của Ceph MON
    - **Duy trì Cluster Maps:** Ceph MON lưu trữ và cập nhật trạng thái của các bản đồ chính (Master Maps), bao gồm:
        - Mon Map: Danh sách và trạng thái của các MON.
        - OSD Map: Danh sách, trạng thái hoạt động (in/out, up/down) của tất cả OSD (đơn vị lưu trữ dữ liệu).
        - PG Map: Trạng thái của các Placement Group (nhóm lưu trữ).
        - MDS Map: Trạng thái của các Metadata Server (dành cho CephFS).
        - CRUSH Map: Qui tắc phân bổ và vị trí dữ liệu trên hạ tầng phần cứng.
    - **Đồng thuận và Nhất quán dữ liệu:** Ceph MON sử dụng thuật toán đồng thuận Paxos để đảm bảo tất cả các node Monitor trong cluster đều giữ chung một phiên bản thông tin bản đồ (Cluster Map) chính xác và nhất quán.
    - **Hướng triển khai Ceph MON:**  Monitor là lightweight daemon không yêu cầu nhiều tài nguyên tính toán. Một Server thuộc dòng entry-server, CPU, RAM vừa phải và card mạng 1 GbE là đủ. Monitor node cần có đủ không gian lưu trữ để chứa các cluster logs, gồm OSD, MDS và monitor log.
        - Nếu hạn chế về tài chính, monitor daemon có thể chạy cùng trên OSD node. Tuy nhiên, cần trang bị nhiều CPU, RAM và ổ cứng hơn để lưu monitor logs.
        - Đối với các hệ thống lớn, nên sử dụng node monitor chuyên dụng. Đặt các node montior trên các rack riêng biệt với switch và nguồn điện riêng biệt. Nếu có nhiều DC với đường mạng tốc độ cao, có thể đặt monitor node trên nhiều DC.
    - **Số lượng MON:** cần triển khai số lượng Ceph MON là số lẻ (thường là 3 hoặc 5 node). Để đảm bảo tính sẵn sàng cao (High Availability) và duy trì sự đồng thuận Paxos



- **Ceph OSD:** là thành phần chịu trách nhiệm lưu trữ dữ liệu thực tế, xử lý các thao tác đọc/ghi (I/O), nhân bản (replication) và tự khôi phục dữ liệu (self-healing) trong cụm Ceph. Quy tắc thiết kế: 1 OSD = 1 Ổ đĩa vật lý (HDD/SSD/NVMe). Đặc điểm: OSD làm việc trực tiếp với Client sau khi Client lấy bản đồ cụm (Cluster Map) từ Ceph MON, hoàn toàn không thông qua MON khi đọc/ghi data.
    - **Kiến Trúc Lưu Trữ Nội Tại: BlueStore Engine:** Từ phiên bản Ceph Luminous trở đi, tất cả các OSD mặc định sử dụng engine lưu trữ có tên là BlueStore. cho phép Ceph OSD ghi trực tiếp dữ liệu lên ổ đĩa cứng thô (Raw Block Device) mà không cần thông qua Hệ thống tập tin (File System) của Linux (như ext4 hay xfs). Điều này giúp tăng gấp đôi hiệu năng và loại bỏ chi phí trung gian. Cấu trúc lưu trữ trong đĩa của OSD gồm 3 thành phần:
        - Raw Data (Data Block): Chiếm ~95–98% dung lượng đĩa, dùng để lưu trữ dữ liệu thực tế của người dùng.
        - Block DB (RocksDB): Chiếm ~1–4% dung lượng đĩa, lưu trữ toàn bộ Metadata (tên object, vị trí khối đĩa, checksum, snapshot).
        - WAL (Write-Ahead Log): Lưu nhật ký thao tác Metadata của RocksDB. Đối với các dữ liệu nhỏ ($<$ min_alloc_size), WAL đóng vai trò làm vùng đệm ghi tạm để tối ưu tốc độ I/O.


- **Ceph MGR:** chạy song song cùng với hệ thống và làm nền giám sát chính,
nhằm cung cấp hệ thống giám sát và giao diện bổ sung cho các hệ thống quản lý và giám sát bên ngoài. Nhận toàn bộ trách nhiệm thu thập số liệu thống kê, báo cáo trạng thái chi tiết và cung cấp các giao diện quản trị (GUI/REST API).
    - **Nguyên lý hoạt động:** hông thường trong một cụm Ceph, người ta sẽ cài đặt tối thiểu 2 node MGR để đảm bảo dự phòng (HA). Các node MGR này thường được cài đặt chung trên cùng máy chủ vật lý với các node Ceph MON.
        - Tại một thời điểm, chỉ có 1 node MGR ở trạng thái Active chịu trách nhiệm thu thập dữ liệu và phục vụ Dashboard/API.
        - Các node MGR còn lại ở trạng thái Standby. Nếu node MGR Active bị sập hoặc mất kết nối, một node MGR Standby sẽ tự động nhảy lên làm Active ngay lập tức mà không gây gián đoạn dữ liệu I/O của Client.
    - **Cơ chế lưu trữ:** Khi các daemon (OSD, MON, MDS) gửi thông số hiệu năng (IOPS, độ trễ latency, dung lượng, băng thông) về cho MGR Active, MGR sẽ giữ toàn bộ các chỉ số này trên bộ nhớ RAM.
        - MGR chỉ duy trì một cửa sổ thời gian ngắn (mặc định khoảng vài phút đến vài giờ tùy cấu hình plugin) để hiển thị biểu đồ thời gian thực trên Ceph Dashboard. Các dữ liệu cũ hơn sẽ tự động bị ghi đè/xóa bỏ để giải phóng RAM.
        - Hậu quả nếu MGR bị sập/restart: Khi node MGR Active bị restart, toàn bộ lịch sử metric trong RAM sẽ mất. MGR mới (Standby nhảy lên làm Active) sẽ thu thập lại dữ liệu mới từ các OSD/MON ngay từ đầu mà không làm ảnh hưởng đến hoạt động của cụm.

- **Pool:** là một phân vùng lưu trữ logic (Logical Partition) được tạo ra bên trong cụm Ceph để tổ chức, quản lý và lưu trữ các object dữ liệu. Mỗi Pool sẽ có quy định riêng về cách bảo vệ dữ liệu chống mất mát khi hỏng ổ đĩa
    - Mỗi Pool sẽ có quy định riêng về cách bảo vệ dữ liệu chống mất mát khi hỏng ổ đĩa:
        - Replicated Pool (Mặc định): Lưu trữ bằng cách tạo nhiều bản sao (ví dụ: replica size = 3 sẽ lưu 3 bản sao giống hệt nhau trên 3 OSD/Server khác nhau).
        - Erasure Coded (EC) Pool: Lưu trữ bằng cơ chế phân chia dữ liệu và tính toán mã sửa lỗi (tương tự RAID-5/RAID-6). Cơ chế này giúp tiết kiệm 30% - 50% dung lượng so với Replication nhưng tốn tài nguyên CPU hơn.
    - Quy định nơi dữ liệu được lưu:
        - Pool có thể gắn với CRUSH Rule.
        - CRUSH Rule quyết định dữ liệu được phân bố trên OSD nào. Ví dụ: Pool SSD → chỉ lưu trên OSD dùng SSD. Pool HDD → chỉ lưu trên OSD dùng HDD.
    - Quản lý dung lượng và quyền truy cập:
        - Có thể đặt quota để giới hạn dung lượng hoặc số lượng object.
        - Có thể sử dụng CephX để giới hạn user nào được phép đọc/ghi vào Pool nào.
    - Chứa các Placement Group (PG):
        - Mỗi Pool được chia thành nhiều PG.
        - Object được đưa vào PG dựa trên cơ chế phân bố của Ceph.
        - Sau đó CRUSH quyết định PG sẽ nằm trên OSD nào.

- **PG:** là khái niệm trung gian cực kỳ quan trọng trong Ceph, đóng vai trò làm lớp kết nối/quy gom giữa Pool (Logic) và OSD (Vật lý).
    - **Tại sao Ceph cần PG:** Nếu không có PG: Hệ thống sẽ phải quản lý vị trí của từng Object một trên từng OSD. Việc này khiến bản đồ cụm (Cluster Map) phình to hàng trăm Gigabyte RAM, gây sập các node MON/MGR và không thể đồng bộ giữa các node. Với PG, Ceph không quản lý vị trí trực tiếp của từng Object, mà gom hàng ngàn Object vào chung một PG, sau đó chỉ quản lý vị trí của vài nghìn PG này trên các OSD.
    - Quản lý Trạng thái (PG States):PG là nơi Ceph báo cáo sức khỏe chi tiết của dữ liệu. Một số trạng thái phổ biến khi gõ ceph status:
        - active+clean: Trạng thái hoàn hảo nhất (Dữ liệu hoạt động tốt, đủ số bản sao).
        - degraded: Đang bị thiếu bản sao (do có 1 OSD bị sập).
        - peering / recovering / backfilling: Đang trong quá trình kiểm tra, đồng bộ hoặc chép bù dữ liệu sang OSD mới.


- **CRUSH:** là thuật toán tính toán vị trí dữ liệu của Ceph. Bản chất của CRUSH là thay thế việc Tra cứu (Lookup) bằng Tính toán (Calculation).Mọi thành phần (Client, MON, OSD) tự dùng CPU của mình chạy một hàm toán học để tự tính ra object nằm ở OSD nào.
    - Cách hoạt động của CRUSH: Lấy 3 đầu vào gồm CRUSH Map, PG ID, CRUSH Rule để tính toán xem object nằm ở OSD nào mà không cần hỏi Server trung tâm 
    - Weight: Ceph sử dụng 2 chỉ số Weight độc lập để điều phối lượng dữ liệu thực tế lưu trên từng OSD, giúp phân bổ dữ liệu theo kích thước đĩa và chủ động xử lý lệch tải. 
        - CRUSH Weight (Trọng số phần cứng): : Khai báo khả năng lưu trữ lý thuyết của OSD dựa theo dung lượng đĩa thô (1.0 ~ 1TB)
            - Cách hoạt động: Thuật toán CRUSH dựa vào tỷ lệ này để chia PG. Đĩa to nhận nhiều PG, đĩa nhỏ nhận ít PG (Ví dụ: Ổ 4TB weight = 4.0 sẽ chứa lượng dữ liệu gấp 4 lần ổ 1TB weight = 1.0).
            - Thời điểm thiết lập: Tự động gán khi cắm đĩa mới vào cụm dựa trên kích thước đĩa.
        - OSD Reweight / In-Out Weight (Trọng số Điều tiết Thực tế): Tỷ lệ phần trăm tham gia chứa dữ liệu thực tế của OSD (dải giá trị từ 0.0 đến 1.0).
            - 1.0 (In - Normal): Hoạt động bình thường, nhận 100% tải lý thuyết từ CRUSH.
            - 0.1 -> 0.9 (In - Throttled): Giảm tải chủ động. Giữ lại X% dữ liệu, đẩy 100 -X% dữ liệu dư thừa sang OSD khác gánh giúp (Dùng khi đĩa bị chạm ngưỡng kịch trần Near-Full 85%–90% hoặc cần ép cân bằng lại cụm).
            - 0.0 (Out): Tắt nhận dữ liệu. CRUSH rút sạch 100% dữ liệu ra khỏi OSD này mà không làm gián đoạn dịch vụ (Dùng khi rút đĩa ra để bảo trì/thay thế).
    
- **Data Placement:** 

![Data placement](image.png)
 
- **Bước 1: Striping — Xé nhỏ dữ liệu thành các Object** Khi Client gửi một tập tin lớn (hoặc một khối đĩa ảo RBD 100GB):
    - Ceph xé nhỏ dữ liệu này thành các Object có kích thước cố định (mặc định là 4MB).
    - Mỗi Object được gán một Object ID (OID)
- **Bước 2: ObjectObject -> PG (Hashing):**Để tránh việc phải lưu trữ vị trí của hàng tỷ Object, Ceph quy gom các Object vào một số lượng PG (Placement Group) cố định của Pool. Client thực hiện phép tính toán băm ngay trên CPU local:
    - Băm tên Object: Chạy OID qua hàm băm Jenkins Hash thu được một chuỗi số 32-bit: **Hash Value = Hash(ObjectID)**
    - Xác định PG ID: Chia lấy phần dư (Modulo) cho tổng số pg_num của Pool đó: **PG_index = Hash Value (mod pg_num)**
    - Trả về PG ID có dạng <Pool_ID>.<PG_Index> (Ví dụ: Pool ID = 1, PG Index = 5 -> PG 1.5).
    - Đặc tính: Dữ liệu cùng tên sẽ luôn rơi vào đúng 1 PG cố định. Phép tính này mất chưa đến 1 microsecond và không tốn băng thông mạng.
- **Bước 3:  PG -> OSDs (CRUSH Algorithm)**
    - Sau khi có PG ID, Client áp dụng thuật toán CRUSH để tìm ra tập hợp các OSD vật lý sẽ chứa PG này. Input của hàm toán học CRUSH bao gồm:
        - PG ID (ví dụ: 1.5)
        - CRUSH Map (Sơ đồ topology hạ tầng: Phòng máy -> Rack -> Host ->
         OSD)
        - CRUSH Rule của Pool (Tập quy tắc: chọn 3 bản sao, nằm ở 3 Rack khác nhau, dùng đĩa NVMe)
        - **CRUSH(PG_ID, CRUSH Map, CRUSH Rule) -> [OSD Primary, OSD Secondary 1, OSD Secondary 2]**

---
## 2. Các mô hình lỗi phổ biến trong hệ thống phân tán 
- **Crash-stop:** xảy ra khi một hoặc nhiều daemon OSD trong Ceph đột ngột bị chấm dứt tiến trình (terminate/kill) do gặp lỗi không thể phục hồi ở cấp độ phần cứng, phần mềm, bộ nhớ hoặc hệ điều hành.
    - Nguyên nhân:
        - Lỗi phần cứng: Mất điện đột ngột, lỗi RAM (Kernel Panic), hỏng CPU hoặc chập cháy bo mạch
        - Tác động từ Hệ điều hành (OS): Trình quản lý bộ nhớ của Linux kích hoạt OOM Killer (Out Of Memory) tự động SIGKILL (kill -9) ngay lập tức tiến trình chiếm quá nhiều RAM.
        - Lỗi ứng dụng nghiêm trọng (Fatal Exception): Tiến trình gặp các lỗi bộ nhớ nghiêm trọng như Segmentation Fault, Stack Overflow, hoặc unhandled exception ở tầng nhân khiến process bị ngắt ngang tức thì.
    - Giải pháp:
        - Phát hiện qua Heartbeat & Timeout:Các nút còn lại liên tục gửi tín hiệu nhịp tim (Heartbeat). Nếu quá thời gian phản hồi (Election Timeout), hệ thống xác định nút đó đã Crash Stop.
        - Kích hoạt Bầu chọn & Failover: Nếu nút bị crash là Leader/Master, các nút còn lại sẽ tổ chức bầu chọn Leader mới dựa trên cơ chế Quorum (Số đông).
        - Chuyển giao tải: Các công việc, partition hoặc replica do nút cũ nắm giữ sẽ được phân phối lại cho các nút lành lặn trong cụm.

- **Crash-recovery:**  là quá trình một nút (node) hoặc toàn bộ hệ thống khôi phục lại trạng thái nhất quán và tiếp tục hoạt động sau khi bị sập (crash) hoàn toàn và mất đi toàn bộ dữ liệu trên bộ nhớ tạm (RAM)
    - Giải pháp: 
        - Ghi nhật ký bền vững (Durable Logging / Write-Ahead Log - WAL): Trước khi thực hiện thay đổi dữ liệu trên RAM, hệ thống ghi lại lịch sử thao tác vào nhật ký lưu trên ổ đĩa. Khi phục hồi, hệ thống đọc lại file log này để tái hiện (redo) hoặc hoàn tác (undo) giao dịch nhằm đảm bảo không mất dữ liệu đã xác nhận
        - Bầu lại nhóm và Đồng thuận : Sử dụng các thuật toán như Raft hoặc Paxos để xác định xem nút vừa hồi phục có còn đủ điều kiện làm lãnh đạo hay phải đồng bộ lại dữ liệu từ các nút khác trước khi phục vụ người dùng.

- **Network partion** Là tình trạng mạng truyền thông bị gián đoạn vật lý hoặc logic, làm cho một tập hợp các nút (nodes) trong cụm bị chia tách thành 2 hoặc nhiều phân vùng riêng biệt (sub-clusters). Điểm đặc biệt:
    - Các nút trong cùng một phân vùng vẫn có thể liên lạc với nhau bình thường.
    - Các nút thuộc hai phân vùng khác nhau hoàn toàn mất liên lạc.
    - Tất cả các nút đều đang sống và hoạt động (không bị crash), nhưng chúng không thể đàm thoại với nhau.
    - Nguyên nhân: Lỗi thiết bị phần cứng mạng, lỗi cấu hình phần mềm, mạng bị nghẽn nặng
    - Hệ quả & Đánh đổi định luật CAP: Theo Định lý CAP (Brewer's Theorem), mạng chập chờn hay Network Partition là điều không thể tránh khỏi trong thực tế. Khi xảy ra Network Partition, hệ thống phân tán bắt buộc phải đánh đổi giữa 2 lựa chọn:
        - Hệ thống CP (Consistent): Ưu tiên Tính nhất quán dữ liệu, Phân vùng nào không có đủ đa số nút (không đạt Quorum) sẽ tự đóng băng I/O hoặc từ chối yêu cầu từ client.
        - Hệ thống AP (Availability): Ưu tiên Tính sẵn sàng (Availability). cả 2 phân vùng đều tiếp tục ghi/đọc độc lập, hệ thống chấp nhận dữ liệu bị bất đồng bộ tạm thời và chấp nhận quy trình hợp nhất dữ liệu phức tạp về sau
- **Split-brain:** Do mất kênh liên lạc chung, mỗi phân đoạn không thể nhận biết trạng thái của phân đoạn còn lại. Kết quả là cả hai bên đều tự coi mình là phân đoạn sống duy nhất và đồng thời tự bầu chọn lên làm Leader (Master) để tiếp tục phục vụ ứng dụng.
    - Nguyên nhân: Do network partition, tín hiệu kiểm tra trạng thái hoạt động giữa các máy chủ bị gián đoạn
    - Hệ quả: 
        - Xung đột và hỏng dữ liệu: Cả 2 Master đều nhận và ghi các dữ liệu khác nhau từ khách hàng. Khi mạng khôi phục, hai tập dữ liệu này bị lệch chuẩn nghiêm trọng và không thể tự động gộp lại.
        - Mất tính nhất quán: Hệ thống mất đi nguồn sự thật duy nhất (single source of truth), dẫn đến lỗi ứng dụng hoặc mất dữ liệu nghiêm trọng
    - Giải pháp:
        - Quorum: Quy định một phân đoạn chỉ được phép hoạt động hoặc bầu Leader nếu nó nắm giữ quá bán tổng số nút: Quorum >= [N/2] + 1. Phân đoạn nằm ở phe thiểu số (Minority) sẽ tự động chuyển sang chế độ Read-Only hoặc tự ngắt dịch vụ để bảo vệ dữ liệu. 
        - Cơ chế FENCING/ STONITH: Khi một nút phát hiện nguy cơ Split-Brain, nó sẽ chủ động gửi tín hiệu tắt nguồn phần cứng hoặc cô lập cổng mạng của nút nghi vấn thông qua giao diện quản trị từ xa (IPMI/iLO) trước khi nhận quyền Master.

- **Silent Data Coruption/ Bit Rot** là hiện tượng dữ liệu bị thay đổi, biến dạng hoặc hỏng hóc trên thiết bị lưu trữ mà hệ điều hành, hệ thống tệp (file system) hoặc phần cứng không phát hiện hay báo lỗi.
    - Nguyên nhân: 
        - Lỗi môi trường vạt lý lưu trữ: HDD: Sự suy giảm từ tính (magnetic decay) trên các đĩa từ theo thời gian, hoặc hiện tượng ghi đè chéo (cross-talk) giữa các rãnh ghi mật độ cao (SMR/PMR). SSD / Flash Memory: Rò rỉ điện tích (charge leakage) trong các ô nhớ NAND Flash khi không được cấp điện trong thời gian dài, hoặc hiện tượng mòn ô nhớ (wear-out) do chu kỳ Ghi/Xóa (P/E cycles).
        - Lỗi Firmware / Driver: Bug trong Controller của ổ cứng, RAID controller, hay SAS/SATA bus dẫn đến việc ghi sai dữ liệu hoặc ghi sai vị trí block (phantom writes, misdirected writes) mà không báo lỗi
    - Hệ quả: 
        - Hỏng tệp tin vĩnh viễn: Các định dạng media (ảnh JPEG, video MP4) bị nhiễu sọc hoặc không thể mở; các tệp nén (ZIP, TAR) bị lỗi checksum và không thể giải nén.
        - Sai lệch dữ liệu doanh nghiệp: Dữ liệu trong cơ sở dữ liệu (Database) bị thay đổi giá trị số học âm thầm, làm sai lệch báo cáo tài chính hoặc thông tin giao dịch mà hệ thống không hay biết.
        - Sao lưu dữ liệu hỏng (Corrupted Backups): Do hệ thống không nhận biết dữ liệu nguồn đã hỏng, tiến trình backup tiếp tục sao lưu phiên bản hỏng này và ghi đè lên các bản backup sạch trước đó
    - Giải pháp: 
        - Hệ thống tệp tự sửa lỗi: Các File System tiên tiến như ZFS hoặc Btrfs sử dụng cơ chế End-to-End Checksumming.Khi đọc dữ liệu, hệ thống tự tính lại checksum. Nếu phát hiện sai lệch (SDC), nó sẽ tự động lấy bản sao lành lặn từ cơ chế Mirroring/RAID-Z để phục hồi (Self-healing) và ghi đè lại block bị hỏng.
        - Tiến trình Scrubbing: Quá trình chạy ngầm định kỳ quét toàn bộ các block dữ liệu trên ổ đĩa, kiểm tra tính vẹn toàn checksum nhằm phát hiện và chủ động sửa lỗi Bit Rot trước khi ứng dụng đọc tới.

---
## 3. Các mô hình lỗi phổ biến trong Ceph 

- **Lỗi tiến trình OSD:** Tiến trình ceph-osd bị dừng, crash hoặc ngưng phản hồi do tràn bộ nhớ (OOM), lỗi phần mềm hoặc treo thread I/O. Dấu hiệu: ceph health báo 1 osd down, OSD chuyển sang trạng thái down/in hoặc down/out.
    - down + in: Tiến trình OSD bị dừng/crash, nhưng Ceph chưa chuyển dữ liệu đi đâu cả. Ceph sẽ đợi một khoảng thời gian (mặc định 600 giây - mon_osd_down_out_interval) để xem OSD có tự khôi phục không (ví dụ trường hợp máy reboot).
    - down + out: Cụm Ceph lập tức kích hoạt tiến trình Self-healing (Tự chữa lành): Lấy các bản sao dữ liệu của OSD hỏng từ các OSD còn sống để nhân bản sang vị trí mới, đưa cụm về lại trạng thái an toàn active+clean.
    - Nguyên nhân phổ biến:
        - Lỗi phần cứng: Ổ cứng chứa dữ liệu OSD bị hỏng, bad sector, hoặc quá nhiệt khiến kernel ngắt kết nối I/O
        - Tiến trình Crash/OOM: Tiến trình ceph-osd bị crash do lỗi phần mềm (bug) hoặc bị hệ thống Linux kill do thiếu bộ nhớ RAM
        - Sự cố mạng: Mất kết nối mạng công cộng hoặc riêng tư (cluster network) giữa các node, làm gián đoạn gói tin heartbeat.
        - Hiện tượng Flapping OSD: OSD liên tục ngắt kết nối rồi kết nối lại trong thời gian ngắn, gây biến động nặng cho cụm do Ceph liên tục phải tính toán lại CRUSH Map.
    - Cách khắc phục cơ bản:
        - Kiểm tra trạng thái cụm: Chạy lệnh ceph health detail hoặc ceph osd tree để định danh chính xác OSD nào đang gặp sự cố.
        - Kiểm tra log hệ thống và OSD: Xem tệp log tại /var/log/ceph/ hoặc kiểm tra thông điệp kernel qua dmesg -T để phát hiện lỗi ổ cứng hay phân vùng
            - OSD bị ngắt tạm thời (thử khởi động lại)
            - Tránh hiện tượng Flapping OSD khi đang chẩn đoán: Tạm thời ngắt cơ chế tự động đánh dấu Down/Out để kiểm tra mạng/ổ đĩa
            - Nếu ổ đĩa OSD hỏng hoàn toàn Nếu đĩa hỏng vật lý, cần khai tử OSD cũ để Ceph hoàn tất xả dữ liệu (backfill), sau đó thay đĩa mới

- **Lỗi Node:** xảy ra khi một máy chủ (node) vật lý hoặc ảo trong cụm lưu trữ ngừng hoạt động, mất kết nối mạng hoặc sập nguồn, khiến toàn bộ các tiến trình OSD (Object Storage Device), MON (Monitor) hay MGR (Manager) trên node đó không thể truy cập được.
    - Hàng loạt OSDs bị đánh dấu Down: Tất cả OSD thuộc Node đó ngay lập tức ngừng gửi Heartbeat. Cụm Monitor (MON) chuyển trạng thái toàn bộ OSD trên Node đó thành Down/In.
    - Các Placement Groups (PG) bị ảnh hưởng: Các PG có chứa bản sao (replica) nằm trên Node hỏng sẽ chuyển sang trạng thái degraded (giảm số lượng bản sao) hoặc undersized. Tuy nhiên, nhờ quy tắc bầu vị trí dữ liệu CRUSH Rule (mặc định phân bổ bản sao nằm trên các Node khác nhau - failure-domain = host), dữ liệu vẫn đọc/ghi bình thường từ các bản sao còn sống trên các Node khác.
    - Mất daemon quản lý (MON / MGR / MDS): Nếu Node bị sập chứa MON hoặc MGR, cụm sẽ kích hoạt cơ chế Bầu chọn (Election) để chuyển vai trò sang các Node lành lặn còn lại (miễn là hệ thống còn đủ Quorum số đông MON: >= [N/2] + 1).
    - Quy tắc quan trọng:
        - Quy tắc failure-domain: Luôn đảm bảo CRUSH Map được cấu hình failure-domain = host (mặc định) hoặc rack. Tránh cấu hình failure-domain = osd vì nếu rớt 1 Node chứa nhiều OSD, dữ liệu có nguy cơ bị mất vĩnh viễn (Data Loss).
        - Quy tắc số lượng Node tối thiểu: Với cơ chế Replication (RF=3): Cần tối thiểu 3 Node.
        - Phân bổ MON hợp lý: Luôn triển khai số lượng MON là số lẻ (3, 5) trên các Node vật lý độc lập nhau.

- **Lỗi Disk:** xảy ra khi ổ cứng chứa Ceph OSD (Object Storage Daemon) gặp sự cố phần cứng hoặc phần mềm khiến tiến trình OSD không thể đọc/ghi dữ liệu
    - Cơ chế xử lý tự động của Ceph:
        - Đánh dấu Down: Sau khoảng 20 giây không có tín hiệu phản hồi, Ceph đánh dấu OSD đó ở trạng thái down. Các Placement Group (PG) trên đĩa đó chuyển sang 'degraded'. Client vẫn đọc/ghi bình thường nhờ các bản sao trên đĩa khác.
        - Hết thời gian chờ mặc định: Ceph đánh dấu OSD là out khỏi bản đồ CRUSH Map. Cụm bắt đầu tái định tuyến và khôi phục (recovering) bản sao dữ liệu (PG - Placement Group) sang các OSD lành lặn khác để đảm bảo an toàn dữ liệu
        - Kiểm tra trạng thái cụm: Chạy lệnh ceph -s hoặc ceph health detail để xác định OSD nào đang gặp sự cố hoặc chậm.
        - Thay thế phần cứng: Tiến hành gỡ bỏ OSD cũ (ceph osd out, dừng dịch vụ, xóa khỏi Crush map), thay thế ổ cứng vật lý mới và khởi tạo lại OSD (ceph osd crush remove, tạo OSD mới)


- **Network / ToR failure:** Sự cố này khiến toàn bộ các Node trong một Rack bị mất kết nối đồng thời với phần còn lại của cụm, cắt đứt hoàn bộ giao tiếp giữa các OSD, Monitor và Client.
    - Tác động: 
        - Hàng loạt OSD rơi vào trạng thái Down:  Tất cả OSD thuộc các Node nằm dưới switch ToR đó sẽ rớt kết nối Heartbeat.Cụm Monitor lập tức đánh dấu danh sách OSD này ở trạng thái Down/In
        - Nguy cơ mất Quorum Monitor: Nếu các nút MON (Monitor) được xếp nằm chung trong Rack bị hỏng ToR, cụm Ceph có thể bị mất số đông (Quorum) nếu số MON còn sống không đạt quá bán. Khi mất Quorum MON, toàn bộ cụm Ceph sẽ lập tức đóng băng (Block I/O) để bảo vệ tính vẹn toàn dữ liệu
        - Hiện tượng nghẽn mạng do Backfill: Nếu ToR chết lâu hơn thời gian mon_osd_down_out_interval (mặc định 10 phút), Ceph sẽ chuyển tất cả OSD trong Rack bị hỏng thành Out và kích hoạt Backfill / Self-healing. Toàn bộ dữ liệu của Rack hỏng sẽ được nhân bản lại trên các Rack khác, làm nghẽn băng thông của các switch ToR còn lại.
    - Giải pháp: 
        - Để sống sót qua thảm họa ToR Failure, kiến trúc Ceph bắt buộc phải cấu hình CRUSH Map với tham số quy định vùng chịu lỗi ở cấp Rack: failure-domain = rack
        - Kiến trúc LACP / Bonding: Mỗi Server trong Rack cắm 2 dây mạng về 2 Switch ToR độc lập (Stacking/MLAG) chạy song song theo chế độ LACP (802.3ad). Nếu 1 switch ToR chết, switch còn lại lập tức gánh toàn bộ traffic mà không rơi Node nào.
        - Phân bổ MON lẻ trên các Rack



- **MON quorum loss:**  xảy ra khi số lượng thành phần Monitor (MON) hoạt động trong cụm rơi xuống dưới mức tối thiểu cần thiết để duy trì hoạt động. Khi mất quorum, toàn bộ cụm Ceph sẽ bị đóng băng (freeze). 
    - Tác động: Chặn toàn bộ I/O của Client: Các dịch vụ như RBD (Block Device), CephFS, hay RGW (Object Storage) không thể xác thực trạng thái OSD. Toàn bộ ứng dụng (Virtual Machines, Kubernetes Pods...) bị treo I/O (I/O Pause/Hang).
    - Nguyên nhân: Mất kết nối mạng hàng loạt, Sập nhiều Node vật lý cùng lúc: Mất nguồn điện datacenter, cháy mainboard làm dừng đồng thời 2/3 hoặc 3/5 nút MON, Bất đồng bộ thời gian: Lệnh NTP/Chrony bị dừng, thời gian giữa các nút MON lệch nhau quá ngưỡng cho phép (mặc định 0.05s - mon_clock_drift_allowed), khiến Paxos từ chối đồng bộ.
    - Hướng xử lý:
        - Đồng bộ lại thời gian giữa các node bằng Chrony hoặc NTP.
        - Dọn dẹp ổ đĩa nếu phân vùng dữ liệu của MON bị đầy.
        - Cứu dữ liệu bằng cách ép buộc Quorum: Nếu các node MON khác đã chết hoàn toàn và không thể khôi phục ngay, bạn phải ép buộc MON duy nhất còn sống tạo thành một quorum mới bằng cách loại bỏ các MON đã chết khỏi bản đồ. Sau khi cụm hoạt động trở lại với 1 MON, cần tiến hành bổ sung thêm các node MON mới để đảm bảo tính sẵn sàng cao (High Availability)

- **PG degraded | unclean | inconsistent:**
    - **PG degraded:** Một PG ở trạng thái degraded khi số lượng bản sao (replicas) thực tế đang sống ít hơn số lượng bản sao được cấu hình cho Pool đó (ví dụ cấu hình RF = 3 nhưng hiện chỉ có 2 bản sao khả dụng).
        - Nguyên nhân: Một hoặc nhiều OSD chứa bản sao của PG đó bị hỏng (Down), ngắt mạng hoặc đang reboot.Cụm Ceph đang trong quá trình Backfill/Recovery sau khi thay ổ đĩa hoặc thêm/bớt Node.
        - Client VẪN ĐỌC/GHI ĐƯỢC bình thường thông qua các bản sao còn sống (như OSD 1, OSD 2). Dịch vụ không bị gián đoạn.
    - **PG unclean:** không phải là một trạng thái độc lập, mà là thuật ngữ tổng quát đại diện cho bất kỳ PG nào chưa đạt trạng thái hoàn hảo active+clean. Thực tế, degraded là một tập con của trạng thái unclean
        - Tùy thuộc vào trạng thái con đi kèm. Nếu chỉ là active+degraded hay active+recovering, I/O vẫn chạy bình thường. Nếu ở trạng thái peering hoặc stale, I/O đối với các object thuộc PG đó có thể bị block tạm thời
    - **PG inconsistent:** khi tiến trình kiểm tra vẹn toàn dữ liệu (Scrubbing / Deep Scrubbing) phát hiện sự sai lệch về nội dung hoặc Metadata giữa các bản sao của cùng một Object nằm trên các OSD khác nhau.
        - Nguyên nhân: 
            - Bit Rot / Silent Data Corruption: Ổ đĩa bị lỗi lật bit âm thầm mà phần cứng không báo error.
            - Lỗi ngắt điện đột ngột hoặc crash đĩa đúng thời điểm OSD đang ghi dữ liệu, khiến một OSD ghi dở dang.
        - Ceph sẽ chủ động chặn thao tác Đọc/Ghi đối với Object bị hỏng đó để ngăn ngừa dữ liệu sai lệch lan rộng sang ứng dụng.
        - Xác định chính xác Object và OSD bị lỗi, Yêu cầu Ceph so sánh các bản sao, lấy bản sao chứa số đông Checksum đúng để ghi đè lên bản sao bị hỏng


 
- **Recovery / Backfill / Rebalance:** là ba cơ chế di chuyển dữ liệu tự động, giúp cụm tự khôi phục tính an toàn của dữ liệu sau sự cố phần cứng, hoặc tự động phân bổ lại tài nguyên lưu trữ khi quy mô cụm thay đổi. 
    - **Recovery:** xảy ra khi một hoặc nhiều OSD (Object Storage Daemon) gặp sự cố tạm thời hoặc bị mất dữ liệu một phần, và Ceph cần đưa các PG (Placement Group) về trạng thái khỏe mạnh (active+clean) bằng cách sử dụng các bản ghi lịch sử (PG logs)
        - Cơ chế: Khi một OSD bị sập và hoạt động trở lại, Ceph sẽ đối chiếu nhật ký (logs) giữa OSD đó với các OSD bản sao khác để xác định chính xác những Object nào bị thiếu hoặc sai lệch phiên bản. Nó chỉ sao chép đúng những Object bị thay đổi trong thời gian OSD đó ngoại tuyến
        - Đặc điểm: Tốc độ xử lý rất nhanh vì chỉ truyền tải phần dữ liệu chênh lệch, không cần quét toàn bộ ổ đĩa.
    - **Backfill:** xảy ra khi Ceph cần sao chép toàn bộ dữ liệu của một PG sang một OSD mới. Hiện tượng này xuất hiện khi một OSD bị hỏng hoàn toàn (bị xóa khỏi cụm) hoặc một OSD mới trống hoàn toàn được thêm vào
        - Cơ chế: Khác với Recovery sử dụng PG logs, Backfill được kích hoạt khi lượng dữ liệu sai lệch quá lớn vượt quá khả năng lưu trữ của PG log, hoặc khi OSD đích hoàn toàn chưa có dữ liệu. Ceph sẽ quét toàn bộ không gian tên của PG và sao chép tuyến tính từng Object từ OSD nguồn sang OSD đích
        - Đặc điểm: Tốn rất nhiều băng thông mạng và tài nguyên I/O của ổ đĩa. Trong quá trình Backfill, trạng thái PG thường hiển thị là backfilling
    - **Rebalancing:** là quá trình di chuyển các PG giữa các OSD lành lặn để đảm bảo dung lượng lưu trữ được phân bổ đều trên toàn bộ các đĩa trong cụm.
        - Cơ chế: Hiện tượng này xảy ra khi thêm OSD mới vào cụm. Thuật toán CRUSH sẽ tính toán lại vị trí tối ưu mới cho các PG. Kết quả là một số PG sẽ được di chuyển từ các OSD cũ sang OSD mới.
        - Cơ chế: Hiện tượng này xảy ra khi bạn thêm OSD mới vào cụm hoặc thay đổi trọng số (CRUSH weight) của các OSD hiện tại. Thuật toán CRUSH sẽ tính toán lại vị trí tối ưu mới cho các PG. Kết quả là một số PG sẽ được di chuyển từ các OSD cũ sang OSD mới.
    - Khi cụm Ceph rơi vào các trạng thái này (đặc biệt là Backfill và Rebalance), hiệu năng I/O của khách hàng (Client) có thể bị giảm do nghẽn mạng nội bộ (Cluster Network) và nghẽn ổ đĩa.Có thể kiểm soát tốc độ của các tiến trình này thông qua các tham số cấu hình như:
        - osd_max_backfills: Số lượng tiến trình backfill tối đa được phép chạy đồng thời trên một OSD.
        - osd_recovery_max_active: giới hạn số lượng thao tác recovery tối đa đồng thời trên mỗi OSD
---

