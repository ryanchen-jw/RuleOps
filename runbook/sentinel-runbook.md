# 哨兵 Runbook（TASK-376 L1 檢測層）

> 骨架版（Stage 0-B1）。各章內容隨 0-B2～0-B5 落地陸續補上；未發生的事不預先填。
> 規格出處：`C:/Projects/RuleStox/.claude/daily-plan/sessions/TASK-376-l1/l1-spec.html`

## 1. 適用範圍

Stage 0＝使用者本機 Docker 的 Uptime Kuma，只對公開端點做唯讀 HTTP GET，不接 prod push。Stage 1 才遷哨兵 VM 並接線 prod 排程。

## 2. 建置與首次設定

規格書 §3.3 寫「首次進入是建立管理員畫面」——那是 Uptime Kuma 1.x 的行為。**2.5.3 在建立管理員之前多一個 `/setup-database` 選資料庫畫面**（2026-09-09 實測，`GET /` 會 302 到 `/setup-database`）。首次設定的實際順序：

| # | 步驟 | 要點 |
|---|---|---|
| 0 | `http://localhost:3001` → **Setup Database** 畫面選 **SQLite** | 三個選項是 SQLite／MariaDB／Embedded MariaDB（本 image `isEnabledEmbeddedMariaDB: true`，三個都會出現）。**必須選 SQLite**——第 4、5 章的備份與升級程序建立在單一 SQLite 檔 `/app/data/kuma.db` 上，選了任一種 MariaDB 那兩章整章失效 |
| 1 | 建立管理員（實際採用 `rulestox`，規格書 §3.3 寫的是 `rulestox-admin`）＋ ≥20 字元隨機密碼 | Kuma 沒有忘記密碼流程，遺失只能進容器改 DB；密碼存密碼管理器 |
| 2 | Profile → Security → Two Factor Authentication → Enable | TOTP 掃 QR ＋ 輸入 6 碼；回復碼截圖存密碼管理器 |
| 3 | Settings → General | 顯示時區 ＝ `Asia/Taipei`（**要顯式選，別留 `自動`**）；伺服器時區 ＝ `Asia/Taipei`；搜尋引擎可見度 ＝ `不建議索引`。注意顯示時區存在瀏覽器 `localStorage`、不在 DB，查證要看瀏覽器而非 `setting` 表；伺服器時區才是 `setting.serverTimezone` |
| 4 | Settings → Notifications → Certificate Expiry | 天數清單設 `14`、`7`、`3` |
| 5 | 建 EMAIL-P0 通知、依第 7 章建監測項 | 設定值見規格書 §8.3／§4.2 |

DB 選定後檔案落在容器內 `/app/data/kuma.db`（＝ volume `ruleops_kuma_data`），對應 `server/database.js:139`。

> **Windows／Git Bash 注意**：`docker exec` 帶容器內絕對路徑時會被 MSYS 路徑轉換改寫成 `C:/Program Files/Git/...`。本檔所有 `docker exec`／`docker run` 指令請在前面加 `MSYS_NO_PATHCONV=1`，或改在 PowerShell 執行。

## 3. 啟停

所有指令的工作目錄一律是 **RuleOps repo 根目錄**（`C:/Projects/RuleStox/RuleOps`），
compose 檔以相對路徑 `sentinel/docker-compose.yml` 指定。compose project 名寫死在檔案裡
（`name: ruleops-kuma`），所以不必額外帶 `-p`。

```bash
cd C:/Projects/RuleStox/RuleOps

# 啟動
docker compose -f sentinel/docker-compose.yml up -d
# 停止（volume 保留）
docker compose -f sentinel/docker-compose.yml down
```

## 4. 備份與還原

SQLite 熱備份會拿到半截檔，必須先停容器再打包 volume。

> ✅ **本章指令於 2026-09-09 實跑驗證通過**（存證 `evidence-0909-task376/backup-restore.txt`）。
> 規格書 §3.4 的原文有兩個問題，以下已修正：
> ① 註解寫「在 `RuleOps/sentinel/` 下執行」卻又用 `-f RuleOps/sentinel/...` 的路徑，兩者不自洽；
>    本章統一為 **cwd ＝ RuleOps repo 根**、`-f sentinel/docker-compose.yml`。
> ② `-v "$PWD:/backup"` 在 Git Bash 下會被 MSYS 路徑轉換咬掉（容器內路徑 `/data`、`/backup`
>    會被改寫成 `C:/Program Files/Git/...`）。必須加 `MSYS_NO_PATHCONV=1`，且主機側路徑用
>    `$(pwd -W)` 取 Windows 形式。**PowerShell 沒有這個問題**，可直接用 `${PWD}`。

