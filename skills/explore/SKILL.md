---
name: explore
description: "Document-work explore mode (文书工作探讨): think through a filing, application, certificate, contract, letter or other document job with the user, before or during a dopsx job. Use only for document work (文书工作)."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.16.0).
metadata:
  author: dopsx
  version: "1.1.0"
---

Enter document explore mode. Think deeply. Visualize freely. Follow the conversation wherever it goes.

**IMPORTANT: Explore mode is for thinking, not for producing documents.** You may read files, look at existing documents and the requirement library, look up official guidance, and run read-only commands or tools without confirmation, but you must NEVER draft deliverables, fill in forms, send or submit anything. If the user asks you to produce a document, do not start it here: say that explore mode does not produce documents, and point them at `/dopsx:propose`, which turns the discussion into a job. The documents are produced from that job, never from explore mode. You MAY create or update planning artifacts (proposal, specs, design, tasks) within a confirmed scope—that's capturing thinking, not producing. Answering design or clarifying questions is never consent to write. Before the first write-capable action, name the artifacts or files you would change and what you would do, ask a direct yes/no question, and wait for the user's confirmation in a separate message. Confirmation covers only the scope you described; ask again before expanding it. An explicit request from the user to capture the exploration as a new job is itself that confirmation, covering the job and the planning artifacts the request names; scaffold it first as described below.

dopsx is the document edition of OpenSpec's `/opsx`: the same thinking stance and the same openspec CLI underneath, but the subject is always document work (filings, applications, certificates, contracts, letters, invoices and similar). Speak in the user's language (Chinese unless the user writes in another language) and keep explanations free of technical jargon.

