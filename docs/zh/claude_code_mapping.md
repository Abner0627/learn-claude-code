# 各章節與 Claude Code 對應機制總覽

| 章節 | 核心概念 | Claude Code 對應 | 支援程度 | 差距 |
| :--- | :--- | :--- | :--- | :--- |
| s01 | Agent Loop | 完全內建，透明運作 | ✅ | 使用者無需實作，CC 自動管理 LLM 呼叫循環 |
| s02 | Tool Use / Dispatch Map | 內建工具集 + MCP 自訂工具 | ✅ | CC 工具已沙盒化；自訂工具透過 MCP Server 新增 |
| s03 | Todo / Task 系統 | TodoWrite / TodoRead 工具 | ✅ | 無 Nag Reminder 注入機制，但 CC 有內建行為約束 |
| s04 | Subagent（子代理） | `Agent` 工具 | ✅ | 一次性、上下文隔離、做完即棄，行為完全對應 |
| s05 | Skill Loading | `Skill` 工具 + `.claude/` 目錄 | ✅ | 兩層懶加載模式相同；定義位置：`.claude/skills/` 或 superpowers |
| s06 | Context Compact | `/compact` 指令 + 自動壓縮 | ⚠️ | 有 auto/manual compact；無 micro_compact（逐條工具輸出替換） |
| s07 | Task System（DAG） | `TaskCreate/Update/List/Get` 工具 | ⚠️ | 有持久化任務管理；但無 `blockedBy/blocks` 依賴圖原生支援 |
| s08 | Background Tasks | Bash 工具 `run_in_background` | ⚠️ | 有背景執行；無 notification queue 自動注入機制 |
| s09 | Agent Teams | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | ⚠️ | 實驗性、功能不完整；穩定替代：多個 Subagent 平行執行 |
| s10 | Team Protocols（關機 + 計劃審批） | 無直接對應 | ⚠️ | Shutdown → 子代理生命週期結束；Plan Approval → Hooks 或 `--permission-mode` |
| s11 | Autonomous Agents（自主認領） | 無原生輪詢機制 | ❌ | 近似做法：TodoWrite 任務板 + Subagent + Skill 引導工作流 |
| s12 | Worktree Task Isolation | `Agent` 工具 `isolation: "worktree"` | ✅ | 穩定內建；s12 比 CC 多：任務板雙向綁定、JSONL 事件日誌、崩潰恢復 |

---

## 各章節詳述

### s01 — Agent Loop（代理迴圈）

**概念**：`while True` 迴圈驅動 LLM 呼叫 → 工具執行 → 結果回饋，直到 `stop_reason != "tool_use"` 為止。

**Claude Code 對應**：✅ **完全內建**

Claude Code 本身就是這個 Agent Loop 的生產實作。使用者與 Claude Code 互動時，所有 LLM 呼叫、工具執行、歷史追加均由 CC 自動管理，無需手動實作。

**使用方式**：無需任何設定。直接使用 Claude Code 即運行在 Agent Loop 之上。

**差距**：
- s01 讓開發者理解底層機制，CC 將其完全封裝
- CC 的迴圈邏輯不可自訂（例如無法更改 stop_reason 判斷邏輯）
- 若需要完全自訂的 Loop 行為，需使用 Anthropic SDK 自行實作（如 s01 示範）

---

### s02 — Tool Use（工具使用）

**概念**：Tool Dispatch Map — 將工具名稱對映到 Handler 函式，透過 `safe_path()` 實作路徑沙盒。新增工具不需修改 Agent Loop。

**Claude Code 對應**：✅ **內建工具集 + MCP 自訂擴充**

CC 提供完整的內建工具集，已實作路徑安全與沙盒：

| s02 概念 | Claude Code 對應工具 |
| :--- | :--- |
| `read_file` | `Read` 工具 |
| `write_file` | `Write` 工具 |
| `edit_file`（精確替換） | `Edit` 工具 |
| `bash` | `Bash` 工具 |
| 檔案搜尋 | `Glob` 工具 |
| 內容搜尋 | `Grep` 工具 |

**自訂工具**（新增 Handler + Schema）→ 透過 **MCP Server** 擴充：

```bash
# 透過 CLI 新增（儲存至 ~/.claude.json，預設 local scope）
claude mcp add --transport stdio my-tools -- npx my-mcp-server

# 或新增為專案範圍（儲存至 .mcp.json）
claude mcp add --scope project --transport stdio my-tools -- npx my-mcp-server
```

**差距**：
- CC 工具的路徑沙盒由平台層強制執行，比 s02 的 `safe_path()` 更嚴格
- 無法像 s02 一樣任意修改工具的 Handler 邏輯；自訂工具必須透過 MCP 協定

---

### s03 — Todo / Task 系統

**概念**：`TodoManager` 維護帶狀態的任務清單（pending/in_progress/completed），限制同時只有一個 `in_progress`；搭配 **Nag Reminder**（連續 3 輪未呼叫 todo 工具時自動注入提醒）。

**Claude Code 對應**：✅ **TodoWrite / TodoRead 工具內建**

CC 提供 `TodoWrite` 和 `TodoRead` 工具，功能與 s03 的 `TodoManager` 直接對應：

```
# TodoWrite 建立/更新任務清單
# TodoRead 讀取當前清單狀態
# 狀態：pending / in_progress / completed
# CC 強制執行：同時只有一個 in_progress 任務
```

**使用方式**：直接在對話中要求 Claude Code 規劃多步驟任務，它會自動使用 TodoWrite 建立並追蹤進度。

**差距**：
- s03 的 **Nag Reminder**（自動注入 `<reminder>Update your todos.</reminder>`）在 CC 中沒有對應機制
- CC 透過模型訓練（而非程式碼注入）來維持任務追蹤習慣，效果因任務複雜度而異
- 若需要強制 Nag 行為，可透過 **Hooks** 在每次工具呼叫後注入提醒

---

### s04 — Subagent（子代理）

**概念**：派生獨立的 Agent 執行子任務，上下文完全隔離，完成即棄。

**Claude Code 對應**：✅ **`Agent` 工具（穩定內建）**

```
# 定義位置：.claude/agents/<name>.md
# description 欄位含 "Use proactively" → 自動觸發
# 每個 Subagent 獲得獨立、乾淨的對話歷史
# 完成後僅回傳摘要，不污染主對話
```

**使用方式**：
1. 在 `.claude/agents/` 建立 `<name>.md` 檔案
2. 設定 `description` 欄位描述觸發條件
3. 主代理使用 `Agent` 工具委派任務

**差距**：行為幾乎完全對應 s04。

---