### 備份

```bash
cd C:/Projects/RuleStox/RuleOps
mkdir -p sentinel/backups

docker compose -f sentinel/docker-compose.yml stop

MSYS_NO_PATHCONV=1 docker run --rm \
  -v ruleops_kuma_data:/data \
  -v "$(pwd -W)/sentinel/backups:/backup" \
  alpine tar czf "/backup/kuma-backup-$(date +%Y%m%d-%H%M%S).tar.gz" -C /data .

docker compose -f sentinel/docker-compose.yml start
```

實跑數據：整包 tar 約 **57 KB**、停機時間 **2 秒**（17:45:12 → 17:45:14）。
封存內容共 6 項：`./`、`./kuma.db`、`./db-config.json`、`./screenshots/`、`./upload/`、`./docker-tls/`。

打包完先驗封存可讀，再相信它：

```bash
MSYS_NO_PATHCONV=1 docker run --rm -v "$(pwd -W)/sentinel/backups:/backup" \
  alpine tar tzf /backup/<檔名>.tar.gz
```

### 還原

正式還原（**破壞性**：會砍掉現役 volume，執行前先確認手上的 tar 真的可讀）：

```bash
cd C:/Projects/RuleStox/RuleOps

docker compose -f sentinel/docker-compose.yml down
docker volume rm ruleops_kuma_data
docker volume create ruleops_kuma_data

MSYS_NO_PATHCONV=1 docker run --rm \
  -v ruleops_kuma_data:/data \
  -v "$(pwd -W)/sentinel/backups:/backup" \
  alpine tar xzf /backup/<檔名>.tar.gz -C /data

docker compose -f sentinel/docker-compose.yml up -d
```

> **驗證方式（建議在正式還原前先做一次）**：把同一份 tar 還原到拋棄式 volume、用臨時容器
> 開在另一個 port 確認資料完整，確認無誤再動現役那顆。2026-09-09 即以此法驗證，
> 沒有把當天剛建好的設定押在一次還原上。
>
> ```bash
> docker volume create ruleops_kuma_data_restoretest
> MSYS_NO_PATHCONV=1 docker run --rm \
>   -v ruleops_kuma_data_restoretest:/data \
>   -v "$(pwd -W)/sentinel/backups:/backup" \
>   alpine tar xzf /backup/<檔名>.tar.gz -C /data
>
> MSYS_NO_PATHCONV=1 docker run -d --name ruleops_kuma_restoretest \
>   --add-host host.docker.internal:host-gateway -e TZ=Asia/Taipei \
>   -p 3002:3001 -v ruleops_kuma_data_restoretest:/app/data \
>   louislam/uptime-kuma:2.5.3
>
> # 比對筆數（三張表都要一致）
> MSYS_NO_PATHCONV=1 docker exec ruleops_kuma_restoretest \
>   sqlite3 -readonly /app/data/kuma.db \
>   "SELECT (SELECT COUNT(*) FROM monitor),(SELECT COUNT(*) FROM notification),(SELECT COUNT(*) FROM monitor_notification);"
>
> docker rm -f ruleops_kuma_restoretest
> docker volume rm ruleops_kuma_data_restoretest
> ```
>
> 2026-09-09 實測結果：`monitor 8/8`、`notification 1/1`、`monitor_notification 8/8` 全數一致，
> 臨時容器啟動日誌無 migration 失敗，port 3002 回 302（＝ Kuma 正常起來、導向登入頁）。

備份檔含 SMTP 應用程式密碼與三組 push token，放本機 `sentinel/backups/`，已在 `.gitignore` 排除。
備份頻率：Stage 0 期間「每次改完監測項設定手動跑一次」即可。

## 5. 升級

Kuma 啟動時自動跑 DB migration，不可降版（新版 schema 舊版讀不了）；升級失敗的唯一退路是第 4 章的還原。

