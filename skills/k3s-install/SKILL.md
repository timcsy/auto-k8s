---
name: k3s-install
description: >-
  在 VPS 上安裝單節點 k3s（保留內建 Traefik 與 ServiceLB），設定免 sudo 的 kubeconfig，並安裝 Helm 3。
  當要從零建立 k3s 叢集、或為部署應用準備 Kubernetes 環境時使用。執行前建議先跑 k3s-preflight 確認 80/443
  的處理方式；若 80/443 已被既有服務占用，安裝會即時搶占 port，需搭配 k3s-migrate-proxy。
---

# 安裝 k3s + kubeconfig + Helm

## 需要的輸入
- `SSH_TARGET`
- 是否保留內建 Traefik（預設保留；若要受控接管被占用的 80/443，改用 `--disable traefik servicelb`）。

## 1. 安裝 k3s（單節點，開放 kubeconfig 讀取）
```bash
ssh "$SSH_TARGET" 'curl -sfL https://get.k3s.io | sudo INSTALL_K3S_EXEC="server --write-kubeconfig-mode 644" sh -'
# 等 node Ready
ssh "$SSH_TARGET" 'for i in $(seq 1 12); do kubectl get nodes 2>/dev/null | grep -q " Ready" && break; sleep 5; done; kubectl get nodes -o wide'
```
> ⚠️ 若 80/443 被占用且尚未遷移：Traefik 的 svclb 一起來就會經 DNAT 接管外部 80/443。要嘛安裝時加 `--disable traefik servicelb` 之後受控接管，要嘛把 `k3s-migrate-proxy` 的資源**幾乎同時**備好，把中斷壓到最短。

## 2. kubeconfig 給一般使用者（免 sudo 用 kubectl）
```bash
ssh "$SSH_TARGET" '
  mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $(id -u):$(id -g) ~/.kube/config
  grep -q KUBECONFIG ~/.bashrc || echo "export KUBECONFIG=$HOME/.kube/config" >> ~/.bashrc
  kubectl get ns >/dev/null && echo "kubectl OK(免sudo)"
'
```

## 3. 安裝 Helm 3
```bash
ssh "$SSH_TARGET" 'curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sudo bash; helm version --short'
```

## 驗證
- `kubectl get nodes` → `Ready`
- `kubectl get pods -A` → coredns / local-path / metrics-server / traefik / svclb 起來（80/443 被占用時 svclb 可能卡住，屬預期，遷移後恢復）
- `helm version` 正常

> host 上 `ss` 看不到 80/443 listener 是正常的——klipper-lb 走 iptables DNAT，不是 host socket。以實際 `curl` 為準。
