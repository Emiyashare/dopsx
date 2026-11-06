---
name: bulk-archive
description: "Archive several finished dopsx document jobs at once (批量归档文书工作): the user picks the jobs, each one is checked with the archive rules, overlapping requirements are merged by their checked dates, the batch is confirmed once, then every ready job is archived with its process record. Use when the user wants to close several document jobs together."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.16.0).
metadata:
  author: dopsx
  version: "1.1.0"
---

Archive several finished dopsx document jobs in a single operation.

dopsx is the document edition of OpenSpec's `/opsx`: a job's deliverables are documents in `<planningHome.root>/documents/<name>/`, and archiving never moves them. Every job in the batch goes through the same checks as `/dopsx:archive`, and there is no "archive anyway". Jobs that can be archived are already finished, so when two of them bring requirements for the same library category, the checked dates of those requirements decide the merge order. Write everything the user reads in the user's language (Chinese unless the user writes in another language).

`<capability-path>` is the spec directory relative to `specs/` (for example `company-setup/registration`). Preserve the full path from each delta spec when resolving its main spec in the requirement library.

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.16.0` (requires Node.js 22.12 or newer; dopsx is tested with openspec 1.16.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

**Input**: None required (prompts for selection)

**Steps**

1. **Get active jobs**

   Run `openspec list --json` to get all active jobs.

   If no active jobs exist, inform the user and stop.

2. **Prompt for job selection**

   Ask the user to choose jobs (multi-select):
   - Show each job name and checklist progress from the list output
   - Include an option for "全部"
   - Allow any number of selections (1+ works, 2+ is the typical use case)

   **IMPORTANT**: Do NOT auto-select. Always let the user choose.

   **Load current archive inputs once for the selected root before batch validation:**

   Choose one selected job from this root and run
   `openspec instructions archive --change "<selected-job>" --json`. This lookup is
   advisory and optional: it only supplies extra prompt inputs, so it must never
   block the batch. If it fails or returns invalid JSON — for example on an older
   CLI that does not support this command yet — continue the batch with no context
   and no operation guidance. Do not report an error and do not stop.

   A valid response may omit `context` and `operationGuidance`. Treat `context` as a
   required prompt-level input across the batch: read and consider it, and apply
   relevant facts, conventions, and constraints. Treat `operationGuidance` as
   optional additive advice: read and consider every entry, and follow entries that
   are applicable and compatible with the built-in batch workflow.

   Keep both fields separate from conflict analysis, explicit user choices,
   resolved paths, CLI checks, and command contracts. If context conflicts with one
   of those controlling inputs, report the conflict and preserve the controlling
   value. If guidance is inapplicable or conflicts with a controlling input, do not
   follow it and explain why. Do not infer skipped prompts, replacement paths, or
   flags from either field, and do not copy their text verbatim into requirement
   files, jobs, process records or summaries. These are prompt-level behavior
   contracts, not enforceable checks.

3. **Batch validation - gather status for all selected jobs**

   Run `openspec list --json` once for checklist progress. If the lookup fails,
   returns invalid JSON, or omits any selected job, contains a duplicate selected
   job, or returns invalid counts, report the problem and stop before syncing or
   archiving the batch.

   For each selected job, collect:

   a. **Artifact status** - Run `openspec status --change "<name>" --json`
      - Parse `schemaName`, `artifacts`, `planningHome`, `changeRoot`, `artifactPaths`, and `actionContext`
      - Note artifacts that are neither `done` nor `skipped`

   b. **Open items** - the same checks as `/dopsx:archive` step 3:
      - Find the `changes` entry from the list response whose `name` exactly matches this job. Require nonnegative integer `totalTasks` and `completedTasks`, with `completedTasks <= totalTasks`. Open checklist items = `totalTasks - completedTasks`; the items listed under 未完成 and 等待中 must add up to that number
      - Do not infer checklist completion from artifact status. The CLI counts only `x`/`X` checkbox markers as complete; other markers remain open
      - **未完成**: open checklist items in tasks.md owned by 【我方】, or with no owner marker
      - **等待中**: open checklist items owned by 【客户】 or 【官方】
      - **待确认**: every item under the 待确认 section of design.md that is not a ticked `- [x]` line - unticked `- [ ]` lines, and plain list items or sentences written without a checkbox. Ignore HTML comments, blank lines and a line that only says 无
      - **成果物文件夹**: `<planningHome.root>/documents/<name>/` does not exist

   c. **Personal data** - the same check as `/dopsx:archive` step 4 on every delta spec in `artifactPaths.specs.existingOutputPaths`: keywords such as 在留カード, マイナンバー, 個人番号, 口座番号, パスワード, 暗証番号, 密码, 账号, 生年月日, 旅券番号, passport, password; e-mail addresses; phone numbers; strings that look like card or account numbers; and the names and addresses of the people involved in that job as written in its proposal.md

   d. **Delta specs** - Check `artifactPaths.specs.existingOutputPaths` from status JSON
      - List which requirement categories (`<capability-path>`) exist
      - For each, extract the requirement names (lines matching `### Requirement: <name>`) and each requirement's checked date from its `出处：…（确认日期：YYYY-MM-DD）` line
      - Treat this list as the only delta-spec source. If the `specs` entry is missing or the list is empty (for example a job that declares skip_specs), perform no requirement sync or specs-instruction lookup for that job; do not infer requirements from other artifacts

   e. **Archive target** - Compute each job's target name once and record it as that job's `<target-name>`
      - Use the job name as-is when it already starts with a `YYYY-MM-DD-` prefix; otherwise prepend the current date as `YYYY-MM-DD-<name>` (same rule as `openspec archive`)
      - Check whether `<planningHome.changesDir>/archive/<target-name>` already exists
      - If it exists, or another selected job resolves to the same target name, mark every such job 不能归档 with 归档目录里已经有同名文件夹
      - Such a job is never synced or moved: show it as 不能归档 in the step 6 table, keep it out of the step 4 conflict map like any other 不能归档 job, and record it as 失败 in step 8e
      - Checking here, before any library file is written, matches `openspec archive`: a collision found after sync would leave the library rewritten for an archive that never happened

   A job is **可以归档** only when it has no open items, no personal-data hit, no artifact that is neither `done` nor `skipped`, and a free archive target (3e). Otherwise it is **不能归档**, with every reason.

