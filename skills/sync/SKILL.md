---
name: sync
description: "Merge the requirements checked in a dopsx document job into the requirement library (要求库) without archiving the job. Use when the user wants the library updated now, for example so another document job can already rely on those requirements."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.16.0).
metadata:
  author: dopsx
  version: "1.1.0"
---

Merge the requirements of a dopsx document job (its delta specs) into the requirement library (main specs).

This is an **agent-driven** operation - you will read delta specs and directly edit the library to apply the changes. This allows intelligent merging (e.g., adding a scenario without copying the entire requirement).

dopsx is the document edition of OpenSpec's `/opsx`: the requirement library holds the rules that documents must satisfy (official rules, client requirements, format rules), each with its source and checked date, and never any personal information. Write everything the user reads in the user's language (Chinese unless the user writes in another language).

`<capability-path>` is the spec directory relative to `specs/` (for example `company-setup/registration` or `invoice/qualified-invoice`). Preserve the full path from each delta spec when resolving its main spec.

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.16.0` (requires Node.js 22.12 or newer; dopsx is tested with openspec 1.16.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

**Input**: Optionally specify a job name after `/dopsx:sync` (e.g., `/dopsx:sync sample-company-setup`). If omitted, check if it can be inferred from conversation context. If vague or ambiguous you MUST prompt for available jobs.

**Steps**

1. **Select the job**

   If a name is provided, use it. Otherwise:
   - Infer from conversation context if the user mentioned a job
   - Auto-select if only one active job exists
   - If ambiguous, run `openspec list --json` to get available jobs and ask the user to select one

   When prompting, show jobs that have delta specs (under `specs/` directory).

   Always announce: "使用工作：<name>" and how to override (e.g., `/dopsx:sync <other>`).

2. **Resolve job context**

   Run:
   ```bash
   openspec status --change "<name>" --json
   ```

   The JSON includes `planningHome.root`. The requirement library lives under `<planningHome.root>/openspec/specs/` — use that root for every library path below, not an assumed path.

3. **Find delta specs**

   Use `artifactPaths.specs.existingOutputPaths` from the status JSON as the
   only source of delta spec paths. If the `specs` entry is missing or
   `existingOutputPaths` is empty (for example a job that declares skip_specs),
   report that there are no requirements to merge, do not infer them from other
   artifacts, and stop without requesting artifact instructions or writing to the
   library.

   Sync every path in `existingOutputPaths` unless the caller narrowed the set.
   A caller narrows it by naming an explicit list of complete entries from
   `existingOutputPaths` — copy those absolute values verbatim. Archive does
   this inline, and a user can too (for example, by selecting the entry ending
   in `/specs/company-setup/registration/spec.md`).
   Then sync only the named paths and leave the remaining delta specs untouched:
   the caller deliberately withheld them, and syncing them anyway would write
   library entries the caller did not approve.
   Carry that narrowed selection through the following steps; never widen it back
   to the full list. If a named path is not in `existingOutputPaths`, do not sync
   it — report it and stop, rather than dropping it silently. If the named list is
   empty, report that there is nothing to sync and stop without writing to the
   library.

   Each delta spec file contains sections like:
   - `## ADDED Requirements` - New requirements to add
   - `## MODIFIED Requirements` - Changes to existing requirements
   - `## REMOVED Requirements` - Requirements to remove
   - `## RENAMED Requirements` - Requirements to rename (FROM:/TO: format)

   If no delta specs found, inform user and stop.

4. **Check for personal data before writing**

   If archive invoked this workflow inline and supplied its personal-data check result, reuse it. Otherwise scan every selected delta spec for personal information and credentials: keywords such as 在留カード, マイナンバー, 個人番号, 口座番号, パスワード, 暗証番号, 密码, 账号, 生年月日, 旅券番号, passport, password; e-mail addresses; phone numbers; strings that look like card or account numbers; and the names and addresses of the people involved in this job as written in its proposal.md.

   If anything is found, stop before writing to the library: list each file and line, and recommend replacing it with a generic role (客户、代表者、发起人).

