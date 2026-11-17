# cc-footprint Plugin 與 Hooks 說明

> 依據 [ilwu/cc-footprint](https://github.com/ilwu/cc-footprint) 的 `plugin/`（commit `0c0d8ed`，plugin 版本 `0.2.12`）整理。Claude Code 的 function-hooks API 屬於早期版本，會隨 Claude Code 更新而變動；實際型別以當下版本的 `claude-code.d.ts` 為準。

## 目的

Statusline 必須極度精簡，所以較細的資訊移到 plugin：

1. **Toast 通知**：某一輪（turn）讓 context 暴增、接近自動壓縮、壓縮完成。
2. **`/footprint` 面板**：context 由什麼組成、rate limit、各 session 的記憶體與游離的 MCP server。

Plugin 無法自己量測行程或讀 transcript，所以資料來自 monitor 的 HTTP API（`http://127.0.0.1:19823`），再加上引擎提供的 `$.session.usage()`。

## 檔案結構

```
plugin/
├─ .claude-plugin/plugin.json   名稱、版本、types 入口
├─ hooks/hooks.json             { "modules": ["./register.tsx"] }
├─ hooks/register.tsx           所有 hooks 與面板繪製
├─ hooks/notices.ts             純邏輯：門檻、分組、文字組裝
├─ hooks/strings.ts             各語言文字、width()／padTo()
├─ hooks/register.test.ts       測試（25 個 test）
└─ types/index.d.ts             Part／Session／Outside／Limit／View 與 PluginState 型別
```

為什麼不用 JSON 放文字：hooks 模組讀不了 JSON 檔，所以 plugin 的文字放在 `strings.ts`，和 monitor 的 `i18n/*.json` 互不共用句子，只共用語言代碼。

## 資料來源

| 資料 | 來源 |
|---|---|
| context 組成 `{ tokens, turn, parts }` | `GET /context/<sessionId>` |
| 各 session 記憶體、游離 MCP、`claude_total`、`lang` | `GET /sessions` |
| context 視窗大小、自動壓縮門檻、rate limits | 引擎的 `$.session.usage()`（`context.window`、`breakdown.autoCompactThreshold`、`rateLimits`） |
| session id、現在時間 | `$.session.id()`、`$.clock.now()` |

`ask()` 包住所有 `$.http.fetch`：任何失敗都回傳 `null`，面板就改為顯示「monitor 未執行」的狀態，**plugin 必須能在 monitor 停止時降級**。

## Hooks 一覽

所有 hook 都自己 try/catch：hook 失敗不能影響 session 啟動、一輪結束或壓縮。

| Hook | 觸發時機 | 做什麼 |
|---|---|---|
| `session.start` | session 開始 | 以 `$.command.register` 註冊 `/footprint`（描述依語言）；失敗就只是沒有指令，toast 不受影響 |
| `command.run`（`/footprint`） | 執行 `/footprint` | 面板已開則關閉（取消計時器、`$.ui.close`）；否則 `load()` 讀資料、`$.ui.open({ id: 'footprint', closeOnEscape: true })`，成功後以 `$.clock.every(30_000, …)` 每 30 秒重讀。放不了面板時改回一行摘要 `summaryLine()` |
| `ui.close`（id=footprint） | 面板被關閉 | 取消重讀計時器、標記為關閉 |
| `prompt.submit` | 使用者送出 prompt | 記下這一輪開始時的 `base`（context token 數與各 part），供「這一輪增加了什麼」比較 |
| `turn.complete` | 一輪結束 | 先 `await next(e)`；跳過 subagent（`e.agentId` 有值）；`load()` 後判斷是否發兩種 toast（見下） |
| `session.compact` | 壓縮完成 | 跳過 subagent、`trigger === 'precompute'` 與被略過的壓縮；發出「Context compacted: 前 → 後」toast，並重新武裝警告 |
| `ui.render`（component `Pane`、requestId `footprint`） | 繪製面板 | 用 `$.ui.resolve(e)` 取得 `Box`／`Button`／`Text`，畫出面板 |

### 狀態

`register` 閉包內有四個變數：`base`（這一輪起點）、`hasWarned`（接近壓縮的警告是否已發過）、`refresher`（面板開著時的計時器）、`isOpen`。重新載入 plugin 會忘記這些，後果只是少一次「mostly …」或面板改成每輪結束才更新。另用 `atom({ plugin: 'cc-footprint', key: 'view' })` 保存最後一次讀到的 `View`，供 `ui.render` 讀取；語言則用 `$.store` 持久保存，monitor 停止時面板不會變回英文。

### Toast 規則（`notices.ts`）

**1. 一輪 context 暴增**（`turnNotice`）

- 門檻 `bigTurn(window) = min(window × 5%, 25,000)`：200k 視窗是 10k，1M 視窗是 25k。因為之後每次請求都會重新讀取這一輪增加的內容，視窗越大 5% 越多，所以另設 token 上限。statusline 的黃色 `↑` 用同一門檻。
- 增量優先用 monitor 回報的 `turn`，沒有時用「目前 tokens − `base.tokens`」。
- 若某個 part 的增量 ≥ 該輪增量的 40%，加上「mostly …」指出最大來源；沒有 `base.parts` 時不指責任何來源（否則每個 part 看起來都是新的）。
- 範例：`Context +30k this turn (15% of the window), mostly file reads`。

**2. 接近自動壓縮**（`compactNotice`）

- `tokens / autoCompactThreshold ≥ 0.85`（`NEAR_COMPACT`）時發一次，之後 `hasWarned = true`。
- 降到 `0.70`（`REARM`）以下，或發生壓縮後，重新武裝。
- 自動壓縮關閉或不知道門檻時（`compactAt === null`）不發。

Toast 顯示 8 秒（`NOTICE_MS = 8000`）。

## `/footprint` 面板

由上而下：

1. **標題**：`Footprint`，加上 session 數與總記憶體。
2. **Context window**：圓角卡片，已用／視窗、百分比、進度條、這一輪增減、距離自動壓縮的剩餘量；接近壓縮時邊框與標記變黃。再下面是「What fills it」堆疊條與圖例（兩欄，窄時一欄）。
3. **Limits**：每個 rate limit（`five_hour`→`5h`、`seven_day`→`Week`、`spend_limit`→`Spend`）的進度條與重置倒數。
4. **Memory**：本 session 的記憶體（含 claude 本身、子行程數、MCP server 數與記憶體）、其他 session、以及**游離的 MCP server**（存活 ≥ 7 天的以黃色標出）。
5. **Refresh** 按鈕與關閉提示。

資料在每輪結束、執行 `/footprint`、按 Refresh 時讀取，面板開著時另每 30 秒重讀（`REFRESH_MS`；monitor 每分鐘量測一次，所以與 statusline 最多差半分鐘）。

### 分組與顏色（`groupParts`）

monitor 的 part 名稱被折成九種「kind」，各有固定顏色與堆疊順序：

| kind | 包含的 part | 顏色 |
|---|---|---|
| output | `output` | `#3987e5` |
| think | `think` | `#d95926` |
| shell | `shell` | `#199e70` |
| files | `files` | `#c98500` |
| mcp | 所有 MCP server（圖例列出前兩個名稱） | `#d55181` |
| web | `search`、`web` | `#008300` |
| agents | `agents`、`tools` | `#9085e9` |
| prompts | `prompts`、`summary` | `#e66767` |
| system | `base`、`system`（使用者無法處理） | `#898781` |

`kindOf(part)` 先信 monitor 給的 `kind`；舊版 monitor 沒有 `kind` 時，名稱不屬於內建類別就視為 MCP。名為 `web` 的 MCP server 因此不會被誤分到網頁類。兩個 server 簡稱相同時用 `id` 區分。`allot()` 以最大餘數法把圖例條切成剛好 N 格。

> `context.js` 的類別名稱、`notices.ts` 的分組與 README 的類別說明是同一份契約，三者要一起改。

### 版面細節

- 欄位對齊一律用 `strings.ts` 的 `width()`／`padTo()`（寬字元佔兩欄），**不要用** `.length`／`padEnd`。
- 面板寬度取 `e.props.bodyColumns ?? e.viewport?.columns ?? 80`，卡片內寬 `min(76, columns − 4)`。
- 圖形資料不合預期時，整個 `ui.render` 在 try/catch 內，回傳一行「無法繪製」與 Refresh 按鈕。

## 多語系（`strings.ts`）

英文是基準，其他語言以其型別撰寫，缺一句就無法通過型別檢查。plugin 從 `/sessions` 的 `lang` 得知語言，monitor 沒有提供的語言會退回英文。

## 開發與發佈

- **開發**：編輯 `plugin/` 後在 session 內執行 `/reload-plugins`。從本機 clone 註冊的 marketplace 會「就地載入」，改動不必升版；`claude plugin update` 只是重新記錄版本。
- **驗證**：`claude plugin validate plugin --strict`、`claude plugin test plugin`，並依當前版本的 `claude-code.d.ts` 做型別檢查（tsconfig 放在 repo 外，`include` 指向該檔、`plugin/hooks`、`plugin/types`）。
- **發佈**：repo 根目錄的 `.claude-plugin/marketplace.json` 讓 repo 本身就是 marketplace（`/plugin marketplace add ilwu/cc-footprint`），推到 `main` 即發佈。只要 `plugin/` 有變動，就要同時調高 `plugin/.claude-plugin/plugin.json` 與 marketplace 檔的 `version`；以 `claude plugin validate . --strict` 檢查 marketplace。
- 以 GitHub 加入的 marketplace 是依 `version` 釘版本的，所以沒升版使用者不會收到變更。

## 尚未驗證的部分

依專案 CLAUDE.md：plugin 的 toast 在真實 session 中的呈現只有 `claude plugin test` 的覆蓋，沒有實機驗證。
