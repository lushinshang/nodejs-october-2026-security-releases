# Node.js 10 月安全更新導讀：10 個 CVE、Brotli 的 RCE 與 eMudhra 憑證

以 nodejs.org 2026-10-21 官方公告為主軸，整理 Node.js v22.23.4、v24.22.1、v26.12.1 修補的 10 個 CVE（含 Brotli decoder 越界寫入 CVE-2026-111497），對照上游修補、CERT 與廠商說法，並說明同批更新中的 eMudhra 憑證封鎖。每個說法標明誰說、截至何時；資料截至 2026-10-22 07:14 UTC（台灣 15:14）。

## 200字介紹

Node.js 在 2026-10-21 發布安全更新，修補 10 個 CVE（4 個 high、3 個 medium、3 個 low），其中 Brotli decoder 越界寫入被官方標為可導致遠端程式碼執行。官方公告沒有給 CVSS、利用條件，也沒有提到在野利用；外界則出現 critical、已有 PoC、與 PHP 事件混為一談等說法。本文逐項整理 10 個 CVE 與版本線差異、上游 google/brotli v1.2.1 的修補、同批依賴更新中的 eMudhra 中繼憑證封鎖，把各方說法與官方公告對照，並給出開發維運、企業 IT、資安同業三層行動清單。

## 檔案

| 檔案 | 說明 |
|---|---|
| `index.html` | 發布網頁本體 |
| `nodejs-october-2026-security-releases.md` | 導讀 Markdown 原稿 |
| `images/` | 全貌資訊圖（16:9、9:16 兩版） |
| `share_post.md` | 社群分享文 |

## 原始素材

- 官方公告：https://nodejs.org/en/blog/vulnerability/october-2026-security-releases
- 逐字稿：不適用

## 查證來源（重點）

- Node.js release notes（v22.23.4、v24.22.1、v26.12.1）
- google/brotli v1.2.1：https://github.com/google/brotli/releases/tag/v1.2.1
- Google、Mozilla、Node PR #66871（eMudhra）
- HKCERT 公告、Vercel 公告（廠商自述）

## 狀態

已發布：https://lushinshang.github.io/nodejs-october-2026-security-releases/
