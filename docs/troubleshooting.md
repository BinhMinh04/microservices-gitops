## MetalLB: Service có EXTERNAL-IP nhưng gọi vào bị timeout
- **Triệu chứng:** `kubectl get svc` có EXTERNAL-IP; curl từ ngoài báo `Connection timed out`;
  log và event của MetalLB bình thường (`IPAllocated`, `nodeAssigned`, `serviceAnnounced`).
- **Nguyên nhân:** IPAddressPool nằm ngoài subnet của node. L2 mode dùng ARP, ARP chỉ có tác dụng
  trong cùng subnet, nên client không có đường tới IP đó. MetalLB không kiểm tra điều này.
- **Cách sửa:** đặt pool trong subnet của node, ở dải mà DHCP/IPAM không cấp
  (lab: tạo network kind với `--ip-range`, pool ở phần còn lại của subnet).
- **Phát hiện sớm:** so `kubectl get nodes -o wide` (INTERNAL-IP) với pool trước khi apply;
  trên Mac: `route -n get <IP>` xem gói tin có đi đúng interface không.
- **Bài học chung:** timeout = tầng mạng, refused = tầng ứng dụng. Debug từng tầng
  pod → ClusterIP → NodePort → External IP.