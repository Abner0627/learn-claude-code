# s05_skill_loading.py 執行流程詳解

本章介紹了「隨選知識（On-demand Knowledge）」的注入機制，透過兩層式注入 (Two-layer Injection) 來優化系統提示詞 (System Prompt) 的大小。

### 初始化階段 (System Prompt 構建)

1.  **第一層注入 (Layer 1)**：
    *   `SkillLoader` 掃描 `skills/` 目錄。
    *   將所有技能的「名稱」與「簡短描述」注入 System Prompt。
    *   範例：`"pdf: Process PDF files..."`。

---

### 第一輪迴圈 (Iteration 1: 請求詳細知識)

1.  **使用者請求**：`"Extract all tables from data.pdf."`
2.  **模型發覺需要特定技能**：模型在提示詞中看到了 `pdf` 這個技能，但並不知道具體操作 SOP。
3.  **發出 `load_skill` 請求**：
    ```json
    {
        "type": "tool_use",
        "name": "load_skill",
        "input": {"name": "pdf"}
    }
    ```

---

### 第二輪迴圈 (Iteration 2: 注入詳細內容)

1.  **第二層注入 (Layer 2)**：
    *   工具回傳 `SKILL.md` 的完整 Body 內容（包含詳細步驟、最佳實踐）。
    *   這些詳細指引以 `tool_result` 的形式傳入 `messages`。
2.  **獲得新知**：模型現在完整理解了處理 PDF 的 SOP（例如：先讀取 Meta、使用特定套件解析等）。
3.  **開始工作**：模型依據新注入的知識，決定下一步的工具呼叫。

### 核心總結
本章展示了 **「延遲加載知識」** 的價值。透過先傳遞 metadata，只有在模型真正需要時才注入完整 SOP，有效地將單次對話的 Token 成本降至最低。

---

## SkillLoader 機制詳解

### 技能目錄結構

每個技能以一個目錄表示，目錄中包含 `SKILL.md` 檔案：

```
skills/
  pdf/
    SKILL.md         # 包含 YAML frontmatter + 正文
  code-review/
    SKILL.md
  agent-builder/
    SKILL.md
```

`SKILL.md` 的格式如下：

```yaml
---
name: pdf
description: Process PDF files -- extract tables, text, metadata
---

# PDF Processing Workflow

Step 1: Read file metadata...
Step 2: Extract content using the appropriate library...
```

`SkillLoader` 透過 `rglob("SKILL.md")` 遞歸掃描整個 `skills/` 目錄，並以目錄名作為技能識別符（除非 frontmatter 中有 `name` 欄位覆蓋）。

### 兩層注入的 Token 成本分析

| 注入層 | 內容 | Token 消耗（約估） |
| :--- | :--- | :--- |
| Layer 1（系統提示） | 所有技能的名稱與簡短描述 | ~5-10 token / 技能 |
| Layer 2（tool_result） | 單一技能的完整 SOP | ~500-3000 token / 技能 |

以 10 個技能、每個完整內容 2000 token 為例：
- **全塞進 System Prompt**：每次對話固定多消耗 20,000 token，即使只用到 1 個技能。
- **兩層按需加載**：System Prompt 只多 ~100 token，用到哪個技能才付那個技能的費用。

這種「懶加載（Lazy Loading）」策略在技能數量增多時效益尤為明顯。

### 與 s04 子代理的設計哲學比較

| 維度 | s04 子代理 (Subagent) | s05 技能加載 (Skill Loading) |
| :--- | :--- | :--- |
| **解決問題** | 對話歷史過長 | 系統提示過胖 |
| **隔離單位** | 對話歷史 | 知識內容 |
| **觸發方式** | 父代理主動委派 | 模型識別需求後自主呼叫 |
| **生命週期** | 一次性：做完即丟棄 | 持久存在於 `tool_result` 中直到對話結束 |

### 關鍵設計原則：循環不變

如同 s02，新增 `load_skill` 工具**不需要修改 agent loop**。工具只是加入 dispatch map：

```python
TOOL_HANDLERS = {
    # ...base tools...
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

技能加載的結果以 `tool_result` 回傳，對循環來說與任何其他工具呼叫結果無異。

---

## 常見問題 Q&A
