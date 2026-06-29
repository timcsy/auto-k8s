---
name: k3s-dns-ssl
description: >-
  處理 k3s 上的 DNS 與 Let's Encrypt SSL——診斷 DNS 委派（dig +trace 從 parent 找真權威、用 MX 當指紋比對
  面板 vs 對外權威、SOA serial 驗有沒有真的發布）、讓子網域指向叢集、用 cert-manager 自動簽憑證；並提供上線前
  用 --resolve + 真實 ACME challenge token 預驗整條鏈路的方法。當子網域或 HTTPS 憑證沒生效要排查、DNS「設了卻
  不通」、或上線前要先確認機器端 100% 就緒時使用。
---

# DNS 委派診斷 + 憑證自動簽發 + 上線前預驗

## A. 「設了卻不通」——找出真正的權威 DNS
記錄加了卻不生效，最常見是**改到的系統 ≠ 對外服務的系統**。

```bash
H=<UI_HOST>; ZONE=<zone 如 example.com>
# 1) 從 parent 往下 trace：parent 實際委派給誰（可能跟 zone 內 NS 不一致）
dig +trace A "$H" | tail -15
# 2) 直接問「真權威」：SOA serial 沒前進＝沒 reload/發布
dig @<real-auth> SOA "$ZONE"
dig @<real-auth> A "$H"
# 3) 用一筆已知對外正常的記錄當指紋（如 MX），比對面板 vs 真權威是否同一套
dig @<real-auth> MX "$ZONE"
# 4) 公開遞迴交叉驗證（Let's Encrypt 視角）
dig @1.1.1.1 +short A "$H"; dig @8.8.8.8 +short A "$H"
```
判讀：
- in-zone NS（`ns1.<zone>`）**不一定**是 parent 委派的對象；以 `dig +trace` 看到的為準。
- **SOA serial** 是「有沒有真的發布」的鐵證：改了但 serial 沒前進 → 還沒生效。
- 公開遞迴查不到 → 對外不存在 → HTTP-01 與一般使用者都到不了。

## B. 讓記錄生效（依供應商）
- **自管 BIND**：改 zone → **SOA serial +1** → `rndc reload <zone>`（或重啟 named）。
- **區網/託管面板**：填完常需按「發布 / 套用」；務必確認真權威 serial 前進。
- **Cloudflare（dns01）**：把該層委派給 Cloudflare，domain-manager 設 `cloudflare.enabled=true` + token，可自助簽 **wildcard** 憑證（不依賴 80 port）。
- **萬用 DNS** `*.sub A <IP>`：讓所有子網域一次指向本機，搭配 HTTP-01 各自簽 cert，最省事。

## C. 上線前預驗整條鏈路（DNS 還沒生效就能證明機器端 OK）
用 `--resolve` 假裝 DNS 已指過來，並**扮演 Let's Encrypt** 抓 challenge：
```bash
IP=<node_public_ip>; H=<UI_HOST>
# 取 pending challenge 的 token/key
ssh "$SSH_TARGET" 'kubectl get challenge -A -o jsonpath="{.items[0].spec.token} {.items[0].spec.key}"'
# 扮演 LE：回傳值 == .spec.key 就證明 DNS 一生效 HTTP-01 必過、cert 會自動簽出
curl -s --resolve $H:80:$IP "http://$H/.well-known/acme-challenge/<TOKEN>"
curl -s --resolve $H:80:$IP  -o /dev/null -w "UI HTTP  %{http_code}\n" "http://$H/"
curl -sk --resolve $H:443:$IP -o /dev/null -w "UI HTTPS %{http_code}\n" "https://$H/"
```

## D. 上線後驗證（DNS 生效之後）
```bash
H=<UI_HOST>
dig @<real-auth> A "$H"; dig @1.1.1.1 +short A "$H"       # 對外查得到、serial 前進
ssh "$SSH_TARGET" 'kubectl get certificate -A; kubectl get challenge -A 2>/dev/null'  # cert Ready、無 pending
curl -s -o /dev/null -w "HTTPS %{http_code} verify=%{ssl_verify_result}\n" "https://$H/"  # verify=0
```
通過標準：對外 DNS 解析正確、`certificate READY=True`、無 pending challenge、`verify=0`、頁面正確。
