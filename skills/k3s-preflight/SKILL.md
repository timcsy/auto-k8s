---
name: k3s-preflight
description: >-
  部署 k3s 前偵測 VPS 環境是否就緒——OS/arch、使用者與 sudo、公網 IP、資源、既有 k8s，以及最關鍵的
  80/443 佔用者與既有正式服務。當要在一台 VPS 上裝 k3s、或評估某主機現況、或排查「裝 k3s 會不會打斷既有服務」時使用。
  輸出一份環境摘要與建議的處理路線，供後續 k3s-install / k3s-migrate-proxy 參考。
---

# k3s Preflight 偵測

目標：在動手裝 k3s **之前**，把環境摸清楚並向使用者報告，特別是會影響後續決策的 **80/443 佔用**。

## 需要的輸入
- `SSH_TARGET`（如 `user@host`）。

## 偵測指令
```bash
ssh "$SSH_TARGET" '
  echo "## user/sudo"; whoami; sudo -n true 2>/dev/null && echo "sudo: NOPASSWD" || echo "sudo: 需要密碼"
  echo "## os/arch/res"; . /etc/os-release; echo "$PRETTY_NAME"; uname -m; dpkg --print-architecture 2>/dev/null; nproc; free -h | sed -n 2p; df -h / | sed -n 2p
  echo "## public ip"; curl -s -4 ifconfig.me; echo
  echo "## existing k8s/helm"; which k3s kubectl helm 2>/dev/null; systemctl is-active k3s 2>/dev/null
  echo "## docker(僅遷移既有容器才需要)"; which docker 2>/dev/null && (docker ps --format "{{.Names}} {{.Ports}}" 2>/dev/null || sudo docker ps --format "{{.Names}} {{.Ports}}")
  echo "## PORT 80/443 佔用者（關鍵！）"; sudo ss -tlnp | grep -E ":(80|443) " || echo "80/443 空閒"
'
```

## 判讀與輸出
- **80/443 空閒** → 乾淨主機，後續 `k3s-install` 可保留預設 Traefik，直接接管。
- **80/443 被占用**（nginx/apache/caddy/docker…）→ ⚠️ **重點**：k3s 的 Traefik+ServiceLB 一啟動就會用 iptables DNAT **即時**接管外部 80/443，對既有服務**沒有緩衝期**。必須先盤點既有服務（server_name、upstream、憑證）並規劃 `k3s-migrate-proxy`，或安裝時 `--disable traefik servicelb` 受控接管。
- **sudo 需要密碼** → 後續指令改用 `sudo -S`，或請使用者設免密。
- **已裝 k3s** → 確認是要沿用還是重裝。
- **docker 在跑容器** → 記下容器名與 publish port，遷移時會用到（但 docker 本身不是 k3s 必要條件）。

把摘要交給使用者確認，並標明建議路線（乾淨接管 vs 需要遷移），再進入 `k3s-install`。

## 前置（只檢查、預設不代勞）
非 root 使用者、sudo、SSH（建議 key-based）通常在開主機時已設好。缺項就**報錯並給補救指令**，是否 bootstrap 由使用者決定。
