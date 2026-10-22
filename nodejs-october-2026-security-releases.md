---
title: Node.js 10 月安全更新導讀：10 個 CVE、Brotli 的 RCE 與 eMudhra 憑證，你的服務該查什麼
subtitle: 以 nodejs.org 2026-10-21 官方公告為主軸，對照上游修補、各國機關與廠商的說法，整理升級與暫行措施
date: 2026-10-22
language: zh-Hant-TW
sources:
  - type: 官方公告
    title: "Wednesday, October 21, 2026 Security Releases（nodejs.org）"
    url: https://nodejs.org/en/blog/vulnerability/october-2026-security-releases
  - type: 官方 release 說明
    title: Node.js v26.12.1 release notes
    url: https://nodejs.org/en/blog/release/v26.12.1/
  - type: 上游修補
    title: google/brotli v1.2.1
    url: https://github.com/google/brotli/releases/tag/v1.2.1
---

# Node.js 10 月安全更新導讀：10 個 CVE、Brotli 的 RCE 與 eMudhra 憑證，你的服務該查什麼

> 以 nodejs.org 2026-10-21 官方公告為主軸，對照上游修補、各國機關與廠商的說法，整理升級與暫行措施。
>
> 時間一律先寫 UTC，括號內是台灣時間（UTC+8）。本文資料截至 2026-10-22 07:14 UTC（台灣 15:14），之後的變化沒有涵蓋。

## 三十秒版

