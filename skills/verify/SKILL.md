---
name: verify
description: "Check a dopsx document job before archiving (验收文书): unfinished checklist items, missing deliverables, names, addresses, dates or amounts that differ between documents, unexpected blanks, stale or changed sources, and personal data in the requirement library. Use when the user wants to verify document work."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.16.0).
metadata:
  author: dopsx
  version: "1.1.0"
---

Verify that a dopsx document job's deliverables match its plan (requirements, design, checklist) before archiving.

dopsx is the document edition of OpenSpec's `/opsx`: a job's deliverables are documents in `<planningHome.root>/documents/<name>/`, not code, so this check reads documents and planning artifacts. It never searches a codebase or looks for tests. Write the report in the user's language (Chinese unless the user writes in another language). Verify only reports; it never changes deliverables or planning artifacts.

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.16.0` (requires Node.js 22.12 or newer; dopsx is tested with openspec 1.16.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

**Input**: Optionally specify a job name after `/dopsx:verify` (e.g., `/dopsx:verify sample-company-setup`). If omitted, check if it can be inferred from conversation context. If vague or ambiguous you MUST prompt for available jobs.

**Steps**

1. **Select the job**

   If a name is provided, use it. Otherwise:
   - Infer from conversation context if the user mentioned a job
   - Auto-select if only one active job exists
   - If ambiguous, run `openspec list --json` to get available jobs and ask the user to select one

   When prompting, show all active jobs returned by the list, including jobs with `status: "no-tasks"`.
   Include the schema used for each job if available.
   Mark jobs with open items as "(进行中)".

   Always announce: "使用工作：<name>" and how to override (e.g., `/dopsx:verify <other>`).

2. **Check status to understand the schema**
   ```bash
   openspec status --change "<name>" --json
   ```
   Parse the JSON to understand:
   - `schemaName`: expected to be `doc-driven`
   - `planningHome`, `changeRoot`, `artifactPaths`, and `actionContext`: path and scope context
   - Which artifacts exist for this job
   - The deliverables folder `D = <planningHome.root>/documents/<name>/`
   - The requirement library `L = <planningHome.root>/openspec/specs/`

3. **Load the plan**

   ```bash
   openspec instructions apply --change "<name>" --json
   ```

   This returns the job folder, `contextFiles` (artifact ID -> array of concrete file paths), `taskTrackingConfigured`, and top-level `tasks` and `progress` aggregated from every concrete file matched by the schema's `apply.tracks` configuration that could be read. Read all available artifacts from `contextFiles`. From them, collect:
   - **Expected deliverables**: the file names listed under proposal.md 成果物 and in checklist items that save to `documents/<name>/`
   - **Requirements**: every requirement in the delta specs - marked with `### Requirement:`, or listed as `FROM:`/`TO:` pairs under `## RENAMED Requirements` - with the delta section it sits under (`## ADDED`, `## MODIFIED`, `## REMOVED`, or `## RENAMED Requirements`) and, for ADDED and MODIFIED requirements, its source line (出处) and checked date (确认日期). The section decides what each check looks for. The delta specs are the files of the spec artifacts: those whose `artifactPaths.<id>.outputPath` is under `specs/`, read from `contextFiles.<id>`; artifact ids come from the active schema, so do not assume the id `specs`
   - **Chosen samples and formats**: from design.md 做法
   - **Intended blanks**: from design.md 预期的空白 and any blanks noted in checklist items
   - **Open questions**: every item under design.md 待确认 that is not a ticked `- [x]` line. Ignore HTML comments, blank lines and a line that only says 无. An unticked `- [ ]` line is open, and so is a plain list item or sentence written without a checkbox

   Treat apply `state` and `instruction` as context, not a verification verdict.

4. **Read the deliverables**

   List every file in D. For each file, get its text:
   - `.docx`: `pandoc -t plain "<file>"` (or python-docx if pandoc is unavailable)
   - `.pdf`: `pdftotext -layout "<file>" -`, or open it with the Read tool
   - `.xlsx`, `.csv`, `.txt`, `.md`: read directly
   - A PDF or image with no extractable text is a scan: do NOT guess its content. Only check that it exists, and add it to the 需要人工核对 list

