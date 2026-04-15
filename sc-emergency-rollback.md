---
description: 緊急 rollback/restore SC (ask-wt / Side Conversation) 功能。Ryan 出國時若 SC 壞了，WT/Dans 可用 Phase 1 關掉 SC；Ryan 回國後叫 Claude load 這份 skill，執行 Phase 2 全部復原
---

# SC Emergency Rollback & Restore

## Context

Ryan 從 2026-04-13 開始建置 Devin ↔ WT Side Conversation loop（ask-wt），包含 n8n workflow、Zendesk trigger/webhook/views、Devin skills、KB buffer。

出國期間若此功能造成問題（重複 SC、loop、客人沒回信等），WT 或 Dans 執行 **Phase 1** 緊急 rollback。Ryan 回國後 Claude load 此 skill 執行 **Phase 2** 還原。

---

## Full Inventory（所有被加入/改動的東西）

### n8n Workflow #1: SC Workflow `Fv1YYRI6cVIsz3nG`
**名稱**: `Devin -Side Conversation`
**URL**: https://n8n-support.pg-internal.homes/workflow/Fv1YYRI6cVIsz3nG

**主要區塊**（所有都是 SC 功能的一部分）：
- 🚀 SC Reply Receiver — webhook `/devin-sc-reply`
- 🤖 MCP Server — SC MCP trigger + tools (ask_wt, Create SC1, Follow Up SC, Get SC)
- 🎯 Ask WT sub-workflow — webhook `/devin-ask-wt`
- 🎯 Check WT Status sub-workflow — 給 Devin main 呼叫
- 🧪 POC: Manual Create SC (`/devin-sc-test`)
- 🧪 POC: Manual Reply SC (`/devin-sc-followup`)

### n8n Workflow #2: Devin Main `at6fIDol0ZzhjrXf`
**URL**: https://n8n-support.pg-internal.homes/workflow/at6fIDol0ZzhjrXf

SC 相關新增 nodes：
- `Side Conversation Tools`（MCP Client，連到 Devin 3.0 agent）
- `Check WT Status`（executeWorkflow 呼叫 SC workflow 的 sub-workflow）
- `WT Action Router`（switch node：skip/bypass/continue）
- `5.4-mini (TE)`（Tool Executor 專用 LLM，取代原本 gpt-4o-mini）
- `Trigger Devin` 是 SC workflow 裡的（不是這裡），但會 POST 到這裡的 Webhook1

**被修改的現有 nodes**:
- `Already Replied?` TRUE 分支原本連到別處，被改到 `Post Processor`
- `Tool Executor` 的 ai_languageModel 連線從 `gpt-4o-mini` 換成 `5.4-mini (TE)`

### Zendesk
| 資源 | ID | 用途 |
|------|-----|------|
| Webhook | `01KP2ZGW3BKHFPZECH0JRPM218` | Trigger 打這個通知 SC Reply Receiver |
| Trigger | `45040281751053` | 「Devin - SC Reply Notify」— waiting-wt tag + SC replied 時 fire |
| View | `45040460913165` | SC - Waiting WT |
| View | `45040469979277` | SC - Received (WT Answered) |

### Devin Skills (`dans-huang/devin-skills` repo)

**新建檔案**（可以直接刪）:
- `skills/ask-wt.md`
- `skills/team-slack-ids.md`

**被修改的既有檔案**（要 revert commits）:
- `skills/_index.json` — 加了 ask-wt 和 team-slack-ids 兩個 entry
- `skills/troubleshooting.md` — 加了 `received-wt = Skip Troubleshooting` hard rule + routing 改動
- `skills/ticket-resolution.md` — 加了 `waiting-wt = Stop`、`received-wt = Answer Ready`、`票件移交摘要 ONLY for Escalation` 三條 hard rule
- `skills/ticket-resolution/escalation-templates.md` — 加了 `Escalation = Open` hard rule
- `skills/warranty-service.md` — spare part trigger 擴充（PCB, board 等）
- `bundle.json` — `tool_executor_prompt` 強化

