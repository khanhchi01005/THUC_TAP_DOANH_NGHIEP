# Giải pháp DR cho dịch vụ Block storage: Ceph RBD Mirror

## 1. Tìm hiểu cơ chế Ceph RBD DR, kiến trúc Ceph RBD Mirror, Ceph RBD Mirror hoạt động như thế nào (giải thích bằng flow diagram cho từng dạng mirror: One-way vs Two-way)

## 2. Hands-on Lab

- Mục tiêu triển khai 2 ceph cluster và mirroring rbd mode image (thử lần lượt replication mode: journal và snapshot)
  + Lab 1 — One-way + Image Mode + Journal
  + Lab 2 — One-way + Image Mode + Snapshot
  + Lab 3 — Two-way + Image Mode + Journal
  + Lab 4 — Two-way + Image Mode + Snapshot

- Benchmark perf trước, trong và sau khi quá trình mirror hoàn tất (tổng hợp kết quả benchmark các lần vào 1 file để so sánh đối chiếu, và cho kết luận):
  + IOPS của rbd image 
  + Latency của rbd image 
  + Throughput của rbd image 
  + CPU của host vật lý trong ceph cluster
  + Replication lag
    

- Tài liệu tham khảo:
  + https://docs.ceph.com/en/reef/rbd/rbd-mirroring/
  + https://docs.redhat.com/en/documentation/red_hat_ceph_storage/3/html/block_device_guide/block_device_mirroring#configuring_one_way_mirroring
  + https://docs.redhat.com/en/documentation/red_hat_ceph_storage/3/html/block_device_guide/block_device_mirroring#configuring_two_way_mirroring
