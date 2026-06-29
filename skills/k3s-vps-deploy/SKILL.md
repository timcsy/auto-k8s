---
name: k3s-vps-deploy
description: >-
  端到端在一台 VPS 上部署 k3s + domain-manager + DNS/SSL 的總流程。依序串起六個子 skill：k3s-preflight →
  k3s-install →（條件式）k3s-migrate-proxy → k3s-domain-manager → k3s-dns-ssl → k3s-kubeconfig，並在每個
  關卡驗證、遇到既有正式服務時停下確認。當使用者要「完整一次跑完」整套 VPS 上的 k3s 部署與 DNS/SSL 設定時使用；
  若只想做其中一段，直接呼叫對應的子 skill 即可。
---

# k3s VPS 端到端部署（orchestrator）

把六個子 skill 串成完整流程。**先一次收齊參數**，再依序呼叫各子 skill（用 Skill 工具）；子 skill 共用對話中的參數與偵測結果。

## 步驟 0 — 一次收齊參數（缺的用 AskUserQuestion 問）
| 參數 | 說明 |
|---|---|
| `SSH_TARGET` | `user@host` |
| `ADMIN_EMAIL` | Let's Encrypt 信箱 |
| `UI_HOST` | domain-manager UI 網址（如 `dns.app.example.com`） |
| `SSL_MODE` | `http01`（DNS 已/將指向本機）或 `dns01`（Cloudflare token，可簽 wildcard） |
| `CF_TOKEN` | 僅 `dns01` |
| `KUBECONFIG_DOMAIN` | 可攜 kubeconfig 的 API domain |

## 步驟 1 — `k3s-preflight`
偵測環境，**特別看 80/443 是否被占用**。把摘要與建議路線交使用者確認。
→ 決定變數 `MIGRATE_EXISTING`：80/443 被既有服務占用 = 需要遷移。

## 步驟 2 — `k3s-install`
- 乾淨主機：保留預設 Traefik。
- `MIGRATE_EXISTING=true`：兩種策略擇一——(a) 安裝時 `--disable traefik servicelb`，遷移資源備好後再啟用；或 (b) 保留 Traefik 但把 `k3s-migrate-proxy` 的資源**緊接著**套上，把中斷壓到最短。**動既有正式服務前先向使用者確認。**

## 步驟 3 —（條件式）`k3s-migrate-proxy`
僅當 `MIGRATE_EXISTING=true` 才執行：把既有反代遷到 Traefik、沿用既有憑證、驗證後才停舊 proxy。乾淨主機跳過此步。

## 步驟 4 — `k3s-domain-manager`
Helm 部署 domain-manager + cert-manager，掛 UI ingress。注意 image tag→`main`、admin 初始密碼 `admin/admin`。

## 步驟 5 — `k3s-dns-ssl`
讓 `UI_HOST` 解析到本機（必要時先做委派診斷），用 cert-manager 自動簽憑證。**DNS 由外部單位管時**：先用該 skill 的 `--resolve` 預驗證明機器端就緒，把「加 DNS 記錄 / 發布」這一步交回使用者，待生效後回來完成驗證。

## 步驟 6 — `k3s-kubeconfig`
加 tls-san、產生可攜 kubeconfig、`.gitignore` 保護。

## 收尾驗證（整條鏈路）
```bash
curl -s -o /dev/null -w "UI HTTPS %{http_code} verify=%{ssl_verify_result}\n" "https://$UI_HOST/"
KUBECONFIG=<out>/cluster.yaml kubectl get nodes
```
全綠＝對外 HTTPS `verify=0`、cert Ready、可攜 kubeconfig 能操作叢集。

---

## 編排原則
- **每個關卡都驗證**再進下一步；失敗就停在該步排查，別硬推。
- **會碰到既有正式服務、難回滾或對外發布的動作**（搶 80/443、停舊 proxy、改 DNS、重啟 k3s），先向使用者確認。
- **環境差異用偵測 + 詢問吸收**，不要把任何特定機房的結論寫死。
- 外部相依（DNS 委派、防火牆開 6443）非機器端可控時，**用 `--resolve` 預驗先證明機器端 OK**，再把球交回使用者。

## 通用陷阱速查（詳見各子 skill）
1. k3s Traefik+ServiceLB 經 iptables DNAT **即時**搶 80/443，無緩衝期。
2. 萬用憑證只蓋一層。
3. HTTP→HTTPS 用 per-router middleware，勿全域 redirect（會擋 HTTP-01）。
4. domain-manager image tag 預設可能不存在 → `image.tag=main`。
5. domain-manager admin `--set` 密碼無效，初始 `admin/admin`，登入後改。
6. DNS：`dig +trace` 找真權威、MX 指紋、SOA serial 驗發布；面板要「發布」。
7. 可攜 kubeconfig 要 tls-san + 6443 對外開 + `.gitignore`。
8. host 上 `ss` 看不到 80/443 listener 正常（klipper 走 DNAT），以 `curl` 為準。
