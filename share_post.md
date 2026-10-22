Node.js 10/21 安全更新：10 個 CVE，4 個 high，其中 Brotli decoder 越界寫入官方標題寫「可導致遠端程式碼執行」（CVE-2026-111497）。

官方公告沒有給利用條件，也沒提在野利用；外界有說 critical、有廠商說已有研究者示範，目前都沒有第二個來源確認。同批更新還含 eMudhra 中繼憑證的封鎖清單。

修正版：v22.23.4、v24.22.1、v26.12.1。升級、重啟、容器與打包產物重建。

完整導讀：https://lushinshang.github.io/nodejs-october-2026-security-releases/