4. **Detect requirement conflicts**

   Build a map keyed by `<capability-path>`, the exact path relative to `specs/`, over the 可以归档 jobs:

   ```text
   company-setup/registration -> [job-a, job-b]  <- CONFLICT (2+ jobs)
   invoice/qualified-invoice  -> [job-c]         <- OK
   ```

   A conflict exists when 2+ 可以归档 jobs have delta specs for the exact same `<capability-path>`.

5. **Resolve conflicts by checked date**

   **For each conflict**:

   a. **Read the delta specs** from each conflicting job to understand what each adds, modifies, removes or renames

   b. **Compare requirement by requirement**:
      - Requirements with different names do not collide: merge all of them
      - For the same requirement in 2+ jobs, compare the checked dates on their source lines: merge the job with the older date first and the job with the newer date last, so the newer wording ends up in the library
      - If the wording differs in substance and the dates are equal or missing: when the source is a public page and web access is available, re-open it and keep the wording that matches it; otherwise ask the user which wording is current before going on

   c. **Record the resolution** for each conflict:
      - The merge order of the jobs for that category
      - For every colliding requirement, which job's wording ends up in the library and why (newer checked date, re-checked source, or the user's choice)

6. **Show consolidated status table**

   Display a table summarizing all selected jobs:

   ```markdown
   | 工作            | 计划文件 | 清单  | 要求 | 冲突                         | 状态     |
   |-----------------|----------|-------|------|------------------------------|----------|
   | setup-tokyo     | 齐全     | 12/12 | 2 类 | company-setup/registration   | 可以归档 |
   | setup-osaka     | 齐全     | 11/11 | 1 类 | company-setup/registration   | 可以归档 |
   | invoice-2026-09 | 齐全     | 5/5   | 无   | 无                           | 可以归档 |
   | permit-renewal  | 齐全     | 6/8   | 1 类 | 无                           | 不能归档 |
   ```

   For conflicts, show the resolution:
   ```text
   冲突处理：
   - company-setup/registration：先合并 setup-tokyo，再合并 setup-osaka
     - 「印鉴证明的有效期」以 setup-osaka 为准（确认日期 2026-09-10，较新）
   ```

   For jobs that cannot be archived, list the reasons grouped as in `/dopsx:archive`:
   ```text
   不能归档：
   - permit-renewal
     - 等待中：3.1 【客户】…
     - 待确认：…
   ```

