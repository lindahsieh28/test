# cc-footprint 測試與架構

> 依據 [ilwu/cc-footprint](https://github.com/ilwu/cc-footprint)（commit `0c0d8ed`）整理。

## 為什麼要這樣設計才能測

cc-footprint 依賴的 Claude Code 格式（statusline stdin、`~/.claude/sessions/<pid>.json`、transcript、settings.json）**都沒有官方文件**，而且必須在 statusline 的時間預算內執行。測試設計因此圍繞三件事：

1. **把平台差異壓到最小**：只有「列出行程」是平台專屬，其餘都是共用純函式，單元測試在任何平台都能跑。
2. **用工廠函式注入路徑與狀態**：會讀檔或持有狀態的模組（`config`、`sessions`、`transcripts`、`plugin-state`、`api`、`tray`、`i18n`）由參數傳入路徑，測試指向暫存目錄即可。
3. **用真實擷取的 fixtures 與端對端 smoke test 擋格式變動**。

## 測試分層

| 層 | 工具／指令 | 範圍 |
|---|---|---|
| 單元測試（monitor） | `cd monitor && node --test` | 12 個測試檔，涵蓋 api、config、consistency、context、i18n、proctree、sessions、transcripts 與四個 collector；**不含** `app.js`、`tray.js`、`plugin-state.js` |
| 單元測試（plugin） | `claude plugin test plugin`（`claude-code/testing` 的 `test`／`mock`／`expect`） | `plugin/hooks/register.test.ts`，25 個 test：純邏輯、toast、面板、語言 |
| 型別／驗證 | `claude plugin validate plugin --strict`；對當前版本的 `claude-code.d.ts` 做型別檢查 | plugin 的 manifest 與型別契約 |
| Statusline | `bash statusline/statusline.sh < test/fixtures/statusline-input.json`，再用 `bash:3.2` 容器跑一次 | 真實 stdin 的解析、bash 3.2 相容 |
| 端對端 smoke test | `bash test/smoke.sh`（Docker：`docker run --rm -v "<repo>:/repo:ro" node:22-bookworm bash /repo/test/smoke.sh`） | 安裝 → 量測 → statusline → 解除安裝 |
| CI | `.github/workflows/test.yml` | 見下表 |

專案沒有 lint 或 TypeScript 編譯步驟（除上述 plugin 型別檢查外）。

### 改了什麼就跑什麼

| 變動 | 要跑 |
|---|---|
| 任一 monitor 模組、顯示項目、README 的 config 區塊 | `node --test`（在 `monitor/`） |
| `statusline.sh` | fixture 餵一次 → `bash:3.2` → smoke test |
| `app.js`、`api.js`、`tray.js`、安裝器、`scripts/` | Docker 內的 smoke test；Windows、macOS 靠 CI |
| `plugin/` | `claude plugin validate plugin --strict`、`claude plugin test plugin`、型別檢查 |

## 單元測試重點

| 測試檔 | 驗證什麼 |
|---|---|
| `proctree.test.js` | 行程樹加總、MCP 判定（30 秒存活門檻）、包裝鏈只算一個、pid 重用防護、游離 MCP |
| `context.test.js` | 由 `test/fixtures/transcript.jsonl` 推算 context 組成，數字須與原始列一致；壓縮、縮小、thinking 的處理 |
| `transcripts.test.js` | 只讀新增位元組、不完整行的處理、檔案被改寫時重讀、找不到檔案時的重試間隔、格式異常的列只算 `rows_failed` |
| `sessions.test.js` | session 檔解析、同一 id 在兩個檔時取較晚開始者、`bridgeSessionId` 判斷 Remote Control |
| `api.test.js` | `/session` 回應的鍵（statusline 的 regex 依賴它們）、Host 檢查（403）、過期資料不輸出、`fileWaitMs` 逾時行為 |
| `config.test.js` | 預設值補齊、壞掉的 config.json 不被覆蓋、mtime 重讀 |
| `collectors/*.test.js` | `linux` 用假 `/proc` 目錄；`darwin` 的 `ps`／`vm_stat` 解析；`win32` 的腳本輸出解析；`mcp` 簡稱抽取 |
| `i18n.test.js` | 每個語言檔的 key 與 placeholder 都與英文一致、不含 `&`、系統 locale 對應規則、`tray.js` 用到的每個 key |
| `consistency.test.js` | **項目清單的五處一致**：`ITEMS`、statusline 的 `has` 檢查、statusline 的備援清單、兩份 README 的 config.json 預設值；選單標籤不含 `&` |
| `register.test.ts` | 門檻、分組、堆疊條寬度、各 hook 的行為、面板在各種表面的繪製、語言保留、資料形狀異常時的降級 |

寫法慣例：用假的 `m`、`config`、`sessions`、`transcripts`、`plugin`、`i18n` 組出 `createHandler` 的輸入（見 `api.test.js` 的 `fakes()`）；collector 測試用純函式與假目錄，不呼叫真的系統指令。

## Fixtures（`test/fixtures/`）

- `statusline-input.json`：真實擷取的 statusline stdin（Claude Code 2.1.291），ID、名稱與路徑已替換，數字保持原樣；smoke test 使用。
- `transcript.jsonl`：真實 transcript 的前 120 列（user／assistant／system）；`context.js` 讀取的每個鍵都保留，數字與型別照舊，文字改成等長的 `x`（長度決定增量如何分攤）。
- `README.md` 說明擷取方式。**Claude Code 改版後若格式變動，要重新擷取並重跑測試。**

## Smoke test（`test/smoke.sh`）

在拋棄式的 `HOME`（路徑刻意含空白）內，依序驗證：

1. 安裝器執行（以 README 的方式 `./install.sh --no-plugin`，所以可執行位遺失會在這裡失敗），monitor 開始回應。
2. 使用者原本的 statusline 與設定被備份，第一份備份被保留。
3. 一個假 session（約 80 MB 的行程＋40 MB 的子行程）：`/session`、`/sessions` 量到正確的樹記憶體與行程數。
4. 假 transcript（context 由 50k 長到 68k）：statusline 顯示記憶體、成長量、MCP 佔比、重置倒數。
5. `statusLine.command` 以 shell 執行；context 為 `null` 時維持未知。
6. 餵入真實的 stdin fixture。
7. `/footprint` 提示會跟著 plugin 是否啟用而變。
8. 換行會依終端寬度填滿。
9. 編輯 `config.json` 關掉項目後，下一次 render 就生效。
10. 解除安裝只移除自己裝的東西。
11. Monitor 停止時：statusline 退回只顯示 Claude Code 自有項目、30 秒內不再重試、並寫下 log。

`SMOKE_SERVICE=1` 會改用 systemd user service（Linux）或 launchd agent（macOS），並寫入真實的 `$HOME`，所以**只在 CI 執行**。Windows 的 smoke test 會停掉並解除安裝本機的 monitor，同樣只在 CI（`CI=true`）執行。

## CI 矩陣

| 工作 | 平台 | 內容 |
|---|---|---|
| `unit` | ubuntu／windows／macos | `node --test`（Node 22） |
| `linux` | ubuntu | `bash test/smoke.sh` |
| `linux-systemd` | ubuntu | `SMOKE_SERVICE=1`，先 `loginctl enable-linger` 啟動 user session |
| `windows` | windows | Git Bash 執行 smoke，安裝器走 Windows PowerShell 5.1 |
| `macos` | macos | `/bin/bash` 3.2、`ps` collector |
| `macos-service` | macos | launchd agent |

## 系統架構

```
Claude Code ── statusline stdin ──► statusline.sh ──/dev/tcp──┐
     │                                                       ▼
     └── hooks（session／command／turn／compact／ui）► plugin ──HTTP──► api.js ◄── app.js 60 秒迴圈
                                                                          │            │
                                                          config / sessions / transcripts / plugin-state
                                                                                       │
                                                                  collectors/{win32,linux,darwin} → proctree.js
```

分層與責任：

1. **Claude Code**：提供 stdin、session 檔、transcript、hooks。
2. **Statusline（Bash）**：唯一在使用者互動路徑上、有時間預算的元件。
3. **Plugin（TSX）**：toast 與面板，資料同樣經由 HTTP。
4. **HTTP API（`api.js`）**：兩種消費者的唯一介面，並負責 Host 檢查與讀檔時間預算。
5. **Monitor（Node）**：量測、快取、推算 context。

### 效能預算（來自 CLAUDE.md 的限制）

| 項目 | 預算／成本 |
|---|---|
| statusline 整體 | 官方只說 300 ms debounce，且新的更新會取消仍在跑的腳本；太慢就什麼都不顯示 |
| `/dev/tcp` 取 `/session` | 約 35～46 ms |
| Windows 啟動一個行程 | PowerShell 約 500～900 ms、curl 約 650 ms、cat 約 230 ms |
| MSYS 下每次 `$(...)` fork | 約 30 ms |
| MSYS 連線被拒 | 約 2 秒 → 失敗後 30 秒內不再詢問 |
| API 讀檔等待 | 每個請求最多 300 ms（`fileWaitMs`） |
| monitor 量測間隔 | 60 秒；plugin 面板開著時 30 秒重讀 |

### 容錯原則

- Monitor 掛掉不得拖慢 statusline（30 秒退避與備援清單）。
- 量測過期不輸出舊數字；寧可少顯示。
- 一列格式不明的 transcript 只損失那一列（`rows_failed`），不影響後續列。
- plugin 每個 hook 自行 try/catch；面板資料形狀不對時只顯示一行說明與重新讀取按鈕。
- 喚醒偵測：機器睡眠後 tray helper 可能選單失效，`app.js` 偵測到計時器延遲超過 60 秒就重啟 tray。

## 尚未驗證（誠實清單）

依 CLAUDE.md，以下**沒有**被自動測試證實，不要當成已驗證：

- macOS 選單列圖示（CI 的 macOS 沒有螢幕）；monitor、launchd agent 與 bash 3.2 下的 statusline 則有測。
- macOS／Linux 桌面的真實 Claude Code session（CI 用假 session 代替）。
- plugin 的 toast 在真實 session 的呈現（只有 `claude plugin test` 覆蓋）。
- 「睡眠會讓托盤選單失效」只是由一次現象推論，重啟 monitor 後恢復。
- Windows 的備援：以自己名稱執行的 tray helper 失敗時退回出貨名稱；以及 npm 安裝的 Claude Code（行程名為 node.exe，靠 session 檔辨識）。
