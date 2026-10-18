# microservices-gitops

Repo GitOps cho lab K8s microservice (Moleculer: api-gateway, categories, news).
Code app nằm ở repo riêng `microservices-app`.

## Môi trường
- Local: OrbStack + kind 3 node (`microlab`), Kubernetes 1.36.x pin bằng digest
- Thật: dev/stg on-prem RKE2 + Rancher; uat/prod AKS
- Dựng/xoá local: `make -C local-env up | down | smoke`; kubeconfig `~/.kube/microlab.yaml`
- Network kind cố định 192.168.97.0/24, Docker chỉ cấp .0/25; MetalLB pool .200-.250

## Quy ước
- Pin mọi version; chỉ dùng bản patch đã phát hành ≥ 2 tuần; image pin digest khi có thể
- Phần mềm bên thứ ba: Helm chart chính thức, deploy qua Argo CD
- Manifest app: Kustomize (base / components / overlays)
- Từ Phần 4: không `kubectl apply` tay, mọi thay đổi đi qua Git
- Quyết định kiến trúc → ADR trong `docs/adr/`
- Lỗi đã gặp → `docs/troubleshooting.md` (triệu chứng / nguyên nhân / cách sửa / cách phát hiện sớm)

## Cấu trúc
- `local-env/` cluster local · `bootstrap/` Argo CD · `platform/` add-on · `apps/` manifest app · `docs/`