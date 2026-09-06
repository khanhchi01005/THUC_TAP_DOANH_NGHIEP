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

## 2. Các mô hình lỗi phổ biến trong Ceph 

- **Lỗi tiến trình OSD:** Tiến trình ceph-osd bị dừng, crash hoặc ngưng phản hồi do tràn bộ nhớ (OOM), lỗi phần mềm hoặc treo thread I/O. Dấu hiệu: ceph health báo 1 osd down, OSD chuyển sang trạng thái down/in hoặc down/ou.
    - down + in: Tiến trình OSD bị dừng/crash, nhưng Ceph chưa chuyển dữ liệu đi đâu cả. Ceph sẽ đợi một khoảng thời gian (mặc định 600 giây - mon_osd_down_out_interval) để xem OSD có tự khôi phục không (ví dụ trường hợp máy reboot).
    - down + out: ụm Ceph lập tức kích hoạt tiến trình Self-healing (Tự chữa lành): Lấy các bản sao dữ liệu của OSD hỏng từ các OSD còn sống để nhân bản sang vị trí mới, đưa cụm về lại trạng thái an toàn active+clean.

---

## 3. Các mô hình lỗi phổ biến trong hệ thống phân tán 
- **Crash-stop:** xảy ra khi một hoặc nhiều daemon OSD trong Ceph đột ngột bị chấm dứt tiến trình (terminate/kill) do gặp lỗi không thể phục hồi ở cấp độ phần cứng, phần mềm, bộ nhớ hoặc hệ điều hành.

- **Crash-recovery:** 

- **Network partion/ Split-brain:** Là tình trạng mạng truyền thông bị gián đoạn vật lý hoặc logic, làm cho một tập hợp các nút (nodes) trong cụm bị chia tách thành 2 hoặc nhiều phân vùng riêng biệt (sub-clusters). Điểm đặc biệt:
    - Các nút trong cùng một phân vùng vẫn có thể liên lạc với nhau bình thường.
    - Các nút thuộc hai phân vùng khác nhau hoàn toàn mất liên lạc.
    - Tất cả các nút đều đang sống và hoạt động (không bị crash), nhưng chúng không thể đàm thoại với nhau.
    - Nguyên nhân: Lỗi thiết bị phần cứng mạng, lỗi cấu hình phần mềm, mạng bị nghẽn nặng
    - Hệ quả & Đánh đổi định luật CAP: Theo Định lý CAP (Brewer's Theorem), mạng chập chờn hay Network Partition là điều không thể tránh khỏi trong thực tế. Khi xảy ra Network Partition, hệ thống phân tán bắt buộc phải đánh đổi giữa 2 lựa chọn:
        - Hệ thống CP (Consistent): Ưu tiên Tính nhất quán dữ liệu, Phân vùng nào không có đủ đa số nút (không đạt Quorum) sẽ tự đóng băng I/O hoặc từ chối yêu cầu từ client.
        - Hệ thống AP (Availability): Ưu tiên Tính sẵn sàng (Availability). cả 2 phân vùng đều tiếp tục ghi/đọc độc lập, hệ thống chấp nhận dữ liệu bị bất đồng bộ tạm thời và chấp nhận quy trình hợp nhất dữ liệu phức tạp về sau
    - **Split-brain:** 