5. **Initialize verification report structure**

   Create a report structure with three dimensions:
   - **完整性** (completeness): checklist, deliverables, open questions
   - **正确性** (correctness): requirements met, sources, unexpected blanks, personal data
   - **一致性** (coherence): the same facts written the same way everywhere, structure matching the chosen sample

   Each dimension can have CRITICAL, WARNING, or SUGGESTION issues. Keep two extra lists: **等待中** (unticked client or office items) and **需要人工核对** (scans and unreadable files).

   Verification is advisory. Respect intentional omissions such as `skip_specs: true`, optional design documents, and schemas without task tracking. Do not require or invent optional or intentionally omitted artifacts to obtain a clean report. 未核对 describes a limit of this report, not a new archive prerequisite. `/dopsx:archive` keeps its own checks and user confirmation.

   Mark checks the schema does not define, or artifacts the status reports as intentionally skipped, as **不适用** (not applicable). A job whose readable delta specs contain REMOVED or RENAMED requirements but no ADDED or MODIFIED requirements has nothing to check for ADDED and MODIFIED requirements, so that part of **Requirements met** is also 不适用 (see step 7). Exclude 不适用 checks from skipped-check counts and the archive-readiness assessment. Reserve **未核对** (not verified) for applicable checks whose evidence is missing or unusable.

   If only the checklist is available for applicable checks, verify the checklist and deliverable presence only, and mark the remaining applicable checks as 未核对 with the reason "只有清单可作依据".

   If planning artifacts cannot be read or contain no usable requirements or design content, mark the affected checks as 未核对 with the specific reason. Continue the checks supported by the remaining evidence, but a partially checked input set is not a fully verified check. Missing requirements affect **Requirements met** and **Sources**; missing design content affects **Structure vs the chosen sample** and **Open questions**. A deliverable listed under 需要人工核对 leaves every check that needs its text partially verified.

6. **Verify Completeness (完整性)**

   **Checklist**:
   - If `taskTrackingConfigured` is false, report **Checklist** as 不适用. Do not treat empty `tasks` as missing evidence.
   - Otherwise, use the top-level `tasks` and `progress` fields. They already aggregate every readable concrete file matched by `apply.tracks`, regardless of the tracked artifact's ID; do not infer tracking from a `contextFiles` key.
   - If `unavailableTrackingFiles` is nonempty, mark **Checklist** as 未核对 and include every unavailable path and reason. Continue using any readable items, but do not infer completion from the partial `tasks` and `progress` fields.
   - If `taskTrackingConfigured` is true and `tasks` is empty, mark **Checklist** as 未核对 and record the reason from apply `state` and `instruction`. Nonzero totals alone do not establish evaluable item descriptions.
   - Report complete vs total items from `progress`. Read the owner marker after the item number in each item's `description`
   - Unticked 【我方】 item → CRITICAL: "未完成：<item>". Recommendation: "用 `/dopsx:apply` 完成；如果已经做完，勾选它"
   - Unticked 【客户】 or 【官方】 item → add to 等待中 (not an issue by itself, but the job cannot be archived until it is done)
   - If `progress.remaining` is greater than the number of unticked items listed in `tasks`, the difference is unticked checkboxes without a description → CRITICAL: "未完成：没有说明的复选框 N 个". Recommendation: "用 `/dopsx:update` 写明这些项目的内容、负责人标记和完成判断，然后完成它们". Do not infer completion from the listed items alone
   - Item without an owner marker → WARNING: "缺少负责人标记：<item>". Recommendation: "用 `/dopsx:update` 补上【我方】【客户】或【官方】"

   **Deliverables**:
   - D does not exist → CRITICAL: "成果物文件夹不存在：documents/<name>/"
   - Each expected deliverable missing from D → CRITICAL: "缺少正式版：documents/<name>/<file>"
   - Files in D that the plan does not mention (other than 过程记录.md) → SUGGESTION: list them so the user can confirm they belong there

   **Open questions**:
   - Each unticked `- [ ]` item under design.md 待确认 → WARNING: "待确认未答复：<question>" (archiving is refused while any remain)
   - Each open item written without a checkbox → WARNING: "待确认未答复（未使用复选框）：<question>". Recommendation: "用 `/dopsx:update` 改成 `- [ ] 问题`；答复后写成 `- [x] 问题：答复`" (archiving is refused while any remain)