| 項目 | 內容 |
|---|---|
| 修正版本 | Node.js v22.23.4、v24.22.1、v26.12.1（2026-10-21 發布），見[官方公告](https://nodejs.org/en/blog/vulnerability/october-2026-security-releases) |
| 漏洞數量 | 10 個 CVE：4 個 high、3 個 medium、3 個 low；官方寫明最高嚴重度為 HIGH |
| 最受矚目的一項 | CVE-2026-111497：Brotli decoder 越界寫入，官方標題寫「permits remote code execution」，影響 26.x、24.x、22.x |
| 另外三個 high | `dns.resolve` 遇到 NUL 字元會截斷名稱（CVE-2026-105482），以及兩個 TLS 憑證主機名稱驗證可被繞過的問題（CVE-2026-105483、CVE-2026-106069） |
| 依賴更新 | c-ares、ca-cert（eMudhra 中繼 CA 事件）、OpenSSL、SQLite、Undici |
| 官方沒說的 | 沒有 CVSS 分數、沒有利用條件，也沒有提到在野利用 |
| 今天要做的事 | 查你實際跑的 `node -v`，升到修正版，重啟所有行程，容器映像與打包產物重建 |

官方公告沒有寫利用條件；外界則出現了各種說法：有人把它跟同一天修補的 PHP 事件放在一起講，有廠商說已有研究者示範過，也有媒體把 Brotli 那一項寫成 critical。本文要回答的是：官方到底說了什麼，各方說法哪些對得上、哪些要打折。

做法跟之前的 [PHP2Shell 導讀](https://lushinshang.github.io/php2shell-cve-2026-109618/)一樣：以官方公告為準，其他說法都標明是誰說的、截至什麼時候。

## 一、事件時間線

先把順序排清楚。這次有兩條線同時進行：Node 自己的 10 個 CVE，以及一個從 10/17 開始的憑證事件。

| UTC | 台灣時間 | 事件 | 來源 |
|---|---|---|---|
| 10/17 18:05 | 10/18 02:05 | Google 收到通報：8 張 eMudhra 中繼 CA 憑證的私鑰可被還原 | [Google 部落格](https://blog.google/security/protecting-chrome-and-android-users-from-compromised-emudhra-intermediate-ca-certificates/) |
| 10/17 19:45 | 10/18 03:45 | Chrome 以 CRLSet 依公鑰雜湊封鎖受影響的中繼憑證 | [Google 部落格](https://blog.google/security/protecting-chrome-and-android-users-from-compromised-emudhra-intermediate-ca-certificates/) |
| 10/20（來源只寫日期） | — | Go 先行修補（CVE-2026-111943）；Node 的封鎖清單 PR #66871 合併 | [The Hacker News](https://thehackernews.com/2026/10/eight-emudhra-ca-keys-compromised.html)、[Node PR #66871](https://github.com/nodejs/node/pull/66871) |
| 10/21 00:46 | 10/21 08:46 | BleepingComputer 報導 Mozilla、Google、Apple 封鎖，軟體表中 Node.js 狀態為「Pending」 | [BleepingComputer](https://www.bleepingcomputer.com/news/security/mozilla-google-apple-block-compromised-emudhra-ca-certificates/) |
| 10/21 10:32–10:36 | 10/21 18:32–18:36 | 三個修正版的 SHASUMS256.txt 在 nodejs.org/dist 的檔案時間（公告本身只寫日期，沒有時分） | [nodejs.org/dist](https://nodejs.org/dist/) |
| 10/21 17:36 | 10/22 01:36 | 一位 Mastodon 使用者稱以 Express 應用測試 Brotli 輸入，行程被 SIGBUS 終止（崩潰，未證明是 RCE） | Mastodon 貼文 |
| 10/21 19:27 | 10/22 03:27 | 上游 google/brotli 發布 v1.2.1，release notes 寫明 CVE-2026-111497 | [google/brotli v1.2.1](https://github.com/google/brotli/releases/tag/v1.2.1) |
| —（來源只寫日期） | 10/22 | 香港 HKCERT 發布公告，風險等級列 Medium | [HKCERT](https://www.hkcert.org/security-bulletin/node-js-multiple-vulnerabilities_20261022) |

官方公告頁沒有時間標籤，上表 10:32–10:36 UTC 是下載目錄的檔案時間，只能當線索；`dist/index.json` 對 v22.23.4、v24.22.1 記的日期還是 10/20，各處時間來源並不一致。

## 二、10 個 CVE 逐項看

下表的嚴重度、影響版本線都照官方公告。「說明」欄是官方標題的意譯，官方沒有寫的機制與利用條件，本文也不推測。

| CVE | 嚴重度 | 影響版本線 | 說明（官方標題意譯） |
|---|---|---|---|
| CVE-2026-111497 | high | 26／24／22 | Brotli decoder 越界寫入，可導致遠端程式碼執行 |
| CVE-2026-105482 | high | 26／24／22 | `dns.resolve` 遇到含 NUL 字元的名稱會截斷 |
| CVE-2026-105483 | high | 26／24 | IP 以全形數字書寫時，TLS 憑證主機名稱驗證可被繞過 |
| CVE-2026-106069 | high | 26／24／22 | 主機名稱被「寬鬆解析」成 IP 時，TLS 憑證主機名稱驗證可被繞過 |
| CVE-2026-105481 | medium | 26／24／22 | keep-alive socket 處理了在請求送出前就寫入的回應 |
| CVE-2026-106072 | medium | 26／24／22 | 非標準寫法的 IPv4 主機名稱，讓 DNS SAN 能驗證 IP 端點 |
| CVE-2026-106074 | medium | 26／24／22 | `crypto.randomUUID()` 與 `randomInt()` 的 entropy cache 被存進 startup snapshot 與 SEA，每個行程重複相同的值 |
| CVE-2026-106070 | low | 26 | ZipFile 在 header 驗證失敗後仍回傳攻擊者控制的前綴位元組 |
| CVE-2026-106071 | low | 26 | Permission Model 因萬用字元 radix 查詢而多給檔案系統權限 |
| CVE-2026-106073 | low | 26／24／22 | Permission Model 可透過 `fs.realpathSync` 繞過 |

表中 CVE-2026-106071 的名稱與版本線依安全公告；release 頁沒有直接標示它對應的 commit，拿 release 頁反查會找不到明確標記。

幾個容易看漏的地方：

- **版本線不同**。105483 只影響 26.x 與 24.x；106070、106071 只影響 26.x。你跑 22.x 的話，這三個不用擔心。
- **TLS 主機名稱這一組有三個**（105483、106069、106072）。官方標題都跟 IP 與主機名稱的比對有關，細節沒有展開。106069 與 106072 共用一個修補（`tls: reject non-canonical IP addresses`），105483 是另一個（`tls: fix ip normalization regression`）。
- **106074 可能要重建產物**。這項講的是 snapshot 與 SEA（單一可執行檔應用）裡被存進去的隨機數快取，依 [XenoSpectrum](https://xenospectrum.com/en/nodejs-october-security-tls-snapshot/) 的說法，SEA 與 snapshot 需要重建。如果你的產物是用舊版 Node 打包出來的，升級後用新版重新打包。

## 三、Brotli：官方說了什麼，沒說什麼

CVE-2026-111497 是這次最多人提的一項，因為標題直接寫 remote code execution。

**官方說了**：這是 Node.js Brotli decoder 的越界寫入，可導致遠端程式碼執行，影響 26.x、24.x、22.x，嚴重度 high。回報者是 mcollina，修正者是 aduh95。修補內容是 cherry-pick 上游的一個 commit（`0ed4268a…`，訊息是 Detect and report invalid stream earlier），commit 訊息帶有這個 CVE 編號。

**官方沒說**：攻擊者要怎麼把資料送進 decoder、需要多大的資料、有沒有已知的利用。公告頁沒有 CVSS，也沒有任何一句談利用條件。

**上游**：google/brotli 在 10/21 19:27 UTC（台灣 10/22 03:27）發布 v1.2.1，release notes 的安全欄寫著「CVE-2026-111497: out-of-bounds write permits remote code execution」。有一篇詳細的分析文章（Atlas，10/21 20:45 UTC，台灣 10/22 04:45）寫「上游還沒有打包含修補的 tag」，GitHub 顯示 v1.2.1 的建立時間早於該文標示的發布時間，目前頁面也不符這句話（release notes 的編修歷史未核對）。

### 外界對利用情況的說法

下表記錄外界對利用情況的說法與一則實測，彼此層級不同，目前都沒有第二個來源確認。

| 誰說 | 說了什麼 | 怎麼看 |
|---|---|---|
| [Vercel](https://vercel.com/kb/bulletin/node-js-security-advisory) | 「Researchers demonstrated process-memory disclosure and command execution」，並稱已部署 WAF 規則 | 沒有給出處、沒有具名研究者；官方公告沒有這句。這是 Vercel 自己的陳述 |
| 一位 Mastodon 使用者（10/21 17:36 UTC，台灣 10/22 01:36） | 稱以 Express 應用測試 Brotli 輸入，行程被 SIGBUS 終止 | 描述的是行程崩潰，不是程式碼執行；貼文也沒有證明就是 CVE-2026-111497 |
| 一位日本作者的實測（lilting.ch） | 用修補 commit 的測試資料餵新舊版本，兩者都只回報錯誤，作者明說這樣無法判斷越界寫入是否發生 | 屬於「沒能重現」，不能當成沒有風險的證據 |

另外，iThome、SecurityOnline、Cyber Updates 365、cybernote 等報導都明說官方沒有提到在野利用，CISA KEV 在 10/21 14:04 UTC（台灣 22:04）的版本也不含這 10 個 CVE。這個時點早於多數事件，只能當參考。

結論：**截至本文時點，沒有查到任何來源聲稱在野利用；利用細節沒有公開確認。** 這不等於沒有風險——官方標題就寫了 RCE，只是我們不知道門檻多高。另外，Vercel 稱已部署偵測規則，不等於聲稱觀察到在野攻擊。

### 發行版與部署上的坑

以下幾點來自二手分析（Atlas、Wardio），各發行版的建置設定未逐一核對，請當成「去確認」的提醒：

- 有人指出 Debian、Alpine、Fedora、RHEL 的 nodejs 套件用 `--shared-brotli` 建置，也就是用系統的 brotli 函式庫。如果是這樣，只換 Node 版本不一定換掉 Brotli，要看發行版有沒有更新 brotli 套件。Alpine 追蹤頁在 10/22 07:04 UTC 前後只列出 edge-community 的 `nodejs-current 26.12.1-r0` 已修正；這只說明那個 Node 套件的狀態，各發行版的 Brotli 連結方式與修補方式仍須各自核對。
- Atlas 說，官方內建 Brotli 的 Node 在修補前後的 `process.versions.brotli` 都顯示 1.1.0，不能用來判斷；改用系統 brotli 的建置（例如 Homebrew）則會顯示系統函式庫的版本。Atlas 建議的偵測法是看 `zlib.constants.BROTLI_DECODER_ERROR_FORMAT_BLOCK_SWITCH` 存不存在。這個常數是這次修補一起加進 Node 的，只能判斷 Node 本身的版本；用 `--shared-brotli` 建置的 Node，還要另外看系統 brotli 套件的版本（`node -p "process.config.variables.node_shared_brotli"` 可以看是否為 shared 建置）。
- 有人指出 Express 5 的 body-parser 2.x 預設會解壓 `br` 的請求本體，Express 4 則回 415。這是二手說法，影響範圍尚未逐項驗證。
- Vercel 的說法：Node.js 不會自己解 Brotli 資料，應用程式或其依賴的函式庫解壓攻擊者可控的 Brotli 資料時才會暴露，不論是請求本體，還是自己抓回來的回應。這是廠商自述，但可以拿來判斷你的暴露面。

## 四、TLS 主機名稱驗證這一組

三個 TLS 相關項目（105483、106069、106072）都在講「憑證怎麼和主機名稱、IP 對上」。官方公告只給標題，沒有寫攻擊情境。

我們能確定的有：

- 兩個 high 項目 105483 與 106069 的標題都用了「hostname bypass」；105483 只影響 26.x 與 24.x。
- 106069 與 106072 的修補標題是「拒絕非標準寫法的 IP」；105483 的修補標題是「修正 IP 正規化的 regression」。細節公告沒寫。
- 另外，各 release 頁還有一項沒標 CVE 的安全相關變更：`tls: reject chains containing blocked public keys`，這就是下一節 eMudhra 封鎖清單。

至於「實際上可以怎麼攻擊」，公告與 release 頁都沒寫，本文不推測。如果你的服務會用 IP 直接連線、又依賴憑證驗證，升級是唯一明確的處理方式。

## 五、eMudhra 憑證事件：公告裡那行「ca-cert」

公告的依賴更新裡有一行「ca-cert (eMudhra Intermediate CA Compromise)」。這是從 10/17 開始的另一條線，跟 10 個 CVE 沒有關係。

各方一致的部分：

- **範圍**：Mozilla 與 Google 的說明都是 8 張中繼憑證（6 張 TLS、2 張程式碼簽署）的私鑰被還原出來。
- **成因**：eMudhra 自己的聲明說，2025-04-30 金鑰儀式的 wrap／unwrap 步驟多產生了一組弱金鑰，弱金鑰被送進瀏覽器根庫。這是 eMudhra 的自述，Mozilla 說仍在跟 CA 追根因。
- **處置**：Chrome、Firefox 自 10/17 封鎖，Apple 10/19，Microsoft 10/20（Dennis Jackson 的彙整頁）。Google 說已把 emSign ECC Root CA - G3 從 Chrome Root Store 移除，預計 10/27（週二上午，太平洋夏令時間）對所有裝置生效。
- **Node 的做法**：[PR #66871](https://github.com/nodejs/node/pull/66871) 在程式內加入一份 SPKI SHA-256 雜湊的封鎖清單，命中時以 `X509_V_ERR_CERT_REVOKED` 回報。PR 說明稱，這 8 筆雜湊與 Go 的修補（CVE-2026-111943）和 Electron 的修補逐位元組相同。

為什麼非瀏覽器的程式也要處理？Scott Helme 的文章指出，curl、Python、Node、Go、Java 這些客戶端預設不檢查撤銷，也不檢查 CT，瀏覽器的封鎖不會自動套到它們身上。

對你有什麼影響：在正常的憑證驗證下，憑證鏈上含那 8 把被封鎖公鑰之一的，升級後 Node 會拒絕；PR 說明也提到，升級前已建立、之後恢復的 TLS session 不會重新驗證。這次 release 也把根憑證更新到 NSS 3.131.1，該版本是否包含 emSign ECC Root CA - G3 的移除，未核對。Google 表示，封鎖前沒有這些中繼簽發的 TLS 憑證進入 CT；這只說明強制 CT 的客戶端，不能用來估計 curl、Node 這類預設不檢查 CT 的客戶端有多少連線受影響。eMudhra 向 BleepingComputer 表示多數中繼並未用於正式簽發。請依實際憑證鏈確認。

## 六、各方說法對照與澄清

| 說法 | 誰說 | 與官方 | 怎麼看 |
|---|---|---|---|
| Brotli 那項是 critical | GBHackers | 官方是 high | 同一篇文章後面又寫「four high-severity」，自相矛盾，以官方為準 |
| 逾 6000 萬張憑證不可信 | The Register | 官方沒談 | 該報導談的是 eMudhra 全部簽發量，沒有給出處；Mozilla 與 Google 確認外洩的是 8 張中繼憑證，兩者範圍不同，不能當成外洩規模 |
| 這跟 PHP 同一天修補，是同一件事 | 社群上的混用 | 官方未提及 PHP | 兩者修補日期相同，但我們查到的一手資料沒有指出兩者有技術關聯；SecNews 的評論是「不像 PHP 那樣的危機」，屬該站看法 |
| 風險等級 Medium | HKCERT | 官方是 high（最高） | 各機構自己評級，不必然一致；HKCERT 是目前唯一查到的 CERT 層級公告 |
| 技術細節與 PoC 尚未釋出 | CyberQ（最後修改於 10/21 17:37 UTC，台灣 10/22 01:37） | 官方未說 | 之後有人自稱貼出觸發測試，Vercel 也稱有研究者示範，兩者都沒有第二個來源確認。這句話有時效，引用時要註明時點 |
| 上游尚未打 tag | Atlas | 與 google/brotli v1.2.1 不符 | GitHub 顯示該版本建立於該文標示的發布時間之前 |
| Bun 也受影響 | Lobsters 一則留言、Atlas | 官方只談 Node | 都沒有驗證，不下結論 |

## 七、行動清單

### 開發與維運

1. 查實際在跑的版本：`node -v`。要升到 v22.23.4、v24.22.1 或 v26.12.1 以上。
2. 升級後重啟所有行程，包含 PM2、systemd、Kubernetes 的 Pod。
3. 容器映像重建，並確認基底映像已含修正版。請以映像內實際的 `node -v` 為準。
4. 用 Node 打包的產物（SEA、startup snapshot）要用新版重建，原因見 CVE-2026-106074。
5. 用發行版套件裝 Node 的主機，先確認該套件是否連結系統 Brotli；若是，再核對發行版的 Brotli 修補狀態，不要只看 Node 版本號。
6. 還在跑 EOL 版本線的，官方明說 EOL 版本一律受影響，要換到仍在支援的版本線（目前公告涵蓋 22、24、26）。
7. 暫行措施（二手建議，不取代升級）：在反向代理拒絕帶 `Content-Encoding: br` 的請求。另外有作者實測 `fetch()` 對回應的 `Content-Encoding: br` 會解碼（即使送 `Accept-Encoding: identity`），所以連到不可信伺服器的程式也要注意。

### 企業 IT

1. 先盤點「內含 Node 的東西」：CI runner 與建置映像、內部工具、桌面應用、第三方產品的 Node 內嵌版本。要向廠商確認，不能假設自己的 Node 版本就是產品用的版本。
2. 盤點 eMudhra 簽發的憑證鏈；若含被封鎖的中繼公鑰，安排更換憑證並驗證升級後的連線。Node 本身仍照安全更新時程升級，不必等盤點完成。
3. 看外部的防護設定：Vercel 說已對平台上的專案部署 WAF 規則，但稱不涵蓋函式 `fetch` 外部回應。這是廠商自述，你若用該平台，仍須自己升級依賴。

### 資安同業

1. 官方公告沒有給利用條件。要判斷修補範圍，可以對照 release 頁的 commit 清單與上游 v1.2.1 的 release notes。
2. 評級各家不同（官方 high、HKCERT Medium、部分媒體寫 critical），引用時標明來源。
3. 約 07:02–07:05 UTC（台灣 15:02–15:05），cve.org 的 API 對這 10 個 CVE 全部回 404，NVD 已查詢的 3 個為 0 筆。這是時點性結果，資料庫幾小時內會變，引用前再查一次。
4. 不要把沒有出處的 PoC 說法當成確認。目前有兩則「說有」的說法（Vercel、一位 Mastodon 使用者），另有一則實測沒能重現，都沒有第二個來源。

## 八、結論

這次更新有三件事要分開看：

1. **10 個 CVE 本身**：4 個 high 裡，Brotli 那個因為標題寫了 RCE 最受矚目；官方沒給利用條件，所以風險大小目前無法由公告推出來。
2. **eMudhra 憑證事件**：這是獨立的事，Node 與 Go、Electron 採用同一份封鎖清單，影響面看你的憑證鏈。
3. **各方說法**：「critical」「6000 萬張」「已有 PoC」這幾句，在我們查到的原始資料裡都對不上，或只有單一來源。

該做的事不複雜：升級、重啟，容器與打包產物重建，再盤點內含 Node 的東西。爭議的部分可以等官方或上游補上細節再回頭看。

## 來源清單

**官方與第一手**

- [Node.js：Wednesday, October 21, 2026 Security Releases](https://nodejs.org/en/blog/vulnerability/october-2026-security-releases)（2026-10-21）

> 來源清單的日期，國際來源用 UTC 日期，iThome、HKCERT 用當地日期；僅供查找，時間判斷以正文為準。
- Node.js release notes：[v22.23.4](https://nodejs.org/en/blog/release/v22.23.4/)、[v24.22.1](https://nodejs.org/en/blog/release/v24.22.1/)、[v26.12.1](https://nodejs.org/en/blog/release/v26.12.1/)
- [google/brotli v1.2.1 release](https://github.com/google/brotli/releases/tag/v1.2.1)（2026-10-21 19:27 UTC）
- [Node.js PR #66871：eMudhra 封鎖清單](https://github.com/nodejs/node/pull/66871)
- [Google：Protecting Chrome and Android users from compromised eMudhra intermediate CA certificates](https://blog.google/security/protecting-chrome-and-android-users-from-compromised-emudhra-intermediate-ca-certificates/)（2026-10-20）
- [Mozilla Bugzilla 2083449](https://bugzilla.mozilla.org/show_bug.cgi?id=2083449)
- [eMudhra 聲明](https://emudhra.com/en/press-release/emudhra-statement-distrust-global-tls-intermediate-root-ca)（2026-10-21）
- [HKCERT：Node.js 多個漏洞](https://www.hkcert.org/security-bulletin/node-js-multiple-vulnerabilities_20261022)（2026-10-22）

**廠商與媒體**

- [Vercel：Node.js Security Advisory](https://vercel.com/kb/bulletin/node-js-security-advisory)（廠商自述）
- [iThome：Node.js 安全更新](https://www.ithome.com.tw/news/179887)（2026-10-22）
- [heise online](https://www.heise.de/en/news/Node-js-fixes-its-own-and-inherited-security-vulnerabilities-11496143.html)（2026-10-21）
- [SecurityOnline](https://securityonline.info/nodejs-security-release-october-2026/)（2026-10-21）
- [CyberQ 賽博客](https://cyberq.tw/2026/10/22/nodejs-october-security-update-4-high/)（2026-10-22）
- [GBHackers](https://gbhackers.com/node-js-patches-10-security-vulnerabilities/)（2026-10-22）
- [SecNews](https://www.secnews.gr/en/742493/nodejs-cve-2026-111497-brotli-enimerosi/)（2026-10-21）
- [Brandefense](https://brandefense.io/security-news/next-js-node-js-security-updates-october-2026/)（2026-10-21）
- [Cyber Updates 365](https://cyberupdates365.com/nodejs-security-update-2026/)（2026-10-22）
- [cybernote](https://www.cybernote.click/2026/10/22/nodejs-october-2026-security-release-brotli-rce-emudhra/)（2026-10-22）
- [lilting.ch（日文原文英譯）](https://lilting.ch/en/articles/nodejs-tls-dns-brotli-security-release)（2026-10-21，作者實測）
- [Atlas：CVE-2026-111497 分析](https://atlasum.net/blog/cve-2026-111497-nodejs-brotli-decoder)（二手分析）
- [Wardio：Node.js Brotli CVE-2026-111497](https://wardio.net/guides/nodejs-brotli-cve-2026-111497/)（二手分析）
- [XenoSpectrum](https://xenospectrum.com/en/nodejs-october-security-tls-snapshot/)（2026-10-21）

**eMudhra 事件**

- [Dennis Jackson 彙整頁](https://emudhra.we-have-a-problem.com/)
- [The Hacker News](https://thehackernews.com/2026/10/eight-emudhra-ca-keys-compromised.html)
- [BleepingComputer](https://www.bleepingcomputer.com/news/security/mozilla-google-apple-block-compromised-emudhra-ca-certificates/)
- [Scott Helme：Eight bits of trust](https://scotthelme.co.uk/eight-bits-of-trust-emudhras-ca-keys-cracked-in-milliseconds/)
- [The Register](https://www.theregister.com/security/2026/10/20/indian-certification-authority-emudhras-private-keys-compromised-one-of-them-was-the-number-4/5304312)（2026-10-20）

**社群**

- [Lobsters 討論串](https://lobste.rs/s/gu7bow)
