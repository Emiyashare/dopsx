---
name: new
description: "Create a new dopsx document job without writing its plan yet (新建文书工作): set up the folder if needed, create the job and its documents folder, and show the template of the first planning artifact. Use when the user wants to start a document job step by step."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.16.0).
metadata:
  author: dopsx
  version: "1.1.0"
---

Start a new dopsx document job, one step at a time.

dopsx is the document edition of OpenSpec's `/opsx`: the same steps and the same openspec CLI underneath, but the work is documents (filings, applications, certificates, contracts, letters, invoices and similar), never code. Every job uses the `doc-driven` schema, whose planning artifacts are proposal → specs → design → tasks. Final deliverables live in `<root>/documents/<name>/`, next to `<root>/openspec/`, where `<name>` is the job name. dopsx does not use OpenSpec stores.

Write everything the user reads in the user's language (Chinese unless the user writes in another language).

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.16.0` (requires Node.js 22.12 or newer; dopsx is tested with openspec 1.16.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

**Input**: The argument after `/dopsx:new` is the job name (lowercase kebab-case), OR a description of the document work.

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

   - If it reports `no_openspec_root`, no root exists in the current folder or any folder above it. Check the bundled schema (see above). If the current folder looks like a code workspace rather than a place for document work (it is inside a git repository, or it contains code repositories), do not initialize it: suggest a separate folder for the document work (for example a new folder next to it), ask the user to confirm, then initialize the folder they choose instead of the current one. Otherwise initialize the current working directory as a document-work root:
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

   If a job with that name already exists (check `openspec list --json`), create nothing: suggest `/dopsx:continue <name>` instead and stop.

   From the root, run:
   ```bash
   openspec new change "<name>" --schema doc-driven
   ```
   Always pass `--schema doc-driven`, even when the root's config already defaults to it. This creates the job folder with `.openspec.yaml`.

   Then create the deliverables folder `<root>/documents/<name>/` (create `<root>/documents/` first if it does not exist). If `<root>/documents/<name>/` already exists and is not empty, ask the user whether this job continues that earlier work before using it.

4. **Show the artifact status**
   ```bash
   openspec status --change "<name>" --json
   ```
   Use the returned `planningHome`, `changeRoot`, `artifactPaths`, and `nextSteps` instead of assuming paths. `planningHome.root` is the root that holds `openspec/` and `documents/`.

5. **Get instructions for the first artifact**
   The first artifact is the first one with status "ready" in the status output (`proposal` for doc-driven).
   ```bash
   openspec instructions <first-artifact-id> --change "<name>"
   ```
   This outputs the template and the writing guidance for the first artifact.

6. **STOP and wait for user direction**

**Output**

After completing the steps, summarize in the user's language:
- Job name, the job folder, and the deliverables folder `documents/<name>/`
- Folders created by initialization, if any
- The workflow (`doc-driven`) and its planning artifacts in order: proposal → specs → design → tasks
- Current status (0/4 计划文件)
- The template of the first artifact
- Prompt: "准备好写第一份计划文件了吗？运行 `/dopsx:continue`，或者直接描述这件文书工作，我来起草。"

**Guardrails**
- Do NOT create any planning artifact yet - just show the instructions
- Do NOT produce, draft or fill in any document
- Do NOT advance beyond showing the first artifact template
- If the name is invalid (not lowercase kebab-case), derive a valid one or ask for one
- If a job with that name already exists, suggest using `/dopsx:continue` instead
- Always pass `--schema doc-driven`
- Initialize a root only with `--tools none`; never write `.claude/` files