### s05 — Skill Loading（技能懶加載）

**概念**：兩層注入策略 — Layer 1 在 System Prompt 只放技能名稱（低成本），Layer 2 在需要時透過 `load_skill` 工具按需注入完整 SOP（約 2000 tokens）。

**Claude Code 對應**：✅ **`Skill` 工具 + `.claude/` 目錄**

CC 的技能系統與 s05 的兩層懶加載模式完全對應：

| s05 概念 | Claude Code 實作 |
| :--- | :--- |
| `skills/` 目錄 + `SKILL.md` | `.claude/` 目錄中的 `.md` 技能檔案 |
| Layer 1：技能名稱列於 System Prompt | `<system-reminder>` 中列出可用技能名稱 |
| Layer 2：`load_skill` 按需載入 | `Skill` 工具按需載入完整內容 |
| `rglob("SKILL.md")` 掃描 | CC 平台自動掃描技能目錄 |

**使用方式**：
```
# 呼叫技能
Skill tool → skill: "my-skill-name"
```

**差距**：
- s05 的 `SkillLoader` 允許完全自訂掃描路徑與格式
- CC 的技能路徑由平台決定，開發者只能在指定目錄下建立技能檔
- s05 的 `load_skill` 是顯式工具呼叫；CC 的 Skill 工具功能相同但由助理決定何時觸發

---

### s06 — Context Compact（上下文壓縮）

**概念**：三層壓縮管線：
1. **Micro Compact**：每輪自動將超過 3 輪的舊 `tool_result` 替換為佔位符
2. **Auto Compact**：Token 超過閾值時，LLM 生成摘要並重置 `messages`
3. **Manual Compact**：模型主動呼叫 `compact` 工具

**Claude Code 對應**：⚠️ **有 auto/manual compact，但無 micro_compact**

| s06 層 | Claude Code 對應 | 支援 |
| :--- | :--- | :--- |
| Micro Compact | ❌ 無直接對應 | ❌ |
| Auto Compact | 對話接近 Context 上限時自動壓縮 | ✅ |
| Manual Compact | `/compact` 指令 | ✅ |

**使用方式**：
```bash
# 手動壓縮
/compact

# 自動壓縮：無需設定，CC 在 context 接近上限時自動觸發
```

**差距**：
- **Micro Compact 缺失**：CC 不會逐條替換舊 tool_result 為佔位符，而是等到接近上限才整體壓縮
- 實際效果：CC 的壓縮策略比 s06 更「粗糙」，中間步驟資訊可能消耗更多 token
- 若需要精細的 micro_compact 行為，需自行在工具 Hooks 中實作

---

### s07 — Task System（依賴管理 DAG）

**概念**：持久化磁碟任務圖（DAG）。每個任務為 JSON 檔，含 `blockedBy` / `blocks` 雙向依賴，完成任務時自動解鎖下游任務，跨對話、跨壓縮持久存活。

**Claude Code 對應**：⚠️ **有持久化任務工具，但無原生 DAG 依賴管理**

CC 提供 `TaskCreate`、`TaskUpdate`、`TaskGet`、`TaskList`、`TaskOutput`、`TaskStop` 工具，支援持久化任務管理：

```
TaskCreate  → 建立新任務
TaskUpdate  → 更新任務狀態
TaskList    → 列出所有任務
TaskGet     → 取得單一任務詳情
TaskOutput  → 取得任務輸出
TaskStop    → 停止任務
```

**使用方式**：
```
# 在對話中透過工具操作
TaskCreate(title="Refactor DB", description="...")
TaskUpdate(id="123", status="completed")
TaskList()
```

**差距**：
- CC 的 Task 工具**不支援** `blockedBy` / `blocks` 依賴圖
- 無自動依賴解鎖機制（`_clear_dependency`）
- 無法查詢「現在可以執行的任務」（需 `blockedBy == []`）

**替代方案**：
若需要 DAG 依賴管理，可透過以下方式實現：
1. 使用 JSON 檔案手動維護依賴關係（如 s07 的 `.tasks/` 目錄）
2. 讓 Agent 在每輪開始時讀取任務檔並手動計算可執行集合

---

### s08 — Background Tasks（背景任務）

**概念**：`BackgroundManager` 以守護執行緒非同步執行 Shell 指令，結果透過通知佇列在下一輪 LLM 呼叫前自動注入 Context。

**Claude Code 對應**：⚠️ **有背景執行，但無自動通知注入**

CC 的 `Bash` 工具支援 `run_in_background` 參數：

```python
# Claude Code 的背景執行
Bash(command="pytest long_tests/", run_in_background=True)
# → 立即返回，背景繼續跑
# → 完成時通知主對話（需使用者或工具主動查詢）
```

**使用方式**：
```
# 在工具呼叫中指定背景執行
Bash tool → run_in_background: true
# → CC 會在任務完成時自動通知
```

**差距**：
- s08 的 `notification_queue` 在每輪 LLM 呼叫前**自動排空並注入結果**
- CC 的背景任務完成時會通知，但注入時機的精細控制不如 s08
- s08 的 500 字元截斷、300 秒超時保護需自行在 Prompt 或 Hooks 中設定

---

### s09 — Agent Teams（持久隊員 + 信箱通訊）

**概念**：多個持久存活的 Agent 透過 Append-Drain JSONL 信箱互相通訊，支援跨 session 的隊員狀態。

**Claude Code 對應**：⚠️ **實驗性，預設關閉**

```bash
# 啟用方式
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

**已知限制**：
- `/resume` 後隊員狀態消失
- 無 Append-Drain JSONL 信箱機制
- 功能不完整，不建議生產使用

**穩定替代方案**：多個 Subagent 平行執行（s04 模式），各自回傳摘要。

---

### s10 — Team Protocols（關機 + 計劃審批協議）

**概念**：Shutdown Protocol（隊員優雅關機）+ Plan Approval Protocol（執行前等待人工審批）。

**Claude Code 對應**：⚠️ **無直接對應，有近似機制**

| s10 概念 | Claude Code 近似機制 |
| :--- | :--- |
| Shutdown Protocol | Subagent 生命週期自然結束（做完即棄） |
| Plan Approval Protocol | `--permission-mode auto`（背景分類器）或手動 Human-in-the-loop Hooks |

**替代方案**：

- 透過 Hooks 實作計劃審批
- `settings.json` 中設定 PreToolUse hook，在特定工具執行前暫停並請求確認

---

### s11 — Autonomous Agents（自主認領任務）

**概念**：空閒 Agent 主動掃描任務板，認領未分配任務，自主輪詢執行。

**Claude Code 對應**：❌ **無原生輪詢與自主認領機制**

**推薦替代方案**：

```
主代理（精簡）
  → TodoWrite 作為任務板
  → 逐一建立 Subagent（s04 模式）執行各任務
  → Skill 引導工作流程與品質標準
