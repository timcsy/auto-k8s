---
name: k3s-domain-manager
description: >-
  在 k3s 上用 Helm 部署 domain-manager（github.com/timcsy/domain-manager）與其相依的 cert-manager，
  設定 Traefik ingress class 與 Let's Encrypt ClusterIssuer，並掛上 domain-manager UI 的 ingress。
  當要部署 domain-manager 來用 Web UI/API/MCP 管理子網域→service 對應與自動 SSL 時使用。內含 image tag
  與 admin 密碼的已知 chart 陷阱與規避方式。
---

# 部署 domain-manager（Helm + cert-manager）

## 需要的輸入
- `SSH_TARGET`、`ADMIN_EMAIL`（Let's Encrypt 用）、`UI_HOST`（UI 網址）。
- `SSL_MODE`：`http01`（預設）或 `dns01`（需 `CF_TOKEN`）。

## 1. 取 chart + 相依 cert-manager
```bash
ssh "$SSH_TARGET" '
  cd ~ && rm -rf domain-manager && git clone --depth 1 https://github.com/timcsy/domain-manager.git
  cd ~/domain-manager
  helm repo add jetstack https://charts.jetstack.io && helm repo update jetstack
  helm dependency build ./helm/domain-manager
'
```

## 2. 安裝
```bash
ssh "$SSH_TARGET" 'cd ~/domain-manager && helm install domain-manager ./helm/domain-manager \
  --namespace domain-manager --create-namespace \
  --set ingress.className=traefik \
  --set subdomains.defaultIngressClass=traefik \
  --set ingress.certManager.issuerEmail="'"$ADMIN_EMAIL"'" \
  --set image.tag=main'
  # dns01 另加： --set cloudflare.enabled=true --set cloudflare.apiToken="$CF_TOKEN"
ssh "$SSH_TARGET" 'kubectl rollout status deploy/domain-manager -n domain-manager --timeout=120s; kubectl get clusterissuer letsencrypt-prod'
```

## ⚠️ 已知 chart 陷阱（必處理）
1. **image tag**：chart 預設 `image.tag`（如 `v0.2.10`）在 ghcr 可能不存在 → ImagePullBackOff。先確認可用 tag：
   ```bash
   TOKEN=$(curl -s "https://ghcr.io/token?scope=repository:timcsy/domain-manager:pull" | sed -E 's/.*"token":"([^"]+)".*/\1/')
   curl -s -H "Authorization: Bearer $TOKEN" "https://ghcr.io/v2/timcsy/domain-manager/tags/list"
   ```
   用 `--set image.tag=main` 覆寫（對應 main 分支最新 build）。
2. **admin 密碼**：chart 把 `admin.password` 寫進 Secret 但 **deployment 沒引用** → `--set admin.password` 無效。初始登入固定 **`admin` / `admin`**。**登入後務必到 UI「系統設定 → 帳號管理」改密碼**。
3. **ClusterIssuer**：`letsencrypt-prod` 預設帶 **http01 solver**（class 跟著 `ingress.className`）；只要 Traefik 握有 80 port 即成立。填了 `cloudflare.apiToken` 才會多 dns01 solver（可簽 wildcard）。

## 3. 掛 UI ingress（DNS 指好後 cert 自動簽出）
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: domain-manager-ui
  namespace: domain-manager
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
spec:
  ingressClassName: traefik
  tls: [{hosts: [<UI_HOST>], secretName: ui-tls}]
  rules: [{host: <UI_HOST>, http: {paths: [{path: /, pathType: Prefix, backend: {service: {name: domain-manager, port: {number: 8080}}}}]}}]
```

## 驗證
```bash
# app 健康 + 實測登入（別假設密碼，要實打）
CIP=$(ssh "$SSH_TARGET" 'kubectl get svc -n domain-manager domain-manager -o jsonpath="{.spec.clusterIP}"')
ssh "$SSH_TARGET" "curl -s -o /dev/null -w 'health %{http_code}\n' http://$CIP:8080/health
  curl -s -o /dev/null -w 'login %{http_code}\n' -X POST http://$CIP:8080/api/v1/auth/login -H 'Content-Type: application/json' -d '{\"username\":\"admin\",\"password\":\"admin\"}'"
```
通過：health=200、login=200、`kubectl get pods -n domain-manager` 全 Running（cert-manager 3 pod + domain-manager）、ClusterIssuer Ready。

接著用 `k3s-dns-ssl` 讓 `UI_HOST` 解析到本機並完成憑證。
