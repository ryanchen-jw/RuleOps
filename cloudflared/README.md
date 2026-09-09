# cloudflared

Stage 1 的 Cloudflare Tunnel 設定落點：哨兵機上 Kuma UI 對外的通道，以及其前置的 Access 保護。

Stage 0 不使用——本機 Kuma 直接開 `http://localhost:3001`，不建通道。

建 tunnel、Access policy 與 IAM 屬於執行長閘門項目（交接檔 §4），需使用者親自執行或 attach 放行；在那之前本目錄保持空的。
