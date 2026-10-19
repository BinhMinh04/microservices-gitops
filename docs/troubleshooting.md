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

## Phần 1 — Container image

| Triệu chứng | Nguyên nhân | Cách sửa |
|---|---|---|
| `COPY ... "/code/public": not found` | Build sai thư mục: context là `code/` thay vì gốc repo | Luôn `docker build` ở gốc repo; đường dẫn COPY tính từ context |
| Dòng đầu output build ghi `docker:desktop-linux` | Docker Desktop tự chiếm docker context khi khởi động | `docker context use orbstack`; tắt auto-start Docker Desktop |
| yarn `ENOENT ... chmod /tmp/yarn-cache/...` | 2 stage chạy song song cùng ghi một cache mount (`sharing=shared`) | `--mount=type=cache,...,sharing=locked`; `docker builder prune --filter type=exec.cachemount` |
| `pull access denied` tại `COPY --from=build` + cảnh báo `NoEmptyContinuation` | Dòng `RUN ... \` thừa dấu `\` nuốt mất `FROM ... AS build` | `docker build --check .` trước khi build; ghi đè file bằng heredoc thay vì sửa từng đoạn |
| `COPY --from=deps /app/node_modules: not found` | Stage tồn tại nhưng thiếu bước tạo ra path đó | Kiểm tra các `RUN` của stage nguồn |
| `no matching manifest for linux/amd64` / AKS `ErrImagePull` | Image build trên Mac chỉ có arm64 | `docker buildx build --platform linux/amd64,linux/arm64 --push` |
| `exec format error` / `CrashLoopBackOff` | Manifest đơn lẻ (không có index) sai kiến trúc | Như trên; kiểm tra bằng `docker buildx imagetools inspect` |
| kubectl gọi `localhost:8080` | Không có kubeconfig (biến chưa export hoặc file mất) | `export KUBECONFIG=~/.kube/microlab.yaml`; `kind export kubeconfig` hoặc `make up` |
| Kubelet: `failed to fetch anonymous token ... 401` | Package GHCR private (mặc định khi push lần đầu) | Đổi sang Public và hoàn tất hộp thoại xác nhận; kiểm tra bằng `gh api /user/packages/container/<name> --jq .visibility` |
| `git push`: `ECONNREFUSED .../vscode-git-*.sock` | `GIT_ASKPASS` trỏ vào socket VS Code đã chết | `gh auth setup-git` |
| `curl` in thêm nhiều dòng `HTTP 000` | Zsh tương tác không coi `#` là comment | Bỏ comment khỏi lệnh hoặc `setopt interactivecomments` |