```

**注意事項**：
- `/loop` skill 可重複執行，但每次迭代共享同一 session context，**不適合**多小時長任務（context 無限累積）
- 最省 Token 架構：主代理保持精簡 + 高噪音操作全部委派給隔離子代理
- Claude Code 自動啟用 Prompt Caching（靜態內容節省約 90%），但無法阻止對話歷史增長

---

### s12 — Worktree Task Isolation（目錄隔離）

**概念**：每個任務綁定獨立 Git Worktree，物理隔離防止檔案衝突，支援任務板雙向綁定與 JSONL 事件日誌。

**Claude Code 對應**：✅ **`isolation: "worktree"`（穩定內建）**

```python
# Agent 工具中使用 worktree 隔離
Agent(
    subagent_type="general-purpose",
    isolation="worktree",
    prompt="..."
)
```

**行為**：
- 每個 Subagent 獲得獨立實體目錄
- 無改動 → 自動清除 worktree
- 有改動 → 保留並回傳 worktree 路徑與分支名稱
- 物理上不可能發生跨任務的檔案衝突

**注意**：Agent Teams 不會自動套用 Worktree 隔離，需在每個 Agent 定義檔中顯式加入 `isolation: worktree`。

**差距**（s12 比 CC 多的功能）：
- 任務板雙向綁定（任務與 worktree 路徑互相記錄）
- JSONL 事件日誌（每個操作都有可稽核的日誌）
- 崩潰恢復能力（Agent 重啟後能從 worktree 狀態還原）

---

## 快速決策指引

### 我想要... → 使用什麼

| 需求 | 推薦做法 |
| :--- | :--- |
| 讓 AI 自動執行多步驟任務 | 直接使用 Claude Code（Agent Loop 已內建） |
| 新增自訂工具 | MCP Server（`claude mcp add` CLI 指令或 `.mcp.json`） |
| 追蹤多步驟任務進度 | `TodoWrite` / `TodoRead`（直接使用） |
| 委派子任務並隔離上下文 | `Agent` 工具（s04 模式） |
| 按需載入領域知識 | 建立 Skill 檔 + `Skill` 工具 |
| 壓縮過長的對話 | `/compact` 指令 |
| 持久化任務（跨 session） | `TaskCreate/Update/List` 工具 |
| 有依賴關係的任務圖 | 手動維護 JSON 任務檔 + Agent 讀取計算 |
| 非阻塞背景執行 | `Bash` 工具 `run_in_background: true` |
| 多代理平行執行 | 多個 `Agent` 工具呼叫（並行 subagent） |
| 物理隔離的檔案工作區 | `Agent` 工具 `isolation: "worktree"` |
| 自主輪詢認領任務 | TodoWrite 任務板 + 逐一 Subagent（s11 替代方案） |

---

## 常見問題 Q&A

### 1. 想新增自訂工具，直接更新 `.claude/settings.json` 就好了嗎？

不是。**`settings.json` 無法用來定義自訂工具**，MCP Server 的設定有專屬的儲存位置，且新增工具需要兩個步驟：**準備 MCP Server** + **向 Claude Code 註冊**。

#### 步驟一：準備 MCP Server

MCP Server 是實際執行工具邏輯的程式，可以是：

| 類型 | 說明 | 適用情境 |
| :--- | :--- | :--- |
| **遠端 HTTP Server** | 透過 URL 連接的現有服務 | 使用第三方 MCP 服務（如 Notion、Asana） |
| **本地 Stdio Server** | 本機執行的程式（npx / Python / 執行檔） | 自訂工具、私有 API 封裝 |

不需要從零寫一個完整 Server；如果需求簡單，可以用現有的 MCP library 快速包裝現有函式。

#### 步驟二：向 Claude Code 註冊

透過 `claude mcp add` CLI 指令（**不是** `settings.json`）：

```bash
# 本地 Stdio Server（最常用）
claude mcp add --transport stdio my-tools -- npx my-mcp-server

# 遠端 HTTP Server
claude mcp add --transport http notion https://mcp.notion.com/mcp

# 指定為專案範圍（設定存入 .mcp.json，可提交至 git）
claude mcp add --scope project --transport stdio my-tools -- npx my-mcp-server
```

#### 設定儲存位置

| Scope | 儲存位置 | 指令 |
| :--- | :--- | :--- |
| local（預設） | `~/.claude.json` | `claude mcp add` |
| project | `.mcp.json`（專案根目錄） | `claude mcp add --scope project` |
| user | `~/.claude.json` | `claude mcp add --scope user` |

`.mcp.json` 可以提交至 git，讓團隊共用相同的工具設定。`settings.json` 則是用來設定 Claude Code 的行為選項（如 hooks、permission mode），與工具定義無關。

#### 其他管理指令

```bash
claude mcp list          # 列出所有已註冊的 Server
claude mcp get <name>    # 查看特定 Server 的詳細資訊
claude mcp remove <name> # 移除 Server
/mcp                     # 在 Claude Code session 中查看目前連線狀態
```

### 2. MCP Server 是什麼？如何建立遠端 HTTP Server 和本地 Stdio Server？

#### MCP Server 是什麼

**MCP（Model Context Protocol）** 是開放標準協定，讓 Claude Code 等 LLM 應用能與外部工具和資料來源整合。底層使用 **JSON-RPC 2.0** 格式進行訊息交換。

每個 MCP Server 可以提供三種原語（Primitive）：

| 原語 | 說明 | 比喻 |
| :--- | :--- | :--- |
| **Tools** | 模型可呼叫的函式（如查詢天氣、讀資料庫） | Agent 的「手腳」 |
| **Resources** | 唯讀資料（檔案、API 回應） | Agent 的「眼睛」 |
| **Prompts** | 可重複使用的提示模板 | Agent 的「腳本」 |

Claude Code 作為 **Host** 啟動時，會向每個 MCP Server 發送 `tools/list` 請求取得可用工具清單，之後在需要時呼叫 `tools/call` 執行。

---

#### 遠端 HTTP Server

適用情境：工具需要從外部存取（跨機器、雲端部署、第三方服務）。

**核心要求**：
- 暴露**單一 HTTP 端點**（例如 `https://example.com/mcp`）
- 支援 POST 方法接收 JSON-RPC 請求
- 可選：回傳 `Mcp-Session-Id` header 維持有狀態 session

**請求格式**（Claude Code → Server）：