**SC-related commits（按時間倒序，最新在上）**:
```
fb1abfc  Expand spare parts trigger: PCB, board, chip, speaker etc.
c376adf  Hard rule: 票件移交摘要 only for escalate-to-human scenarios
bcc303b  Strengthen tool_executor_prompt against silent tool-call skipping
c38bfa4  Tighten ask-wt: Email 2+ only, require customer push back
8b67ff7  received-wt: follow JIRA/KB references in WT answer before replying
fe4ac0a  Add waiting-wt prerequisite check to prevent duplicate SC
aea194e  Require JIRA check before ask-wt; bug/malfunction → bug-investigation first
fd5f898  Remove "KB silence" hard rule — was overriding category routing
0362234  Fix loop: add received-wt check to troubleshooting + ask-wt prereqs
9e06e3b  Update ask-wt SC format: Chinese, structured fields, include SN
a81daab  Add hard rules: waiting-wt = stop, received-wt = answer ready
5a9bc39  Add hard rule: never assume not-supported from KB silence
33a54ef  Add ask-wt skill for KB-gap escalation to WT via Side Conversation
3ad9772  Add team Slack IDs reference skill
7c8cc25  Add hard rule: escalation internal note must use Submit ticket as open
fdf1cd4  Make file sharing HC link mandatory when requesting media
```

**Pre-SC baseline commit**: `8ac05ac` (Update NEO TX repair step 2 image) — 2026-04-13 之前，沒有任何 SC 改動。

### KB (`positivegrid-support/support-km` repo)

**新建檔案**:
- `internal/Devin WT Knowledge Buffer.md`

---

## PHASE 1: Emergency Rollback (WT / Dans 執行)

目標：讓 Devin 行為回到「ask-wt 不存在」的狀態，暫時停止 SC 功能。

### Step 1: 斷開 Devin Main 的 SC tool（最快止血）

打開 https://n8n-support.pg-internal.homes/workflow/at6fIDol0ZzhjrXf

找到 `Side Conversation Tools` node（MCP Client Tool）→ 右鍵 → **Disable**（或直接刪掉該 node）。

*完成這步後 Devin 就不能呼叫 ask_wt 了，SC 停止生成。*

### Step 2: 停用 Zendesk Trigger

https://positivegrid.zendesk.com/admin/objects-rules/rules/triggers

搜尋 `Devin - SC Reply Notify`（id: 45040281751053）→ 停用。

*這防止 WT 在 SC 回覆時觸發 n8n workflow。*

### Step 3: 停用 SC Workflow

打開 https://n8n-support.pg-internal.homes/workflow/Fv1YYRI6cVIsz3nG → 右上角 toggle → **Inactive**。

### Step 4: 清理 Devin Main 的 sub-workflow 呼叫

在 Devin Main workflow，找 `Check WT Status`（executeWorkflow node）和 `WT Action Router`（switch node）。

**不要刪**，但 Loop Tickets 的 output 1 要改回直接連到 `Check Devin Reply`（跳過 Check WT Status 和 WT Action Router）。

---

### Step 5（可選，完整清除）: Revert Devin Skills

如果 Devin 還是有奇怪行為（可能因為 skill 指向已停用的功能），執行 git revert：

```bash
cd /Users/ryanc/devin-skills
git checkout 8ac05ac -- skills/ bundle.json  # 退到 SC 前的狀態
# 確認：skills/ask-wt.md 和 skills/team-slack-ids.md 被刪、其他改動被還原
git commit -m "Emergency rollback: remove SC/ask-wt skill changes"
git push
```

*注意：這會同時 revert 某些可能還想保留的改動（如 spare parts trigger、票件移交摘要 rule）。如果要保留其中一些，需要手動 cherry-pick。*

### Step 6: 驗證

- 開一張測試票 → Devin 應該不會發 SC 到 #supt-private
- Check WT Status 不會被呼叫
- `waiting-wt` / `received-wt` tag 不會再出現在新票上

---

## PHASE 2: Restore (Ryan 回國後，Claude 執行)

**前提**：WT 已卸掉手動處理 SC 的工作流程，Ryan 請 Claude load 此 skill。

### Step 1: Reactivate SC Workflow

打開 https://n8n-support.pg-internal.homes/workflow/Fv1YYRI6cVIsz3nG → 右上 toggle → **Active**。

驗證 webhook endpoints 可達：
```bash
curl -s -i "https://n8n-support.pg-internal.homes/mcp/2ca7c0bd-c17a-49e8-9f07-fac47aa95ed6" --max-time 3 | head -3
# 應該回 HTTP/2 200, content-type: text/event-stream
```

### Step 2: Enable Zendesk Trigger