7. **Verify Correctness (正确性)**

   **Requirements met**:
   - If status marks the spec artifact skipped by `skip_specs: true`, or the schema defines no spec artifact (no artifact whose `artifactPaths.<id>.outputPath` is under `specs/`), report **Requirements met** and **Sources** as 不适用.
   - Otherwise, if the spec artifacts' files are absent or empty, mark **Requirements met** and **Sources** as 未核对; do not treat either as clean.
   - If the delta specs are readable and contain at least one REMOVED or RENAMED requirement but no ADDED or MODIFIED requirements, the ADDED and MODIFIED part of this check is 不适用; still check each RENAMED entry below. A delta spec with no parseable requirements at all is unusable evidence, not a removal-only job: mark this check as 未核对.
   - For each ADDED or MODIFIED requirement (for MODIFIED, check the text in the delta, not the old wording), look for evidence in the deliverables (and, for requirements about attachments or procedures, in the checklist) that it is satisfied; cite the file and the passage
   - Clearly not met → CRITICAL: "要求未满足：<requirement>", with where it should appear
   - Cannot tell → WARNING with what to check
   - REMOVED requirement → the requirement no longer applies, so this check is 不适用 to it: never report it as "要求未满足" or recommend meeting it
   - RENAMED entry (`FROM:`/`TO:`) → the name changes but the requirement stays, so check the TO requirement for the unchanged content:
     - Do not report the FROM name as missing
     - If the TO name also appears under MODIFIED, it is checked there against the MODIFIED text; skip it here
     - Otherwise, read the baseline requirement in the main spec at `<planningHome.root>/openspec/specs/<capability-path>/spec.md`, using the same capability path as the delta spec: the requirement under the FROM name, or under the TO name only when the FROM name is absent because the library is already synced. Check the deliverables against its content as for an ADDED requirement; clearly not met → CRITICAL: "改名后的要求未满足：<TO name>（原名：<FROM name>）". Recommendation: "按这条要求的内容修改成果物；改名不改变要求本身"
     - If the baseline requirement cannot be found or read, mark this check as 未核对 for that entry with the reason. Never count an unchecked rename as passing

   **Sources (出处)**:
   - ADDED or MODIFIED requirement without a source line or without a checked date → WARNING
   - Checked date more than 90 days before today → WARNING: "出处确认日期较旧（YYYY-MM-DD），建议重新确认"
   - When the source is a public web page and web access is available, re-open it and compare the statement the requirement relies on. If it has changed → CRITICAL, quoting the old and new wording. If the page cannot be opened → SUGGESTION: "无法自动确认出处，请人工核对"

   **Unexpected blanks**:
   - Search the readable deliverables for unfilled places: empty date patterns (for example `令和　　年　　月　　日` or `年 月 日` without numbers), `○○`, `＿＿`, `□`, empty signature or seal lines, empty amount fields, and leftover template placeholders (`<…>`, `【】`, `XXX`)
   - A blank listed in design.md 预期的空白 or noted in the checklist → do not report it
   - Any other blank in a signature, date or amount position → WARNING with the file and surrounding text; any other leftover placeholder → WARNING

   **Personal data**:
   - Scan the delta specs and every file under L for personal information and credentials: keywords such as 在留カード, マイナンバー, 個人番号, 口座番号, パスワード, 暗証番号, 密码, 账号, 生年月日, 旅券番号, passport, password; e-mail addresses; phone numbers; strings that look like card or account numbers; and the names and addresses of the people involved in this job as written in proposal.md
   - Any hit → CRITICAL: "要求库中含个人信息：<file>:<line>". Recommendation: "替换成「客户」「代表者」「发起人」这类称呼后再归档"
   - Scan proposal.md, design.md and tasks.md for credentials (passwords, PINs, login IDs together with passwords) → CRITICAL