```http
POST /mcp HTTP/1.1
Content-Type: application/json
Accept: application/json, text/event-stream

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "location": "Tokyo" }
  }
}
```

**回應格式**（Server → Claude Code）：

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      { "type": "text", "text": "Tokyo: Sunny, 25°C" }
    ]
  }
}
```

**認證方式**：

| 方式 | 適用場景 | 實作難度 |
| :--- | :--- | :--- |
| OAuth 2.1（PKCE） | 公開服務、需要使用者授權 | 高 |
| API Key（Bearer Token） | 私有/內部服務 | 低 |

API Key 範例（在 Claude Code 的 `.mcp.json` 中設定 header）：

```json
{
  "mcpServers": {
    "my-api": {
      "type": "http",
      "url": "https://example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${MY_API_KEY}"
      }
    }
  }
}
```

向 Claude Code 註冊：

```bash
claude mcp add --transport http my-api https://example.com/mcp
```

---

#### 本地 Stdio Server（用官方 SDK 快速包裝現有函式）

適用情境：本機工具、私有邏輯、不需要網路存取。Stdio Server 以子進程形式啟動，透過 stdin/stdout 和 Claude Code 通訊。

**Python — FastMCP（最快，推薦初學）**

```bash
pip install mcp
```

核心設計：用 `@mcp.tool()` 裝飾器包裝現有函式，**自動從 type hints 生成 JSON Schema**，無需手動定義。

```python
# weather_server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Weather Server")

# 直接包裝現有函式：只需加裝飾器
@mcp.tool()
def get_weather(location: str) -> str:
    """Get current weather for a location."""
    # 這裡可以是任何現有邏輯：呼叫 API、查資料庫、讀檔案…
    return f"Weather for {location}: Sunny, 25°C"

@mcp.tool()
def analyze_text(text: str, language: str = "zh") -> dict:
    """Analyze text and return metrics."""
    return {
        "length": len(text),
        "word_count": len(text.split()),
        "language": language,
    }

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

FastMCP 會自動將 `get_weather(location: str) -> str` 轉換為：

```json
{
  "name": "get_weather",
  "description": "Get current weather for a location.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "location": { "type": "string" }
    },
    "required": ["location"]
  }
}
```

**TypeScript — 官方 SDK（需要更精細控制時）**

```bash
npm install @modelcontextprotocol/sdk zod
```

```typescript
// server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "my-server", version: "1.0.0" });

server.registerTool(
  {
    name: "get_weather",
    description: "Get current weather for a location",
    inputSchema: z.object({
      location: z.string().describe("City name or zip code"),
    }),
  },
  async ({ location }) => ({
    content: [{ type: "text", text: `Weather for ${location}: Sunny, 25°C` }],
  })
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

**本地測試**：

```bash
# Python：使用官方 MCP Inspector（瀏覽器 UI 測試工具）
mcp dev weather_server.py

# TypeScript
npx tsx src/server.ts
```

**向 Claude Code 註冊**：

```bash
# Python
claude mcp add --transport stdio weather -- python /path/to/weather_server.py

# TypeScript (已編譯)
claude mcp add --transport stdio weather -- node /path/to/dist/server.js

# TypeScript (直接用 npx 執行未發佈的本地套件)
claude mcp add --transport stdio weather -- npx tsx /path/to/server.ts
```

---

#### 兩種 Server 的選擇指引

| 維度 | 遠端 HTTP Server | 本地 Stdio Server |
| :--- | :--- | :--- |
| **部署位置** | 雲端或遠端機器 | 本機，與 Claude Code 同機 |
| **啟動方式** | 獨立運行的 HTTP 服務 | Claude Code 啟動時自動作為子進程啟動 |
| **適合場景** | 第三方服務整合、團隊共用工具 | 個人工具、私有 API、快速原型 |
| **認證需求** | 需要（API Key 或 OAuth） | 無（本機進程，天然安全） |
| **開發難度** | 較高（需部署、管理服務） | **低**（FastMCP 裝飾器即可） |

### 3. Hooks 的作用是什麼？

#### 核心概念

Hooks 是 Claude Code 的**確定性自動化機制**。LLM 的行為本質上是機率性的，Hooks 讓你在特定生命週期事件發生時強制執行確定性邏輯，不依賴模型的判斷。

用一句話說：**Claude 決定做什麼，Hooks 決定能不能做、做完要執行什麼**。

#### Hook 的運作流程

```
使用者輸入 → [UserPromptSubmit Hook]
     ↓
Claude 決定使用工具 → [PreToolUse Hook] → 可阻擋
     ↓
工具執行
     ↓
[PostToolUse Hook] → 執行副作用（格式化、日誌…）
     ↓
