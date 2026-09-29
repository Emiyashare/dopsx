---
name: continue
description: "Write the next planning artifact of a dopsx document job, one at a time (继续写文书工作的计划): proposal, requirements with sources, approach, then the checklist with owners. Use when the user wants to move a document job's plan forward step by step."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.12.0).
metadata:
  author: dopsx
  version: "1.0.0"
---

Continue a dopsx document job by writing its next planning artifact.

dopsx is the document edition of OpenSpec's `/opsx`: the same steps and the same openspec CLI underneath, but the work is documents, never code. Every job uses the `doc-driven` schema; final deliverables live in `<planningHome.root>/documents/<name>/`.

Write everything the user reads (progress, questions, summaries) and all artifact content in the user's language (Chinese unless the user writes in another language). Keep official document titles and quoted official wording in their original language.

**Planning boundary**: This workflow writes planning artifacts only. Do not produce, draft or fill in any deliverable; that is `/dopsx:apply`'s job.

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.12.0` (requires Node.js; dopsx is tested with openspec 1.12.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

**Input**: Optionally specify a job name after `/dopsx:continue` (e.g., `/dopsx:continue sample-company-setup`). If omitted, check if it can be inferred from conversation context. If vague or ambiguous you MUST prompt for available jobs.

**Steps**

1. **Select the job**

   If a name is provided, use it. Otherwise:
   - Infer from conversation context if the user mentioned a job
   - Auto-select if only one active job exists
   - If ambiguous, run `openspec list --json` to get available jobs sorted by most recently modified, and ask the user to select one

   When prompting, present the top 3-4 most recently modified jobs as options, showing:
   - Job name
   - Schema (from `schema` field if present, otherwise "spec-driven")
   - Status (e.g., "0/5 项", "complete", "no tasks")
   - How recently it was modified (from `lastModified` field)

   Mark the most recently modified job as "(推荐)" since it's likely what the user wants to continue.

   Always announce: "使用工作：<name>" and how to override (e.g., `/dopsx:continue <other>`).

2. **Check current status**
   ```bash
   openspec status --change "<name>" --json
   ```
   Parse the JSON to understand current state. The response includes:
   - `schemaName`: The workflow schema being used (expected `doc-driven`). If it is not, tell the user this job was not created by dopsx (owner markers and the documents folder may be missing) and ask whether to continue
   - `artifacts`: Array of artifacts with their status ("done", "skipped", "ready", "blocked")
   - `isPlanningComplete`: Boolean indicating if all planning artifacts are complete. Older CLI versions expose the same value as `isComplete`.
   - `planningHome`, `changeRoot`, `artifactPaths`, and `actionContext`: path and scope context. Use these instead of assuming paths. `planningHome.root` is the root that holds `openspec/` and `documents/`.

   If `<planningHome.root>/documents/<name>/` does not exist, create it.

3. **Act based on status**:

   ---

   **If all planning artifacts are complete (`isPlanningComplete: true`, or legacy `isComplete: true`)**:
   - Tell the user the plan is complete
   - Show final status including the schema used
   - Suggest: "计划文件已齐全，可以运行 `/dopsx:apply` 开始制作文书。清单完成后先运行 `/dopsx:verify` 验收，再运行 `/dopsx:archive` 归档。"
   - STOP without writing any file

   ---

   **If artifacts are ready to create** (status shows artifacts with `status: "ready"`):
   - Pick the FIRST artifact with `status: "ready"` from the status output
   - Get its instructions:
     ```bash
     openspec instructions <artifact-id> --change "<name>" --json
     ```
   - Parse the JSON. The key fields are:
     - `context`: Root background (constraints for you - do NOT include in output)
     - `rules`: Artifact-specific rules (constraints for you - do NOT include in output)
     - `template`: The structure to use for your output file
     - `instruction`: Schema-specific guidance
     - `resolvedOutputPath`: Resolved path or pattern to write the artifact
     - `dependencies`: Completed artifacts to read for context (entries with `skipped: true` have no files - do not look for them)
     - `skipped`/`warning`: present when the job declares skip_specs and this artifact must NOT be created - pick another artifact
   - **Create the artifact file**:
     - Read any completed dependency files for context - always re-read them from disk, even if you saw them earlier in the conversation (the user may have edited them)
     - **Inspect the relevant materials before drafting**: look at what already exists: the requirement library under `<planningHome.root>/openspec/specs/`, archived jobs under `<planningHome.root>/openspec/changes/archive/` that handled similar documents, the files the user provided or pointed to (client materials, earlier versions, samples), and `<planningHome.root>/documents/`. Keep inspection read-only and proportional to the job.
       - Official requirements must come from authoritative sources (the responsible government office, bureau or official guidance page). Look them up when the library does not already have them, record the source and the date you checked, and never state a requirement from memory alone. If a source cannot be reached, say so and record the requirement as unconfirmed in design.md 待确认.
       - Distinguish confirmed facts from assumptions; surface conflicts with the existing library instead of silently deciding which is correct.
     - If the `instruction` field delegates creation to a specific skill or command, invoke it to produce the artifact instead of writing the file yourself, then verify the artifact file exists at `resolvedOutputPath`
     - Otherwise use `template` as the structure - fill in its sections
     - Apply `context` and `rules` as constraints when writing - but do NOT copy them into the file
     - Write to the `resolvedOutputPath` specified in instructions. If it is a glob pattern, choose the concrete file path using the schema instruction and the job's context
   - **Right after writing the artifact**:
     - **proposal**: if its 要求 section says 无 (the job has no requirements to check), append the line `skip_specs: true` to the job's `.openspec.yaml` now, then re-run status: `specs` must read `skipped`. Decide this only from the proposal's content, and never invent a requirement to avoid skipping
     - **specs**: run `openspec validate "<name>" --strict`. If it reports problems in the requirement files (for example a requirement without SHALL or MUST, or a scenario that does not use `####`), fix them and validate again
   - Show what was created and what's now unlocked
   - STOP after creating ONE artifact

   ---

   **If no artifacts are ready (all blocked)**:
   - This shouldn't happen with a valid schema
   - Show status and suggest checking for issues

