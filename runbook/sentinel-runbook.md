# 哨兵 Runbook（TASK-376 L1 檢測層）

> 骨架版（Stage 0-B1）。各章內容隨 0-B2～0-B5 落地陸續補上；未發生的事不預先填。
> 規格出處：`C:/Projects/RuleStox/.claude/daily-plan/sessions/TASK-376-l1/l1-spec.html`

## 1. 適用範圍

Stage 0＝使用者本機 Docker 的 Uptime Kuma，只對公開端點做唯讀 HTTP GET，不接 prod push。Stage 1 才遷哨兵 VM 並接線 prod 排程。

## 2. 建置與首次設定

首次啟動後的管理員建立、2FA、時區、通知與監測項建立順序見規格書 §3.3。實測步驟與截圖於 0-B2 補。

## 3. 啟停

```bash
# 啟動
docker compose -f RuleOps/sentinel/docker-compose.yml up -d
# 停止（volume 保留）
docker compose -f RuleOps/sentinel/docker-compose.yml down
```

## 4. 備份與還原

SQLite 熱備份會拿到半截檔，必須先停容器再打包 volume。以下為規格書 §3.4 原文，僅 `-f` 路徑改為 `sentinel/`。

⚠️ 未實跑驗證，0-B2 起容器時修正。

```bash
# 備份（PowerShell 或 Git Bash 皆可；在 RuleOps/sentinel/ 下執行）
docker compose -f RuleOps/sentinel/docker-compose.yml stop
docker run --rm -v ruleops_kuma_data:/data -v "$PWD:/backup" alpine \
  tar czf /backup/kuma-backup-$(date +%Y%m%d).tar.gz -C /data .
docker compose -f RuleOps/sentinel/docker-compose.yml start

# 還原（到空 volume）
docker compose -f RuleOps/sentinel/docker-compose.yml down
docker volume rm ruleops_kuma_data
docker volume create ruleops_kuma_data
docker run --rm -v ruleops_kuma_data:/data -v "$PWD:/backup" alpine \
  tar xzf /backup/kuma-backup-YYYYMMDD.tar.gz -C /data
docker compose -f RuleOps/sentinel/docker-compose.yml up -d
```

備份檔含 bot token 與 SMTP 密碼，放本機 `RuleOps/sentinel/backups/`，已在 `.gitignore` 排除。

## 5. 升級

Kuma 啟動時自動跑 DB migration，不可降版（新版 schema 舊版讀不了）；升級失敗的唯一退路是第 4 章的還原。以下為規格書 §3.5 原文，僅 `-f` 路徑改為 `sentinel/`。

⚠️ 未實跑驗證，0-B2 起容器時修正。

```bash
# 1. 先備份（第 4 章）
# 2. 改 compose 的 image tag 到新版本（例：2.6.0）
# 3. 拉取並重建
docker compose -f RuleOps/sentinel/docker-compose.yml pull
docker compose -f RuleOps/sentinel/docker-compose.yml up -d
# 4. 開 UI 確認版本（右下角）與全部監測項仍為綠
```

## 6. 密鑰位置

實值一律不進 repo。下表為規格書 §9 的位置對照，只記「住哪」，不記值。

| 值 | 實值住哪 | 進 repo |
|---|---|---|
| KUMA_PUSH_RS_PUBLISH | VM `/opt/RuleETLFlow/.env` | ❌ 實值／✅ 空範本 |
| KUMA_PUSH_PG_BACKUP | VM `/etc/cron.d/ruleplatform-pg-backup` 變數行 | ❌ 實值／✅ 步驟文件 |
| KUMA_PUSH_SCHEDULER_URL | VM `/opt/RulePlatform/.env`；本機 `RulePlatform/.env` | ❌ 實值／✅ 空範本 |
| Telegram bot token | Kuma volume `ruleops_kuma_data` 的 SQLite | ❌ |
| Telegram chat id | 同上 | ❌ |
| admin@ Gmail 應用程式密碼 | 同上＋使用者密碼管理器 | ❌ |
| Kuma admin 帳密 | 使用者密碼管理器 | ❌ |
| Kuma 2FA 回復碼 | 使用者密碼管理器 | ❌ |
| Push token（三組） | Kuma volume；並出現在三個 push URL 裡 | ❌ |
| Kuma volume 備份 tar | 本機 `RuleOps/sentinel/backups/` | ❌ |
| `simulate-push.sh`（含 token） | 本機 `RuleOps/sentinel/` | ❌ |
| compose 檔 | `RuleOps/sentinel/docker-compose.yml` | ✅ 全文（無密鑰） |

哨兵機的內網 IP 實值同樣不進 repo；本檔一律以 `<哨兵機內網 IP>` 佔位。

## 7. 監測項清單

六項主動探測（規格書 §4.1）。「目前狀態」待 0-B2 於 Kuma 建立後填。

| # | 名稱 | 類型 | 級別 | Stage | 目前狀態 |
|---|---|---|---|---|---|
| M1 | web-rulestox-home | HTTP(s) - Keyword | P0 | 0 | |
| M2 | api-health | HTTP(s) - Json Query | P0 | 0 | |
| M3 | 憑證到期（附掛 M1／M2） | Certificate Expiry | P1 | 0 | |
| M4 | dns-rulestox-a | DNS | P1 | 0 | |
| M5 | vm-ruleetl-ping | Ping | P0 | 1 | |
| M6 | airflow-scheduler | HTTP(s) - Json Query | P0 | 1 | |

三項 push deadman（P1 `deadman-rs-publish`／P2 `deadman-pg-backup`／P3 `deadman-scheduler`）的契約見規格書 §5，0-B2 起陸續落地。

## 8. 演練紀錄

AC#6 三情境（規格書 §10）。每次演練一列，附實測時間、判定與截圖檔名；未演練前留空。

| 日期 | Stage | 情境 | 實測時間 | 判定 | 截圖檔名 |
|---|---|---|---|---|---|
| | | F1 停 user 容器 → P0 告警 ≤3 分鐘 → 起回 → 復原通知 | | | |
| | | F2 跳過一次 push → 寬限期後告警 | | | |
| | | F3 停 Airflow scheduler 10 分鐘 → /health 監測告警 | | | |