> 路徑已配合第 3、4 章統一為 cwd ＝ RuleOps repo 根。`pull` / `up -d` 兩步與 0-B2 首次
> `up -d` 走的是同一條路徑，該次實跑成功（拉 `2.5.3`、建 volume、容器 healthy）。
> 真正的換版尚未發生過，換版當天請照第 4 章先備份。

```bash
cd C:/Projects/RuleStox/RuleOps

# 1. 先備份（第 4 章）——這一步不可略過，升級失敗只有還原一條路
# 2. 改 sentinel/docker-compose.yml 的 image tag（例：2.5.3 → 2.6.0）
# 3. 拉取並重建
docker compose -f sentinel/docker-compose.yml pull
docker compose -f sentinel/docker-compose.yml up -d

# 4. 確認版本與健康狀態
docker exec ruleops_kuma node -e 'console.log(require("/app/package.json").version)'
docker ps --filter name=ruleops_kuma --format '{{.Status}}'
# 5. 開 UI 確認全部監測項仍為綠
```

> image tag 釘死版本（不用浮動 tag）的理由見 `sentinel/docker-compose.yml` 檔頭：
> 哨兵換版失敗時你不會收到告警——因為壞掉的正是告警系統本身。

## 6. 密鑰位置

實值一律不進 repo。下表為規格書 §9 的位置對照，只記「住哪」，不記值。

| 值 | 實值住哪 | 進 repo |
|---|---|---|
| KUMA_PUSH_RS_PUBLISH | VM `/opt/RuleETLFlow/.env` | ❌ 實值／✅ 空範本 |
| KUMA_PUSH_PG_BACKUP | VM `/etc/cron.d/ruleplatform-pg-backup` 變數行 | ❌ 實值／✅ 步驟文件 |
| KUMA_PUSH_SCHEDULER_URL | VM `/opt/RulePlatform/.env`；本機 `RulePlatform/.env` | ❌ 實值／✅ 空範本 |
| ~~Telegram bot token~~ | — | **不適用**：user 2026-09-09 裁示不做 Telegram |
| ~~Telegram chat id~~ | — | **不適用**：同上 |
| admin@ Gmail 應用程式密碼 | Kuma volume `ruleops_kuma_data` 的 SQLite（`notification.config`）＋使用者密碼管理器 | ❌ |
| Kuma admin 帳密 | 使用者密碼管理器 | ❌ |
| Kuma 2FA 回復碼 | 使用者密碼管理器 | ❌ |
| Push token（三組） | Kuma volume；並出現在三個 push URL 裡 | ❌ |
| Kuma volume 備份 tar | 本機 `RuleOps/sentinel/backups/` | ❌ |
| `simulate-push.sh`（含 token） | 本機 `RuleOps/sentinel/` | ❌ |
| compose 檔 | `RuleOps/sentinel/docker-compose.yml` | ✅ 全文（無密鑰） |
| 監測項設定匯出 | `RuleOps/sentinel/kuma-monitors.json` | ✅ 全文（**已去密**，見第 7 章） |

哨兵機的內網 IP 實值同樣不進 repo；本檔一律以 `<哨兵機內網 IP>` 佔位。

## 7. 監測項清單

六項主動探測（規格書 §4.1）＋三項 push deadman（契約見規格書 §5）＋一項演練用臨時項。

全部九項已於 2026-09-09（Stage 0-B2）建立完成，逐欄設定值與規格書 §4.2 比對 92/92 相符
（存證 `evidence-0909-task376/s2-monitor-config-diff.txt`）。F1 演練用的臨時項
`api-health-local`（id 9）已於演練結束後刪除，現存 8 項：