5. **For each delta spec, apply changes to the library**

   Before the first library write, obtain one current specs-rule snapshot:
   - If archive invoked this workflow inline and supplied a valid snapshot from
     `openspec instructions specs --change "<name>" --json`, reuse it and do not
     fetch the same instructions again.
   - Otherwise run that command once now.
   - If the direct lookup exits non-zero or returns invalid artifact-instruction
     JSON, report the error and stop before writing any library file. Do not treat
     the failure as an absent rule set.
   - A valid response with omitted `rules` means no artifact rules are configured
     and the existing semantic merge continues.

   Apply returned `rules` only to the content and form of the library files produced
   by this merge. Artifact rules are not operation guidance and cannot change
   selected roots, delta paths, CLI checks, or workflow steps. Use their text as
   constraints without copying it verbatim into a library file or summary.

   For each capability delta spec path selected in step 3 — the full `existingOutputPaths` list, or the narrowed subset when a caller supplied one:

   a. **Read the delta spec** to understand the intended changes

   b. **Read the main spec** at `<planningHome.root>/openspec/specs/<capability-path>/spec.md` (may not exist yet)

      **If it does not exist yet** (a new capability), match what `openspec archive` does:
      only ADDED requirements may be applied - step d creates the spec from them.
      MODIFIED and RENAMED have no requirement to act on, so stop the sync for that
      capability and report that its main spec does not exist and only ADDED is allowed
      for a new spec; never invent the missing requirement. REMOVED has nothing to
      remove - skip it and warn.

   c. **Apply changes intelligently**:

      **ADDED Requirements:**
      - If requirement doesn't exist in main spec → add it
      - If requirement already exists → update it to match (treat as implicit MODIFIED)

      **MODIFIED Requirements:**
      - Find the requirement in main spec
      - Apply the changes - this can be:
        - Adding new scenarios the main spec does not have yet
        - Modifying existing scenarios
        - Changing the requirement description
        - Updating its source line and checked date
      - Preserve scenarios/content not mentioned in the delta

      **Source lines:**
      - Every requirement keeps its `出处：…（确认日期：YYYY-MM-DD）` line in the library, directly under the requirement text
      - When the delta and the library cite the same source, keep the later checked date

      **REMOVED Requirements:**
      - Remove the entire requirement block from main spec
      - Retiring the capability. Delete the whole `spec.md` - and the directory once
        nothing else is left in it - only when ALL of these hold:
        1. removing the requirements *this run* left no requirement blocks;
        2. the rest of the spec is well-formed (it still has a `## Purpose`);
        3. the main spec was not already empty before this sync - if you removed
           nothing, change nothing;
        4. every other nonblank line in the whole file is accounted for as the
           title, Purpose, Requirements header, or a canonical requirement's
           statement, source line, scenarios, or fenced examples;
        5. the job's `.openspec.yaml` declares `retire_capabilities: true`;
        6. the `spec.md` resolves inside the real library root (do not follow a
           capability-directory symlink to delete an external file).
        If removing the selected requirements would leave no requirement blocks and
        any retirement condition is not satisfied, do not modify the main spec. Stop
        the sync for that capability, report the blocking condition, and tell the user
        how to resolve it. Never write or leave an empty `## Requirements` section.
        When only the marker is missing, say that too - it is the one thing the user
        can add to make the retirement go through.
      - Deleting the file also deletes its `## Purpose`; any other section blocks
        retirement. Name Purpose when you report the retirement. If the root is under
        version control (git), include a pasteable `git checkout`; otherwise tell the
        user that the removed text remains in the job's delta spec, which moves to the
        archive with the job.

      **RENAMED Requirements:**
      - Find the FROM requirement, rename to TO

      **`## Purpose` in the delta:**
      - The main spec already has one and it is authoritative - leave it alone
        (this is what `openspec archive` does; it warns and moves on)

   d. **Create new main spec** if capability doesn't exist yet:
      - Only when the delta has ADDED requirements to put in it and no MODIFIED or
        RENAMED requirements blocked this capability in step b. Otherwise create nothing
        and leave the library untouched. For a REMOVED-only delta, if the job's
        `.openspec.yaml` declares `retire_capabilities: true`, report it as already retired
        and continue without recreating the spec. Without that marker, report the sync as blocked:
        `openspec archive` rejects it with `Spec must have at least one requirement`.
        An empty delta has no operations to sync; report it as blocked too.
        Never write an empty `## Requirements` section.
      - Create `<planningHome.root>/openspec/specs/<capability-path>/spec.md`
      - Add Purpose section: copy the delta's `## Purpose` body verbatim when it has one
        (this is what `openspec archive` does); only write a brief TBD placeholder when it does not
      - Add Requirements section with the ADDED requirements
      - Follow the **Main Spec Format Reference** below