8. **Verify Coherence (一致性)**

   **Same facts across documents**:
   - From each readable deliverable, extract company or organization names, personal names, addresses, dates, amounts, counts and periods (for example capital, number of shares, fiscal year end, validity periods)
   - Compare each fact across all deliverables, and against proposal.md and design.md
   - Different forms of the same fact (for example `2丁目` vs `二丁目`, half-width vs full-width digits, a different name order, a different amount) → WARNING, listing every original text with its file
   - If a requirement or design.md prescribes the notation, recommend that one; otherwise recommend the notation of the official sample or the official record

   **Structure vs the chosen sample**:
   - If the schema defines no design artifact (no artifact with id `design`, and none whose `artifactPaths.<id>.outputPath` is or ends in `design.md`), report this check as 不适用.
   - Otherwise, if the design artifact's `contextFiles.<id>` file is absent or empty, mark this check as 未核对.
   - If design.md names no official sample or base document for any deliverable, report this check as 不适用.
   - For each deliverable whose design.md names an official sample or base document, compare the structure: title, order of sections, required items, date line, signature block
   - Missing required element → WARNING; order or wording differences that do not change the meaning → SUGGESTION
   - If the sample itself is not available, mark this check as 未核对 for that file and say why

9. **Generate Verification Report**

   **Summary Scorecard**:
   ```markdown
   ## 验收报告：<name>

   ### 概要
   | 维度   | 结果                                        |
   |--------|---------------------------------------------|
   | 完整性 | 清单 X/Y 项，成果物 A/B 份，待确认 N 条       |
   | 正确性 | 要求 M/N 条已满足，出处问题 K 条              |
   | 一致性 | 不一致 J 处                                  |
   ```

   In each 结果 cell, report the results of checks that ran and `未核对（<原因>）` for every skipped check. If all checks in a dimension were skipped, start the cell with `未核对`. Never score a skipped check as passing. Treat every 未核对 or partially verified check as skipped in the final assessment. Count only ADDED and MODIFIED requirements in N, and report REMOVED and RENAMED requirements separately (for example, "删除 1 条，改名 1 条已核对", or "改名 1 条未核对（找不到原要求）"). For a job that only removes or renames requirements, write `不适用（没有新增或修改的要求）` in place of the M/N count.

   **Issues by Priority**:

   1. **CRITICAL**（归档前必须处理）:
      - Unfinished 【我方】 items, unticked checkboxes without a description, missing deliverables, requirements not met (including renamed ones), changed sources, personal data
      - Each with a specific, actionable recommendation

   2. **WARNING**（建议处理）:
      - Inconsistent facts, unexpected blanks, stale sources, unanswered questions, missing owner markers
      - Each with a specific recommendation

   3. **SUGGESTION**（可以改进）:
      - Unlisted files, minor structure differences, sources that could not be checked automatically
      - Each with a specific recommendation

   **等待中**: every unticked 【客户】 or 【官方】 item, with who and what.

   **需要人工核对**: scans and files that could not be read.

   **Final Assessment**:
   - If CRITICAL issues: "发现 X 个严重问题，归档前必须处理。" If any check was skipped, also name every skipped check and its reason.
   - If no CRITICAL issues, one or more warnings or waiting items, and no checks were skipped: "没有严重问题。有 Y 个警告需要考虑；等待中的项目和待确认事项处理完之前不能归档。"
   - If only suggestions (no warnings or waiting items) and no checks were skipped: "没有严重问题和警告。有 Z 条建议可以考虑；可以运行 `/dopsx:archive` 归档。"
   - If no issues, no waiting items, and no checks were skipped: "全部检查通过，可以运行 `/dopsx:archive` 归档。"
   - If any check was skipped and there are no CRITICAL issues: do not claim readiness. Say "已执行的检查中没有严重问题。<检查>未核对：<原因>。" Include the warning count when nonzero.
   - Include the suggestion count when nonzero in every final assessment.

**Verification Heuristics**

- **Completeness**: Focus on objective items (checkboxes, file presence, open questions)
- **Correctness**: Quote the passage that satisfies or violates a requirement - don't assume
- **Coherence**: Flag real differences in facts and notation, not stylistic choices
- **False Positives**: When uncertain, prefer SUGGESTION over WARNING, WARNING over CRITICAL
- **Actionability**: Every issue names the file and passage (`documents/<name>/<file>` with the quoted text, or `<artifact>.md:<line>`) and gives a specific recommendation. No vague suggestions like "请检查一下"

**Output Format**

Use clear markdown with:
- Table for summary scorecard
- Grouped lists for issues (CRITICAL/WARNING/SUGGESTION), then 等待中 and 需要人工核对
- File references like `documents/<name>/就任承諾書.docx` with the quoted passage
- Specific, actionable recommendations
