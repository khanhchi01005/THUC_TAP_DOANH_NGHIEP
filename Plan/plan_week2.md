# Week 2 – Ceph Fundamentals & Disaster Recovery

Sau khi tìm hiểu, nghiên cứu về các dịch vụ: Block, File, Object cần nắm được các kiến thức architecture trong Ceph cluster:

## 1. Ceph Fundamentals**
 - [ ] Ceph là gì? Tại sao dùng Distributed Storage?
 - [ ] Ceph MON
 - [ ] Ceph OSD
 - [ ] Ceph Manager
 - [ ] Pool
 - [ ] PG
 - [ ] CRUSH
 - [ ] Replication / Erasure Coding 
 - [ ] Data placement

## 2. Failure Model 

Requirement: Tìm đọc, nghiên cứu, lab thử nghiêm các model 

**Failure Model tổng quan**
 - [ ] Crash-stop
 - [ ] Crash-recovery
 - [ ] Network Partition
 - [ ] Split-brain
 - [ ] Silent Data Corruption / Bit Rot

**Failure Model trên Ceph**
 - [ ] OSD failure
 - [ ] Node failure
 - [ ] Disk failure
 - [ ] Network / ToR failure
 - [ ] MON quorum loss
 - [ ] PG degraded | unclean | inconsistent
 - [ ] Recovery / Backfill / Rebalance 
