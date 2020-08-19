**Network**

Ceph cluster "phải" dùng một "public" (front side) network. Nếu chúng ta không chỉ định "cluster" (back side) network thì Ceph sẽ chỉ sử dụng "public" network.
Ceph hoàn toàn có thể vận hành chỉ với public network nhưng sẽ tối ưu hơn nếu chúng ta dùng cluster network ở hệ thống lớn.
Có thể dùng bond hoặc dùng nhiều NIC.

![](ceph-public-cluster-network.png)

Có nhiều lý do để tách public network và cluster network ra:
- Performance: Ceph OSD xử lý replicate data, và khi ceph xử lý replicate thì sẽ ngốn kha khá băng thông và điều này ảnh hưởng tới network load giữa Ceph client và Ceph cluster. Khiến latency giữa client và ceph storage cluster tăng.
- Security: Giả sử public network bị tấn công DoS, khi đó traffic giữa các OSDs bị gián đoạn, latency giữa các OSD tăng cao khiến các PG không còn ổn định. Và điều này làm giảm hiệu suất đọc/ghi data. Vì vậy để ngăn tình huống này thì nên tách biệt cluster network ra và cluster network không ra được internet.
