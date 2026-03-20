# 每個 ADK 開發者都應該知道的 5 種 Agent Skill 設計模式

當涉及到 `SKILL.md` 時，開發者往往專注於格式——正確設定 YAML、構建目錄並遵循規範。但隨著超過 30 種代理工具（如 Claude Code、Gemini CLI 和 Cursor）在相同的佈局上標準化，格式問題實際上已經過時了。

**現在的挑戰是內容設計**。規範解釋了如何封裝一個技能（skill），但對於**如何構建其中的邏輯**卻沒有提供任何指導。例如，一個封裝了 FastAPI 慣例的技能與一個四步文件管道的運作方式完全不同，儘管它們的 `SKILL.md` 檔案在外部看起來一模一樣。

通過研究整個生態系統中技能的構建方式——從 Anthropic 的存儲庫到 Vercel 和 Google 的內部指南——有五種重複出現的設計模式可以幫助開發者構建代理。

由 [@Saboo_Shubham_](https://x.com/@Saboo_Shubham_) 和 [@lavinigam](https://x.com/@lavinigam) 撰寫

本文涵蓋了每一種模式，並附帶可運行的 ADK 程式碼：

- **工具封裝器 (Tool Wrapper)**：讓您的代理立即成為任何函式庫的專家
- **生成器 (Generator)**：從可重複使用的模板產生結構化文件
- **審查器 (Reviewer)**：根據檢查表按嚴重程度為程式碼評分
- **反轉 (Inversion)**：代理在行動前對您進行訪談
- **管道 (Pipeline)**：強制執行帶有檢查點的嚴格多步驟工作流

## 模式 1：工具封裝器 (The Tool Wrapper)
工具封裝器為您的代理提供特定函式庫的隨選上下文。與其將 API 慣例硬編碼到系統提示中，不如將它們封裝成一個技能。您的代理**僅在實際處理該技術時才載入此上下文**。

![1](https://pbs.twimg.com/media/HDoDoIeXAAUmQoy?format=jpg&name=small)

這是最容易實現的模式。`SKILL.md` 檔案偵聽使用者提示中的特定函式庫關鍵字，從 `references/` 目錄**動態**載入您的內部文件，並將這些規則視為絕對真理。這正是您用來將團隊的內部編碼指南或特定框架最佳實踐直接分發到開發者工作流中的機制。

這是一個教導代理如何編寫 FastAPI 程式碼的工具封裝器示例。請注意指令如何明確告訴代理**僅在開始審查或編寫程式碼時**載入 `conventions.md` 檔案：

```
# skills/api-expert/SKILL.md
---
name: api-expert
description: FastAPI development best practices and conventions. Use when building, reviewing, or debugging FastAPI applications, REST APIs, or Pydantic models.
metadata:
  pattern: tool-wrapper
  domain: fastapi
---

You are an expert in FastAPI development. Apply these conventions to the user's code or question.

## Core Conventions

Load 'references/conventions.md' for the complete list of FastAPI best practices.

## When Reviewing Code
1. Load the conventions reference
2. Check the user's code against each convention
3. For each violation, cite the specific rule and suggest the fix

## When Writing Code
1. Load the conventions reference
2. Follow every convention exactly
3. Add type annotations to all function signatures
4. Use Annotated style for dependency injection
```

## 模式 2：生成器 (The Generator)
雖然工具封裝器應用知識，但**生成器強制執行一致的輸出**。如果您為代理在每次運行時產生不同的文件結構而苦惱，生成器通過**編排填空過程**來解決這個問題。

![2](https://pbs.twimg.com/media/HDoEJdZbEAEdYMo?format=jpg&name=small)

它利用了兩個可選目錄：**`assets/` 存放您的輸出模板**，而 **`references/` 存放您的風格指南**。指令充當專案經理的角色。它們告訴代理載入模板、閱讀風格指南、向使用者詢問缺失的變數並填充文件。這對於產生可預測的 API 文件、標準化提交訊息或構建專案架構腳手架非常實用。

在這個技術報告生成器示例中，技能檔案不包含實際的佈局或語法規則。**它只是協調這些資產的檢索，並強制代理逐步執行它們**：

```
# skills/report-generator/SKILL.md
---
name: report-generator
description: Generates structured technical reports in Markdown. Use when the user asks to write, create, or draft a report, summary, or analysis document.
metadata:
  pattern: generator
  output-format: markdown
---

You are a technical report generator. Follow these steps exactly:

Step 1: Load 'references/style-guide.md' for tone and formatting rules.

Step 2: Load 'assets/report-template.md' for the required output structure.

Step 3: Ask the user for any missing information needed to fill the template:
- Topic or subject
- Key findings or data points
- Target audience (technical, executive, general)

Step 4: Fill the template following the style guide rules. Every section in the template must be present in the output.

Step 5: Return the completed report as a single Markdown document.
```

## 模式 3：審查器 (The Reviewer)
審查器模式**將檢查什麼與如何檢查分開**。與其寫一段詳述每一種程式碼異味的長系統提示，不如將模組化的評分標準存儲在 `references/review-checklist.md` 檔案中。

![3](https://pbs.twimg.com/media/HDoEa51XEAIKSnO?format=jpg&name=small)

當使用者提交程式碼時，代理會載入此檢查表並有條不紊地為提交內容評分，按嚴重程度對發現結果進行分組。如果您將 Python 風格檢查表替換為 OWASP 安全檢查表，您將使用完全相同的技能基礎設施獲得一個完全不同的、專業的稽核。這是自動化 PR 審查或在人工查看程式碼之前捕獲漏洞的一種非常有效的方法。

以下程式碼審查器技能展示了這種分離。指令保持不變，但代理**從外部檢查表動態載入特定的審查標準，並強制產生結構化、基於嚴重程度的輸出**：

```
# skills/code-reviewer/SKILL.md
---
name: code-reviewer
description: Reviews Python code for quality, style, and common bugs. Use when the user submits code for review, asks for feedback on their code, or wants a code audit.
metadata:
  pattern: reviewer
  severity-levels: error,warning,info
---

You are a Python code reviewer. Follow this review protocol exactly:

Step 1: Load 'references/review-checklist.md' for the complete review criteria.

Step 2: Read the user's code carefully. Understand its purpose before critiquing.

Step 3: Apply each rule from the checklist to the code. For every violation found:
- Note the line number (or approximate location)
- Classify severity: error (must fix), warning (should fix), info (consider)
- Explain WHY it's a problem, not just WHAT is wrong
- Suggest a specific fix with corrected code

Step 4: Produce a structured review with these sections:
- **Summary**: What the code does, overall quality assessment
- **Findings**: Grouped by severity (errors first, then warnings, then info)
- **Score**: Rate 1-10 with brief justification
- **Top 3 Recommendations**: The most impactful improvements
```

## 模式 4：反轉 (Inversion)
代理天生就想立即猜測並產生。反轉模式翻轉了這種動態。與其由使用者驅動提示並由代理執行，不如讓代理充當面試官。

![4](https://pbs.twimg.com/media/HDoEo5XbEAUaaFG?format=jpg&name=small)

反轉依賴於明確的、不可協商的門控指令（如「在所有階段完成之前不要開始構建」），以強制代理**首先收集上下文**。它按順序詢問結構化問題，並在進入下一階段之前等待您的回答。在完整了解您的需求和部署約束之前，代理拒絕綜合最終輸出。

為了了解其實際運作情況，請看這個專案規劃器技能。這裡的關鍵要素是嚴格的分階段和明確的門控提示，它**阻止代理在收集到所有使用者回答之前綜合最終計劃**：

```
# skills/project-planner/SKILL.md
---
name: project-planner
description: Plans a new software project by gathering requirements through structured questions before producing a plan. Use when the user says "I want to build", "help me plan", "design a system", or "start a new project".
metadata:
  pattern: inversion
  interaction: multi-turn
---

You are conducting a structured requirements interview. DO NOT start building or designing until all phases are complete.

## Phase 1 — Problem Discovery (ask one question at a time, wait for each answer)

Ask these questions in order. Do not skip any.

- Q1: "What problem does this project solve for its users?"
- Q2: "Who are the primary users? What is their technical level?"
- Q3: "What is the expected scale? (users per day, data volume, request rate)"

## Phase 2 — Technical Constraints (only after Phase 1 is fully answered)

- Q4: "What deployment environment will you use?"
- Q5: "Do you have any technology stack requirements or preferences?"
- Q6: "What are the non-negotiable requirements? (latency, uptime, compliance, budget)"

## Phase 3 — Synthesis (only after all questions are answered)

1. Load 'assets/plan-template.md' for the output format
2. Fill in every section of the template using the gathered requirements
3. Present the completed plan to the user
4. Ask: "Does this plan accurately capture your requirements? What would you change?"
5. Iterate on feedback until the user confirms
```

## 模式 5：管道 (The Pipeline)
對於複雜的任務，您無法承受跳過的步驟或被忽視的指令。管道模式強制執行帶有硬性檢查點的嚴格順序工作流。

指令本身**充當工作流定義**。通過實施明確的金剛石門控條件（例如在從 docstring 生成轉向最終組裝之前需要使用者批准），管道**確保代理不能繞過複雜任務並呈現未經驗證的最終結果**。

![5](https://x.com/GoogleCloudTech/article/2033953579824758855/media/2033943506906189824)

此模式利用了所有可選目錄，僅在需要它們的特定步驟引入不同的參考檔案和模板，從而保持上下文視窗的清潔。

在這個文件管道示例中，請注意明確的門控條件。**明確禁止代理進入組裝階段，直到使用者在上一步中確認生成的 docstring**：

```
# skills/doc-pipeline/SKILL.md
---
name: doc-pipeline
description: Generates API documentation from Python source code through a multi-step pipeline. Use when the user asks to document a module, generate API docs, or create documentation from code.
metadata:
  pattern: pipeline
  steps: "4"
---

You are running a documentation generation pipeline. Execute each step in order. Do NOT skip steps or proceed if a step fails.

## Step 1 — Parse & Inventory
Analyze the user's Python code to extract all public classes, functions, and constants. Present the inventory as a checklist. Ask: "Is this the complete public API you want documented?"

## Step 2 — Generate Docstrings
For each function lacking a docstring:
- Load 'references/docstring-style.md' for the required format
- Generate a docstring following the style guide exactly
- Present each generated docstring for user approval
Do NOT proceed to Step 3 until the user confirms.

## Step 3 — Assemble Documentation
Load 'assets/api-doc-template.md' for the output structure. Compile all classes, functions, and docstrings into a single API reference document.

## Step 4 — Quality Check
Review against 'references/quality-checklist.md':
- Every public symbol documented
- Every parameter has a type and description
- At least one usage example per function
Report results. Fix issues before presenting the final document.
```

## 選擇正確的代理技能模式
每種模式回答不同的問題。使用此決策樹為您的用例找到合適的模式：

![Pattern Decision Tree](https://pbs.twimg.com/media/HDoFWovXAAsbb8C?format=jpg&name=small)

## 最後，模式是可以組合的
這些模式並非互斥。它們可以組合。
一個管道技能可以在最後包含一個審查器步驟來雙重檢查其自身的工作。一個生成器可以在最開始依賴反轉來收集必要的變數，然後再填充其模板。得益於 ADK 的 SkillToolset 和漸進式揭露，您的代理在運行時僅在它需要的精確模式上花費上下文令牌。

**不要再試圖將複雜且脆弱的指令擠進單個系統提示中**。分解您的工作流，應用正確的結構模式，並構建可靠的代理。

## 立即開始
代理技能規範是開源的，並在整個 ADK 中得到原生支援。您已經知道如何封裝格式。現在您知道如何設計內容。快去使用 [Google Agent Development Kit](https://google.github.io/adk-docs/) 構建更聰明的代理吧。
