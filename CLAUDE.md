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

## App image (từ Phần 1)
- Image: ghcr.io/binhminh04/microservices-app (public), tag = git short SHA, multi-arch amd64+arm64. Manifest pin digest INDEX.
- Dockerfile ở gốc repo app, build context = gốc repo. Luôn chạy `docker build --check .` trước khi build.
- Runtime: tini PID 1, USER 1000:1000, port mặc định 3001 (env PORT), SERVICEDIR=dist/services đã có sẵn trong image.
- Env của app: SERVICES (api|categories|news), NODE_ENV (testing|production; dev sẽ tắt migration),
  TRANSPORTER, CACHER, DB_NAME/DB_HOST/DB_PORT/DB_USER/DB_PASSWORD (KHÔNG phải DB_DATABASE), RUN_MIGRATIONS=false.
- Lệnh migration: `sequelize-cli db:migrate` (đã có trong PATH, cwd /app vì .sequelizerc dùng đường dẫn tương đối).
- Docker context phải là `orbstack`. kubectl luôn dùng KUBECONFIG=~/.kube/microlab.yaml.
- Khi đưa lệnh shell cho user: không đặt comment `#` trong lệnh (zsh tương tác không hiểu); file dài thì đưa nguyên file bằng heredoc.