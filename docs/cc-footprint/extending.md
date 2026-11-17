# cc-footprint 擴充指南

> 依據 [ilwu/cc-footprint](https://github.com/ilwu/cc-footprint)（commit `0c0d8ed`）的專案慣例整理。範例中的檔名與函式名稱都來自原始碼；片段程式碼是示意，不是可直接貼上的 patch。

四種常見擴充：新增顯示項目、新增語言、新增平台、新增 context 類別。每一種都有**多處要同步**，漏改的地方多半會被測試擋下（見 [testing-architecture.md](./testing-architecture.md)）。

## 1. 新增顯示項目（statusline item）

項目清單目前在五個地方各有一份，`monitor/consistency.test.js` 會比對它們是否一致。

1. **`monitor/config.js`**：在 `ITEMS` 加一行，托盤選單會自動帶出。

   ```js
   { id: 'my_item', label: 'My Item', group: 'usage', default: false },
   ```

   - `group` 是 `memory`、`usage`、`session` 其中之一。
   - label 不可含 `&`（Windows 選單會把它當快捷鍵標記，字就不會顯示）。
   - 若數值由 monitor 供應，加進 `api.js` 的 `statusFor()` 或 `/session` handler，鍵名**不可與其他欄位的名稱互相包含**，且要保持扁平（statusline 用 regex 解析）。
2. **`statusline/statusline.sh`**：加上 regex 解析與 `if has my_item; then add_item … fi`。
   - 寫在腳本中的位置就是它顯示的順序（不是選單順序）。
   - `add_item` 第二個參數是量測寬度用的純文字；長條與箭頭請用 ASCII 代替（`#`、`^`）。
   - 如果只靠 Claude Code 自己的 stdin 就能提供，考慮是否加進 monitor 停止時的備援 `display='…'` 清單。
   - 遵守效能規則：不用 `$(...)`、不呼叫外部指令、相容 bash 3.2。
3. **`monitor/i18n/*.json`**：每個語言都要有 `item.my_item`（英文必須與 `ITEMS` 的 label 完全相同）。
4. **README.md 與 README.zh-TW.md**：項目表的列，以及 `config.json` 預設值區塊（兩份 README 要同步）。
5. **`test/smoke.sh`**：如果新項目預設開啟，100 欄換行檢查的結果可能改變，要一併確認。

最後執行 `cd monitor && node --test`；若動到 `statusline.sh`，另外在 `bash:3.2` 容器與 smoke test 下驗證。

## 2. 新增語言

monitor（托盤）與 plugin（面板、toast）的文字**各自獨立**，兩邊都要加：

1. **`monitor/i18n/<code>.json`**：複製 `en.json`，翻譯所有 key。
   - `_name` 填該語言的自稱，托盤的 Language 選單會用它列出（選單本身的標題固定是 `Language`，使用者選錯語言也找得到回頭的路）。
   - 保留所有 `{placeholder}`；`_one` 結尾的鍵是計數為 1 的單數形，其他語言可省略。
   - 任何字串都不可含 `&`。
2. **`plugin/hooks/strings.ts`**：照英文物件寫出該語言的完整版本，加進 `TABLES`。英文是型別基準，缺一句就無法通過型別檢查。
3. **`fromLocale()`（`monitor/i18n.js`）**：只有在該語言無法由代碼自動比對時才需要改（中文以文字別 Hant／Hans 與地區 TW／HK／MO 判斷；`pt-PT` 會落到 `pt-BR`）。
4. 執行兩套測試：`cd monitor && node --test` 與 plugin 測試（`claude plugin test plugin`）。

注意：

- statusline 不放任何需翻譯的字（只有 `Ctx`、`5h`、`Sys` 等短標籤與數字），新詞彙請放在 plugin 面板。
- 在 macOS 上由 launchd 啟動的 monitor，`lang: auto` 會是英文（Node 的 `Intl` 只讀 `LANG`，launchd 代理人沒有），這是刻意保留的行為，使用者從選單選語言即可。

## 3. 新增平台

「列出行程」是唯一的平台專屬量測程式。新平台（例如 FreeBSD）只要產生同樣格式的行程表。

1. **新增 `monitor/collectors/<platform>.js`**，匯出：

   ```js
   // prepare(configDir)：啟動時呼叫一次
   // collect(sessionPids, done)：done(err, { table, clock, cols, systemPct })
   module.exports = { prepare, collect };
   ```

   - `table` 每列：`{ pid, ppid, mem, born, name, mcp }`。`mem` 為常駐記憶體位元組、`born` 為啟動時間（毫秒；任何 epoch 皆可，但要與 `clock` 同一個時鐘）、`mcp` 用 `require('./mcp').label(cmdline) || null`。
   - `clock`：以 `born` 所用時鐘表示的「現在」。
   - `cols`：`Map(pid → 終端欄數)`，只需涵蓋 session 行程，取不到就不放。
   - `systemPct`：系統記憶體使用率 0–100，不知道就 `null`。
   - 查詢失敗要 `done(err)`，**不要回傳空表**（空表會被視為查詢失敗，但錯誤路徑更清楚）。
   - 不要自己加總行程樹、判斷 MCP server 或 session 根行程：這些都在 `proctree.js`。
2. **註冊**：在 `monitor/collectors/index.js` 的 `modules` 對照表加上 `process.platform` → 檔案。
3. **只有兩處可以放其他平台分支**：`tray.js`（無托盤、`.ico`／`.png`、helper 檔名）與 `statusline.sh` 在 macOS 的 `date` 後備。不要在別處加分支。
4. **解析函式寫成純函式並匯出**（像 `linux.parseStat`、`darwin.parsePs`、`win32.parseOutput`），再寫 `<platform>.test.js`：用假資料（例如假 `/proc` 目錄）餵進去，這樣測試在任何平台都能跑。
5. **安裝器**：`install.sh`／`uninstall.sh` 需要加入開機啟動方式（systemd、launchd 或其他）；共用的設定修改邏輯仍走 `scripts/`。
6. **CI**：在 `.github/workflows/test.yml` 加入該平台的 `unit` 與 smoke 工作。

要驗證的事項：session 樹加總是否正確、`MCP_MIN_AGE_MS` 判斷、`born` 的時鐘是否與 `clock` 一致（pid 重用防護依賴它）、讀不到的 `cols` 是否安全略過。

## 4. 新增 context 類別（`context.js` 的 `categoryOf`）

類別名稱是 monitor 與 plugin 之間的契約，**三處要一起改**：

1. `monitor/context.js` 的 `categoryOf()`（以及必要時 `labelOf()`）。
2. `plugin/hooks/notices.ts` 的 `GROUPS`：決定它在面板歸入哪個顏色群組（沒列出的名稱會被當成 MCP server，除非 monitor 的 `kind` 另有說明）；`strings.ts` 的 `kinds` 與 `words` 要有對應文字（所有語言）。
3. README 的類別清單（兩份）。

並補 `context.test.js` 與 `register.test.ts` 的案例。

## 5. 新增監控指標（monitor 端）

1. 在平台 collector 或 `proctree.js` 取得原始資料；加總與判斷邏輯放在純函式，方便測試。
2. 把結果放進 `m`（見 `app.js` 的狀態物件），由 `api.js` 的對應路徑輸出：給 statusline 用就走 `/session/:sid`，給 plugin 用就走 `/sessions` 或 `/context/:sid`。
3. 一定要考慮「資料過期」：`api.js` 對超過 `staleMs` 的量測不再輸出，新指標也要遵守，不能把舊數字當現值。
4. 在 `api.test.js` 檢查新鍵確實出現在 statusline 讀的位置，且沒有鍵名重複。
5. 計數類資訊可以放進 `m.stats`，由 `/status` 輸出，用來在 Claude Code 版本改格式時及早發現（`rows_failed` 增加、量測停擺）。

## 不要做的事

- 不要讓任何東西依賴 token 價格；成本欄位只轉述 Claude Code 回報的值。
- 不要在 statusline 加 curl、`$(...)`、`sed`／`grep`。
- 不要把 `.sh` 存成 CRLF（`.gitattributes` 已強制 LF）；新的、README 會直接執行的 `.sh` 要設可執行位：`git update-index --chmod=+x <file>`。
- 不要在 `.ps1` 用 `ConvertTo-Json` 改 settings.json；統一用 node 腳本。`.ps1` 需能在 Windows PowerShell 5.1 執行，非註解部分保持 ASCII。
- 托盤更新用 systray2 的 `update-item`，不要用 `update-menu`（會讓 onClick 失效）。
