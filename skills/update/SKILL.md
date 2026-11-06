---
name: update
description: "Revise the plan of a dopsx document job (修改文书工作的计划): fold in new decisions, answered questions, changed deadlines or new checklist items, and keep proposal, requirements, design and checklist consistent. Use when the user wants to change or reconcile a document job's plan. Never edits the deliverables."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.16.0).
metadata:
  author: dopsx
  version: "1.1.0"
---

Revise a dopsx document job's existing planning artifacts and keep them coherent. Never edit the deliverables in `documents/<name>/`.

dopsx is the document edition of OpenSpec's `/opsx`. Write everything the user reads in the user's language (Chinese unless the user writes in another language).

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.16.0` (requires Node.js 22.12 or newer; dopsx is tested with openspec 1.16.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

**Input**: Optionally specify a job name after `/dopsx:update` (e.g., `/dopsx:update sample-company-setup`). If omitted, check if it can be inferred from conversation context. If vague or ambiguous you MUST prompt for available jobs.

**Steps**

1. **Select the job**

   If a name is provided, use it. Otherwise:
   - Infer from conversation context if the user mentioned a job
   - Auto-select if only one active job exists
   - If ambiguous, run `openspec list --json` to get available jobs sorted by most recently modified, and ask the user to select one

   When prompting, present the top 3-4 most recently modified jobs as options, showing:
   - Job name
   - Status (e.g., "0/5 项", "complete", "no tasks")
   - How recently it was modified (from `lastModified` field)

   Mark the most recently modified job as "(推荐)" since it's likely what the user wants to update.

   Always announce: "使用工作：<name>" and how to override (e.g., `/dopsx:update <other>`).

2. **Get the job's artifacts**
   ```bash
   openspec status --change "<name>" --json
   ```
   Parse the JSON to understand current state. The response includes:
   - `schemaName`: The workflow schema being used (expected `doc-driven`)
   - `artifacts`: Array of artifacts with their status ("done", "skipped", "ready", "blocked")
   - `isPlanningComplete`: Boolean indicating if all planning artifacts are complete. Older CLI versions expose the same value as `isComplete`.
   - `planningHome`, `changeRoot`, `artifactPaths`, and `actionContext`: path and scope context. Use these instead of assuming paths.

   The artifact ids and paths come from the active schema - do NOT assume them, and do NOT branch on hardcoded artifact names.

   The files to edit are `artifactPaths.<id>.existingOutputPaths` - the concrete files that exist on disk, already glob-expanded for glob artifacts (e.g. `specs/**/*.md`), plus any confirmed concrete path allowed under a partly populated glob artifact (step 4). Do NOT write to `resolvedOutputPath`: for a glob artifact it is still the glob pattern, not a real file.

3. **Understand the request**
   - If the user asked for a specific revision ("客户决定改成纸质申请", "期限提前到 10 月 1 日"), that is the starting edit.
   - If the user answered an open question, the starting edit is in design.md 待确认: that line changes from `- [ ]` to `- [x]`, with the answer written after it. Then check whether the answer changes the requirements, the approach or the checklist.
   - If they only said "update" / "对一下" / "make this coherent", treat it as a coherence review: read the existing artifacts and check them against each other for contradictions, gaps, and duplication.

4. **Read and reconcile**
   - Read the artifact(s) the request touches and the job's other existing artifacts.
   - Draft the requested edit in the conversation, not in files. Work out exactly what it changes; step 5 owns every write. Then check every other existing artifact against the drafted edit - in ANY direction: an edit to a later artifact may require revising an earlier one, not only the other way around. Build order is a useful reading order, not a constraint on which artifacts may be revised.
   - Also check the document-specific points:
     - every checklist item still has an owner marker (【我方】【客户】【官方】) and a completion check
     - deliverable file names agree between proposal.md 成果物 and tasks.md
     - the intended blanks in design.md 预期的空白 match what the checklist expects
     - requirements still carry a source line and checked date, and contain no personal information
     - no planning file contains credentials
     - open questions in design.md 待确认 are written as `- [ ] 问题`, and answered ones as `- [x] 问题：答复`
   - Note everything that is now inconsistent, missing, or contradictory.
   - Propose revisions to files that already exist (`existingOutputPaths`). If an artifact has no existing output files and status `ready` or `blocked`, note it and point the user to `/dopsx:continue <name>` to create them. Leave `skipped` artifacts untouched; do not treat them as missing or defer them to `/dopsx:continue`.
   - A glob artifact (e.g. `specs/**/*.md`) is marked `done` after at least one file matches, and `/dopsx:continue` only handles `ready` artifacts. When reconciliation identifies a missing file for a glob artifact whose `existingOutputPaths` is non-empty (for example a requirement category in proposal.md 要求, including one this revision adds, that has no `specs/<capability-path>/spec.md` yet):
     1. Run `openspec instructions "<artifact-id>" --change "<name>" --json` and use its `instruction` and `template`. Treat `context` and `rules` as constraints; do not copy them into the file. If instructions report `skipped: true`, do not create the file. Read current dependency files from disk; if a required non-skipped dependency is missing, stop and ask the user to restore it first.
     2. Choose a concrete path inside `changeRoot` that matches `artifactPaths.<id>.outputPath` and does not already exist. Verify it remains inside `changeRoot` after resolving any symlinked parent directories. The glob `resolvedOutputPath` is not a valid target.
     3. Include the new file in step 5's proposed revisions and create it only after the user confirms.
     4. After confirmation, immediately before creation, refresh status and instructions. Verify the artifact is still in scope, not skipped, and partially populated; repeat the concrete-path checks above.
     5. Use a create operation that fails if the target already exists. If `instruction` delegates creation to another skill or command, invoke it only if it can honor the confirmed path and these guardrails; otherwise stop. If any check fails or the confirmed draft is no longer valid, stop and reconcile with the user rather than replacing existing content or choosing a different path.
   - If the job is already coherent, say so and propose no revisions.

5. **Confirm and apply, one artifact at a time**
   - This step performs every artifact write in this workflow; no earlier step edits an artifact.
   - Show each proposed revision and why - including the requested edit drafted in step 4. Write only after the user confirms.
   - If the user rejects a revision, do not write it - leave that artifact unchanged.
   - When a substantial rewrite is needed, get that artifact's rules and template first:
     ```bash
     openspec instructions "<artifact-id>" --change "<name>" --json
     ```

6. **Point to the next step (guidance only - NEVER act on it)**
   - Artifacts with empty `existingOutputPaths` and status `ready` or `blocked` -> suggest `/dopsx:continue <name>` to create them.
   - Items already ticked whose documents are affected by the revision -> suggest unticking those items (as part of this update, after confirmation) and redoing them with `/dopsx:apply`.
   - Everything done -> suggest `/dopsx:verify`, then `/dopsx:archive`.

**Output**

After each invocation, show in the user's language:
- Which artifacts were revised (and which proposed revisions were rejected)
- Any file created under a glob artifact that was already partially populated
- Anything deferred to `/dopsx:continue` (artifacts with no files yet and status `ready` or `blocked`, never `skipped` artifacts)
- Documents in `documents/<name>/` that may need to be redone
- Where the job stands and the recommended next command

**Guardrails**
- Planning artifacts only - NEVER edit the deliverables in `documents/<name>/`. If the revised plan means a document must change, stop and point to `/dopsx:apply`.
- Use the artifact ids and paths reported by `openspec status`; never branch on hardcoded artifact names.
- Edit only the concrete files in `existingOutputPaths`, plus any confirmed concrete path allowed under a partly populated glob artifact (step 4); never write to a glob `resolvedOutputPath`.
- Do not advance the build frontier: if an artifact has empty `existingOutputPaths` and status `ready` or `blocked`, creating it is `/dopsx:continue`'s job. Leave `skipped` artifacts untouched. The only new-file scope is a confirmed concrete path under a glob artifact whose `existingOutputPaths` is non-empty.
- Confirm every edit with the user before writing.
- Keep personal information out of requirement files and credentials out of every planning file.
- If the request changes the job's *intent* rather than refining it (different documents, a different client, a different purpose), recommend starting a new job with `/dopsx:new` under a new, distinct name.
