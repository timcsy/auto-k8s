# auto-k8s — VPS 上自動部署 k3s + DNS + SSL 的 Claude Code Skills

一組 [Claude Code](https://claude.com/claude-code) skills，把「在一台 VPS 上從零部署單節點 k3s、安全遷移既有
80/443 服務、用 [domain-manager](https://github.com/timcsy/domain-manager) + cert-manager 自動化子網域與
Let's Encrypt SSL、最後產生可攜 kubeconfig」做成可重複、可單獨取用的流程。

> 設計原則：**通用核心**（每次都跑）與**環境特例**（偵測到才處理）分開；收錄的是**偵測與驗證的「做法」**，
> 不是任何特定機房的「答案」。會碰到既有正式服務、難回滾或對外發布的動作，一律先向使用者確認。

---

## 架構：1 個總控 + 6 個子 skill

打包成一個 plugin（`k3s-vps`），真實檔案放在 `skills/`；`.claude/skills` 是指向 `skills/` 的 symlink，讓 clone 當專案時也能自動載入（單一份檔案、不重複維護）。

```
auto-k8s/
├── .claude-plugin/
│   ├── marketplace.json     ← plugin marketplace 目錄（讓別人 /plugin install）
│   └── plugin.json          ← plugin 清單
├── skills/                  ← 真實 skills（plugin 標準位置）
│   ├── k3s-vps-deploy/      ← orchestrator：收齊參數、依序編排、每關驗證、遇正式服務先確認
│   ├── k3s-preflight/       ← 環境偵測（OS/sudo/IP/資源/既有 k8s，重點抓 80/443 佔用）
│   ├── k3s-install/         ← 安裝 k3s + 設定 kubeconfig + 安裝 Helm
│   ├── k3s-migrate-proxy/   ← 【條件式】把既有反代（nginx 等）遷到 Traefik，不中斷服務
│   ├── k3s-domain-manager/  ← Helm 部署 domain-manager + cert-manager（含 chart 陷阱規避）
│   ├── k3s-dns-ssl/         ← DNS 委派診斷 + cert-manager 自動簽憑證 + 上線前 --resolve 預驗
│   └── k3s-kubeconfig/      ← 加 tls-san、產生可攜 kubeconfig、.gitignore 保護
└── .claude/skills -> ../skills   （symlink，供 clone 當專案時自動載入）
```

| Skill | 做什麼 | 何時單獨用 |
|---|---|---|
| `k3s-vps-deploy` | 端到端串起全部 | 想完整一次跑完 |
| `k3s-preflight` | 偵測主機是否就緒、80/443 佔用 | 評估某台 VPS 現況 |
| `k3s-install` | 裝 k3s/kubeconfig/Helm | 只想建叢集 |
| `k3s-migrate-proxy` | 既有反代遷進 Traefik | k3s 要接管已被占用的 80/443 |
| `k3s-domain-manager` | 部署 domain-manager | 只想裝這套 ingress/SSL 管理工具 |
| `k3s-dns-ssl` | DNS 排查 + 簽憑證 | DNS/HTTPS「設了卻不通」要 debug |
| `k3s-kubeconfig` | 可攜 kubeconfig | 要從本機/CI 遠端操作叢集 |

---

## 安裝（三種方式）

### A. Plugin marketplace（推薦，可跨專案、可更新）
在 Claude Code 中：
```
/plugin marketplace add timcsy/auto-k8s
/plugin install k3s-vps@auto-k8s
```
安裝後 skill 會帶 plugin namespace 前綴觸發，例如 `/k3s-vps:k3s-vps-deploy`、`/k3s-vps:k3s-dns-ssl`。

### B. 手動複製成個人 skills（最簡單、無前綴）
```bash
git clone https://github.com/timcsy/auto-k8s.git
cp -r auto-k8s/skills/* ~/.claude/skills/
```
之後直接 `/k3s-vps-deploy`、`/k3s-dns-ssl`（無 namespace 前綴）。

### C. 當成專案目錄直接用（零設定）
```bash
git clone https://github.com/timcsy/auto-k8s.git && cd auto-k8s && claude
```
`.claude/skills`（symlink→`skills/`）會自動載入，`/k3s-vps-deploy` 等即可用。

> 需求：plugin marketplace 方式建議 Claude Code ≥ 2.1.0；手動 / 專案方式任何版本皆可。

---

## 使用方式

**完整部署**：呼叫 `/k3s-vps-deploy`（plugin 安裝時為 `/k3s-vps:k3s-vps-deploy`）。它會先收齊參數，再依序叫各子 skill；`k3s-migrate-proxy` 只在偵測到 80/443 被占用時才執行。

**只做其中一段**：直接呼叫對應子 skill，例如
```
/k3s-dns-ssl        # 只排查 DNS / 簽憑證
/k3s-kubeconfig     # 只產生可攜 config
```

### 常用參數
| 參數 | 說明 | 範例 |
|---|---|---|
| `SSH_TARGET` | 登入字串 | `user@host.example.com` |
| `ADMIN_EMAIL` | Let's Encrypt 信箱 | `you@example.com` |
| `UI_HOST` | domain-manager UI 網址 | `dns.app.example.com` |
| `SSL_MODE` | `http01`（DNS 已指向本機）/ `dns01`（Cloudflare token，可簽 wildcard） | `http01` |
| `KUBECONFIG_DOMAIN` | 可攜 kubeconfig 的 API domain（會加進 tls-san） | `host.example.com` |

---

## 前提（skill 只「檢查」，預設不代勞）

- 可 SSH 登入的 Linux VPS（以 Ubuntu 22.04 / amd64 驗證）。
- 已有非 root 使用者 + sudo（免密最順）。
- SSH 已設好（建議 key-based）。
- **docker 不是 k3s 的必要條件**（k3s 自帶 containerd）；只有要把既有 docker 容器遷進叢集時才需要。

缺項時 skill 會報錯並給補救指令，是否 bootstrap 由你決定。

---

## 通用陷阱速查（血淚換來的）

1. **k3s Traefik+ServiceLB 經 iptables DNAT 即時搶 80/443，沒有緩衝期** —— 裝前先偵測既有服務並規劃遷移。
2. **萬用憑證 `*.x` 只蓋一層**（不蓋 `a.b.x`）。
3. **HTTP→HTTPS 用 per-router middleware，勿全域 redirect** —— 全域 redirect 會擋掉 cert-manager 的 HTTP-01 challenge。
4. **domain-manager chart 預設 image tag 可能不存在** → 覆寫 `--set image.tag=main`。
5. **domain-manager `--set admin.password` 無效**（chart bug）→ 初始登入固定 `admin/admin`，登入後從 UI 改。
6. **DNS「設了卻不通」** → `dig +trace` 從 parent 找真權威、用 MX 當指紋比對、看 SOA serial 有沒有前進；託管面板填完常要按「發布」。
7. **可攜 kubeconfig** 要加 `tls-san` + 確認 6443 對外開放 + `.gitignore` 保護。
8. **host 上 `ss` 看不到 80/443 listener 是正常的**（klipper-lb 走 DNAT），以實際 `curl` 為準。

---

## 安全

- `kubeconfig/` 內含 **cluster-admin 憑證**（等同叢集 root），已由 `.gitignore` 排除，**切勿提交或外流**。
- 要給 CI/他人請改發**權限受限的 ServiceAccount token**，而非 admin kubeconfig。

---

## 相關

- [domain-manager](https://github.com/timcsy/domain-manager) — 本流程部署的核心工具（子網域 → k8s service 對應 + 自動 SSL，支援 Web UI / API / MCP）。
- [k3s](https://k3s.io) · [cert-manager](https://cert-manager.io) · [Traefik](https://traefik.io)
