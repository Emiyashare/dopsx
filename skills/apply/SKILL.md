---
name: apply
description: "Carry out a dopsx document job (执行文书工作): produce the documents on its checklist, save final versions to documents/<job>/, tick our own items, and pause at items owned by the client or an office. Use when the user wants to start or continue the document work of a planned job."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.16.0).
metadata:
  author: dopsx
  version: "1.1.0"
---

Carry out the checklist of a dopsx document job.

dopsx is the document edition of OpenSpec's `/opsx`: the same openspec CLI underneath, but the work is documents, never code. Final versions go to `<planningHome.root>/documents/<name>/`; drafts and working files stay in the job folder. Write everything the user reads in the user's language (Chinese unless the user writes in another language).

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.16.0` (requires Node.js 22.12 or newer; dopsx is tested with openspec 1.16.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

**Input**: Optionally specify a job name (e.g., `/dopsx:apply sample-company-setup`). If omitted, check if it can be inferred from conversation context. If vague or ambiguous you MUST prompt for available jobs.

**Steps**

1. **Select the job**

   If a name is provided, use it. Otherwise:
   - Infer from conversation context if the user mentioned a job
   - Auto-select if only one active job exists
   - If ambiguous, run `openspec list --json` to get available jobs and ask the user to select one

   Always announce: "使用工作：<name>" and how to override (e.g., `/dopsx:apply <other>`).

2. **Check status to understand the schema**
   ```bash
   openspec status --change "<name>" --json
   ```
   Parse the JSON to understand:
   - `schemaName`: expected to be `doc-driven`. If it is not, tell the user this job was not created by dopsx (owner markers and the documents folder may be missing) and ask whether to continue
   - `planningHome`, `changeRoot`, and `actionContext`: planning scope and edit constraints. The deliverables folder is `<planningHome.root>/documents/<name>/`
   - Which artifact contains the checklist (`tasks` for doc-driven, check status for others)

3. **Get apply instructions**

   ```bash
   openspec instructions apply --change "<name>" --json
   ```

   This returns:
   - `contextFiles`: artifact ID -> array of concrete file paths (proposal, specs, design, tasks for doc-driven)
   - Progress (total, complete, remaining)
   - Task list with status, source path, and source line
   - Dynamic instruction based on current state
   - Optional `context`: current required instruction input from the selected root
   - Optional `operationGuidance`: current advisory guidance for apply
   - `missingArtifacts` (when present): required artifact ids with no output

   **Handle states:**
   - If `state: "blocked"`: show the message and pause the work.
     - If `missingArtifacts` is non-empty: suggest `/dopsx:continue <name>` to create them.
     - Otherwise, follow the CLI instruction to create or repair the checklist file the schema tracks, from the existing planning artifacts. Do not assume another artifact is ready or start the document work while blocked.
   - If `state: "all_done"`: report that all checklist items are complete, and suggest `/dopsx:verify` and then `/dopsx:archive`
   - Otherwise: proceed

   Treat `context` as a required prompt-level input. Read and consider it, and
   apply relevant facts, conventions, and constraints while doing the work.
   Treat `operationGuidance` as optional additive advice. Read and consider every
   entry, and follow entries that are applicable and compatible with the built-in
   workflow.

   Keep both fields separate from CLI-returned state, missing artifacts, tasks,
   progress, `contextFiles`, and the built-in `instruction`. They are not
   evidence of task completion, do not replace the built-in instruction, and do
   not permit bypassing a blocked state. If context conflicts with the built-in
   instruction, an explicit user choice, or a CLI-controlled value, report the
   conflict and preserve the controlling value. If guidance is inapplicable or
   conflicts with those controlling inputs, do not follow it and explain why.
   These are prompt-level behavior contracts, not enforceable checks.

4. **Read context files**

   Read every file path listed under `contextFiles` from the apply instructions output. In design.md, note the chosen official samples or base documents, format, signing and submission methods, file naming, the 预期的空白 (intended blanks) and the 待确认 (open questions).

   Do not copy `context` or `operationGuidance` verbatim into documents or planning artifacts unless the user separately asks for that content.

5. **Show current progress**

   Display:
   - Schema being used
   - Progress: "N/M 项已完成"
   - Remaining items, grouped by owner (【我方】, 【客户】, 【官方】)
   - Dynamic instruction from CLI

6. **Work through the checklist (loop until done, waiting or blocked)**

   Take the pending items in order. For each item:
   - Read the owner marker at the start of its description:
     - **【我方】** - do it (see below)
     - **【客户】 or 【官方】** - do NOT tick it. Stop and report who we are waiting for, exactly what they need to do or send, and the item's completion check. If the user says it has already happened (for example the client has sent the file or the office has accepted the filing), look at the evidence, confirm the completion check is met, then tick it and continue
     - **No owner marker** - pause and ask who owns it; suggest `/dopsx:update` to add the marker
   - For a 【我方】 item:
     - Follow design.md (official sample or base document, format, signing, submission, naming) and satisfy the requirements in specs
     - Save every final version to `<planningHome.root>/documents/<name>/` under the file name the item names; create the folder if it does not exist
     - Keep drafts, intermediate versions and working notes in the job folder (`changeRoot`)
     - Treat every file outside `documents/<name>/` and the job folder as read-only source material: the client's materials, earlier versions and samples the user provided. To change one, copy it into the job folder or `documents/<name>/` and edit the copy. Overwrite, move or delete a file outside those two folders only when the item names that file as the one to change and the user confirms
     - Use the tools and skills suited to the file format (for example the docx skill for Word files, the pdf skill for PDF); when practical, render or re-open the result to check it
     - Leave the intended blanks from design.md 预期的空白 blank; never invent dates, signatures, seals or amounts
     - Never write credentials into any file; steps that need the client's account are done by the client
   - Before ticking, confirm the checkbox at the item's returned `sourcePath` and `line` still matches the item's description; if it does not, rerun the apply instructions and use the refreshed location
   - Mark the item complete at its returned `sourcePath` and `line`: `- [ ]` → `- [x]`, only when its completion check is fully met
   - Rerun the apply instructions and confirm that the item is now done and progress changed
   - Continue to the next item

   **Pause if:**
   - The next pending item belongs to 【客户】 or 【官方】 and is not done yet
   - An item is unclear, or it depends on an open question in design.md 待确认 → ask for clarification
   - Doing the item reveals a problem with the plan → suggest updating the artifacts with `/dopsx:update`
   - An item needs work beyond what the requirements and checklist describe, or you are tempted to drop, narrow, defer, or accept exceptions to a requirement to make it fit → surface the added scope and ask; do not absorb it silently
   - An item would overwrite, move or delete a file outside `documents/<name>/` and the job folder → name the file and ask the user first
   - Error or blocker encountered → report and wait for guidance
   - User interrupts

7. **On completion or pause, show status**

   Display:
   - Items completed this session
   - Overall progress: "N/M 项已完成"
   - Files written to `documents/<name>/`
   - If waiting: who we are waiting for and what they need to do
   - If all done: suggest `/dopsx:verify`, then `/dopsx:archive`
   - If paused for another reason: explain why and wait for guidance

**Output During Execution**

```
## 执行中：<name>（流程：<schema-name>）