6. **Validate updated library**

   Run `openspec validate --specs`.
   If validation fails, report the problems and do not claim the sync succeeded.

7. **Show summary**

   After applying all changes, summarize in the user's language:
   - Which capabilities were updated
   - What changes were made (requirements added/modified/removed/renamed)
   - Any new library file left with a TBD Purpose placeholder, so it gets written
     now rather than lingering
   - Any capability retired, naming the deleted `spec.md`, its Purpose, and the
     recovery guidance from step 5

**Delta Spec Format Reference**

```markdown
## Purpose

Only on a delta that introduces a brand-new capability. Seeds the new main spec.

## ADDED Requirements

### Requirement: 本店只写到市区町村时附本店所在场所决定书
设立登记申请 MUST 附发起人决定本店所在场所的书面，写到门牌号。
出处：https://houmukyoku.moj.go.jp/homu/syougyou_tenpu_kabu_02.html（确认日期：2026-09-14）

#### Scenario: 定款只写到区
- **WHEN** 定款的本店所在地只写到市区町村
- **THEN** 申请时必须另附本店所在场所决定书

## MODIFIED Requirements

### Requirement: 印鉴证明的有效期
印鉴届出所附的印鉴证明 MUST 为发行后3个月以内。
出处：https://houmukyoku.moj.go.jp/homu/syougyou_tenpu_kabu_02.html（确认日期：2026-09-14）

#### Scenario: Scenario the main spec already has
- **WHEN** 印鉴证明发行已超过3个月
- **THEN** 必须重新开具

#### Scenario: New scenario to add
- **WHEN** 在线申请并由申请人电子签名
- **THEN** 登记附件不需要另附印鉴证明

## REMOVED Requirements

### Requirement: Deprecated Requirement

## RENAMED Requirements

- FROM: `### Requirement: Old Name`
- TO: `### Requirement: New Name`
```

**Main Spec Format Reference**

The library is what the delta merges INTO. It must never contain delta operation headers (`## ADDED/MODIFIED/REMOVED/RENAMED Requirements`) - after syncing, every requirement lives under a single `## Requirements` section:

```markdown
# <capability> Specification

## Purpose
这类要求管什么、为什么需要。

## Requirements

### Requirement: 本店只写到市区町村时附本店所在场所决定书
设立登记申请 MUST 附发起人决定本店所在场所的书面，写到门牌号。
出处：https://houmukyoku.moj.go.jp/homu/syougyou_tenpu_kabu_02.html（确认日期：2026-09-14）

#### Scenario: 定款只写到区
- **WHEN** 定款的本店所在地只写到市区町村
- **THEN** 申请时必须另附本店所在场所决定书
```

**Key Principle: Intelligent Merging**

Unlike programmatic merging, you merge rather than overwrite:
- A MODIFIED block carries the whole requirement - body, source line, plus every scenario that survives the change. `openspec validate` and `openspec archive` both reject one that drops a scenario the main spec still has.
- Keep anything the delta does not mention, in the main spec's existing order
- Use your judgment to merge changes sensibly

**Output On Success**

```markdown
## 要求已汇总：<name>

更新的要求库：

**<capability-1>**：
- 新增要求："…"
- 修改要求："…"（新增 1 个场景）

**<capability-2>**：
- 新建要求文件
- 新增要求："…"

要求库已更新。工作仍在进行中，完成后运行 `/dopsx:archive` 归档。
```

**Guardrails**
- Read both delta and main specs before making changes
- Preserve existing content not mentioned in delta
- Never copy a delta file into a main spec as-is - merge its content so the main spec keeps the Main Spec Format Reference structure, with no delta operation headers
- Never write personal information or credentials into the library
- If something is unclear, ask for clarification
- Show what you're changing as you go
- The operation should be idempotent - running twice should give same result
- Use only `artifactPaths.specs.existingOutputPaths`; never infer delta specs from unrelated artifacts
- Honor a caller-supplied subset of `existingOutputPaths`; never widen it back to the full list
- Fetch specs instructions once for direct sync, or reuse the archive-supplied snapshot inline
- Stop before every library write on a non-zero or invalid JSON specs-instruction response
- Artifact rules constrain only the specs being written and are never copied into output files
