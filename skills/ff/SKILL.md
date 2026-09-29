---
name: ff
description: "Fast-forward a new dopsx document job (一次写完文书工作的计划): create the job and its documents folder and write the whole plan in one go, asking only when key facts are missing. Use when the user wants the full plan of a document job right away."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.12.0).
metadata:
  author: dopsx
  version: "1.0.0"
---

Fast-forward through planning - create a document job and write everything needed to start producing its documents.

dopsx is the document edition of OpenSpec's `/opsx`: the same steps and the same openspec CLI underneath, but the work is documents (filings, applications, certificates, contracts, letters, invoices and similar), never code. Every job uses the `doc-driven` schema:
- proposal.md (what documents, for whom, by when)
- `specs/<capability-path>/spec.md` (requirements the documents must satisfy, each with its source and checked date - a delta, not the requirement library itself)
- design.md (how: official sample, format, signing, submission, naming, intended blanks, open questions)
- tasks.md (checklist; every item has an owner 【我方】, 【客户】 or 【官方】 and a completion check)

`<capability-path>` is the spec directory relative to `specs/` (for example `company-setup/registration` or `invoice/qualified-invoice`). Preserve an existing capability's full path and follow the root's established organization for new capabilities.

Final deliverables live in `<root>/documents/<name>/`, next to `<root>/openspec/`, where `<name>` is the job name. Drafts and working files stay in the job folder under `openspec/changes/`. dopsx does not use OpenSpec stores.

Write everything the user reads (progress, questions, summaries) and all artifact content in the user's language (Chinese unless the user writes in another language). Keep official document titles and quoted official wording in their original language.

**Planning boundary**: This workflow writes planning artifacts only, even if the request asks to draft, fill in, sign, send or file documents. Do not produce any deliverable. After the planning artifacts are complete, stop and wait for the user to start `/dopsx:apply`.

