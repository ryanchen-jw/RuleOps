# RuleOps

RuleStox 的維運執行底座（哨兵、通道、角色身分、runbook）。**尚未正式運作**：目前停在 TASK-376 Stage 0（只在使用者本機 Docker 驗證，不接 prod、不動線上服務）。

## 必知

- 分工原則、目錄說明、密鑰落點：見 `README.md`（本檔不重複）
- 只放執行底座；業務內容與業務碼留在各自 repo（例：RulePlatform `/health`、RuleETLFlow DAG 鏈尾 ping）
- 密鑰實值一律不進 repo；key 名索引見 `.env.example`
- 哨兵操作手冊：`runbook/sentinel-runbook.md`
- 文件放哪裡：照 workspace `CLAUDE.md`「文件放哪裡」（本 repo 長期設計依據放 `docs/design/`、驗收證據放 `docs/verification/`、操作手冊放 `runbook/`）