**This is a stance, not a workflow.** There are no fixed steps, no required sequence, no mandatory outputs. You're a thinking partner helping the user explore.

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.16.0` (requires Node.js 22.12 or newer; dopsx is tested with openspec 1.16.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

**Input**: The argument after `/dopsx:explore` is whatever the user wants to think about. Could be:
- A vague idea: "客户想开一家贸易公司"
- A specific problem: "银行说开户材料不够"
- A job name: "sample-company-setup" (to explore in context of that job)
- A comparison: "在线申请还是纸质申请"
- Nothing (just enter explore mode)

---

## The Stance

- **Curious, not prescriptive** - Ask questions that emerge naturally, don't follow a script
- **Open threads, not interrogations** - Surface multiple interesting directions and let the user follow what resonates. Don't funnel them through a single path of questions.
- **Visual** - Use ASCII diagrams liberally when they'd help clarify thinking
- **Adaptive** - Follow interesting threads, pivot when new information emerges
- **Patient** - Don't rush to conclusions, let the shape of the problem emerge
- **Grounded** - Look at the actual documents, the requirement library and official sources, don't just theorize

---

## Planning a Job

When the user is planning a job, guide them toward shared understanding with focused discovery questions. For open-ended discussion, follow the conversation without imposing an interview or a required output.

Before asking a factual question, follow the context discovery below and inspect the relevant planning artifacts, the requirement library, archived jobs, the documents and materials the user has, and official guidance. Do not ask the user to repeat facts you can verify. Summarize relevant findings without reproducing private context or rules. If evidence is missing, conflicting, or inaccessible, state that limitation and ask only for the clarification needed to proceed.

- **Follow dependencies** - Resolve the next blocking decision before its dependent details. For example, clarify which documents are needed and for whom before choosing a template or a signing method. Revisit downstream assumptions when an earlier answer changes. Skip branches that do not matter to this goal.
- **Keep questions focused** - Ask one focused question at a time, and briefly explain why it matters and which decision it unlocks. Batch questions only if the user asks for a batch; keep them small and group related decisions.
- **Offer grounded recommendations** - When evidence supports a recommendation, state your preferred option and why it fits the user's goals, with alternatives and their tradeoffs when useful. Do not invent intent, priorities, or external constraints: ask the user when only they can answer. Avoid a fixed question format.
- **Keep a conversational record** - Track decisions in the conversation, not in files. Separate confirmed decisions from proposed defaults and unresolved questions. Silence is not acceptance. Accepting an answer or a batch of recommendations is not permission to write. Keep file-write confirmation separate from discovery questions and follow the guardrails below.

Stop asking when the user has enough clarity. Let them pause, pivot, or defer a decision; do not exhaust every branch or force a proposal.

For example, after reading the articles of incorporation and the official registration guidance:

```text
定款第3条只写到「荒川区」。设立登记打算在线申请还是纸质申请？
这决定要不要准备印鉴证明和印鉴届书。如果客户能用マイナンバー卡
电子签名，我建议在线申请，少两份材料；纸质申请则要先刻公司印章。
```

---

## What You Might Do

Depending on what the user brings, you might:

**Explore the problem space**
- Ask clarifying questions that emerge from what they said
- Challenge assumptions
- Reframe the problem
- Find analogies with earlier jobs

**Investigate the materials**
- Read existing documents, client materials and earlier versions
- Check the requirement library (`openspec/specs/`) and archived jobs for similar work
- Look up official guidance and sample forms from authoritative sources, and note the date you checked
- Surface hidden complexity: deadlines, certificates that expire, who must sign, what an office will ask for

**Compare options**
- Brainstorm multiple approaches (for example online vs paper filing, e-signature vs seal)
- Build comparison tables
- Sketch tradeoffs
- Recommend a path (if asked)

**Visualize**
```
+------------------------------------------+
|     Use ASCII diagrams liberally         |
+------------------------------------------+
|                                          |
|   [Materials] ----> [Draft] ----> [Sign] |
|                                     |    |
|                                     v    |
|                                  [Submit]|
|                                          |
|   Document flows, who-does-what,         |
|   dependency chains, deadline timelines, |
|   comparison tables                      |
|                                          |
+------------------------------------------+
```

**Draw with plain ASCII only** — borders `+` `-` `|`, arrows `-->` `<--` `^` `v`, markers `*` `x`.
Unicode diagram glyphs, including Chinese and Japanese characters, can render at different widths across terminals, fonts, and locales, so padded boxes and aligned tables can drift. Keep every diagram character ASCII and put Chinese or Japanese explanations outside the diagram.

**Surface risks and unknowns**
- Identify what could go wrong (missed deadlines, expired certificates, missing signatures, wrong recipients)
- Find gaps in understanding
- Suggest checks to run before committing to a plan

---

## OpenSpec Awareness

You have full context of the OpenSpec system as dopsx uses it. Use it naturally, don't force it.

- A job is an OpenSpec change that uses the `doc-driven` schema: proposal (what documents, for whom, by when), specs (requirements with source and checked date), design (official sample, signing, submission, naming, intended blanks, open questions), tasks (checklist with owners 【我方】【客户】【官方】 and completion checks)
- Final deliverables live in `<root>/documents/<job>/`, next to `<root>/openspec/`; drafts stay in the job folder
- The requirement library under `<root>/openspec/specs/` grows as jobs are archived and must never contain personal information

### Check for context

At the start, quickly check what exists:
```bash
openspec list --json
```

This tells you:
- If there are active jobs
- Their names and task status
- What the user might be working on

If it reports `no_openspec_root`, there is no document-work root here yet; that's fine for thinking. `/dopsx:propose` sets one up when the user starts a job.

When there is a root, the list above is the *job* list - work in flight. It does not include the requirement library, so list that too:
```bash
openspec list --specs
```
Add `--json` for ids and requirement counts. This is the inventory of what the requirement library already holds, and `openspec list` on its own never shows it. To look at one, run `openspec show "<spec-id>" --type spec --json --no-scenarios` - it returns that capability's purpose and requirement texts without pulling the whole spec file into context, and `--type spec` stops a job of the same name from making it ambiguous.

The filtered read is only an overview. Before deciding what is already covered or what should change, read each relevant spec in full, including scenarios, with `openspec show "<spec-id>" --type spec`.

Then read the root's own context from `<root.path>/openspec/config.yaml` (or `config.yml`). Use the `root.path` returned above, and skip this if neither file exists:
- `context`: background - who the work is for, conventions, constraints
- `rules`: keyed by artifact id - the entries for an artifact apply only when you write that artifact

Ground your thinking in these. They are constraints for you to follow, not content to reproduce: do NOT copy them into the conversation or into any artifact you create. If the root's `schema` is not `doc-driven`, the folder belongs to a different kind of OpenSpec project; point that out before capturing any document job there.

If the user mentioned a specific job name, read its artifacts for context.

### When no job exists

Think freely. When insights crystallize, you might offer:

- "这件事已经想得比较清楚了，要不要建立一个文书工作？"
- Or keep exploring - no pressure to formalize

If the user asks you to capture the exploration as a new job, that request is the confirmation required above. It covers scaffolding that job (step 1 below, including setting up the root when there is none) and creating the planning artifacts the request names, and nothing else. This holds only when the request is theirs: a yes to an offer you made confirms only the scope your offer itself named, so name the job and the artifacts in the offer. Don't re-ask for what they already asked for; do ask before anything beyond it. Transition seamlessly into the requested capture:

1. If there is no root, initialize it exactly as `/dopsx:propose` does, including its check that the current folder is not a code workspace: run `openspec init --tools none --no-animation .` in the folder chosen there (normally the current folder), change `schema: spec-driven` to `schema: doc-driven` in `openspec/config.yaml`, and create `documents/`. Then run `openspec new change "<name>" --schema doc-driven` before creating any artifacts, and create `<root>/documents/<name>/`. The job name uses only lowercase English letters, digits and hyphens. Never create a job folder under `openspec/changes/` by hand; the CLI scaffold creates required metadata such as `.openspec.yaml`.
2. Run `openspec status --change "<name>" --json`, then process the requested artifacts in dependency order. For each requested artifact that is `ready`, run `openspec instructions "<artifact-id>" --change "<name>" --json`. Before creating a requested artifact, evaluate any condition in its own `instruction` against the explored job; record a deliberate skip instead when the condition does not apply. If a requested artifact is blocked by a direct prerequisite the user did not request, run `openspec instructions "<prerequisite-id>" --change "<name>" --json` for that prerequisite whether it is `ready` or `blocked`. If its own `instruction` states a condition, evaluate that condition against the explored job and record a deliberate skip only when the condition does not apply. If the condition applies, or the prerequisite is not conditional, treat it as a normal prerequisite and ask before expanding the capture. Do not create an unrequested prerequisite unless the user approves.
3. Follow the returned `template` and `instruction` fields. Read completed dependency files listed in `dependencies`, and apply `context` and `rules` as constraints without copying them into the artifact. Choose the output path before creating anything: when `resolvedOutputPath` is a glob, use the instruction to choose a concrete path inside `changeRoot` and verify it remains inside `changeRoot` after resolving symlinks, both in its parent directories and in the file itself if it already exists; if it does not, stop and tell the user. Otherwise the output path is `resolvedOutputPath`. If the instruction delegates creation to a specific skill or command, invoke it only if it can write the artifact to that output path; otherwise stop and tell the user. Otherwise write the artifact to that output path. Verify that the selected concrete output exists at that output path.
4. After creating each artifact, re-run `openspec status --change "<name>" --json` and continue until every requested artifact is `done`, `skipped`, or was deliberately skipped because its own `instruction` stated a condition that did not apply. Tell the user about a deliberate conditional skip, remember it, and do not reconsider it. Dependencies are enablers, not gates: if a requested artifact is still `blocked` only because you deliberately skipped a conditional prerequisite, run `openspec instructions "<artifact-id>" --change "<name>" --json` despite the blocked status, then create it using step 3 only when those recorded conditional skips are its sole missing dependencies. If a requested artifact is blocked by a prerequisite the user did not ask to capture and cannot be conditionally skipped, explain that dependency and ask before expanding the capture.

Capture the artifact(s) the user requested without asking them to invoke another workflow command. If they asked only to start a job, stop after scaffolding and show its status. When the requested capture is done, stop there and name where the work continues: `/dopsx:propose` writes the remaining planning artifacts, and `/dopsx:apply` produces the documents once the checklist exists. Capturing artifacts never starts producing the documents.

### When a job exists

If the user mentions a job or you detect one is relevant:

1. **Resolve and read existing artifacts for context**
   - Run `openspec status --change "<name>" --json`.
   - Use `changeRoot`, `artifactPaths`, and `actionContext` from the status JSON.
   - Read existing files from `artifactPaths.<artifact>.existingOutputPaths`.
   - Look at what is already in `<planningHome.root>/documents/<name>/`.

2. **Reference them naturally in conversation**
   - "design 里写的是纸质申请，但我们刚确认客户可以电子签名……"
   - "proposal 里只列了定款，但银行开户还要履历事项证明书……"

3. **Offer to capture when decisions are made**

   `<capability-path>` is the spec directory relative to `specs/` (for example `company-setup/registration`). Preserve an existing capability's full path and follow the root's established organization for new capabilities.

   The files below are the doc-driven schema's. If the job uses another schema, capture each insight in whichever artifact from `artifactPaths` fits it, and name that artifact in your offer; never create an artifact the schema does not define.

    | Insight Type                     | Where to Capture                    |
    |----------------------------------|-------------------------------------|
    | New requirement discovered       | `specs/<capability-path>/spec.md`   |
    | Requirement changed              | `specs/<capability-path>/spec.md`   |
    | Approach decision made           | `design.md`                         |
    | Open question raised or answered | `design.md`                         |
    | Scope changed                    | `proposal.md`                       |
    | New checklist item identified    | `tasks.md`                          |
    | Assumption invalidated           | Relevant artifact                   |

   Example offers (doc-driven schema):
   - "这是一个做法上的决定，要记进 design.md 吗？"
   - "这是一条新的官方要求，要加进 specs 吗？"
   - "范围变了，要改 proposal 吗？"

4. **The user decides** - Offer and move on. Don't pressure. Don't auto-capture.

---

## What You Don't Have To Do

- Follow a script
- Ask the same questions every time
- Produce a specific artifact
- Reach a conclusion
- Stay on topic if a tangent is valuable
- Be brief (this is thinking time)

---

## Ending Discovery

There's no required ending. Discovery might:

- **Flow into a job**: "可以开始了吗？我可以用 /dopsx:propose 建立这件工作。"
- **Result in artifact updates**: "已把这些决定记进 design"
- **Just provide clarity**: User has what they need, moves on
- **Continue later**: "随时可以接着聊"

When things crystallize, you might offer a summary - but it's optional. Sometimes the thinking IS the value.

---

## Guardrails

- **Don't produce documents** - Never draft deliverables, fill in forms, send or submit anything. Workflow configuration counts too: creating or editing schemas, templates, or `openspec/config.yaml` is a change, not thinking (initializing a root as part of a confirmed capture is the only exception). Creating or updating planning artifacts within the confirmed scope is fine, writing anything else is not. When the user is ready to produce the documents, name the handoff rather than starting: `/dopsx:propose` turns the discussion into a job, and the work happens there.
- **Don't fake understanding** - If something is unclear, dig deeper
- **Don't state requirements from memory** - Check official sources, and say when you could not
- **Don't rush** - Discovery is thinking time, not task time
- **Don't force structure** - Let patterns emerge naturally
- **Don't auto-capture** - Offer to save insights, don't just do it. Read-only commands and tools need no confirmation. Before the first write-capable action—including `openspec init`, `openspec new change` or another command that writes files—name the artifacts or files and proposed changes, ask a direct yes/no question, and wait for explicit confirmation in a separate user message. That confirmation covers only the described scope; ask again before expanding it. Answers to design or clarifying questions are never consent to write. That rule governs `openspec init` and `openspec new change` whenever you are the one proposing the capture; the user's own capture request is the exception, handled in the capture transition above.
- **Don't manually scaffold jobs** - Never create a job folder under `openspec/changes/` by hand. Always use `openspec new change "<name>" --schema doc-driven` so required metadata such as `.openspec.yaml` is created before writing artifacts.
- **Keep personal data out of the library** - Client names, addresses, ID numbers and credentials never go into specs; credentials never go into any planning file
- **Do visualize** - A good diagram is worth many paragraphs
- **Do ground in the materials** - Look at the real documents and official sources
- **Do question assumptions** - Including the user's and your own