| # | 名稱 | Kuma id | 類型 | 級別 | Stage | 目前狀態 |
|---|---|---|---|---|---|---|
| M1 | web-rulestox-home | 1 | HTTP(s) - Keyword | P0 | 0 | ✅ 運行中・UP |
| M2 | api-health | 2 | HTTP(s) - Json Query | P0 | 0 | ✅ 運行中・UP |
| M3 | 憑證到期（附掛 M1／M2） | — | Certificate Expiry | P1 | 0 | ✅ M1／M2 皆已勾選；全域門檻 14／7／3 |
| M4 | dns-rulestox-a | 3 | DNS | P1 | 0 | ✅ 運行中・UP |
| M5 | vm-ruleetl-ping | 7 | Ping | P0 | 1 | ⏸ Pause（目標為 placeholder `10.0.0.0`） |
| M6 | airflow-scheduler | 8 | HTTP(s) - Json Query | P0 | 1 | ⏸ Pause（目標為 placeholder `http://10.0.0.0:8080/health`） |
| P1 | deadman-rs-publish | 4 | Push | P0 | 0 模擬 | ✅ 運行中・UP（間隔 86400／重試 3×1800） |
| P2 | deadman-pg-backup | 5 | Push | P1 | 0 模擬 | ✅ 運行中・UP（間隔 86400／重試 2×1800） |
| P3 | deadman-scheduler | 6 | Push | P0 | 0 | ⏸ Pause（間隔 300／重試 1×300），F2 演練後改回 |

八項全部綁 EMAIL-P0（`monitor_notification` 8 筆）。

### 設定匯出：`sentinel/kuma-monitors.json`

Kuma 2.5.3 **已移除** Settings → Backup 的匯出／匯入（前端路由表沒有 `backup`、沒有 Backup 元件，
後端 `socket-handlers/` 也沒有對應 handler；bundle 裡殘留的 `uploadBackup` 是沒人呼叫的死程式碼）。
因此匯出改由 SQLite 白名單欄位產生：

```bash
docker exec ruleops_kuma sqlite3 -readonly -json /app/data/kuma.db "SELECT ... FROM monitor;"
```

該檔**是稽核與設定基準，不是還原用的備份**——`push_token` 與 `notification.config`
（含 SMTP 應用程式密碼）刻意不匯出，所以無法單獨還原成可運作的哨兵。真正的還原路徑是第 4 章的 volume tar。

去密自檢（2026-09-09，存證 `evidence-0909-task376/s3-export-secret-check.txt`）：
三組 push token 實值以 `grep -F` 逐一比對 → 全部未出現；19 個憑證欄位無一以 JSON key 形式匯出；
32 碼英數字串命中 0 筆；SMTP 主機與 email 位址命中 0 筆；`notification` 只留
`{"id":1,"name":"EMAIL-P0","active":1,"is_default":0}`。

**M5／M6 的 placeholder**：Stage 1 上線前必須把 `10.0.0.0` 換成 ruleetl-vm 的 VPC 內網 IP
（取值指令見規格書 §4.2），換完才 Resume。目前 Pause 是為了不讓打不到的目標產生噪音。

**P3 為什麼暫停**：P3 的來源是 RulePlatform 的 APScheduler heartbeat，那條線要到 0-B4 才接。
在接線之前 P3 每 10 分鐘就會轉 DOWN 一次、且 Resend 設 10，等於每 50 分鐘寄一封誤報。
**0-B4 heartbeat 接線後再 Resume**。F2 演練期間會臨時 Resume，演練結束立即改回 Pause。

### Push token 與規格書 §4.2 的出入

規格書 §4.2 要求 push token 命名為 `rspub`／`pgbak`／`schbt` ＋ 隨機碼。**Kuma 2.5.3 做不到**：
新增 Push 監測項時，UI 只給一個 `disabled` 的「Push 網址」欄位與一顆「重設權杖」按鈕，
沒有可編輯的 token 輸入格（`monitor.push_token` 是 `varchar(32)`，由後端產生）。
要自訂前綴只能直改 SQLite 再重啟容器，代價高於「好認」這點效益，故**沿用 Kuma 自動產生的 token**。
三組 token 已寫入本機 `sentinel/simulate-push.sh`（已 gitignore），該檔以名稱 `rspub`／`pgbak`／`schbt`
對應三個監測項，人肉操作介面與規格書一致。

### 通知綁定

全部監測項一律綁 **EMAIL-P0**（user 2026-09-09 裁示：只用 email，Telegram 整段不做，
規格書 §8.1／§8.2／TG-P0 段落作廢）。綁定方式是建 EMAIL-P0 時勾「套用到目前所有的監測器」，
一次綁完；「啟用預設」維持關閉（新監測項的通知要一項一項想清楚，沿用規格書 §8.3 的紀律）。
抖動防護仍靠各監測項的 Retries，P0 級 Resend ＝ 10、P1 級 ＝ 0（§4.2 已逐項設妥）。

