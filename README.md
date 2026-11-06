**English** | [中文](README.zh-CN.md) | [日本語](README.ja.md)

# dopsx: OpenSpec commands for document work

dopsx is a Claude Code plugin for document work such as company registrations, license applications, contracts, invoices and notices. It follows the same flow as OpenSpec's `/opsx` (explore → plan → apply → verify → archive) and runs on the same openspec CLI, but the work is documents instead of code.

`/opsx` is built for changing code: apply edits code, and verify searches code and tests. Used for paperwork, its checks do not fit, and it has no way to wait for a client or an office. dopsx fills that gap:

- Every checklist item names who does it: 【我方】 (us), 【客户】 (the client) or 【官方】 (an office such as a government agency or a bank).
- Verification checks the documents themselves: whether every deliverable exists, whether names, addresses, dates and amounts agree across documents, and whether anything was left blank.
- Official requirements you have checked are kept in a requirement library with their source and the date you checked them, ready for the next job of the same kind.

> dopsx is an unofficial community extension made by an individual. It is not affiliated with OpenSpec or its maintainers (Fission AI).

## Prerequisites

- Claude Code
- Node.js 22.12 or newer
- openspec CLI 1.16.0 (the version dopsx is tested with)

## Install, update, uninstall

Install the openspec CLI first:

    npm install -g @fission-ai/openspec@1.16.0

Then install the plugin from a terminal:

    claude plugin marketplace add Emiyashare/dopsx
    claude plugin install dopsx@dopsx

You can also type `/plugin marketplace add Emiyashare/dopsx` and then `/plugin install dopsx@dopsx` in a Claude Code session. New sessions then have the twelve `/dopsx:*` commands. The steps are the same on Windows, macOS and Linux, including remote hosts used over SSH; there is no install script to run.

Update to the latest version (new sessions use the new version):

    claude plugin marketplace update dopsx
    claude plugin update dopsx@dopsx

Uninstall:

    claude plugin uninstall dopsx@dopsx

Uninstalling removes only the plugin. Document jobs you have created (`openspec/` and `documents/`) stay as they are.

## The twelve commands

**Everyday**

| Command | What it does |
|---|---|
| `/dopsx:explore` | Explore: think the document work through together; no documents are produced |
| `/dopsx:propose` | Create a job and write all four planning files at once: proposal, specs, design, tasks |
| `/dopsx:apply` | Apply: produce the documents on the checklist; stops and waits at items owned by the client or an office |
| `/dopsx:verify` | Verify: checklist and deliverables complete, names, addresses and dates consistent across files, no unexpected blanks, no outdated sources, no personal information in the requirement library |
| `/dopsx:archive` | Archive: refuses while anything is still open; moves the plan to the archive and leaves the deliverables in place |

**Writing the plan step by step**

| Command | What it does |
|---|---|
| `/dopsx:new` | Create a job without writing the plan; shows how to write the first planning file |
| `/dopsx:continue` | Write the next planning file, one at a time, so you can review each one before going on |
| `/dopsx:ff` | Create a job and write the whole plan at once; asks only when key facts are missing and records reasonable assumptions for the rest |

**Other**

| Command | What it does |
|---|---|
| `/dopsx:update` | Revise the plan midway and keep the four planning files consistent |
| `/dopsx:sync` | Merge checked requirements into the requirement library without archiving |
| `/dopsx:bulk-archive` | Archive several finished jobs at once; each job gets the same checks, jobs with open items are not archived, and when jobs word the same requirement differently the more recently checked one wins |
| `/dopsx:onboard` | Guided tutorial: walks through a complete job with a small, real document task from the current folder |

You can also just say what you want in plain words; Claude picks the matching command.

Typical order:

- Plan in one go: `/dopsx:propose` → `/dopsx:apply` (possibly several times while waiting for the client or an office) → `/dopsx:verify` → `/dopsx:archive`
- Plan file by file: `/dopsx:new` → `/dopsx:continue` (repeat until all four are written) → `/dopsx:apply` → `/dopsx:verify` → `/dopsx:archive`
- To talk it through first, use `/dopsx:explore`; if you are new, run `/dopsx:onboard` once

How propose, new, continue and ff differ:

| Command | Creates the job | Planning files | When something is unclear |
|---|---|---|---|
| propose | yes | all four at once | asks first about anything that affects which documents, for whom, or deadlines |
| new | yes | none | — |
| continue | no (continues an existing job) | one at a time | asks first |
| ff | yes | all four at once | asks only when key facts are missing; assumes the rest and records it in the plan |

What the four planning files contain:

| File | Contents |
|---|---|
| proposal | which documents, for whom, deadlines, the list of deliverables, open questions |
| specs | requirements the documents must meet (official rules, client requests, format rules), each with its source and the date it was checked |
| design | how: which official template, format, signing method, submission method, file names, intentional blanks, open questions |
| tasks | the checklist: each item with an owner and a completion check |

For simple documents with nothing to check against (for example a routine invoice), specs is skipped automatically.

## Folder layout

The first time you create a job in a folder (`/dopsx:propose`, `/dopsx:new` or `/dopsx:ff`), dopsx sets up this structure there; no preparation is needed:

```
<start folder>
+-- openspec
|   +-- config.yaml          schema: doc-driven
|   +-- schemas/doc-driven   document workflow definition (copied from the plugin)
|   +-- specs                requirement library
|   +-- changes
|       +-- <job-name>       plan, drafts, working files
|       +-- archive          archived plans
+-- documents
    +-- <job-name>           final documents; they stay here after archiving
```

