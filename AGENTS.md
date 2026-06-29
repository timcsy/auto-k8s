# AGENTS.md

給任何 AI coding agent（Claude Code / OpenAI Codex / Cursor / GitHub Copilot / Gemini CLI / Windsurf…）
與人類維護者的入口說明。本 repo 是一組**可重複執行的部署技能**：在一台 VPS 上部署單節點 k3s、安全遷移既有
80/443 服務、用 [domain-manager](https://github.com/timcsy/domain-manager) + cert-manager 自動化子網域與
Let's Encrypt SSL，並產生可攜 kubeconfig。

## 這個 repo 怎麼用

技能（[Agent Skills 開放標準](https://agentskills.io) 的 `SKILL.md`）放在 `skills/`：

| 技能 | 用途 |
|---|---|
| `skills/k3s-vps-deploy/` | **總控**：端到端串起下面六個，先收齊參數再依序執行 |
| `skills/k3s-preflight/` | 環境偵測（OS/sudo/IP/資源，**重點：80/443 佔用**） |
| `skills/k3s-install/` | 安裝 k3s + kubeconfig + Helm |
| `skills/k3s-migrate-proxy/` | 【條件式】既有反代遷到 Traefik，不中斷服務 |
| `skills/k3s-domain-manager/` | Helm 部署 domain-manager + cert-manager |
| `skills/k3s-dns-ssl/` | DNS 委派診斷 + 自動簽憑證 + 上線前 `--resolve` 預驗 |
| `skills/k3s-kubeconfig/` | 加 tls-san、產生可攜 kubeconfig |

**要執行完整部署**：讀 `skills/k3s-vps-deploy/SKILL.md` 並照它編排；只做某段就讀對應 `SKILL.md`。
每個 `SKILL.md` 都是自給自足的步驟 + 驗證 + 該階段陷阱。

## 操作守則（所有 agent 都必須遵守）

1. **會碰到既有正式服務、難回滾或對外發布的動作，先停下來向使用者確認**——尤其：搶占 80/443、停用既有 reverse proxy、改 DNS、重啟 k3s。
2. **每個步驟都先驗證再進下一步**；失敗就停在該步排查，不要硬推。
3. **環境差異用「偵測 + 詢問」吸收，不要把任何特定機房的結論寫死**。收錄的是偵測/驗證的「做法」，不是某次的「答案」。
4. **外部相依（DNS 委派、防火牆開 6443）非機器端可控時**，先用 `--resolve` + 真實 ACME challenge token 預驗，證明機器端已就緒，再把球交回使用者。
5. 所有指令以 SSH 對遠端 VPS 執行；需要的參數見 `k3s-vps-deploy` 的「步驟 0」。

## 通用陷阱（血淚速查；細節在各 SKILL.md）

1. k3s 的 Traefik + ServiceLB 經 **iptables DNAT 即時搶占 80/443，沒有緩衝期** → 裝前先偵測既有服務。
2. 萬用憑證 `*.x` **只蓋一層**（不蓋 `a.b.x`）。
3. HTTP→HTTPS 用 **per-router middleware，勿全域 redirect**（會擋 HTTP-01 challenge）。
4. domain-manager chart 預設 image tag 可能不存在 → `--set image.tag=main`。
5. domain-manager `--set admin.password` 無效（chart bug）→ 初始登入固定 `admin/admin`，登入後改。
6. DNS「設了卻不通」→ `dig +trace` 從 parent 找真權威、MX 當指紋、SOA serial 驗發布；託管面板填完常要按「發布」。
7. 可攜 kubeconfig 要 `tls-san` + 6443 對外開放 + `.gitignore` 保護。
8. host 上 `ss` 看不到 80/443 listener 是正常的（klipper-lb 走 DNAT），以 `curl` 為準。

## 安全

- `kubeconfig/` 內含 **cluster-admin 憑證**，已被 `.gitignore` 排除，**切勿提交或外流**。要給 CI/他人請改發權限受限的 ServiceAccount token。

## 純人類 / 無 AI 工具

每個 `SKILL.md` 也是可照著手動執行的 runbook：依 `k3s-vps-deploy` 的順序，逐段把各 `SKILL.md` 裡的指令（替換佔位符後）貼到終端執行即可。