正在处理第 3/7 项：【我方】<事项>
[...制作文书...]
✓ 已完成，正式版：documents/<name>/<文件名>

正在处理第 4/7 项：【我方】<事项>
[...制作文书...]
✓ 已完成
```

**Output When Waiting**

```
## 暂停：等待【客户】

**工作：** <name>
**流程：** <schema-name>
**进度：** 4/7 项已完成

### 等待中
- 3.1 【客户】<事项>
  - 需要对方做的事：<…>
  - 完成判断：<…>

对方完成后告诉我，我确认后继续。
```

**Output On Completion**

```
## 清单已全部完成

**工作：** <name>
**流程：** <schema-name>
**进度：** 7/7 项已完成 ✓

### 本次完成
- [x] 1.1 …
- [x] 1.2 …
...

### 成果物（documents/<name>/）
- <文件名>

下一步：运行 `/dopsx:verify` 验收，通过后运行 `/dopsx:archive` 归档。
```

**Output On Pause (Issue Encountered)**

```
## 暂停：遇到问题

**工作：** <name>
**流程：** <schema-name>
**进度：** 4/7 项已完成

### 问题
<问题说明>

**可选做法：**
1. <做法 1>
2. <做法 2>
3. 其他做法

要怎么处理？
```

**Guardrails**
- Keep going through items until done, waiting or blocked
- Always read context files before starting (from the apply instructions output)
- Only 【我方】 items are done by you; never tick a 【客户】 or 【官方】 item without evidence that its completion check is met
- Final versions go to `documents/<name>/`; drafts and working files stay in the job folder
- Never overwrite, move or delete files outside `documents/<name>/` and the job folder without the user's confirmation; edit copies of the materials the user provided
- If an item is ambiguous, pause and ask before doing it
- If execution reveals problems with the plan, pause and suggest artifact updates
- Keep document changes minimal and scoped to each item
- Update the checkbox immediately after completing each item
- Pause on errors, blockers, or unclear requirements - don't guess
- When an item needs work beyond what the requirements describe, surface the added scope and pause - never silently narrow, defer, or simplify away a requirement
- Only mark an item `- [x]` when its completion check is fully met, not when it is partially done or deferred
- Use contextFiles from CLI output, don't assume specific file names
- Use each item's sourcePath and line to update its exact checkbox
- Do not use context or operation guidance as proof that an item is complete
- Never write credentials into any file
- Apply relevant context; report conflicts with controlling workflow inputs
- Consider every guidance entry; explain any inapplicable or conflicting advice
- Do not copy runtime context or operation guidance into documents or planning artifacts
- Preserve CLI-controlled blocked/ready/all-done behavior and completion criteria

**Fluid Workflow Integration**

This workflow supports the "actions on a job" model:

- **Can be invoked anytime**: Before all artifacts are done (if tasks exist), after partial execution, after a wait ends, interleaved with other actions
- **Allows artifact updates**: If execution reveals problems with the plan, suggest `/dopsx:update` - not phase-locked, work fluidly