4. **After creating an artifact, show progress**
   ```bash
   openspec status --change "<name>"
   ```

**Output**

After each invocation, show in the user's language:
- Which artifact was created (for proposal, also whether specs is now skipped)
- The workflow being used (`doc-driven`)
- Current progress (N/M 计划文件)
- What artifacts are now unlocked
- For design, the open questions recorded in 待确认
- Prompt: "运行 `/dopsx:continue` 写下一份计划文件。"

**Artifact Creation Guidelines**

The `instruction` field from the instructions output is the authoritative guidance for each artifact - follow it even for familiar names (proposal.md, tasks.md, etc.), since the doc-driven schema defines document content for the same file names.

If the `instruction` field directs you to use a specific skill or command to create the artifact, invoke it instead of writing the artifact directly.

**Guardrails**
- Create ONE artifact per invocation
- Planning artifacts only - never produce, draft or fill in a deliverable in `documents/<name>/`
- Always read dependency artifacts before creating a new one - re-read from disk, not from conversation memory (files may have changed since you last saw them)
- Never skip artifacts or create out of order; a `skipped` artifact is satisfied and must not be created
- If context is unclear, ask the user before creating
- Verify the artifact file exists after writing before marking progress
- Use the schema's artifact sequence, don't assume specific artifact names
- **Personal data**: requirement files must never contain a client's personal information; credentials (account IDs, passwords, PINs) never go into any planning file - tasks only say where they are kept or that the client operates it personally
- **IMPORTANT**: `context` and `rules` are constraints for YOU, not content for the file
  - Do NOT copy `<context>`, `<rules>`, `<project_context>` blocks into the artifact
  - These guide what you write, but should never appear in the output
