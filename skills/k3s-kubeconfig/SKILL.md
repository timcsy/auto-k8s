---
name: k3s-kubeconfig
description: >-
  產生可攜的 k3s kubeconfig，讓本機/CI 從任何地方操作叢集——幫 k3s API 憑證加 tls-san、把 server 從
  127.0.0.1 改成 domain、確認 6443 對外可達、把 context/cluster/user 改名避免與其他 kubeconfig 撞名，
  並用 .gitignore 保護內含的 cluster-admin 憑證。當要遠端 kubectl/helm 操作 k3s、或把 kubeconfig 交給
  其他機器使用時用。
---

# 可攜 kubeconfig（tls-san + domain + 保護）

預設 k3s kubeconfig 的 `server` 是 `https://127.0.0.1:6443`，且 API 憑證 SAN 不含主機名 → 不能直接搬到別台用。

## 需要的輸入
- `SSH_TARGET`、`KUBECONFIG_DOMAIN`（API 對外網址，須已解析到本機）、`OUT_DIR`（存放資料夾，例如 `./kubeconfig`）、`CTX`（context 名）。

## 1. 確認 6443 對外可達（區網常擋）
```bash
nc -z -G 8 <host> 6443 && echo "6443 open" || echo "6443 被擋 → 只能用 SSH tunnel"
curl -sk -m8 -o /dev/null -w "API %{http_code}\n" https://<host>:6443/ping
```
> 若 6443 被擋，退而求其次：`ssh -L 6443:127.0.0.1:6443 "$SSH_TARGET"`，kubeconfig 的 server 維持 `127.0.0.1:6443`（憑證本就含 127.0.0.1，零改動）。

## 2. 幫 API 憑證加 tls-san 並重簽（會重啟 k3s）
```bash
ssh "$SSH_TARGET" 'printf "write-kubeconfig-mode: \"0644\"\ntls-san:\n  - %s\n" "'"$KUBECONFIG_DOMAIN"'" | sudo tee /etc/rancher/k3s/config.yaml
  sudo rm -f /var/lib/rancher/k3s/server/tls/dynamic-cert.json
  sudo systemctl restart k3s
  for i in $(seq 1 24); do kubectl get --raw=/readyz >/dev/null 2>&1 && break; sleep 5; done; kubectl get nodes'
# 驗證新 SAN 含 domain
echo | openssl s_client -connect "$KUBECONFIG_DOMAIN":6443 2>/dev/null | openssl x509 -noout -ext subjectAltName
```
> 重啟只動 control plane，跑著的 pod（domain-manager/Traefik/遷移的站）由 containerd 顧、不重啟；仍宜先告知使用者並於重啟後 `curl` 驗證對外服務未受影響。

## 3. 下載、改寫 server + context 名、存進受保護資料夾
```bash
mkdir -p "$OUT_DIR"
scp "$SSH_TARGET":/etc/rancher/k3s/k3s.yaml "$OUT_DIR/cluster.yaml"
sed -i '' "s#https://127.0.0.1:6443#https://$KUBECONFIG_DOMAIN:6443#g; \
  s/: default/: $CTX/g; s/name: default/name: $CTX/g; s/current-context: default/current-context: $CTX/g" "$OUT_DIR/cluster.yaml"
# (Linux 的 sed 用 -i 不帶 ''）
```

## 4. 本機實測 + 保護
```bash
KUBECONFIG="$OUT_DIR/cluster.yaml" kubectl get nodes   # 真正證明可攜
# 確保 .gitignore 排除（內含 cluster-admin 憑證）
grep -q "$(basename "$OUT_DIR")/" .gitignore 2>/dev/null || printf "%s/\n*.kubeconfig\n" "$(basename "$OUT_DIR")" >> .gitignore
```

## ⚠️ 安全
這份 kubeconfig 內嵌 **cluster-admin 憑證**（等同叢集 root）。**切勿提交 git 或外流**；要給 CI/他人請改發**權限受限的 ServiceAccount token**，而非這份 admin config。