---

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.12.0` (requires Node.js; dopsx is tested with openspec 1.12.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

**Input**: The argument after `/dopsx:ff` is the job name (lowercase kebab-case), OR a description of the document work.

**Steps**

1. **If no input provided, ask what document work they want to start**

   Ask the user (open-ended, no preset options):
   > "要做什么文书工作？请描述一下：做哪些文书、给谁、什么时候要。"

   From their description, derive a job name using only lowercase English letters, digits and hyphens - the CLI rejects Japanese, Chinese, uppercase letters and underscores (e.g., "示例公司设立的登记材料" → `sample-company-setup`). Tell the user the name; the deliverables folder uses the same name.

   **IMPORTANT**: Do NOT proceed without understanding which documents the user needs.

2. **Resolve the planning root (initialize it when missing)**

   ```bash
   openspec context --json
   ```

   The plugin ships the doc-driven schema in `${CLAUDE_PLUGIN_ROOT}/schemas/doc-driven` (the bundled schema). Before any step below copies it, check that `${CLAUDE_PLUGIN_ROOT}/schemas/doc-driven/schema.yaml` exists. If that path still contains the `CLAUDE_PLUGIN_ROOT` placeholder or the file is missing, dopsx is not running as an installed plugin: stop without writing anything and point the user to the dopsx README (https://github.com/Emiyashare/dopsx), which explains how to install the plugin or copy the schema by hand.

   - If it reports `no_openspec_root`, no root exists in the current folder or any folder above it. Check the bundled schema (see above), then initialize the current working directory as a document-work root:
     ```bash
     openspec init --tools none --no-animation .
     ```
     Then edit `openspec/config.yaml`: change the line `schema: spec-driven` to `schema: doc-driven` and leave the rest of the file unchanged. Copy the bundled schema folder to `openspec/schemas/doc-driven/` (for example `mkdir -p openspec/schemas && cp -r "${CLAUDE_PLUGIN_ROOT}/schemas/doc-driven" openspec/schemas/` in bash, or `Copy-Item -Recurse` in PowerShell), so the folder carries its own copy of the workflow definition. Create the `documents/` folder next to `openspec/`. Always keep `--tools none`: it creates no `.claude/` command or skill files. Tell the user which folders were created.
   - If a root exists, read `<root.path>/openspec/config.yaml`. If its `schema` is not `doc-driven` (for example a code project using `spec-driven`), explain that this folder already belongs to another OpenSpec project and ask the user to confirm creating a document job in it. If they decline, stop without writing anything.

   Then make sure doc-driven resolves from the root:
   ```bash
   openspec schema which doc-driven --json
   ```
   The JSON is printed on stdout; a note that schema commands are experimental may appear on stderr.
   - It resolves (source `project` or `user`): use that copy as it is. Never overwrite or change an existing copy.
   - It does not resolve (an existing root, for example a folder copied from another computer): check the bundled schema (see above), copy it to `<root>/openspec/schemas/doc-driven/`, tell the user that the document workflow definition was added to this folder, and run the command again.
   - If it still does not resolve, stop and explain why.

3. **Create the job**

   If a job with that name already exists, ask whether to continue it with `/dopsx:continue <name>` or to create a new job under a different name.

   From the root, run:
   ```bash
   openspec new change "<name>" --schema doc-driven
   ```
   Always pass `--schema doc-driven`, even when the root's config already defaults to it. This creates the job folder with `.openspec.yaml`.

   Then create the deliverables folder `<root>/documents/<name>/` (create `<root>/documents/` first if it does not exist). If `<root>/documents/<name>/` already exists and is not empty, ask the user whether this job continues that earlier work before using it.

4. **Get the artifact build order**
   ```bash
   openspec status --change "<name>" --json
   ```
   Parse the JSON to get:
   - `applyRequires`: array of artifact IDs needed before execution (e.g., `["tasks"]`)
   - `artifacts`: list of all artifacts, each with its `status` and its `requires` edges (the artifact IDs it directly depends on)
   - `planningHome`, `changeRoot`, `artifactPaths`, and `actionContext`: path and scope context. Use these instead of assuming paths. `planningHome.root` is the root that holds `openspec/` and `documents/`.

5. **Create every artifact in the required set**

   Use a todo list to track progress through the artifacts.

   Loop through artifacts in dependency order (artifacts with no pending dependencies first):

   a. **For each artifact that is `ready` (dependencies satisfied)**:
      - Get instructions:
        ```bash
        openspec instructions <artifact-id> --change "<name>" --json
        ```
      - The instructions JSON includes:
        - `context`: Root background (constraints for you - do NOT include in output)
        - `rules`: Artifact-specific rules (constraints for you - do NOT include in output)
        - `template`: The structure to use for your output file
        - `instruction`: Schema-specific guidance for this artifact type
        - `skipped`/`warning`: present when the job declares skip_specs and this artifact must NOT be created - stop and pick another artifact
        - `resolvedOutputPath`: Resolved path or pattern to write the artifact
        - `dependencies`: Completed artifacts to read for context
      - Read any completed dependency files for context - always re-read them from disk, even if you saw them earlier in the conversation (the user may have edited them)
      - **Inspect the relevant materials before drafting**: Read `context` and `rules` first, then look at what already exists: the requirement library under `<root>/openspec/specs/`, archived jobs under `<root>/openspec/changes/archive/` that handled similar documents, the files the user provided or pointed to (client materials, earlier versions, samples), and `<root>/documents/`. Keep inspection read-only and proportional to the job; reuse findings for later artifacts and inspect more only as needed.
        - Official requirements must come from authoritative sources (the responsible government office, bureau or official guidance page). Look them up when the library does not already have them, record the source and the date you checked, and never state a requirement from memory alone. If a source cannot be reached, say so and record the requirement as unconfirmed in design.md 待确认.
        - Distinguish confirmed facts from assumptions and proposed additions; surface conflicts with the existing library instead of silently deciding which is correct.
        - Do this discovery now, rather than leaving generic "look into it" items for execution. Keep any necessary follow-up investigation specific to an unresolved question.
      - If the `instruction` field delegates creation to a specific skill or command, invoke it to produce the artifact instead of writing the file yourself, then verify the artifact file exists at `resolvedOutputPath`
      - Otherwise create the artifact file using `template` as the structure and write it to `resolvedOutputPath`. If `resolvedOutputPath` is a glob, follow `instruction` to choose the concrete file path
      - Apply `context` and `rules` as constraints - but do NOT copy them into the file
      - Show brief progress: "✓ 已创建 <artifact-id>"

   b. **Continue until every artifact in the required set exists (not just `apply.requires`)**
      - After creating each artifact, re-run `openspec status --change "<name>" --json`
      - The required set is `applyRequires` plus every artifact reachable from those by following the `requires` edges in `status --json` - walk them transitively (doc-driven closes over proposal, specs, design, tasks). Leave artifacts outside that set alone
      - `status` is file-existence only, so an `applyRequires` artifact reading `done` does NOT mean its dependencies exist - writing `tasks.md` early marks `tasks` done while `specs` was never written. Use each artifact's `requires` edges, not its `status`, to build the required set: a `done` artifact still lists what it depends on
      - **skip_specs**: when the proposal concludes that the job has no requirements to check (its 要求 section says 无), append the line `skip_specs: true` to the job's `.openspec.yaml` before moving past the proposal, then re-run status: `specs` must now read `skipped`. Decide this only from the proposal's content, and never invent a requirement to avoid skipping
      - An artifact already reading `status: "skipped"` is satisfied: the job declares `skip_specs` in `.openspec.yaml`, so its files must NOT exist. Never try to create one
      - Create every artifact in the required set that is missing, then re-check - creating one can unblock others
      - Dependencies are enablers, not gates: if a required artifact is still `blocked` only because specs was skipped, write it anyway
      - Stop when every artifact in the required set is `done` or `skipped`

   c. **If an artifact requires user input**:
      - Ask only when a fact that decides which documents are produced, their recipients, the deadline or what counts as done is missing and cannot be found in the materials
      - For anything else, make a reasonable assumption, write it into the plan (proposal 前提与待确认, or design 待确认 as `- [ ] 问题`) and keep going
      - Official requirements are never assumptions: each one needs its source and checked date, or it goes into design.md 待确认 as unconfirmed

6. **Validate and show final status**
   ```bash
   openspec validate "<name>" --strict
   openspec status --change "<name>"
   ```
   If validation reports problems in the requirement files (for example a requirement without SHALL or MUST, or a scenario that does not use `####`), fix the artifact and validate again.