### M2 的 `Headers` 為什麼填 curl UA（2026-09-09 實測修正）

規格書 §4.2 的口徑是「Cloudflare 擋非 curl 的 User-Agent、回 403 error code 1010」。實測結果**不是這樣**：CF 只擋 `Python-urllib/*` 這一個簽名，其餘一律放行。

| 送出的 User-Agent | `https://api.rulestox.com/health` |
|---|---|
| `curl/8.4.0`、空 UA 標頭 | 200 |
| `Uptime-Kuma/2.5.3`、`Uptime-Kuma/1.23.13` | 200 |
| `axios/1.7.7`、`python-httpx/0.27.0`、`Go-http-client/2.0`、Chrome UA | 200 |
| `Python-urllib/3.11`、`Python-urllib/3.12` | **403 `error code: 1010`** |

Kuma 自己送的 UA 是 `Uptime-Kuma/<版本>`（`server/uptime-kuma-server.js`），也就是上表回 200 的那一個——**M2 的 `Headers` 留空也會通**。仍照填 `{"User-Agent": "curl/8.4.0"}` 是執行長 2026-09-09 的裁決，作用是保險（CF 規則若變動時多一層緩衝），**不是硬性依賴**。日後若這格被清空而 M2 仍為綠，屬正常，不必回填。

連帶結論（供 0-B4 接線用）：對 `api.rulestox.com` 發請求時，`httpx` 可通、`urllib` 會被 CF 擋掉——
RulePlatform user 容器無 curl，若要打自家 API 必須走 httpx。

> ⚠️ **適用範圍**：上表與這條結論**只對 `api.rulestox.com`（Cloudflare 後面）成立**。
> push deadman 打的是哨兵機自己的 Kuma（Stage 0 是 `http://localhost:3001`，Stage 1 是內網位址），
> **不經過 Cloudflare**，因此 UA 完全不影響 push——用 urllib 打 push URL 也會成功。
> 0-B4 選 httpx 的真正理由是 user 容器裡沒有 curl、而 httpx 已是既有相依，不是為了繞 CF。

存證：`.claude/daily-plan/sessions/evidence-0909-task376/s1-ua-matrix.txt`（不在 repo 內）。

## 8. 演練紀錄

AC#6 三情境（規格書 §10）。每次演練一列，附實測時間、判定與截圖檔名。

| 日期 | Stage | 情境 | 實測時間 | 判定 | 存證檔名 |
|---|---|---|---|---|---|
| 2026-09-09 | 0（本機等價版） | F1 停服務 → P0 告警 → 起回 → 復原通知 | DOWN **87 s**（門檻 180 s）／UP **55 s**（預期 ≤60 s） | ✅ 通過 | `f1-drill.txt`、`f1-01-kuma-events.png` |
| 2026-09-09 | 0 | F2 跳過一次 push → 寬限期後告警 | PENDING **T+302 s**（不發通知）／DOWN **T+602 s**（預期 600±60 s）／補打後 **0 s** 復原 | ✅ 通過 | `f2-drill.txt`、`f2-01-kuma-events-p3.png` |
| — | 1 | F3 停 Airflow scheduler 10 分鐘 → /health 監測告警 | 未執行 | ⏳ 留 Stage 1 | — |

存證位於 `C:/Projects/RuleStox/.claude/daily-plan/sessions/evidence-0909-task376/`（不在 repo 內）。

### F1 · 本機等價版做了什麼、為什麼

規格書 §10.1 的 Stage 0 版寫「停本機 dev user 容器」。**本次沒有這樣做**——當天
`ruleplatform_user` 等六個既有容器都在服役（其中還有其他任務的測試在跑），停任何一個都會波及別人。
改為**另起一個拋棄式容器當演練對象**：`ruleops_drill_target`（`python:3.12-slim`，主機 port 8899，
回 `{"status":"healthy",...}`），Kuma 端建臨時監測項 `api-health-local` 打
`http://host.docker.internal:8899/health`，其餘設定同 M2、Headers 留空。演練即
`docker stop` / `docker start` 這個拋棄式容器，結束後 `docker rm` 並刪掉臨時監測項。

實測心跳序列（UTC；台北時間 ＝ +8）：

