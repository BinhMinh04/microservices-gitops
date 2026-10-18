# ADR-001: Môi trường local — OrbStack + kind 3 node + MetalLB L2

- Trạng thái: Đã chấp nhận
- Ngày: 2026-10-18

## Bối cảnh
Cần một cluster local trên Mac M4 (16GB RAM) để tập triển khai microservice theo best practice,
kỹ năng phải chuyển được sang AKS (uat/prod) và RKE2 + Rancher on-prem (dev/stg).
Yêu cầu: Kubernetes upstream, nhiều node (để tập PDB, drain, topology spread),
Service type LoadBalancer hoạt động như on-prem, dựng lại được bằng một lệnh.

## Quyết định
- Container runtime trên Mac: OrbStack (Docker engine), giới hạn 10GB RAM, tắt Kubernetes tích hợp,
  tắt expose port ra LAN.
- Cluster: kind v0.33.0, 1 control-plane + 2 worker, node image kindest/node v1.36.4 pin bằng digest
  (kind v0.33 mặc định 1.37; dự án khớp AKS 1.36). Worker gắn label zone-a / zone-b.
- LoadBalancer: MetalLB chart 0.16.1, chế độ L2.
- Docker network "kind" tạo trước với subnet cố định 192.168.97.0/24 và --ip-range 192.168.97.0/25;
  dải 192.168.97.200-250 dành riêng cho MetalLB.
- Kubeconfig riêng ~/.kube/microlab.yaml, không ghi vào ~/.kube/config.
- Toàn bộ quy trình đóng gói trong local-env/Makefile.

## Phương án đã cân nhắc
- Docker Desktop: nặng hơn, không truy cập thẳng IP container từ Mac, license cho công ty lớn.
- Colima: miễn phí, nhẹ, nhưng cần cấu hình thêm để Mac tới được IP trong network của container.
- Kubernetes tích hợp của OrbStack / minikube: chỉ 1 node.
- k3d: nhiều node nhưng chạy k3s (bản rút gọn), khác Kubernetes upstream của AKS/RKE2.

## Hệ quả
- (+) Kubernetes upstream, giống CI và giống cách dựng cluster thật (kubeadm).
- (+) Mac truy cập thẳng IP do MetalLB cấp, mô phỏng on-prem.
- (+) `make up` dựng lại toàn bộ môi trường, kết quả lặp lại được.
- (−) OrbStack miễn phí cho cá nhân; dùng cho công việc thương mại cần license.
- (−) MetalLB L2: mỗi IP chỉ do 1 node nhận traffic, failover mất vài giây; không phải cân bằng tải thật.
- (−) Subnet 192.168.97.0/24 cố định; nếu trùng với mạng khác trên máy thì phải đổi đồng bộ
  ở Makefile và ipaddresspool.yaml.