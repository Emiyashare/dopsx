---
name: archive
description: "Archive a finished dopsx document job (归档文书工作): refuse while checklist items, client or office waits, or open questions remain; move the plan to the archive, keep documents/<job>/ in place with a process record, and merge the checked requirements into the requirement library. Use when the user wants to close a document job."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.12.0).
metadata:
  author: dopsx
  version: "1.0.0"
---

Archive a finished dopsx document job.

dopsx is the document edition of OpenSpec's `/opsx`: the same openspec CLI underneath, but a job's deliverables are documents in `<planningHome.root>/documents/<name>/`, and archiving never moves them. Only the plan and its working files move to the archive. Write everything the user reads in the user's language (Chinese unless the user writes in another language).

`<capability-path>` is the spec directory relative to `specs/` (for example `company-setup/registration`). Preserve the full path from each delta spec when resolving its main spec in the requirement library.

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.12.0` (requires Node.js; dopsx is tested with openspec 1.12.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

**Input**: Optionally specify a job name after `/dopsx:archive` (e.g., `/dopsx:archive sample-company-setup`). If omitted, check if it can be inferred from conversation context. If vague or ambiguous you MUST prompt for available jobs.

**Steps**

1. **Select the job**

   If a name is provided, use it. Otherwise:
   - Infer from conversation context if the user mentioned a job
   - Auto-select if only one active job exists
   - If ambiguous, run `openspec list --json` to get available jobs and ask the user to select one

   When prompting, show only active jobs (not already archived).
   Include the schema used for each job if available.

   Always announce: "使用工作：<name>" and how to override (e.g., `/dopsx:archive <other>`).

   **Load current archive inputs before the checks:**

   After resolving the selected job and planning root, run:
   ```bash
   openspec instructions archive --change "<name>" --json
   ```
   This lookup is advisory and optional: it only supplies extra prompt inputs, so it must never block archiving by itself.
   If it exits non-zero or returns invalid JSON — for example on an older CLI that
   does not support this command yet — continue with no context and no operation
   guidance. Do not report an error and do not stop.

   A successful response may omit both optional fields. Treat `context` as a
   required prompt-level input: read and consider it, and apply relevant
   facts, conventions, and constraints. Treat `operationGuidance` as optional
   additive advice: read and consider every entry, and follow entries that are
   applicable and compatible with this archive workflow.

   Keep both fields separate from built-in steps, explicit user choices, resolved
   paths, CLI checks, and command contracts. If context conflicts with one of those
   controlling inputs, report the conflict and preserve the controlling value. If
   guidance is inapplicable or conflicts with a controlling input, do not follow it
   and explain why. Do not infer replacement paths, skipped prompts, or flags from
   either field, and do not copy their text verbatim into specs, job artifacts,
   the process record or summaries unless the user separately asks for it.

2. **Check artifact completion status**

   Run `openspec status --change "<name>" --json` to check artifact completion.

   Parse the JSON to understand:
   - `schemaName`: The workflow being used (expected `doc-driven`)
   - `planningHome`, `changeRoot`, `artifactPaths`, and `actionContext`: path and scope context
   - `artifacts`: List of artifacts with their status (`done`, `skipped`, or other)

   **If any artifacts are neither `done` nor `skipped`** (skipped artifacts satisfy the requirement - the job declares skip_specs):
   - Display warning listing incomplete artifacts
   - Prompt user for confirmation to continue
   - Proceed if user confirms

3. **Refuse while anything is still open**

   This is stricter than `/opsx:archive`: dopsx never archives a job that still has open items.

   Collect every open item:
   - **未完成**: unticked `- [ ]` items in tasks.md owned by 【我方】, or with no owner marker
   - **等待中**: unticked `- [ ]` items owned by 【客户】 or 【官方】
   - **待确认**: every item under the 待确认 section of design.md that is not a ticked `- [x]` line - unticked `- [ ]` lines, and plain list items or sentences written without a checkbox. Ignore HTML comments, blank lines and a line that only says 无
   - **成果物文件夹**: `<planningHome.root>/documents/<name>/` does not exist

   If any open item exists, do NOT archive. Show the full list grouped as above, with the next step for each group:
   - 未完成 → finish it with `/dopsx:apply`, or tick it if it is already done
   - 等待中 → wait for the client or office; tick it once their part is done
   - 待确认 → answer the question and record it with `/dopsx:update` as `- [x] 问题：答复`
   - 成果物文件夹 → run `/dopsx:verify` to see what is missing

   Then stop. There is no "archive anyway" option. If the user wants to drop an item, it must first be removed from the plan with `/dopsx:update`.

4. **Check the requirements for personal data**

   Use `artifactPaths.specs.existingOutputPaths` from the status JSON. Scan every delta spec for personal information and credentials: keywords such as 在留カード, マイナンバー, 個人番号, 口座番号, パスワード, 暗証番号, 密码, 账号, 生年月日, 旅券番号, passport, password; e-mail addresses; phone numbers; strings that look like card or account numbers; and the names and addresses of the people involved in this job as written in proposal.md.

   If anything is found, do not sync and do not archive: list each file and line, recommend replacing it with a generic role (客户、代表者、发起人), and stop.

5. **Assess delta spec sync state**

   Use `artifactPaths.specs.existingOutputPaths` from status JSON as the only
   delta-spec source. If the `specs` entry is missing or
   `existingOutputPaths` is empty (for example a job that declares skip_specs),
   proceed without a sync prompt and do not infer delta specs from other artifacts.

   **If delta specs exist:**
   - Compare each delta spec with its corresponding main spec at `<planningHome.root>/openspec/specs/<capability-path>/spec.md`
   - Determine what changes would be applied (adds, modifications, removals, renames)
   - Show a combined summary before prompting

   **Prompt options:**
   - If changes needed: "现在汇总进要求库（推荐）", "不汇总，直接归档"
   - If already synced: "直接归档", "再汇总一次", "取消"

   Route on the answer:
   - "取消" — stop, do not archive
   - "不汇总，直接归档" or "直接归档" — proceed to archive
   - "现在汇总进要求库" or "再汇总一次" — sync, then verify (below)
   - Anything else — ask again rather than archiving

   Before a selected sync writes any main spec, run
   `openspec instructions specs --change "<name>" --json` once. Require a zero
   exit status and valid artifact-instruction JSON. If the lookup fails or returns
   invalid JSON, report the error and stop before writing any main spec or moving
   the job. A valid response with omitted `rules` is the no-rules case. Apply
   returned `rules` only to the content and form of main specs produced by this
   merge; do not use them as archive guidance, change CLI behavior, or copy the
   rule text into any output file.

   Then run the `/dopsx:sync` workflow inline (agent-driven intelligent merge) for job '<name>', passing the delta spec analysis, the personal-data check result from step 4, and the fetched specs-rule snapshot from above, and wait for it to finish. The inline sync must reuse that snapshot without fetching `specs` instructions again. Do not delegate it to a background task — step 6 would move `changeRoot` out from under a sync that is still reading it, leaving the job archived and the library never updated. If your agent can only run it by delegation, delegate synchronously and wait for the result.

   Then re-run the comparison from the top of this step against every capability that has a delta spec in `artifactPaths.specs.existingOutputPaths` — not only the ones the sync reports it touched. A successful sync leaves nothing left to apply, so each capability must now read as already synced:
   - ADDED requirements present
   - MODIFIED requirements carrying the scenario and description changes named in the delta, with their other scenarios intact
   - REMOVED requirements gone — and where this sync retired a capability (removed its last requirement, leaving `## Requirements` empty), its main spec deleted rather than left empty; a spec the sync deliberately kept and reported is also a match
   - RENAMED requirements present under the new name and absent under the old one

   If the sync failed, or any capability does not match, report what differs and stop — do not archive. Nothing has moved and `changeRoot` is intact, so the user can fix the mismatch or re-run the sync and start the archive again.