```
09:30:15  UP       important=0
09:30:28  ← docker stop ruleops_drill_target
09:31:15  PENDING  important=0   connect ECONNREFUSED   ← 第 1 次失敗，不發通知
09:31:35  PENDING  important=0   connect ECONNREFUSED   ← 第 2 次失敗，不發通知
09:31:55  DOWN     important=1   connect ECONNREFUSED   ← 第 3 次失敗 → 判 DOWN、發通知
09:32:01  ← docker start ruleops_drill_target
09:32:55  UP       important=1                          ← 復原、發通知
```

### F2 · 為什麼用 P3 而不是規格書 §10.2 的時間壓縮法

規格書 §10.2 要把 P1 的間隔暫改 120 s、重試間隔 60 s，驗完再改回 86400／1800，
並自陳「漏改回＝哨兵天天誤報」。**P3 `deadman-scheduler` 的原生設定（間隔 300 s、
重試 1×300 s）本身就是 5 分寬限、10 分告警**，不必改任何一格就能在十分鐘內跑完整條鏈路，
也就沒有漏改回的風險。執行長 2026-09-09 裁決採此法（Q2-A）。

實測心跳序列（UTC）：

```
09:33:46  UP       important=0   stage0_sim_schbt        ← 基準 push
09:38:48  PENDING  important=0   No heartbeat in the time window   ← T+302 s，**不發通知**
09:43:48  DOWN     important=1   No heartbeat in the time window   ← T+602 s，發通知
09:44:02  UP       important=1   stage0_sim_schbt        ← 補打，即時復原、發通知
```

**抖動防護的判定依據**：Kuma 只在 `isImportantForNotification()` 為真時送通知
（`server/model/monitor.js:1420-1445`）。該函式的完整條件是：

```
isFirstBeat                        ← 監測項的「第一拍」，不論狀態一律送
| MAINTENANCE -> DOWN
| UP          -> DOWN
| DOWN        -> UP
| PENDING     -> DOWN
```

`UP -> PENDING` **不在清單裡**，所以轉 PENDING 不會叫人——這就是抖動防護的實作位置。
F1 與 F2 兩次演練的 PENDING 心跳全部是 `important = 0`，實證相符：

```sql
-- 兩個演練對象的 PENDING 心跳，沒有任何一筆是 important=1
-- （注意：api-health-local 已於演練後刪除，其心跳連帶被清掉，本查詢現在實際只涵蓋
--   deadman-scheduler；F1 那一半的證據在 evidence 的 f1-drill.txt 心跳序列裡）
SELECT COUNT(*) FROM heartbeat
 WHERE status = 2 AND important = 1
   AND monitor_id IN (SELECT id FROM monitor WHERE name IN ('api-health-local','deadman-scheduler'));
-- → 0
```

> ⚠️ **會誤判的自檢寫法**：如果把上面那句的 `monitor_id IN (...)` 拿掉、改成全庫掃描，
> 2026-09-09 這台機器會回 **2** 而不是 0。那兩筆是 M5 `vm-ruleetl-ping` 與 M6
> `airflow-scheduler` **建立當下的第一拍**（08:26:57、08:27:06 UTC）——它們的目標是
> placeholder `10.0.0.0`，第一拍就打不通而落在 PENDING，而 Kuma 把「第一拍」無條件標成
> important。這不是抖動防護失效。

### 由此得到的操作紀律（Stage 1 建新監測項時要記得）

因為 `isFirstBeat` 會無條件觸發通知，**在已綁定通知的情況下新建一個目標打不通的監測項，
會立刻寄出一封告警信**。M5／M6 這次沒有寄信純粹是時序上的僥倖：它們建於 16:26，
而 EMAIL-P0 到 17:26 才存檔。Stage 1 換上 ruleetl-vm 真 IP、或日後新增任何監測項時，
安全順序是 **先確認目標打得通，再建監測項**；若無法先確認，就先建成 Pause、
確認目標可達後再 Resume（Resume 不算第一拍，不會觸發通知）。

收尾自檢：P3 演練後已改回 Pause，且 `interval=300 maxretries=1 retry_interval=300
resend_interval=10` 一格未動——本演練全程沒有更動任何設定值，故不存在「改回 86400／1800」這個收尾項。
