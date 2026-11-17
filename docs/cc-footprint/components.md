# cc-footprint 元件說明

> 依據 [ilwu/cc-footprint](https://github.com/ilwu/cc-footprint) 原始碼（commit `0c0d8ed`）整理。專案行為與原始碼不一致時，以原始碼為準。

cc-footprint 讓使用者在 Claude Code 裡看到每個 session 的記憶體（RAM）與 context 用量。它由四個元件組成：

| 元件 | 語言 | 位置 | 職責 |
|---|---|---|---|
| Monitor（背景服務） | Node.js | `monitor/` | 每 60 秒量測各 session 的行程樹，並用 HTTP API 回答 statusline 與 plugin |
| Statusline | Bash | `statusline/statusline.sh` | Claude Code 每次回覆後執行，組出一至多行狀態列 |
| Plugin（mod） | TypeScript/TSX | `plugin/` | toast 通知與 `/footprint` 面板 |
| 安裝／輔助腳本 | Bash、PowerShell、Node | `install.*`、`uninstall.*`、`scripts/` | 安裝、設定 settings.json、啟動服務 |

Windows 是系統匣（tray）程式，macOS 是選單列圖示，Linux 為無介面（headless）。

## 整體資料流

```
Claude Code ──stdin JSON──► statusline.sh ──/dev/tcp GET /session/<sid>──► Monitor (127.0.0.1:19823)
Claude Code ◄─ hooks ─────► plugin ─────────HTTP /context/<sid>、/sessions──► Monitor
                                                                              │
                         ~/.claude/sessions/<pid>.json ─► sessions.js ─────────┤
                         ~/.claude/projects/**/<sid>.jsonl ─► transcripts.js ──┤
                         作業系統行程表 ─► collectors/ ─► proctree.js ─────────┘
```

Monitor 停止時，statusline 只顯示 Claude Code 自己提供的項目（見下方「降級行為」）。

## 1. Monitor（`monitor/`）

### 模組一覽

| 檔案 | 角色 |
|---|---|
| `app.js` | 接線與 60 秒迴圈。監聽 `127.0.0.1:19823`，`INTERVAL = 60000`；port 被占用時直接結束。另有睡眠喚醒偵測：計時器跳過超過 60 秒就呼叫 `tray.restart()` |
| `api.js` | HTTP API（見下） |
| `config.js` | `ITEMS` 項目清單與 `~/.cc-footprint/config.json` 的讀寫 |
| `i18n.js`、`i18n/*.json` | 托盤選單的多語系 |
| `sessions.js` | 讀 `~/.claude/sessions/<pid>.json`，建立 sessionId → pid 對照 |
| `plugin-state.js` | 讀 `~/.claude/settings.json`，判斷 `enabledPlugins` 是否有 `cc-footprint@…` |
| `collectors/` | 各平台列出行程表（唯一的平台專屬量測程式） |
| `proctree.js` | 把行程表加總成每個 session 的行程樹，並找出 MCP server |
| `transcripts.js` | 追蹤 `~/.claude/projects/**/<sid>.jsonl`，只讀新增的位元組 |
| `context.js` | 由 transcript 列推算 context 的組成 |
| `tray.js` | 系統匣／選單列圖示（systray2），Linux 無 |

除 `proctree`、`context` 與各 collector 的解析函式是純函式外，其餘會讀檔或持有狀態的模組（`config`、`sessions`、`transcripts`、`plugin-state`、`api`、`tray`、`i18n`）都是**工廠函式**，路徑與狀態由參數傳入，因此測試可以指向暫存目錄。

### 量測流程（`app.js` 的 `collect()`）

1. `plugin.check()`、`config.refresh()`（config.json 以 mtime 判斷是否重讀）。
2. `sessions.scan()` 重讀 session 檔，`transcripts.prune()` 忘掉已結束 session 的 transcript。
3. 平台 collector 回傳 `{ table, clock, cols, systemPct }`。表為空代表查詢失敗，只計入 `collect_errors`，不會被當成「全部結束」。
4. `proctree.measure(table, ages, clock)` 算出每個 session 的記憶體與 MCP server。
5. 結果放進狀態物件 `m`，再 `tray.update()`。

`collecting` 旗標避免慢速的 PowerShell 讓兩次收集互相超車；`collectSoon()` 則在「session 啟動後第一次被問到」時最多每 10 秒觸發一次提早收集。

### Collectors（`monitor/collectors/`）

每個平台模組匯出 `prepare(configDir)` 與 `collect(sessionPids, done)`；`done(err, { table, clock, cols, systemPct })`。行程表每列為 `{ pid, ppid, mem, born, name, mcp }`，其中 `mcp` 是 MCP server 的簡稱（`collectors/mcp.js`），其他行程為 `null`。

| 平台 | 來源 | 備註 |
|---|---|---|
| Windows `win32.js` | 內嵌的 PowerShell／CIM 腳本（`Get-CimInstance Win32_Process`），寫成 `~/.cc-footprint/collect.ps1` | `wmic` 已在 Windows 11 24H2 移除；系統記憶體用 Node `os`。終端寬度＝視窗像素 ÷ 8（高估） |
| Linux `linux.js` | 直接讀 `/proc`（不產生子行程） | `CLK_TCK`、`PAGESIZE` 以 `getconf` 取得；系統記憶體用 `MemAvailable`；終端寬度用 `stty -F` |
| macOS `darwin.js` | 一次 `ps -axww -o pid=,ppid=,rss=,etime=,tty=,command=` | 系統記憶體用 `vm_stat`（active＋wired＋compressor），因為 `os.freemem()` 在 macOS 不準 |

`collectors/index.js` 依 `process.platform` 選模組，不支援的平台為 `null`，`app.js` 會印出訊息並結束。

### `proctree.js`：行程樹與 MCP 判定

- **Session 根行程**：名稱為 `claude`／`claude.exe`（區分大小寫，桌面版 `Claude.exe` 不算），且不是 `chrome-native-host` 或 `claude mcp serve`；或 session 檔指到的 pid（npm 安裝時行程名是 node）。
- **pid 重用防護**：行程比 session 檔記錄的開始時間年輕超過 60 秒（`PID_REUSE_SLACK_MS`），視為 pid 已被他人沿用；子行程若比父行程老（`born` 較早）也不算子行程。
- **Session 記憶體＝整棵行程樹加總**（`mem`），同時給出 `self`（claude 本身）、`procs`、`mcp_mem`、`mcp_count`。
- **MCP server 判定**：命令列符合 `/mcp|modelcontextprotocol|chrome-native-host/i`，且存活 ≥ 30 秒（`MCP_MIN_AGE_MS`，避免剛好帶 "mcp" 字樣的工具 shell 被誤判）。包裝鏈（cmd → npx → node）只算一個 server。
- **`outside`**：不屬於任何 session 樹的 MCP server（已結束 session 遺留、Chrome 橋接等），各自含子樹記憶體，依大小排序。

### Transcript 與 context 推算（`transcripts.js`、`context.js`）

Claude Code 的 transcript 沒有分項的 token 帳，所以 `context.js` 用「相鄰兩次回應的 context 大小差」反推：

- 每次 API 回應的 context 大小＝`input_tokens + cache_creation_input_tokens + cache_read_input_tokens`。
- 兩次回應之間的增量，先歸給上一次回應的輸出（再拆出 `thinking`），剩下的依據各筆 tool result／prompt 的**字元長度**按比例分攤（圖片以 6000 字元計）。
- 類別（`categoryOf`）：`mcp:<server>`、`files`（Read）、`shell`（Bash／PowerShell）、`search`（Grep／Glob）、`web`、`agents`、`tools`，另有 `base`、`system`、`output`、`thinking`、`prompts`、`summary`（壓縮後）。這些類別名稱是與 plugin 之間的**契約**。
- 遇到 `compact_boundary` 後，重設為 `base` ＋ `summary`；context 變小但沒壓縮時，用 `shrink()` 先扣 thinking，其餘按比例扣。
- sidechain（subagent）的列忽略，只算主對話。
- `transcripts.js` 以子字串預先過濾（`"type":"assistant"` 等），這依賴 Claude Code 寫出緊湊 JSON；只讀新增位元組，單次最多讀 4 MB，並以 `fs.promises` 非同步讀取。

`context.detail()` 回傳 `{ tokens, turn, parts }`，`context.summarize()` 回傳 statusline 要的 `turn`、`src`、`src_pct`、`mcp_pct`（`base`、`system` 不參與最大來源排名）。

### HTTP API（`api.js`）

所有請求的 `Host` 必須是 `127.0.0.1:19823`、`localhost:19823` 或 `l`（statusline 用的最短值），否則回 403，用來防範 DNS rebinding。

| 路徑 | 使用者 | 內容 |
|---|---|---|
| `GET /session/:sid` | statusline | 該 session 的記憶體、`claude_total`、`system_pct`、MCP 數據、`ctx_turn`／`ctx_src`／`ctx_src_pct`／`ctx_mcp_pct`、`remote_control`、`plugin`、`cols`、`display`（啟用項目清單） |
| `GET /context/:sid` | plugin | context 組成 `{ tokens, turn, parts[] }` |
| `GET /sessions` | plugin、`install.ps1` | 所有執行中 session（依記憶體大到小）、`claude_total`、`outside`、`lang` |
| `GET /status`、`/status/:pid` | 手動檢視 | 量測狀態與計數器（`rows_read`、`rows_failed`、`transcripts`、`collect_errors`）、各 session 樹 |
| `GET /config` | 手動檢視 | 項目、設定值、`display`、語言 |

`sid` 只接受十六進位與連字號，其他一律 404。資料超過 `staleMs`（3 × 60 秒）未更新時不再回傳量測值，避免舊數字被當成現值。

**檔案讀取預算**：每個請求有 `fileWaitMs = 300` 毫秒的預算讀 transcript 與 session 檔；逾時就不帶那些欄位回答，讀取在背景繼續，供下一次請求使用。原因是 Windows 上開檔曾被卡住數秒（疑似防毒），同步讀取會讓所有 session 的 statusline 同時卡死。

**Statusline 以 regex 解析回應**，所以 `/session` 回應的鍵必須維持扁平且全域唯一。

### 設定與多語系

- `~/.cc-footprint/config.json`：每個項目一個布林值，加上 `lang`（`auto` 或語言代碼）。Linux 沒有托盤，直接編輯此檔即可，mtime 變動時會被重讀。檔案壞掉時保留原樣（備份為 `config.json.broken`），沿用上一份有效設定。
- 18 個項目定義在 `config.js` 的 `ITEMS`（分 memory／usage／session 三組）。
- 語系：en、de、es、fr、pt-BR、ja、ko、zh-CN、zh-TW。`lang: auto` 由 `Intl` 判斷，中文按文字（Hant／Hans、TW／HK／MO）區分。

## 2. Statusline（`statusline/statusline.sh`）

Claude Code 在每次回覆後以 stdin 送入 JSON 並執行此腳本，輸出直接成為狀態列。

**效能限制**（文件只說有 300 ms debounce，新的更新會取消仍在執行的腳本）：

- 不使用 `curl`，改用 `/dev/tcp` 自行送 HTTP/1.0（約 35～46 ms）。
- 不使用 `$(...)`（MSYS 下每次 fork 約 30 ms），輔助函式改設全域變數（`FMT`、`PC`、`BAR`、`RESP`、`TOK`、`LEFT`、`W`）。
- 用 bash regex 解析 JSON，不呼叫 sed／grep。
- 必須相容 **bash 3.2**（macOS 內建）：`read -t` 只接受整數秒、`{` 寫成 `[{]`、沒有 `EPOCHSECONDS`（macOS 退回 `date +%s`）。
- 重複出現的鍵要在自己的物件內找（`used_percentage` 同時出現在 `context_window` 與各 rate limit）。

**Monitor 掛掉時**：連線失敗後 30 秒內不再詢問（`/tmp/claude-sl-monitor.down` 記錄失敗時間），第一次失敗寫入 `/tmp/claude-sl-monitor.log`（`refused`／`no-header`／`no-body`）。這段期間只顯示 Claude Code 自己提供的預設項目 `ctx`、`five_hour`、`week`、`resets`、`session_id`、`path`；由 monitor 量測的項目不顯示，不用快取。

**排版**：依 `COLUMNS − 4` 自動換行（monitor 回傳的 `cols` 只是備援）。顯示寬度由 `width()` 計算，CJK 算兩欄；`add_item` 的第二個參數是用 ASCII 代替長條與箭頭的「量測用字串」。statusline 本身不含需翻譯的文字（只有 `Ctx`、`5h`、`Sys`、`mcp` 等短標籤），新文字應放在 plugin 面板。

## 3. Plugin（`plugin/`）

詳見 [plugin-hooks.md](./plugin-hooks.md)。

## 4. 安裝與腳本

- `install.sh`／`install.ps1`：複製 statusline、設定 `statusLine`、啟動 monitor（Linux 為 systemd user service、macOS 為 launchd agent、Windows 為開機啟動捷徑）、註冊 plugin（可用 `--no-plugin`／`-NoPlugin` 略過）。可重複執行。
- 共用腳本放在 `scripts/`：`statusline-file.js`（放置 statusline.sh、備份使用者自己的）、`statusline-setting.js`（編輯 settings.json，保留鍵順序與格式）、`optional.js`（列出選用優化）、`remove-global-rule.js`（移除 CLAUDE.md 內由標記圈起的區塊）。**不要在 PowerShell 或 bash 再寫第二份**。
- 既有的非本工具 statusLine 會備份為 `.bak`，已有 `.bak` 就保留（它存的是安裝本工具之前的狀態）。`statusLine` 會加上 `refreshInterval: 30`，因為 session 閒置時事件停止，而 monitor 的數字仍會變動。
- 安裝／解除安裝只會停掉**自己的** monitor；port 被別人占用時會明說並停止。
- 選用的「瀏覽器 subagent」優化（`claude/agents/browser.md`、`claude/global-rule.md`）不會被安裝器自動套用。

## 設計取捨摘要

| 決策 | 原因 |
|---|---|
| Monitor 常駐、statusline 只問 HTTP | 在 Windows 啟動一個 PowerShell 要 500～900 ms，遠超 statusline 預算 |
| Session 記憶體＝整棵行程樹 | MCP server 與工具 shell 都是子行程 |
| 平台差異只放在 collector | 其他分支僅限 `tray.js` 與 statusline 在 macOS 的 `date` 後備 |
| 不顯示價格 | 價格會變、本工具無法追蹤；成本欄位只轉述 Claude Code 回報的值 |
| 檔案讀取非同步且有 300 ms 預算 | 避免開檔被卡住時拖垮所有 statusline |