https://positivegrid.zendesk.com/admin/objects-rules/rules/triggers

找 `Devin - SC Reply Notify` → 啟用。

### Step 3: Re-connect Side Conversation Tools in Devin Main

打開 https://n8n-support.pg-internal.homes/workflow/at6fIDol0ZzhjrXf

找到 `Side Conversation Tools`（MCP Client Tool）→ 啟用 → 確認它接到 `Devin 3.0` agent 的 `ai_tool` input（**不是接到 Tool Executor**）。

SSE endpoint 應該是：`https://n8n-support.pg-internal.homes/mcp/2ca7c0bd-c17a-49e8-9f07-fac47aa95ed6`（不帶 `/sse`）。

### Step 4: Restore Loop Tickets Routing

在 Devin Main，確認 Loop Tickets 的 output 1 連到 `Check WT Status`（不是直接接 Check Devin Reply）。

完整路徑：`Loop Tickets → Check WT Status → WT Action Router → [skip/bypass/continue]`。

- skip → 回到 Loop Tickets（loop back）
- bypass → Read Zendesk Ticket（received-wt 的票跳過 Already Replied）
- continue → Check Devin Reply（正常路徑）

### Step 5: Restore Skill Files（如果 Phase 1 做了 git revert）

```bash
cd /Users/ryanc/devin-skills
git pull  # 確保最新

# 套回所有 SC 相關 commits（如果已被 revert）
# 最乾淨的方式：cherry-pick 各個 commit
for sha in fdf1cd4 7c8cc25 3ad9772 33a54ef 5a9bc39 a81daab 9e06e3b 0362234 fd5f898 aea194e fe4ac0a 8b67ff7 c38bfa4 bcc303b c376adf fb1abfc; do
  git cherry-pick $sha
done
git push
```

或更簡單：git reset 回最新 SC-enabled 的 commit：
```bash
git reset --hard fb1abfc  # 最新 SC commit
git push -f origin master  # 注意 -f
```

### Step 6: 驗證完整 loop

測試票（跟 4/14 驗證時的步驟一樣）：

1. 建一張 Devin 回不了的 product 問題 ticket
2. 客人回一次 push back
3. Devin 應該呼叫 ask_wt → 發 SC 到 #supt-private tag WT
4. 在 SC 回「To Devin: ...」
5. 確認：internal note + tag swap `waiting-wt` → `received-wt` + Devin 回客人 + KB buffer 更新

### Step 7: 通知

Slack #supt-ai channel 發一則告知 WT/Dans「SC 功能已 restore」。

---

## 常見問題

**Q: Phase 1 後 Devin 會繼續運作嗎？**
A: 會。Devin 只是少了 ask_wt 這個 tool，其他所有功能正常。

**Q: Phase 1 後 Zendesk 上還有 waiting-wt / received-wt 的票怎辦？**
A: 這些票 Devin 會以為是普通 tag 忽略。WT 想手動處理就人工接手；不處理的話，客人若再回信，會正常觸發 Devin（因為 tag 不再改變 routing）。

**Q: Phase 2 前要確認什麼？**
A: 
1. n8n 和 Zendesk credentials 都還在有效
2. `support-km` repo 的 `Devin WT Knowledge Buffer.md` 仍存在
3. WT 已通知會開始收 SC
4. Ryan 親自在場或已授權 Claude 執行

**Q: Zendesk Views 要刪嗎？**
A: 不用。View 只是篩選器，裡面沒有資料會過期。保留著待 restore 時直接用。

---

## Key Files Reference

| 路徑 | 內容 |
|------|------|
| `~/.claude/projects/-Users-ryanc/memory/project_ask_wt_loop.md` | Ryan 端完整 project 狀態 |
| `~/.claude/skills/ask-wt-operations.md` | Claude Code 操作 SC 系統的指南 |
| `/Users/ryanc/devin-skills/skills/ask-wt.md` | Devin 的 ask-wt skill（被 Devin 即時 load） |
| `/Users/ryanc/support-km/internal/Devin WT Knowledge Buffer.md` | KB buffer 檔 |

---

## Commit Timeline Reference

**SC 開發期間（2026-04-13 to 2026-04-15）**的所有 SC 相關改動。詳見 `full inventory` 的 commit list。Phase 1/2 皆以此 commit list 為依據。

Pre-SC baseline: `8ac05ac` (2026-04-13 之前)