7. **Confirm batch operation**

   If no job is 可以归档, show the reasons, tell the user what to do for each group, and stop:
   - 未完成 → finish it with `/dopsx:apply`, or tick it if it is already done
   - 等待中 → wait for the client or office; tick it once their part is done
   - 待确认 → answer the question and record it with `/dopsx:update` as `- [x] 问题：答复`
   - 成果物文件夹 → run `/dopsx:verify` to see what is missing
   - 个人信息 → replace it with a generic role (客户、代表者、发起人) and try again
   - 归档目录里已经有同名文件夹 → 把已有的归档文件夹改名，或改天再归档

   Otherwise ask the user a single confirmation question with these options:
   - "归档 N 件可以归档的工作"
   - "取消"

   There is no option to archive a job that cannot be archived.

   Route on the answer by intent, not by exact label — you wrote these labels,
   so match what the user picked rather than the wording above:
   - "取消" — stop, archive nothing and write nothing. Report that nothing was archived and skip the remaining steps.
   - The archive option — proceed with the 可以归档 jobs only, and record the others as 不能归档 in step 8e, except jobs that step 3e marked with 归档目录里已经有同名文件夹, which are recorded as 失败 with that reason
   - Anything else — ask again rather than archiving

   Before step 8 writes the first library file or moves any job, fetch every
   required specs-rule snapshot for the confirmed batch. For each job that will
   sync concrete `artifactPaths.specs.existingOutputPaths`, run
   `openspec instructions specs --change "<name>" --json` exactly once. Obtain all
   snapshots before the first write or move. If any lookup exits non-zero or
   returns invalid artifact-instruction JSON, identify the affected job, report the
   error, and stop the whole batch before any library write or job move. Do not
   treat lookup failure as omitted rules. A valid response without `rules` is the
   no-rules case.