**Output**

After completing all artifacts, summarize in the user's language:
- Job name, the job folder, and the deliverables folder `documents/<name>/`
- List of artifacts created with a one-line description each; if specs was skipped, say so and why
- The assumptions made along the way, and the open questions recorded in design.md 待确认
- "计划文件已就绪，可以审阅。准备好后运行 `/dopsx:apply` 开始制作文书。"

**Artifact Creation Guidelines**

- Follow the `instruction` field from `openspec instructions` for each artifact type - it is the authoritative guidance, even for familiar artifact names
- If the `instruction` field directs you to use a specific skill or command to create the artifact, invoke it instead of writing the artifact directly
- The schema defines what each artifact should contain - follow it
- Read dependency artifacts for context before creating new ones
- Use `template` as the structure for your output file - fill in its sections
- **IMPORTANT**: `context` and `rules` are constraints for YOU, not content for the file
  - Do NOT copy `<context>`, `<rules>`, `<project_context>` blocks into the artifact
  - These guide what you write, but should never appear in the output
- **Personal data**: requirement files must never contain a client's personal information; credentials (account IDs, passwords, PINs) never go into any planning file - tasks only say where they are kept or that the client operates it personally

**Guardrails**
- Create every artifact the execution phase transitively depends on, not just the ids listed in `apply.requires`
- Always read dependency artifacts before creating a new one - re-read from disk, not from conversation memory (files may have changed since you last saw them)
- If context is critically unclear, ask the user - but prefer making reasonable decisions to keep momentum, and write every assumption into the plan
- Planning artifacts only - do NOT produce deliverables or start `/dopsx:apply`
- If a job with that name already exists, ask if the user wants to continue it or create a new one
- Initialize a root only with `--tools none`; never write `.claude/` files
- Verify each artifact file exists after writing before proceeding to next