6. **Perform the archive**

   Create an `archive` directory under `planningHome.changesDir` if it doesn't exist:
   ```bash
   mkdir -p "<planningHome.changesDir>/archive"
   ```

   Generate the target name: use the job name as-is when it already starts with a `YYYY-MM-DD-` prefix; otherwise prepend the current date as `YYYY-MM-DD-<name>`. Never stack a second date (same rule as `openspec archive`).

   **Check if target already exists:**
   - If yes: Fail with error, suggest renaming existing archive or using different date
   - If no: Move `changeRoot` to the archive directory

   ```bash
   mv "<changeRoot>" "<planningHome.changesDir>/archive/<target-name>"
   ```

   Do NOT move, copy, rename or edit anything in `<planningHome.root>/documents/<name>/`, apart from writing the process record in the next step.

7. **Write the process record**

   Write `<planningHome.root>/documents/<name>/过程记录.md`. If the file already exists (for example the job was archived before and reopened), append a new dated section instead of overwriting it:

   ```markdown
   # 过程记录：<name>

   ## 归档 YYYY-MM-DD

   - 计划与过程材料：`openspec/changes/archive/<target-name>/`（相对于 <planningHome.root>）
   - 清单：<X>/<X> 项已完成
   - 要求库：<已汇总：列出 capability 路径 ／ 本次未汇总 ／ 本工作没有需要汇总的要求>
   ```

