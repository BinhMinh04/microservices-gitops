# Dựng / xoá môi trường lab local.
# Chạy từ gốc repo:  make -C local-env <target>    (make -C local-env help để xem danh sách)

SHELL := /bin/bash
.DEFAULT_GOAL := help

# ---- Version: chỉ sửa ở đây. Node image pin trong kind-config.yaml ----
CLUSTER_NAME          := microlab
METALLB_CHART_VERSION := 0.16.1

# ---- Mạng cố định để IP pool của MetalLB luôn đúng ----
# Docker chỉ được cấp IP cho node trong KIND_NET_IP_RANGE (.1 - .127).
# Dải .200 - .250 dành riêng cho MetalLB (platform/metallb/config/ipaddresspool.yaml).
KIND_NET_SUBNET   := 192.168.97.0/24
KIND_NET_GATEWAY  := 192.168.97.1
KIND_NET_IP_RANGE := 192.168.97.0/25

# Mọi lệnh kubectl/helm trong Makefile dùng kubeconfig riêng của lab
export KUBECONFIG := $(HOME)/.kube/$(CLUSTER_NAME).yaml

.PHONY: help up down network cluster metallb smoke smoke-clean

help: ## Liệt kê các target
	@grep -E '^[a-z-]+:.*## ' $(MAKEFILE_LIST) | awk -F':.*## ' '{printf "  %-12s %s\n", $$1, $$2}'

up: network cluster metallb ## Dựng toàn bộ môi trường

network: ## Tạo docker network "kind" với subnet cố định; báo lỗi nếu network cũ sai cấu hình
	@if docker network inspect kind >/dev/null 2>&1; then \
	  docker network inspect kind --format '{{range .IPAM.Config}}{{.IPRange}} {{end}}' | grep -q '$(KIND_NET_IP_RANGE)' \
	    || { echo "LỖI: network kind đã tồn tại nhưng không có ip-range $(KIND_NET_IP_RANGE)."; \
	         echo "Xoá cluster rồi chạy: docker network rm kind"; exit 1; }; \
	  echo "network kind OK"; \
	else \
	  docker network create kind --driver bridge \
	    --subnet $(KIND_NET_SUBNET) --gateway $(KIND_NET_GATEWAY) --ip-range $(KIND_NET_IP_RANGE); \
	fi

cluster: ## Tạo cluster kind (bỏ qua nếu đã có)
	@if kind get clusters | grep -qx '$(CLUSTER_NAME)'; then \
	  echo "cluster $(CLUSTER_NAME) đã tồn tại"; \
	else \
	  kind create cluster --config kind-config.yaml --kubeconfig $(KUBECONFIG); \
	fi
	kubectl get nodes -o wide -L topology.kubernetes.io/zone

metallb: ## Cài MetalLB + IP pool
	helm repo add metallb https://metallb.github.io/metallb >/dev/null 2>&1 || true
	helm repo update metallb
	helm upgrade --install metallb metallb/metallb --version $(METALLB_CHART_VERSION) \
	  --namespace metallb-system --create-namespace --wait
	@for i in $$(seq 1 12); do \
	  kubectl apply -f ../platform/metallb/config/ipaddresspool.yaml && exit 0; \
	  echo "webhook chưa sẵn sàng, thử lại sau 5s ($$i/12)"; sleep 5; \
	done; exit 1

smoke: ## Deploy whoami và gọi 6 lần qua EXTERNAL-IP
	kubectl apply -f smoke-test/whoami.yaml
	kubectl -n lb-test rollout status deploy/whoami --timeout=120s
	@for i in $$(seq 1 24); do \
	  IP=$$(kubectl -n lb-test get svc whoami -o jsonpath='{.status.loadBalancer.ingress[0].ip}'); \
	  [ -n "$$IP" ] && break; sleep 5; \
	done; \
	echo "EXTERNAL-IP: $$IP"; \
	for i in 1 2 3 4 5 6; do curl -s -m 5 http://$$IP | head -1; done

smoke-clean: ## Xoá smoke test
	kubectl delete ns lb-test --ignore-not-found

down: ## Xoá cluster (giữ lại network "kind")
	kind delete cluster --name $(CLUSTER_NAME) --kubeconfig $(KUBECONFIG)
	rm -f $(KUBECONFIG)