- `openspec/schemas/doc-driven`: the document workflow definition. It is copied from the plugin when the folder is set up, so the folder keeps working if you move it to another computer, and jobs in progress keep the version they started with.
- `openspec/specs`: the requirement library; checked requirements collect here for the next job of the same kind
- `openspec/changes/<job-name>`: the job's planning files, drafts and working files
- `openspec/changes/archive`: archived plans
- `documents/<job-name>`: final documents. They stay in place after archiving, with an added `过程记录.md` (process record) that says where the plan was archived

Notes:

- If you start in a subfolder, the existing `openspec` above it is reused rather than created again.
- The requirement library belongs to its `openspec` folder: jobs in the same folder share one library; jobs started in different folders have separate libraries.
- To also save the final documents to a client's company folder, make "copy to the company folder" the last checklist item.
- `/dopsx:apply` does not change your original materials (files outside `documents` and `openspec`): it copies a file before editing it, and asks you first if an original really must be changed.
- If the folder already belongs to an OpenSpec code project, dopsx asks before creating a document job there.
- If the folder has no `openspec` yet but is inside a git repository or contains code repositories, dopsx does not set up there: it suggests a separate folder (for example a new one next to it) and sets up in the folder you confirm.

## Job names

openspec accepts only lowercase letters, digits and hyphens. Japanese, Chinese, uppercase letters and underscores are rejected.

- Valid: `sample-company-setup`, `2026-office-move`
- The folder under `documents` has exactly the same name as the job
- The documents themselves can have Japanese or Chinese file names, for example `documents/sample-company-setup/就任承諾書.docx`

## Owner tags

Each checklist item is written as `- [ ] 1.1 【我方】事项；完成判断：…` (owner, task, and when it counts as done).

- 【我方】 (us): we do it; `/dopsx:apply` carries it out
- 【客户】 (the client): the client does it personally, for example providing materials, signing, or appearing in person
- 【官方】 (an office): waiting for a government office, bank or similar to act, for example accepting, reviewing, or issuing a certificate

While a client or office item is not done, `/dopsx:apply` stops and tells you who and what it is waiting for. Tell Claude when they have done it; the item is ticked only after the evidence has been checked. `/dopsx:archive` and `/dopsx:bulk-archive` refuse to archive while any item is unticked.

## Personal information

- The requirement library (`openspec/specs`) holds only the rules, their sources and the dates they were checked, never a client's name, address, date of birth, ID numbers, accounts and the like.
- Accounts, passwords and PINs never go into any planning file; the checklist only says "done by the client personally" or where the credentials are kept.
- Client names and addresses may appear in the planning files inside a job folder. Keep these folders on your own computer or in private storage, and never push them to a public Git repository.
- Verify and archive check the requirement library for personal information; if any is found, nothing is merged or archived.

## Language

The planning templates, their writing instructions and the owner tags (【我方】【客户】【官方】) are currently in Chinese. Claude talks with you and fills in the plan in your language, but the template headings and the owner tags stay in Chinese.

## Using only the schema

If you do not use Claude Code and only want the doc-driven workflow definition in your own OpenSpec project, follow the OpenSpec convention for community schemas: copy this repository's `schemas/doc-driven/` to your project's `openspec/schemas/doc-driven/`, then set `schema: doc-driven` in `openspec/config.yaml` (or pass `--schema doc-driven` when creating a change).

To check:

    openspec schema which doc-driven
    openspec schema validate doc-driven

This gives you the workflow definition only (the four planning templates and their instructions). Waiting for the client or an office, document verification and the checks before archiving live in the `/dopsx:*` commands, which need the Claude Code plugin.

## Professional qualifications

Some documents may, by law, be prepared or filed on someone else's behalf only by licensed professionals (in Japan, for example, gyoseishoshi, shihoshoshi, certified tax accountants or labor and social security attorneys). dopsx is a tool for organizing plans and producing documents; it does not decide who is qualified to handle a document and does not give legal advice. Check this yourself before use, and hand the work to a licensed professional when required.

## Maintenance

**Changing the schema**: in `schemas/doc-driven/`, change only the wording of the instructions and templates. Never rename the four planning files (proposal, specs, design, tasks) or change their order; that is what lets older copies in existing folders keep working with newer commands.

**Using new templates in an existing folder (optional)**: overwrite that folder's `openspec/schemas/doc-driven/` with `schemas/doc-driven/` from the new version. Jobs in progress are not affected, because the names and order of the four planning files never change.

**After upgrading openspec**:

1. Upgrade openspec: `npm install -g @fission-ai/openspec@latest`
2. In an empty temporary folder, run a small fictional job through `/dopsx:propose` → `/dopsx:apply` → `/dopsx:verify` → `/dopsx:archive`, including one client wait; check that nothing fails and that the requirement library holds the job's requirements after archiving
3. Compare with the workflows of the new OpenSpec `/opsx` for improvements worth bringing over; if the new version adds a workflow, add a matching skill under `skills/` to keep the one-to-one mapping
4. Update the tested openspec version in the three README files and in each skill's `compatibility`

**Releasing a new version**: after changing `skills/` or `schemas/`, raise the version together in `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` and each skill's `metadata`, run `claude plugin validate . --strict`, and update all three README files in the same commit.

## License and credits

- License: MIT, see [LICENSE](LICENSE).
- The command texts are adapted from the `/opsx` workflows of OpenSpec (https://github.com/Fission-AI/OpenSpec, MIT License); LICENSE keeps OpenSpec's copyright notice.
- dopsx is an unofficial community extension and is not affiliated with OpenSpec or its maintainers.
