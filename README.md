# RuleOps

RuleStox 的**維運執行底座**：哨兵、通道、角色身分與 runbook 的落腳處。

## 分工原則

來源：`C:/Projects/RuleStox/.claude/daily-plan/sessions/handoff-TASK-376-stage0b.md` §2（使用者 2026-09-08 裁示）。要點內嵌於此，避免離開本機時斷鏈。

**放進 RuleOps**：哨兵 compose、Uptime Kuma、cloudflared、systemd＋tmux、角色身分、日誌、runbook。

**不放進 RuleOps**：業務內容與業務碼。RulePlatform 的 `/health` 深化、APScheduler heartbeat job、`pg_backup.sh` 尾端 ping，以及 RuleETLFlow 的 DAG 鏈尾 ping，都留在各自的 repo——它們是業務碼，只是剛好被哨兵消費。

## 目錄

| 目錄 | 內容 | 現況 |
|---|---|---|
| `sentinel/` | 哨兵容器定義（Uptime Kuma 2.5.3 compose）＋日後的 Kuma 監測項匯出 | compose 已落地 |
| `cloudflared/` | Stage 1 的 Cloudflare Tunnel 設定 | 佔位，見該目錄 README |
| `runbook/` | 哨兵 runbook（建置／啟停／備份／升級／密鑰位置／監測項／演練紀錄） | 骨架 |
| `roles/` | TASK-378 L2 角色身分預留 | 佔位，見該目錄 README |

規格書 §3.1 寫的 `kuma/` 與 `docs/sentinel-runbook.md` 路徑，依交接檔 §3 0-B1 改為 `sentinel/` 與 `runbook/sentinel-runbook.md`。

## 密鑰

實值一律不進 repo。三個落點：Kuma volume（通知憑證）、各機的 `.env` 與 cron 檔（push URL）、使用者密碼管理器（Kuma 帳密與 2FA 回復碼）。位置對照見 `runbook/sentinel-runbook.md` 第 6 章，key 名索引見 `.env.example`，排除規則見 `.gitignore`。

## 出處

- 任務：TASK-376（維運哨兵 Phase 1／L1 檢測層）
- 規格書：`C:/Projects/RuleStox/.claude/daily-plan/sessions/TASK-376-l1/l1-spec.html`
- 交接檔：`C:/Projects/RuleStox/.claude/daily-plan/sessions/handoff-TASK-376-stage0b.md`

目前階段：Stage 0——全部在使用者本機 Docker 驗證，不接 prod push、不動線上服務。