Claude 完成回覆 → [Stop Hook]
```

每個 Hook 接收 **JSON（stdin）**，透過 **exit code + stdout/stderr** 回傳決定：

| Exit Code | 意義 | 輸出用途 |
| :--- | :--- | :--- |
| `0` | 允許 / 成功 | stdout → 注入 Claude 的上下文（或 JSON 決策） |
| `2` | **阻擋**此動作 | stderr → 傳給 Claude 作為阻擋原因 |
| 其他 | 非阻擋性錯誤 | stderr → 僅在 verbose 模式顯示 |

#### 主要 Hook 事件

| 事件 | 觸發時機 | 可阻擋？ | 常見用途 |
| :--- | :--- | :--- | :--- |
| `SessionStart` | Session 啟動或從 compact 恢復 | 否 | 注入專案背景知識 |
| `UserPromptSubmit` | 使用者送出 prompt | 是 | 驗證 / 補充 prompt 內容 |
| `PreToolUse` | 工具執行**前** | **是** | 阻擋危險指令、保護檔案 |
| `PostToolUse` | 工具執行成功後 | 否 | 自動格式化、寫日誌 |
| `PermissionRequest` | 權限確認視窗即將出現 | 是 | 自動核准 / 拒絕特定操作 |
| `Stop` | Claude 完成回覆 | 是 | 強制跑測試後才能停止 |
| `Notification` | Claude 需要使用者注意 | 否 | 桌面推播通知 |
| `PostCompact` | Context 壓縮完成後 | 否 | 重新注入被壓縮掉的關鍵資訊 |

#### 設定格式

Hooks 設定在 `.claude/settings.json`（專案）或 `~/.claude/settings.json`（全域）：

```json
{
  "hooks": {
    "事件名稱": [
      {
        "matcher": "工具名稱的 Regex（空字串 = 全符合）",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/script.sh",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

Hook 腳本透過 stdin 接收事件的 JSON 資料：

```json
{
  "session_id": "abc123",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "rm -rf dist/" }
}
```

#### 範例 1：阻擋危險指令

```bash
#!/bin/bash
# .claude/hooks/guard.sh
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if echo "$CMD" | grep -qE "rm -rf|DROP TABLE|DROP DATABASE"; then
  echo "已阻擋：此指令不允許在本專案執行" >&2
  exit 2   # exit 2 = 阻擋，stderr 傳給 Claude
fi

exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": ".claude/hooks/guard.sh" }]
      }
    ]
  }
}
```

**結果**：Claude 收到 stderr 的原因說明，可調整做法後重試。

#### 範例 2：Context 壓縮後重新注入資訊

```bash
#!/bin/bash
# .claude/hooks/reinject.sh
cat << 'EOF'
【重新注入：Context 壓縮後的關鍵資訊】
- 套件管理器：Bun（禁用 npm）
- 測試指令：bun test
- 目前任務：Auth 系統重構
EOF
exit 0   # stdout 文字會被注入 Claude 的上下文
```

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [{ "type": "command", "command": ".claude/hooks/reinject.sh" }]
      }
    ]
  }
}
```

**結果**：Context 壓縮後重啟 session 時，Claude 自動讀到補充資訊。

#### 範例 3：編輯後自動格式化

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write 2>/dev/null; exit 0",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

**結果**：Claude 每次修改 JS/TS 檔後自動執行 Prettier，無需提醒。

#### Hooks 與 MCP 的差異

| 維度 | Hooks | MCP Tools |
| :--- | :--- | :--- |
| **觸發方式** | 生命週期事件自動觸發 | Claude 主動決定呼叫 |
| **能否阻擋** | ✅ 可以（exit 2） | ❌ 不行（執行後才回報） |
| **用途** | 強制規則、副作用、監控 | 擴充 Claude 的能力 |
| **執行者** | Claude Code 平台（確定性） | Claude 模型（機率性決策） |

**簡單判斷**：「我想限制 Claude 的行為」→ Hook；「我想讓 Claude 能做更多事」→ MCP Tool。

### 4. s07 的 DAG 依賴管理在 Claude Code 中如何用替代方案實現？

#### 問題背景

Claude Code 的 `TaskCreate/Update/List` 工具只支援基本狀態（建立、更新、列出），沒有 `blockedBy` / `blocks` 依賴欄位，也沒有自動解鎖下游任務的機制。

替代方案的核心思路是：**把 s07 的 `TaskManager` 邏輯拆成兩部分，分別由「JSON 檔案」和「Claude 自身的讀取 + 推理」來承擔。**

---

#### 第一步：用 JSON 檔案手動維護依賴關係

在專案中建立 `.tasks/` 目錄，每個任務用一個 JSON 檔表示。這些檔案存在磁碟上，不受 context 壓縮影響。

**單一任務的 JSON 格式**（仿照 s07）：

```json
{
  "id": "task_002",
  "title": "Update API endpoints",
  "status": "pending",
  "blockedBy": ["task_001"],
  "blocks": ["task_003"],
  "owner": null,
  "notes": ""
}
```

| 欄位 | 說明 |
| :--- | :--- |
| `id` | 唯一識別碼，慣例用 `task_001`、`task_002` 等 |
| `status` | `pending` / `in_progress` / `completed` |
| `blockedBy` | 「我必須等哪些任務完成才能開始」的 ID 列表 |
| `blocks` | 「我完成後哪些任務可以解鎖」的 ID 列表（反向索引，加速查詢） |
| `owner` | 認領此任務的 Agent 名稱（多代理場景使用） |

**一個完整的任務組範例**（auth 系統重構，四個任務含依賴）：

```
.tasks/
  task_001.json   ← Refactor DB schema（無依賴，可立即執行）
  task_002.json   ← Update API endpoints（等 task_001）
  task_003.json   ← Write unit tests（等 task_001）
  task_004.json   ← Update documentation（等 task_002 和 task_003）
```

```json
// task_001.json
{ "id": "task_001", "title": "Refactor DB schema",
  "status": "pending", "blockedBy": [], "blocks": ["task_002", "task_003"] }

// task_002.json
{ "id": "task_002", "title": "Update API endpoints",
  "status": "pending", "blockedBy": ["task_001"], "blocks": ["task_004"] }

// task_003.json
{ "id": "task_003", "title": "Write unit tests",
  "status": "pending", "blockedBy": ["task_001"], "blocks": ["task_004"] }

// task_004.json
{ "id": "task_004", "title": "Update documentation",
  "status": "pending", "blockedBy": ["task_002", "task_003"], "blocks": [] }
```

依賴關係視覺化：

```
task_001
├── task_002 ──┐
└── task_003 ──┴── task_004
```

---

#### 第二步：讓 Agent 每輪開始時讀取並計算可執行集合

s07 的 `TaskManager` 有自動計算邏輯，在 Claude Code 中改由 Claude 自己執行這個邏輯。關鍵是透過 **CLAUDE.md** 給出明確的工作流程指示。

**`.claude/CLAUDE.md` 中加入的規則**：

```markdown
## 任務管理規則

每次開始工作前，必須執行以下流程：

1. 讀取 `.tasks/` 目錄下所有 `task_*.json`
2. 找出「可立即執行」的任務：status = "pending" 且 blockedBy = []
3. 從中選一個，將其 status 改為 "in_progress"
4. 執行任務
5. 完成後：
   a. 將此任務的 status 改為 "completed"
   b. 讀取此任務的 "blocks" 欄位，找出所有下游任務
   c. 對每個下游任務，從其 "blockedBy" 陣列中移除剛完成的 task ID
   d. 儲存所有修改過的 JSON 檔

