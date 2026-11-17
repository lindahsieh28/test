# OpenRig 使用指南（繁體中文）

> 本文依據 [mvschwarz/openrig](https://github.com/mvschwarz/openrig) 的 README（`v0.7.17`）整理。指令與行為以官方文件為準；安裝前請先核對你使用的版本（`rig --version`）。

## 1. OpenRig 是什麼

OpenRig 是開源軟體，用來建立並管理「一組 AI 程式代理（agent）」。官方的說法是：*harness 包住模型，rig 包住你的 harness*。你用 YAML 定義代理團隊，一行指令啟動，讓 Claude Code 與 Codex 可以放在同一個 rig 裡，當成一套系統管理。

它解決的問題：把散落的終端機 session，變成有組織、可持續運作的團隊。你只要對主導代理（lead）說明想要的成果，它會協調各個專長代理，並把結果與需要你決定的事項帶回來。

## 2. 事前需求

- Node.js 22 或 24（Apple silicon 請用 22）
- tmux
- macOS 或 Linux（Windows 請用 WSL2）
- 已登入的 Claude Code 或 Codex（只要有其中一個就能開始）

安裝前請先執行 `node --version` 確認版本。

## 3. 安裝

### 方式 A：讓你的程式代理幫你安裝（官方建議）

在你的專案資料夾，把 OpenRig 的 `rigs` skill 加到你使用的代理：

```sh
# Claude Code
npx skills add https://github.com/mvschwarz/openrig/tree/v0.7.17/skills/rigs --skill rigs --agent claude-code
# Codex
npx skills add https://github.com/mvschwarz/openrig/tree/v0.7.17/skills/rigs --skill rigs --agent codex
```

接著在 Claude Code 輸入 `/rigs`（Codex 輸入 `$rigs`），說明你想做什麼。代理會先告訴你它要執行與更動什麼，再安裝 OpenRig、開啟 OpenRig 畫面，並把你的目標交給 OpenRig 的 operator。

### 方式 B：自己安裝

一鍵安裝腳本（第一行是 dry-run，只印出計畫、不做任何更動）：

```bash
curl -fsSL https://raw.githubusercontent.com/mvschwarz/openrig/v0.7.17/scripts/install.sh | sh -s -- --dry-run
curl -fsSL https://raw.githubusercontent.com/mvschwarz/openrig/v0.7.17/scripts/install.sh | sh
```

或分步驟：

```bash
npm install -g @openrig/cli
rig setup --dry-run     # 先預覽
rig setup               # 實際套用
```

若只用一個供應商，請加上 `--providers claude` 或 `--providers codex`，另一個就會顯示為「未選擇」而不是「失敗」。若 `rig setup` 回報 `INSTALLED, SETUP NOT COMPLETE`，依輸出的 Fix 處理後重跑 `rig setup` 即可，不必重裝 CLI。

安裝後檢查：

```bash
tmux -V
claude --version && claude auth status    # 使用 Claude Code 時
codex --version && codex login status     # 使用 Codex 時
```

需要時用 `claude auth login` 或 `codex login` 登入；不要為沒用到的供應商安裝或登入。

## 4. 安裝會更動你電腦的哪些東西

在你同意之前請先了解，並**事先備份**相關檔案（官方不保證能還原）：

| 時機 | 更動內容 |
| --- | --- |
| `npm install -g` | 全域安裝 `@openrig/cli`（指令 `rig`、`openrig-tui`） |
| `rig setup` | 在 `~/.tmux.conf` 加入 OpenRig 區塊；可能安裝 tmux（macOS 用 Homebrew）與 Herdr（可用 `--no-herdr` 略過）；可能安裝缺少的 Claude Code / Codex |
| 啟動 daemon | 在 `~/.openrig`（`OPENRIG_HOME`）建立狀態與資料庫；把探索用 skill 放進 `~/.claude/skills`、`~/.agents/skills` |
| 啟動代理 | Codex：`~/.codex/config.toml` 的 hooks 與信任設定；Claude Code：`~/.claude.json` 的工作區信任，以及專案內 `.claude/settings.local.json` 的 hooks 與 status line |
| 在專案啟動團隊 | 在 `AGENTS.md` 或 `CLAUDE.md` 加入受管理的指引，以及團隊的 skills 與 plugins |

**權限提醒：** 預設下，OpenRig 自己的 kernel 權限較廣（Codex 為 `-s danger-full-access -a never`；Claude Code 的檔案工具、網頁擷取與操作指令，包括讀取家目錄，不會逐一詢問）。專案團隊預設較保守：一般 `rig` 指令、讀取專案與常見測試不會詢問，但 `rig up`、`rig down` 這類生命週期指令仍會詢問。`workshop` 團隊則附帶較寬的權限。YOLO 預設為**關閉**。

完整清單請見官方 README 的「What OpenRig changes on your machine」。

## 5. 第一次使用

安裝完成後，代理會幫你開啟 OpenRig 畫面。之後若要回到畫面：

```sh
rig daemon start
rig terminal open saved:kernel --window
```

與 **operator** 對話就算完成安裝。operator 會詢問你要做什麼，從三種團隊挑一個推薦，並配合你有的供應商調整。

| 團隊 | 對話對象 | 組成與用途 |
| --- | --- | --- |
| `starter` | `dev-build@starter` | Claude Code 建置者 + Codex 審查者，處理一個明確的小變更 |
| `workshop` | `orch-lead@workshop` | lead、建置、QA、審查，適合單一儲存庫的持續工作（rig bundle，由 operator 從固定版本的清單安裝） |
| `factory` | `orch-lead@factory` | 七位代理（lead、顧問、建置、QA、設計、兩位獨立審查），適合長期產品開發 |

`starter` 預設同時用到 Claude Code 與 Codex；若你只有其中一個，請 operator 幫你改成適合的版本。

## 6. 手動啟動 starter 團隊

```bash
cd /path/to/your/repository
rig specs preview starter --kind rig     # 預覽
rig up starter --cwd . --plan            # 看啟動計畫
rig up starter --cwd .                   # 真正啟動
```

確認各座位（seat）就緒，並先處理任何登入、信任或權限提示：

```bash
rig ps --nodes --rig starter
```

若某個座位卡在提示，等你回應後可用 `rig seat continue <seat>` 把啟動內容送進去。接著給建置者**一個明確的目標**：

```bash
rig send dev-build@starter 'Implement <one useful change>. Track the task in the queue and return its ID. Keep it local, verify the behavior, ask dev-review in this rig to check the exact candidate, and record the result and how I can try it.'
rig queue list --destination dev-build@starter --limit 1000
```

注意：送出訊息本身不會建立 queue 項目，是建置者去記錄任務。請閱讀最終成果與審查結果，再回到同一位建置者進行下一個變更。

## 7. 常用指令速查

| 目的 | 指令 |
| --- | --- |
| 啟動 / 還原 rig | `rig up <name>` |
| 停止並存快照 | `rig down --snapshot` |
| 查看狀態 | `rig ps`、`rig tui`（團隊儀表板） |
| 傳訊息給代理 | `rig send <seat> '<訊息>'`、`rig broadcast`、`rig chatroom` |
| 動態調整 | `rig grow`、`rig shrink`、`rig launch`、`rig remove` |
| 探索並納管既有 tmux session | `rig discover`、`rig adopt` |
| 瀏覽團隊範本庫 | `rig specs ls`、`rig specs preview <name> --kind rig` |
| 開啟多個終端機視窗 | `rig terminal open <rig> --provider herdr`（或 `cmux`） |
| 診斷 | `rig doctor`、`rig setup --dry-run` |
| 座位權限 | `rig policy permissions list\|show\|current\|apply`、`rig seat set-permissions <seat> --mode <mode> --reason <text>` |

在 TUI 內可用 `rig tui commands` 列出命令列導覽。

## 8. 核心概念

- **RigSpec**：以 YAML 宣告的多代理團隊定義（pods、成員、edges、延續策略、culture file）。
- **AgentSpec**：可重複使用的代理藍圖（skills、指引、hooks、profiles、啟動契約）。
- **Seat（座位）**：rig 中穩定的角色與位址，例如 `dev-build@starter`；佔用座位的對話可以更換，身分與既有脈絡不變。
- **Pod**：一組相關座位，共享指引與脈絡；每位代理仍有各自的 context window。
- **Discovery**：`rig discover` 辨識既有 tmux session，`rig adopt` 納入管理。
- **Snapshot / Restore**：`rig down --snapshot` 保存完整狀態，`rig up <name>` 從最新快照還原；還原時會逐一回報 resumed、fresh-primed、awaiting-decision（原對話無法續接，需選 `--fresh`）、attention_required 或 failed。

架構概觀：CLI / TUI / MCP → Hono HTTP daemon → 領域服務 → SQLite + tmux + runtime adapters。每個終端機座位都是你可以 attach、檢查的 tmux session。

## 9. 常見狀況

- **安裝失敗或權限被擋：** 這不是 OpenRig 的安裝結果。請讓代理說出被擋的指令與原因，再針對該步驟處理。
- **看不到畫面：** 桌面環境可能需要允許 Automation；SSH / 無顯示環境請自行開新終端機視窗，再向代理索取連線指令。畫面太小（如 80×24）會擠壓；Herdr 首次執行可能出現引導面板。備援方式是 `rig tui --shared`（只有儀表板，沒有 operator 對話）。
- **權限提示太多：** 可考慮 `workshop`（權限較寬，並支援 non-interruptive mode），但請先看清楚它提供的存取範圍。
- **關閉檢視視窗：** 關掉終端機視窗不代表要重新啟動團隊。
- **npm 11 顯示 `npm warn install-scripts`：** 屬預期現象，只是略過了 Node.js / SQLite 檢查。可自行執行 `node "$(npm root -g)/@openrig/cli/scripts/check-abi.mjs"`。

## 10. 延伸資源

- 官方 README：<https://github.com/mvschwarz/openrig>
- 入門指南：`docs/reference/quickstart.md`、`docs/reference/getting-started.md`（官方儲存庫內）
- 問題排解：`docs/reference/help.md`
- 問答討論區：<https://github.com/mvschwarz/openrig/discussions/92>
- 影片：<https://www.youtube.com/@openrig>