8. **Execute archive for each confirmed job**

   Process jobs in the order determined by step 5 (jobs without conflicts in any order).

   a. **Sync delta specs into the requirement library**:
      - Run the `/dopsx:sync` workflow inline (agent-driven intelligent merge) for each job with delta specs, passing its personal-data check result from step 3 and its fetched specs-rule snapshot. Inline sync must reuse them without fetching or scanning again. Wait for it to finish.
      - If the sync reports any stop or blocking condition, treat the sync as failed. Stop processing that job immediately. Before continuing to the next job, record this job's outcome as 失败 in the batch results, including the sync blocking/error condition.
      - Do not perform the post-sync content comparison and do not move its `changeRoot`; leave the job intact.
      - For conflicts, merge in the resolved order so the step 5 resolution holds.
      - Apply artifact rules only to the library files produced by that job's merge. They do not change conflict resolution, archive behavior, or CLI contracts, and their text is not copied into an output file.
      - Do not delegate to a background task — step 8c would move `changeRoot` out from under a sync that is still reading it.
      - If a job has no delta specs, do not run the sync workflow for it.

   b. **Verify the library before moving changeRoot**:
      - Re-run the comparison for that job's delta specs against `<planningHome.root>/openspec/specs/<capability-path>/spec.md` (use `planningHome.root` from the step 3 status JSON, not a hardcoded path):
        - ADDED requirements present
        - MODIFIED requirements carrying the scenario and description changes named in the delta, with their other scenarios intact
        - REMOVED requirements gone — and where this sync retired a capability (removed its last requirement, leaving `## Requirements` empty), its main spec deleted rather than left empty.
        - RENAMED requirements present under the new name and absent under the old one
        - A requirement that a later job in the batch overrides by the step 5 resolution is checked against the winning job's wording after that later job has merged
      - If the sync failed or anything does not match, report what differs and do not move that job's `changeRoot` — record it as 失败. `changeRoot` remains intact.

   c. **Perform the archive**:

      Target name: use the `<target-name>` recorded for this job in step 3e, unchanged. Never recompute it here: a batch that runs past midnight would check one date in step 3 and move to another.

      **Check if target already exists:**
      - Check again immediately before the move, even though step 3 already checked: the target can appear mid-batch
      - If yes: record this job as 失败 with 归档目录里已经有同名文件夹, leave `changeRoot` where it is, report any library files step 8a already updated for it, and continue with the remaining jobs
      - If no: move `changeRoot` to the archive directory

      ```bash
      mkdir -p "<planningHome.changesDir>/archive"
      mv "<changeRoot>" "<planningHome.changesDir>/archive/<target-name>"
      ```

      **Confirm the move did not nest:** `mv` exits 0 even when the target appeared after the check, moving the job *inside* it. If `<planningHome.changesDir>/archive/<target-name>/<job-directory-name>` now exists (the last path segment of `changeRoot`), move that directory back to `changeRoot` and record this job as 失败 with 归档目录里已经有同名文件夹. Never report it as archived.

      Do NOT move, copy, rename or edit anything in `<planningHome.root>/documents/<name>/`, apart from writing the process record in the next step.

   d. **Write the process record** - as in `/dopsx:archive` step 7: write `<planningHome.root>/documents/<name>/过程记录.md`, or append a new dated section when the file already exists:

      ```markdown
      # 过程记录：<name>

      ## 归档 YYYY-MM-DD

      - 计划与过程材料：`openspec/changes/archive/<target-name>/`（相对于 <planningHome.root>）
      - 清单：<X>/<X> 项已完成
      - 要求库：<已汇总：列出 capability 路径 / 本工作没有需要汇总的要求>
      - 批量归档：与 <同批归档的其他工作> 一起归档<；冲突处理：…>
      ```

   e. **Track outcome** for each job:
      - 已归档: archived successfully
      - 失败: error during sync, library verification or archive (record the error)
      - 不能归档: open items or personal data found in step 3 (record the reasons)

9. **Display summary**

   Show final results:

   ```markdown
   ## 批量归档完成

   已归档 3 件：
   - setup-tokyo → openspec/changes/archive/2026-09-15-setup-tokyo/（documents/setup-tokyo/ 原地保留，已写入 过程记录.md）
   - setup-osaka → openspec/changes/archive/2026-09-15-setup-osaka/（documents/setup-osaka/ 原地保留，已写入 过程记录.md）
   - invoice-2026-09 → openspec/changes/archive/2026-09-15-invoice-2026-09/（documents/invoice-2026-09/ 原地保留，已写入 过程记录.md）

   没有归档 1 件：
   - permit-renewal（等待中：3.1 【客户】…）

   要求库：
   - company-setup/registration：合并 2 件工作；「印鉴证明的有效期」以确认日期较新的 setup-osaka 为准
   ```

   If any failures:
   ```text
   失败 1 件：
   - some-job：归档目录里已经有同名文件夹
   ```

**Conflict Resolution Examples**

