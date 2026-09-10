# Week 3 – Disaster & DR

Sau khi nghiên cứu các loại dịch vụ lưu trữ, mô hình lưu trữ dữ liệu, sau khi hiểu failure mới học DR

## 1. Khái niệm
 - [x] Disaster definition
 - [x] Disaster classification
 - [x] Prevention
 - [x] Detection
 - [x] Response
 - [x] Recovery
 - [x] Testing/Auditing
 - [x] RPO | RTO

Bổ sung tìm hiểu thêm các khái niệm sau:

 - [ ] Backup vs DR
 - [ ] HA vs DR
 - [ ] Replication vs Backup
 - [ ] Local failure vs Site failure


## 2. DR Architecture

> **Mục tiêu:** RPO/RTO → replication → architecture -> Nghiên cứu nguyên lý hoạt động, tác động RPO/RTO, ưu/nhược điểm và kịch bản áp dụng cho từng mô hình.

### Nhóm trạng thái sẵn sàng (Standby Models)
- [ ] **Cold Standby** 
- [ ] **Warm Standby** 
- [ ] **Hot Standby** 


### Nhóm mô hình vận hành (Operating Models)
- [ ] **Active-Passive:** 
- [ ] **Active-Active:** 


Cần nghiên cứu thêm về **Replication model**:

- [ ] Synchronous replication
- [ ] Asynchronous replication
- [ ] Snapshot-based replication
- [ ] Log-based replication


Ví dụ:
```
RPO = 0
RTO = vài giây
        ↓
Active-Active / synchronous replication


RPO = vài phút
RTO = vài phút
        ↓
Warm Standby / asynchronous replication


RPO = vài giờ
RTO = vài giờ
        ↓
Cold Standby / Backup
```