禁止跳過步驟 5b-5d，否則下游任務永遠不會被解鎖。
```

**執行一個完整循環的範例**：

```
[讀取階段]
Claude 使用 Read 工具讀取所有 .tasks/*.json

[計算階段]
Claude 判斷：
  task_001 → status=pending, blockedBy=[] → ✅ 可執行
  task_002 → status=pending, blockedBy=["task_001"] → ❌ 被阻擋
  task_003 → status=pending, blockedBy=["task_001"] → ❌ 被阻擋
  task_004 → status=pending, blockedBy=["task_002","task_003"] → ❌ 被阻擋

[執行階段]
Claude 選擇 task_001，用 Edit 工具將 status 改為 "in_progress"
→ 執行 DB schema 重構工作
→ 完成後將 status 改為 "completed"

[解鎖階段]
Claude 讀取 task_001.blocks = ["task_002", "task_003"]
→ 修改 task_002.json：blockedBy 從 ["task_001"] 改為 []
→ 修改 task_003.json：blockedBy 從 ["task_001"] 改為 []

[下一輪]
task_002 → blockedBy=[] → ✅ 可執行
task_003 → blockedBy=[] → ✅ 可執行（可平行）
```

---

#### 與 s07 原始實作的差距比較

| 功能 | s07 TaskManager | CC 替代方案 |
| :--- | :--- | :--- |
| 依賴關係存儲 | ✅ JSON 檔（相同） | ✅ JSON 檔 |
| 自動解鎖下游 | ✅ `_clear_dependency()` 自動執行 | ⚠️ 依賴 Claude 遵循 CLAUDE.md 指示 |
| 跨 session 持久 | ✅（磁碟） | ✅（磁碟） |
| 計算可執行集合 | ✅ 程式碼保證正確 | ⚠️ Claude 推理，偶爾可能出錯 |
| 多代理並行認領 | ✅ `owner` 欄位 + 原子寫入 | ⚠️ 可用 `owner` 欄位，但無原子鎖 |

**什麼時候這個替代方案夠用**：
- 任務數量少（< 20 個）
- 依賴關係簡單（不超過 3 層）
- 單一 Agent 執行（不需要多代理並行搶任務）

**什麼時候應該改用 s07 的完整實作**（自訂 Python TaskManager）：
- 依賴圖複雜、層數多
- 多個 Subagent 並行，需要原子性認領（避免兩個 Agent 同時搶到同一個任務）
- 對正確性有高要求，不能接受 Claude 偶爾漏執行解鎖步驟

### 5. s11 的「自主認領任務」替代方案如何實際運作？

#### 問題背景

s11 的核心是 Agent 在「閒置時主動輪詢任務板、認領未分配任務並執行」。Claude Code 沒有背景輪詢機制，無法讓 Agent 自主等待並認領工作。

替代方案改變了**驅動方式**：從「Agent 主動輪詢」改為「主代理顯式迭代並委派」。三個元件各司其職：

| 元件 | s11 對應 | 作用 |
| :--- | :--- | :--- |
| `TodoWrite` | 任務板（共享狀態） | 記錄所有任務的標題與狀態 |
| `Agent` 工具（Subagent） | 執行任務的工作 Agent | 獨立上下文執行單一任務，完成後回傳摘要 |
| Skill | 工作 SOP | 告訴 Subagent 如何處理特定類型的任務 |

---

#### 完整執行流程

```
Step 1：主代理接收任務清單，用 TodoWrite 建立任務板
Step 2：主代理逐一（或批次）選出 pending 任務
Step 3：對每個任務，用 Agent 工具派出 Subagent 執行
Step 4：Subagent 完成後回傳摘要，主代理用 TodoWrite 標記完成
Step 5：重複 Step 2～4 直到所有任務完成
```

**視覺化**：

```
主代理
  │
  ├─ TodoWrite([任務A pending, 任務B pending, 任務C pending])
  │
  ├─ Agent(任務A) ──────────────────────────────► 執行 A，回傳摘要
  │     完成後 TodoWrite(任務A → completed)
  │
  ├─ Agent(任務B) ──────────────────────────────► 執行 B，回傳摘要
  │     完成後 TodoWrite(任務B → completed)
  │
  └─ Agent(任務C) ──────────────────────────────► 執行 C，回傳摘要
        完成後 TodoWrite(任務C → completed)
```

---

#### 為什麼要用 Subagent 而不是主代理直接做？

這是整個替代方案的核心設計決策：

**直接在主代理做（不推薦）**：
```
主代理執行任務A（讀大量檔案，輸出工具結果）
→ 執行任務B（再讀更多檔案）
→ 執行任務C（context 已很長）
→ Context 膨脹，速度變慢，注意力下降
```

**委派給 Subagent（推薦）**：
```
主代理保持精簡（只有任務清單 + 摘要）
Subagent A：獨立 context，做完任務A，回傳 100 字摘要
Subagent B：獨立 context，做完任務B，回傳 100 字摘要
Subagent C：獨立 context，做完任務C，回傳 100 字摘要
→ 主代理 context 始終精簡
```

Subagent 的 context 完全隔離，執行完畢後連同所有工具呼叫記錄一起丟棄，不污染主代理。

---

#### Skill 的角色

Skill 解決的問題是：Subagent 每次都是全新的 context，它怎麼知道「這個任務應該怎麼做」？

做法是在 `Agent` 工具的 `prompt` 中明確引用 Skill，或在 Subagent 定義檔的 `description` 中指定應使用的 Skill：

**方式 A：在 prompt 中引用（臨時）**

```
Agent(
  prompt="""
  請處理以下任務：替換 auth.py 中的 JWT 驗證邏輯。

  執行前請先使用 code-review Skill 確認修改範圍。
  執行後請使用 testing Skill 確認測試覆蓋率。

  完成後回傳：已修改的檔案列表 + 測試結果摘要。
  """
)
```

**方式 B：在 Agent 定義檔中指定（複用）**

```markdown
---
# .claude/agents/refactor-agent.md
name: refactor-agent
description: 負責程式碼重構任務的 Agent
---

執行重構任務前，使用 code-review Skill 分析影響範圍。
執行後使用 testing Skill 確認無回歸問題。
回傳格式：已修改檔案 + 變更摘要 + 測試結果。
```

Skill 的作用是把「怎麼做」從每次的 prompt 中抽出來，統一定義在一個地方，讓所有 Subagent 遵循相同的品質標準。

---

#### 順序執行 vs 平行執行

**順序執行**（任務間有依賴，或需要人工確認）：

```
主代理逐一等待每個 Subagent 完成後再派下一個
優點：可以根據前一個任務的結果調整後續計劃
缺點：慢
```

**平行執行**（任務彼此獨立）：

```
主代理在同一輪對話中同時發出多個 Agent 工具呼叫
優點：快，與 s11 的多 Agent 並行最接近
缺點：無法根據某一任務的結果動態調整其他任務
```

平行執行的寫法（在同一輪發出多個 Agent 呼叫）：

```
Agent(prompt="重構 auth.py")       ← 同時發出
Agent(prompt="重構 payment.py")    ← 同時發出
Agent(prompt="重構 user.py")       ← 同時發出
→ 三個 Subagent 並行執行，全部完成後主代理收到三份摘要
```

若任務有依賴，可結合 Q&A 4 的 JSON 任務檔方案：先平行執行無依賴的任務，等它們完成後再解鎖並平行執行下一批。

---

#### 與 s11 的差距

| 維度 | s11 原始實作 | CC 替代方案 |
| :--- | :--- | :--- |
| **驅動方式** | Agent 主動輪詢（IDLE 狀態等待） | 主代理顯式迭代委派 |
| **新任務感知** | Agent 自動偵測並認領 | 需要重新啟動主代理迴圈 |
| **多 Agent 並行** | 多個 Agent 同時從任務板認領 | 主代理在同一輪同時派出多個 Subagent |
| **長時間運行** | Agent 可持續等待數小時 | 受限於單次 session 長度 |
| **Context 效率** | ✅ 每個 Agent 獨立 | ✅ 每個 Subagent 獨立（相同優勢） |

**什麼情況下替代方案夠用**：任務清單在開始前已知、不需要 Agent 長時間待命等待外部事件觸發新任務。

**什麼情況下需要 s11 的完整實作**：需要 Agent 在背景持續運行、監控外部事件（如 CI 失敗、新 PR 開啟）並自動認領處理。

### 6. 如何把 s07 的完整 Python 實作整合進 Claude Code？

#### 拆解 s07 的結構

讀完 `s07_task_system.py` 可以看到它分兩層：

| 層 | 程式碼位置 | 作用 | 在 CC 中的對應 |
| :--- | :--- | :--- | :--- |
| **核心邏輯** | `TaskManager` 類別（第 47～124 行） | DAG 依賴計算、自動解鎖、持久化 | **保留** → 包裝成 MCP |
| **執行 Harness** | `agent_loop` + `TOOL_HANDLERS` + `TOOLS`（第 179～248 行） | 驅動獨立 Agent 運行 | **替換** → Claude Code 本身 |

整合的核心思路：**只取 `TaskManager`，用 FastMCP 重新包裝成 Claude Code 可呼叫的工具。**

---

#### 建立 MCP Server（`task_mcp_server.py`）

在專案根目錄新增此檔，從 s07 複製 `TaskManager` 類別，再用 FastMCP 裝飾器包裝四個工具：

```python
# task_mcp_server.py
import json
from pathlib import Path
from mcp.server.fastmcp import FastMCP

# ── 直接複製 s07 的 TaskManager（第 47～124 行），無需修改 ──
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1

    def _max_id(self) -> int:
        ids = [int(f.stem.split("_")[1]) for f in self.dir.glob("task_*.json")]
        return max(ids) if ids else 0

    def _load(self, task_id: int) -> dict:
        path = self.dir / f"task_{task_id}.json"
        if not path.exists():
            raise ValueError(f"Task {task_id} not found")
        return json.loads(path.read_text())

    def _save(self, task: dict):
        path = self.dir / f"task_{task['id']}.json"
        path.write_text(json.dumps(task, indent=2))

    def create(self, subject: str, description: str = "") -> str:
        task = {
            "id": self._next_id, "subject": subject, "description": description,
            "status": "pending", "blockedBy": [], "blocks": [], "owner": "",
        }
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)

    def get(self, task_id: int) -> str:
        return json.dumps(self._load(task_id), indent=2)

    def update(self, task_id: int, status: str = None,
               add_blocked_by: list = None, add_blocks: list = None) -> str:
        task = self._load(task_id)
        if status:
            if status not in ("pending", "in_progress", "completed"):
                raise ValueError(f"Invalid status: {status}")
            task["status"] = status
            if status == "completed":
                self._clear_dependency(task_id)
        if add_blocked_by:
            task["blockedBy"] = list(set(task["blockedBy"] + add_blocked_by))
        if add_blocks:
            task["blocks"] = list(set(task["blocks"] + add_blocks))
            for blocked_id in add_blocks:
                try:
                    blocked = self._load(blocked_id)
                    if task_id not in blocked["blockedBy"]:
                        blocked["blockedBy"].append(task_id)
                        self._save(blocked)
                except ValueError:
                    pass
        self._save(task)
        return json.dumps(task, indent=2)

    def _clear_dependency(self, completed_id: int):
        for f in self.dir.glob("task_*.json"):
            task = json.loads(f.read_text())
            if completed_id in task.get("blockedBy", []):
                task["blockedBy"].remove(completed_id)
                self._save(task)

    def list_all(self) -> str:
        tasks = []
        for f in sorted(self.dir.glob("task_*.json")):
            tasks.append(json.loads(f.read_text()))
        if not tasks:
            return "No tasks."
        lines = []
        for t in tasks:
            marker = {"pending": "[ ]", "in_progress": "[>]", "completed": "[x]"}.get(t["status"], "[?]")
            blocked = f" (blocked by: {t['blockedBy']})" if t.get("blockedBy") else ""
            lines.append(f"{marker} #{t['id']}: {t['subject']}{blocked}")
        return "\n".join(lines)
# ── TaskManager 結束 ──

TASKS = TaskManager(Path(".tasks"))
mcp = FastMCP("Task System")

@mcp.tool()
def task_create(subject: str, description: str = "") -> str:
    """建立新任務。自動分配 ID，初始狀態為 pending。"""
    return TASKS.create(subject, description)

@mcp.tool()
def task_list() -> str:
    """列出所有任務及其狀態與被阻擋情形。"""
    return TASKS.list_all()

@mcp.tool()
def task_get(task_id: int) -> str:
    """取得單一任務的完整 JSON 詳情。"""
    return TASKS.get(task_id)

@mcp.tool()
def task_update(task_id: int, status: str = None,
                add_blocked_by: list[int] = None,
                add_blocks: list[int] = None) -> str:
    """更新任務狀態或依賴關係。
    status: pending / in_progress / completed
    完成時自動從所有下游任務的 blockedBy 中移除此 ID。
    """
    return TASKS.update(task_id, status, add_blocked_by, add_blocks)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

---

#### 向 Claude Code 註冊

```bash
# 安裝依賴（只需一次）
pip install mcp

# 以專案範圍註冊（設定存入 .mcp.json，可提交至 git）
claude mcp add --scope project --transport stdio task-system -- python task_mcp_server.py
```

重啟 Claude Code session 後，可用 `/mcp` 確認連線狀態。

---

#### Claude Code 現在擁有什麼能力

註冊後，Claude Code 可以直接呼叫這四個工具，行為與 s07 原版完全相同：

```
task_create("Refactor DB schema")
→ 建立 task_1.json，status=pending，blockedBy=[]

task_update(task_id=2, add_blocked_by=[1])
→ task_2.blockedBy = [1]，同時更新 task_1.blocks = [2]（雙向自動維護）

task_update(task_id=1, status="completed")
→ task_1 標記完成
→ _clear_dependency(1) 自動執行：從 task_2.blockedBy 移除 1
→ task_2.blockedBy 變為 []，解鎖

task_list()
→ [x] #1: Refactor DB schema
→ [ ] #2: Update API endpoints   ← 已解鎖，可執行
```

**關鍵差異**：`_clear_dependency()` 是 Python 程式碼保證執行，不依賴 Claude 有沒有遵循 CLAUDE.md 的指示。

---

#### 兩種整合路線比較

| | Q&A 4 的替代方案（JSON 手動維護） | Q&A 6 的完整實作（MCP 包裝） |
| :--- | :--- | :--- |
| **自動解鎖** | ❌ Claude 手動執行，可能漏 | ✅ `_clear_dependency()` 程式碼保證 |
| **雙向依賴同步** | ❌ 需 Claude 同時更新兩個檔案 | ✅ `add_blocks` 自動同步 `blockedBy` |
| **設定成本** | 低（只需 CLAUDE.md） | 中（需寫 MCP Server + 安裝 mcp） |
| **適用任務數** | < 20 個，依賴關係簡單 | 不限，依賴複雜亦可 |
| **適合場景** | 快速原型、一次性任務 | 長期專案、正式工作流 |

### 7. 改用 Skill 並把 s07 腳本放進 Skill 資料夾，和 MCP 方案哪個比較好？

#### 先釐清：Skill 能做什麼

Skill 的本質是 **SOP 文字**（`SKILL.md` 的內容），不是執行能力。Claude 讀完 Skill 後，仍需透過自己擁有的工具（`Bash`、`Read`、`Write`…）來執行動作。

因此「Skill + 腳本」方案的執行路徑是：

```
Skill + 腳本方案
Claude 載入 SKILL.md（讀到「用 Bash 呼叫 task_cli.py」的指示）
  → Claude 決定呼叫 Bash 工具
    → Bash 生成新 Python 子進程
      → Python 執行 TaskManager 邏輯
        → 輸出結果作為 Bash 的 stdout 回傳 Claude

MCP 方案
Claude 呼叫 task_create 工具（直接）
  → MCP 協定傳遞到常駐子進程
    → Python 執行 TaskManager 邏輯
      → 結果作為 tool_result 回傳 Claude
```

這個路徑差異決定了兩個方案在可靠性上的根本不同。

---

#### Skill + CLI 腳本方案的完整樣貌

需要對 s07 做兩件事：

**① 將 s07 改寫為 CLI 工具（`task_cli.py`）**

s07 原本是互動式 REPL，需加上 `argparse` 讓它能被 Bash 呼叫：

```python
# .claude/skills/task-system/task_cli.py
import argparse, sys
from pathlib import Path
# [貼入 TaskManager 類別，完全不改]

TASKS = TaskManager(Path(".tasks"))

p = argparse.ArgumentParser()
sub = p.add_subparsers(dest="cmd")

c = sub.add_parser("create")
c.add_argument("subject"); c.add_argument("description", nargs="?", default="")

u = sub.add_parser("update")
u.add_argument("task_id", type=int); u.add_argument("--status")
u.add_argument("--blocked-by", type=int, nargs="*")
u.add_argument("--blocks", type=int, nargs="*")

sub.add_parser("list")

g = sub.add_parser("get")
g.add_argument("task_id", type=int)

args = p.parse_args()
if args.cmd == "create":   print(TASKS.create(args.subject, args.description))
elif args.cmd == "update": print(TASKS.update(args.task_id, args.status, args.blocked_by, args.blocks))
elif args.cmd == "list":   print(TASKS.list_all())
elif args.cmd == "get":    print(TASKS.get(args.task_id))
else: p.print_help(); sys.exit(1)
```

**② 撰寫 `SKILL.md` 告訴 Claude 怎麼呼叫**

```markdown
---
name: task-system
description: Manage tasks with DAG dependency tracking using task_cli.py
---

## 任務管理 SOP

建立任務：
  Bash: python .claude/skills/task-system/task_cli.py create "任務標題"

列出任務（含依賴狀態）：
  Bash: python .claude/skills/task-system/task_cli.py list

完成任務（自動解鎖下游）：
  Bash: python .claude/skills/task-system/task_cli.py update <id> --status completed

設定依賴（task_id 被 blocked_by 阻擋）：
  Bash: python .claude/skills/task-system/task_cli.py update <id> --blocked-by <前置id>
```

---

#### 可靠性對比：關鍵差距

| 維度 | Skill + CLI | MCP |
| :--- | :--- | :--- |
| **工具介面** | Claude 自行組裝 Bash 字串 | typed JSON schema，CC 驗證參數 |
| **錯誤處理** | Bash 回傳 stdout/stderr，Claude 自行判斷 | MCP 協定層標準化錯誤 |
| **參數出錯** | argparse 報錯 → Claude 看到 stderr → 可能重試錯誤方向 | schema 驗證失敗 → CC 攔截，不執行 |
| **腳本路徑** | Claude 必須知道正確的絕對/相對路徑 | 註冊時固定，Claude 不需知道路徑 |
| **Python 邏輯保證** | ✅ `_clear_dependency()` 照常執行 | ✅ `_clear_dependency()` 照常執行 |

兩個方案都能保證 `_clear_dependency()` 執行（因為都是跑 Python），**差距在於呼叫層的可靠性**：MCP 有類型驗證和協定保障；Skill + CLI 依賴 Claude 正確解讀 SKILL.md 並組裝出正確的 Bash 指令。

#### 效率對比

| 維度 | Skill + CLI | MCP |
| :--- | :--- | :--- |
| **每次呼叫** | 生成新 Python 子進程（~100ms 啟動） | IPC 到常駐子進程（< 10ms） |
| **Token 額外消耗** | 載入 SKILL.md 需消耗 token | 只有 tool schema（在每輪固定成本內） |
| **Skill 載入時機** | 按需（懶加載），不用時為 0 | 始終存在（但 schema 很小） |

任務管理屬於低頻操作，每次呼叫的 100ms 啟動時間幾乎無感。實務上效率差距可以忽略。

---

#### 結論：選哪個？

```
需要 _clear_dependency() 保證 + 不想安裝 mcp 套件
  → Skill + CLI（設定最簡單，可靠性夠用）

需要最高可靠性 + 工具介面乾淨 + 長期專案
  → MCP（類型驗證、協定保障、無路徑問題）

只是快速原型或任務數 < 20
  → Q&A 4 的 JSON 手動方案（零依賴，CLAUDE.md 幾行搞定）
```

**一句話**：Skill 擅長傳遞「怎麼做」的知識；MCP 擅長提供「能做什麼」的能力。對 s07 這種有明確 CRUD 介面的工具，MCP 比 Skill 更適合，因為問題從來不是 Claude 不知道怎麼管理任務，而是執行層需要程式碼保證。