Example 1: The same requirement with different checked dates
```text
冲突：company-setup/registration 同时出现在 [setup-tokyo, setup-osaka]

setup-tokyo：「印鉴证明的有效期」（确认日期：2026-07-01）
setup-osaka：「印鉴证明的有效期」（确认日期：2026-09-10），多了在线申请的场景

处理：先合并 setup-tokyo，再合并 setup-osaka。该要求以 setup-osaka 的写法为准（确认日期较新）。
```

Example 2: Different requirements in the same category
```text
冲突：invoice/qualified-invoice 同时出现在 [invoice-a, invoice-b]

invoice-a 新增「登录号码的写法」，invoice-b 新增「税率区分的写法」，没有同名要求。

处理：两条都合并，没有需要取舍的内容。
```

Example 3: The same checked date but different wording
```text
冲突：notary/articles-certification 同时出现在 [job-x, job-y]

两件工作的「定款认证的份数」确认日期都是 2026-09-01，写法不同。
重新打开出处页面核对：与 job-y 的写法一致。

处理：以 job-y 为准。（无法打开出处时，先问用户以哪一份为准。）
```

**Output On Success**

```markdown
## 批量归档完成

已归档 N 件：
- <job-1> → openspec/changes/archive/<target-name-1>/（documents/<job-1>/ 原地保留，已写入 过程记录.md）
- <job-2> → openspec/changes/archive/<target-name-2>/（documents/<job-2>/ 原地保留，已写入 过程记录.md）

要求库：
- 汇总了 N 份要求文件
- 没有冲突（或：处理了 M 处冲突）
```

**Output On Partial Success**

```markdown
## 批量归档完成（部分）

已归档 N 件：
- <job-1> → openspec/changes/archive/<target-name-1>/

没有归档 M 件：
- <job-2>（等待中：…）

失败 K 件：
- <job-3>：归档目录里已经有同名文件夹
```

**Output When No Jobs**

```markdown
## 没有可以归档的工作

没有进行中的文书工作。可以用 `/dopsx:propose` 或 `/dopsx:new` 建立一件。
```

**Guardrails**
- Allow any number of jobs (1+ is fine, 2+ is the typical use case)
- Always prompt for selection, never auto-select
- Check every selected job with the `/dopsx:archive` rules; never archive a job with open items or personal-data hits, and never offer to archive it anyway
- Resolve requirement conflicts by checked date; re-check the source or ask the user when the dates cannot decide
- Show clear per-job status before confirming
- Use a single confirmation for the entire batch
- Never archive after the user cancels the confirmation — a cancelled batch archives nothing and writes nothing
- Track and report all outcomes (已归档 / 不能归档 / 失败)
- Never move or edit `documents/<name>/`, apart from writing each archived job's process record
- Preserve .openspec.yaml when moving to archive
- Archive directory target uses the current date, computed once in step 3e and reused at the move: YYYY-MM-DD-<name>; a name that already starts with a `YYYY-MM-DD-` prefix is used as-is (never stack a second date)
- If an archive target exists, fail that job but continue with others
- Check every archive target in step 3, before the first library write; a job whose target exists is never synced or moved
- Run the `/dopsx:sync` workflow inline for each job with delta specs, and verify the library at `<planningHome.root>/openspec/specs/<capability-path>/spec.md` before moving that job's `changeRoot`
- Fetch archive inputs once per selected root before inspection or moves
- Fetch all required specs-rule snapshots before the batch's first library write or move
- A failed archive-inputs lookup never blocks the batch; it proceeds with no context or guidance
- A failed specs instruction lookup stops the whole batch atomically
- Jobs without concrete `artifactPaths.specs.existingOutputPaths` continue without requirement sync
- Apply relevant runtime context across the batch and report conflicts
- Operation guidance remains advisory; consider every entry and explain rejected advice
- Artifact rules constrain only the library files being written
- Never copy runtime input or artifact-rule text verbatim into output files
