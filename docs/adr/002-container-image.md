# ADR-002: Container image cho app Moleculer

- Trạng thái: Accepted
- Ngày: 2026-10-19 (Phần 1)
- Repo liên quan: BinhMinh04/microservices-app (Dockerfile, .dockerignore)

## Bối cảnh
Image gốc (2021): Node 12 (EOL), 1 stage, chạy root, `yarn start` làm PID 1 (nuốt SIGTERM),
devDeps và compiler TypeScript nằm trong image chạy, migration chạy bên trong pod api.
Image mới phải chạy trên Mac M4 / kind (arm64), AKS (amd64) và RKE2 on-prem.

## Quyết định
1. Base `node:24.21.0-trixie-slim`, pin tag + digest INDEX, khai báo một chỗ qua `ARG NODE_IMAGE`.
   Chọn patch đã phát hành ≥ 2 tuần (24.22.0 ra 14/10 nên chưa dùng).
2. 4 stage: `base` → `deps` (production deps, TARGETPLATFORM) → `build` (tsc, BUILDPLATFORM) → `runtime`.
3. Runtime: `tini` làm PID 1, CMD dạng exec gọi `moleculer-runner`, `PATH` gồm `node_modules/.bin`,
   `SERVICEDIR=dist/services`, `USER 1000:1000` (UID số), file thuộc root (app chỉ đọc).
4. Multi-arch `linux/amd64` + `linux/arm64` trong một index; tag = git short SHA; registry GHCR (public).
5. Patch app tối thiểu: `RUN_MIGRATIONS=false` tắt migration trong app; `sequelize-cli` chuyển sang
   dependencies (Job migration cần), `typescript` chuyển sang devDependencies.

## Lựa chọn đã cân nhắc
- **alpine**: nhỏ hơn ~117MB (giải nén) nhưng dùng musl; native module và hành vi DNS khác glibc.
- **distroless**: bề mặt tấn công nhỏ nhất, nhưng app `exec()` qua `/bin/sh` → chưa dùng được.
  Xem lại khi migration chỉ còn chạy bằng Job.
- **Giả lập toàn bộ build amd64** thay vì chạy stage build ở BUILDPLATFORM: đơn giản hơn nhưng chậm,
  trong khi output của tsc không phụ thuộc kiến trúc.
- **Tag `latest`/semver** thay vì SHA: tag có thể bị push đè; SHA truy ngược được về đúng commit.

## Hệ quả
- (+) Graceful shutdown đo được: docker stop 0.15s, kubectl delete 0.87s (trước đây phải chờ SIGKILL).
- (+) Một tag dùng cho mọi kiến trúc; manifest K8s pin digest index (Phần 3).
- (+) Image non-root sẵn sàng cho Pod Security `restricted` và `readOnlyRootFilesystem` (Phần 6).
- (−) Pin digest nên không tự nhận bản vá Debian/Node → cần bot bump digest (Phần 8).
- (−) Deps 2021 và yarn v1 (maintenance) → Trivy sẽ báo CVE (Phần 8).
- (−) Package GHCR đã public thì không chuyển lại private được.

## Lab ↔ Prod
- AKS: build trên agent Azure DevOps, push ACR, cluster pull bằng managed identity (`--attach-acr`).
  Base image mirror qua ACR artifact cache để tránh rate limit của Docker Hub.
- RKE2: Harbor (proxy cache cho Docker Hub + project riêng), node pull bằng robot account trong `registries.yaml`.
