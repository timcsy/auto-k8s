---
name: k3s-migrate-proxy
description: >-
  把 VPS 上既有的反向代理（systemd nginx/apache/caddy 等，正佔用 80/443 服務正式網站）安全遷移到 k3s 的
  Traefik——後端用 external Service+Endpoints 指向原本的容器/程序、沿用既有 TLS 憑證、用 per-router middleware
  做 HTTP→HTTPS 轉址，驗證無誤後才停用舊 proxy，不中斷服務。當 k3s 要接管已被占用的 80/443、或要把既有
  reverse proxy 收進叢集時使用。這是條件式步驟：只有偵測到既有 80/443 服務才需要。
---

# 遷移既有反向代理到 Traefik（不中斷服務）

目標：Traefik 接管 80/443，但既有正式站不斷線。既有後端（常是 docker 容器）**保持原樣**，只把「路由 + TLS 終結」從舊 proxy 搬到 Traefik。

## 1. 盤點既有 proxy
讀設定取得每個站的 `server_name`、`upstream`（host:port）、所用憑證：
```bash
ssh "$SSH_TARGET" 'sudo nginx -T 2>/dev/null | grep -E "server_name|proxy_pass|ssl_certificate|listen"'
```
逐一確認每個後端**實際在不在跑**（`ss -tlnp` / `docker ps`）。設定還在但服務早掛是常見的——別把別人原本就壞的站算成自己造成的。

## 2. 匯入既有憑證成 TLS secret
```bash
ssh "$SSH_TARGET" '
  kubectl create namespace legacy --dry-run=client -o yaml | kubectl apply -f -
  sudo install -m644 <crt路徑> /tmp/c.crt; sudo install -m644 <key路徑> /tmp/c.key
  kubectl create secret tls wildcard-cert -n legacy --cert=/tmp/c.crt --key=/tmp/c.key --dry-run=client -o yaml | kubectl apply -f -
  sudo rm -f /tmp/c.crt /tmp/c.key
'
```
> 萬用憑證 `*.example.com` **只蓋一層**（蓋 `a.example.com`，不蓋 `a.b.example.com`）。多層名需另簽。

## 3. 每個站：external Service+Endpoints + 雙 Ingress
重點：
- 後端容器須 publish 在 `0.0.0.0`（非 `127.0.0.1`），pod 才連得到 node IP:port。
- **HTTP→HTTPS 用 per-router middleware，切勿做全域 entrypoint redirect**——全域 redirect 會把 cert-manager 的 HTTP-01 challenge（走 80）一起導走。

```yaml
apiVersion: v1
kind: Service
metadata: {name: <APP>, namespace: legacy}
spec: {ports: [{port: <PORT>, targetPort: <PORT>}]}
---
apiVersion: v1
kind: Endpoints
metadata: {name: <APP>, namespace: legacy}
subsets: [{addresses: [{ip: <NODE_IP>}], ports: [{port: <PORT>}]}]
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata: {name: redirect-https, namespace: legacy}
spec: {redirectScheme: {scheme: https, permanent: true}}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <APP>
  namespace: legacy
  annotations: {traefik.ingress.kubernetes.io/router.entrypoints: websecure}
spec:
  ingressClassName: traefik
  tls: [{hosts: [<HOST>], secretName: wildcard-cert}]
  rules: [{host: <HOST>, http: {paths: [{path: /, pathType: Prefix, backend: {service: {name: <APP>, port: {number: <PORT>}}}}]}}]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <APP>-redirect
  namespace: legacy
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
    traefik.ingress.kubernetes.io/router.middlewares: legacy-redirect-https@kubernetescrd
spec:
  ingressClassName: traefik
  rules: [{host: <HOST>, http: {paths: [{path: /, pathType: Prefix, backend: {service: {name: <APP>, port: {number: <PORT>}}}}]}}]
```

## 4. 切換前先驗證，再停舊 proxy
```bash
# 驗證 Traefik 已正確路由＋用對憑證
curl -sk -o /dev/null -w "%{http_code}\n" https://<HOST>/
echo | openssl s_client -connect <HOST>:443 -servername <HOST> 2>/dev/null | openssl x509 -noout -subject
# 確認無誤後停用舊 proxy（設定保留可回滾）
ssh "$SSH_TARGET" 'sudo systemctl stop nginx && sudo systemctl disable nginx'
```

## 驗證
- 每個站 HTTPS 回應正常、憑證為原本的、HTTP 301 轉址。
- 舊 proxy `inactive` + `disabled`，設定檔保留。