8. **Display summary**

   Show archive completion summary including:
   - Job name
   - Schema that was used
   - Archive location
   - Deliverables folder (left in place) and the process record
   - Requirement library status (汇总 / 未汇总 / 没有要求)
   - Note about any warnings (incomplete artifacts the user confirmed)

**Output On Success**

```markdown
## 归档完成

**工作：** <name>
**流程：** <schema-name>
**计划已归档到：** openspec/changes/archive/<target-name>/
**成果物：** documents/<name>/（原地保留，已写入 过程记录.md）
**要求库：** ✓ 已汇总

清单全部完成，没有待确认事项。
```

**Output On Success (No Delta Specs)**

```markdown
## 归档完成

**工作：** <name>
**流程：** <schema-name>
**计划已归档到：** openspec/changes/archive/<target-name>/
**成果物：** documents/<name>/（原地保留，已写入 过程记录.md）
**要求库：** 本工作没有需要汇总的要求

清单全部完成，没有待确认事项。
```

**Output On Refusal (Open Items)**

```markdown
## 暂不能归档

**工作：** <name>

### 未完成
- <事项>

### 等待中
- <事项>（等待：客户／官方）

### 待确认
- <问题>

处理完以上事项后再运行 `/dopsx:archive`。
```

**Output On Error (Archive Exists)**

```markdown
## 归档失败

**工作：** <name>
**目标：** openspec/changes/archive/<target-name>/

归档目录里已经有同名文件夹。

**可选做法：**
1. 把已有的归档文件夹改名
2. 如果是重复的，删除已有的归档文件夹
3. 改天再归档
```

**Guardrails**
- Announce the selected job; prompt for selection when it is ambiguous
- Use artifact graph (openspec status --json) for completion checking
- Refuse while any checklist item, wait or open question remains; never offer to archive anyway
- Never move or edit the deliverables in `documents/<name>/`, apart from writing the process record
- Check the delta specs for personal data before any sync or move
- Preserve .openspec.yaml when moving to archive (it moves with the directory)
- Show clear summary of what happened
- If sync is requested, run the `/dopsx:sync` workflow inline (agent-driven)
- Never archive while a spec sync is still in flight — run the sync inline and verify the library before moving `changeRoot`
- If delta specs exist, always run the sync assessment and show the combined summary before prompting
- Apply relevant runtime context and report conflicts; operation guidance remains advisory
- Consider every guidance entry and explain any inapplicable or conflicting advice
- Artifact rules constrain only the specs being written and are never operation guidance
- Never copy runtime context, operation guidance, or artifact-rule text verbatim into output files
