---
name: onboard
description: "Guided first run of dopsx (文书工作入门教程): pick a small real document task in the current folder and walk through exploring, creating the job, the four planning artifacts, producing the document, verification and archiving, explaining each step. Use when the user wants to learn dopsx by doing."
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires the openspec CLI (tested with 1.12.0).
metadata:
  author: dopsx
  version: "1.0.0"
---

Guide the user through their first complete dopsx document job. This is a teaching experience—you'll do real document work in their folder while explaining each step.

dopsx is the document edition of OpenSpec's `/opsx`: the same steps and the same openspec CLI underneath, but the work is documents, never code. Every job uses the `doc-driven` schema; final deliverables live in `<root>/documents/<name>/`. Speak in the user's language (Chinese unless the user writes in another language) and in plain words: the user is not technical, so explain what each step is for, not how the tool works inside.

Each phase follows the rules of the matching `/dopsx` command. When a phase says to follow a command, apply that command's steps and guardrails for that part of the work.

---

**Prerequisite - openspec CLI**: Before the first `openspec` command below, check that the CLI is available (`openspec --version`). If the command is not found, stop immediately without creating or changing any file, and tell the user, in their language, to install it with `npm install -g @fission-ai/openspec@1.12.0` (requires Node.js; dopsx is tested with openspec 1.12.0) and to see the dopsx README: https://github.com/Emiyashare/dopsx

## Preflight

Before starting, check that the openspec CLI is available and that the doc-driven schema bundled with this plugin is present:

```bash
openspec --version
```

Then check that `${CLAUDE_PLUGIN_ROOT}/schemas/doc-driven/schema.yaml` exists. The schema is copied into the folder when the job is created (see Phase 4), so it does not need to be installed anywhere else.

**If the CLI is not installed:**
> 这台电脑还没有安装 openspec 程序。请先运行 `npm install -g @fission-ai/openspec@1.12.0` 安装（需要 Node.js），再回来运行 `/dopsx:onboard`。详见 dopsx 说明文档：https://github.com/Emiyashare/dopsx

**If the bundled schema is missing** (the path above still contains the `CLAUDE_PLUGIN_ROOT` placeholder, or the file does not exist):
> 找不到文书版流程定义：dopsx 需要作为 Claude Code 插件安装。请按 dopsx 说明文档（https://github.com/Emiyashare/dopsx）安装后，再回来运行 `/dopsx:onboard`。

Stop here if either check fails.

Then check where the job would be created:

```bash
openspec context --json
```

If a root exists and its `openspec/config.yaml` schema is not `doc-driven` (for example a code project), say that this folder belongs to another OpenSpec project and suggest opening the folder that holds the documents instead. Continue only if the user confirms.

---

## Phase 1: Welcome

Display:

```
## 欢迎使用 dopsx

我会用一件真实的小文书工作，带你把完整流程走一遍：从想清楚要做什么，到做出文书、验收、归档。边做边讲。

**我们会做：**
1. 在这个文件夹的材料里挑一件小的文书工作
2. 简单探讨一下要做什么
3. 建立一件工作（放计划和成果物的地方）
4. 写四份计划：提案 → 要求 → 做法 → 清单
5. 按清单制作文书
6. 验收
7. 归档

**时间：** 大约 15–30 分钟（取决于文书本身）

先来看看有什么可以做。
```

---

## Phase 2: Task Selection

### Look at the materials

Look through the current folder (read-only) for small, real document tasks:

1. **Status tables and checklists** (`.csv`, `.xlsx`, `.md`, `.txt`) that list documents still missing or not yet signed
2. **Templates or drafts with blanks** - `○○`, `＿＿`, `令和　　年　　月　　日`, empty signature or seal lines, `<…>` placeholders
3. **Official samples or forms** that were downloaded but not filled in yet
4. **Earlier versions** that need a small update (a changed address, a new date, a corrected name)
5. **Correspondence** that asks for a document (a request, a notice with a deadline)

If the preflight found a root, also run `openspec list --json` and look at `<root>/openspec/specs/` and `<root>/documents/`: an existing root may already have jobs, a requirement library and finished documents to learn from.

Do not open files that look like they hold credentials (for example a text file of account IDs and passwords). Never quote personal data or credentials in your suggestions.

### Present Suggestions

From your look at the materials, present 3-4 specific suggestions:

```
## 可以做的文书工作

根据这个文件夹里的材料，下面几件适合用来练习：

**1. [最合适的一件]**
   依据：`<文件名>`（例如：状态表里写着「委任状：缺」）
   规模：1 份文书
   为什么合适：[简短理由]

**2. [第二件]**
   依据：`<文件名>`
   规模：[估计]
   为什么合适：[简短理由]

**3. [第三件]**
   依据：[位置]
   规模：[估计]
   为什么合适：[简短理由]

**4. 其他？**
   告诉我你想做哪份文书。

选哪一件？（回复编号，或者描述你想做的）
```

**If nothing suitable is found:** Ask what the user needs:
> 这个文件夹里没找到现成的小工作。你最近需要做哪份文书？例如一封信、一张请求书、一份委任状。

### Scope Guardrail

If the user picks or describes something too large (many documents, several parties, a long official procedure):

```
这件工作很有价值，但作为第一次练习有点大。

练习时越小越好：能完整看到一遍流程，不会卡在细节里。

**可以这样：**
1. **切小一点** - 先只做其中一份，例如 [具体的一份]
2. **换一件** - 选上面其他建议，或者另一件小文书
3. **照样做** - 如果你就想做这件，也可以，只是会花更长时间

你想怎么做？
```

Let the user override if they insist—this is a soft guardrail.

---

## Phase 3: Explore Demo

Once a task is selected, briefly demonstrate explore mode:

```
建立工作之前，先演示一下「探讨」：动手之前先把事情想清楚。
```

Spend 1-2 minutes on the relevant materials (read-only, following `/dopsx:explore`):
- Read the file(s) involved
- List what the document needs: who it is for, what must be in it, what is still unknown
- Draw a quick diagram with plain ASCII characters if it helps

```
## 简单探讨

[你的简短分析：发现了什么、需要注意什么]

+------------------------------------------+
|   [可选：用 ASCII 画一张示意图]            |
+------------------------------------------+

探讨（`/dopsx:explore`）就是用来做这种思考的：动手之前先调查、先想清楚。任何时候都可以用。

接下来建立一件工作，把这件事装进去。
```

**PAUSE** - Wait for user acknowledgment before proceeding.

---

## Phase 4: Create the Job

**EXPLAIN:**
```
## 建立工作

在 dopsx 里，一件「工作」是放这件文书的计划和过程材料的地方。做好的正式版单独放在 documents 文件夹里，归档之后也留在原处。

我来为这件事建立一件工作。
```

**DO:** Follow the `/dopsx:new` steps: derive a lowercase kebab-case job name, resolve the root (when there is none, initialize it with `openspec init --tools none`, switch its config to `doc-driven`, and copy the bundled schema `${CLAUDE_PLUGIN_ROOT}/schemas/doc-driven` to `openspec/schemas/doc-driven/`; in an existing root, copy it only when `openspec schema which doc-driven --json` does not resolve, and never overwrite a copy that resolves), confirm that doc-driven resolves, run `openspec new change "<name>" --schema doc-driven`, and create `<root>/documents/<name>/`.

**SHOW:**
```
已建立：<changeRoot>

文件夹的样子：
<root>/
+-- openspec/
|   +-- changes/<name>/     这件工作的计划和过程材料
|       +-- proposal.md     为什么做、做什么（待写）
|       +-- specs/          必须满足的要求（待写）
|       +-- design.md       怎么做（待写）
|       +-- tasks.md        清单（待写）
+-- documents/<name>/       做好的正式版放这里

先来写第一份计划：提案。
```

---

## Phase 5: Proposal

**EXPLAIN:**
```
## 提案

提案写清楚「为什么做、做什么」：做哪些文书、给谁、什么时候要、最后交出哪些文件。后面三份计划都以它为基础。

我先根据这件工作起草一份。
```

**DO:** Get the proposal instructions and template:
```bash
openspec instructions proposal --change "<name>" --json
```
Draft the proposal from the template and the materials (don't save yet), and show it:

```
这是提案的草稿：

---

[按模板填好的提案]

---

意思对吗？保存之前还可以改。
```

**PAUSE** - Wait for user approval/feedback.

After approval, write the content to the `resolvedOutputPath`. If its 要求 section says 无, append `skip_specs: true` to the job's `.openspec.yaml`. Then say:

```
提案已保存。

[写了「无」时：这件工作没有需要核对的官方或客户要求，所以跳过「要求」这一份，直接写做法。]
[否则：下一份是「要求」。]
```

---

## Phase 6: Requirements

Skip this phase when specs is skipped.

**EXPLAIN:**
```
## 要求

要求写的是「文书必须满足什么」：官方规定、客户要求、格式要求。每一条都写明出处和确认日期，以后同类的工作可以直接用。

像这样的小工作，通常一份要求文件就够了。
```

**DO:** Get the instructions (`openspec instructions specs --change "<name>" --json`). Look up each official rule on its authoritative source and record the source and the date you checked; never write a requirement from memory alone, and record a source you cannot reach as unconfirmed in design.md 待确认. Draft the requirement file, keeping personal information out of it:

```
这是要求：

---

## ADDED Requirements

### Requirement: <名称>
<要求内容，含 MUST 或 SHALL>
出处：<链接或文件名称>（确认日期：YYYY-MM-DD）

#### Scenario: <场景>
- **WHEN** <条件>
- **THEN** <结果>

---

每条要求都配一个「WHEN…THEN…」的场景，验收时就照着它检查。
```

Save to the concrete file path chosen from `resolvedOutputPath`, then run `openspec validate "<name>" --strict` and fix any problem it reports.

---

## Phase 7: Design

**EXPLAIN:**
```
## 做法

做法写清楚「怎么做」：用哪个官方样本或哪份旧文件做底版、格式、签名方式、提交方式、文件名，还有故意留空的地方和还没确认的问题。

小工作的做法可以写得很短。
```

**DO:** Draft design.md from `openspec instructions design --change "<name>" --json`, show it briefly, and save it to the `resolvedOutputPath`. Point out 预期的空白 and 待确认 in plain words.

---

## Phase 8: Tasks

**EXPLAIN:**
```
## 清单

最后把工作拆成清单。每一项写明谁来做——【我方】【客户】【官方】——以及怎样算完成。执行时照着它一项一项做。
```

**DO:** Draft tasks.md from `openspec instructions tasks --change "<name>" --json`, following its format exactly:

```
这是清单：

---

## 1. 准备材料

- [ ] 1.1 【我方】…；完成判断：…

## 2. 制作文书

- [ ] 2.1 【我方】…，正式版保存为 documents/<name>/<文件名>；完成判断：…

## 3. 签署与提交

- [ ] 3.1 【客户】…；完成判断：…

---

【客户】和【官方】的项目我不会替别人打勾：轮到它们时会停下来，告诉你在等谁、等什么。准备好开始做了吗？
```

**PAUSE** - Wait for user to confirm they're ready to start.

Save to the `resolvedOutputPath`.

---

## Phase 9: Apply (Producing the documents)

**EXPLAIN:**
```
## 制作文书

现在按清单一项一项做，做完一项勾一项。我会顺便说明是计划里的哪一条决定了怎么做。
```

**DO:** Follow the `/dopsx:apply` rules for each item:

1. Announce: "正在处理第 N 项：[事项]"
2. For a 【我方】 item: produce the document as design.md says, save the final version to `documents/<name>/`, keep drafts in the job folder, leave the intended blanks blank, and never change the original materials in the folder (copy first, then work on the copy)
3. Reference the plan naturally: "要求里写着 X，所以这里写 Y"
4. Mark complete in tasks.md: `- [ ]` → `- [x]`, only when its completion check is met
5. Brief status: "✓ 第 N 项完成"

When the next item belongs to 【客户】 or 【官方】 and is not done yet, do not tick it. Explain:

```
## 轮到【客户】了

第 N 项是 [事项]，需要 [谁] 来做：[要做的事]。完成判断是：[…]。

这就是 dopsx 的「等待」：别人负责的事我不会替他打勾。对方做完后告诉我，我看过凭证再勾选，然后继续。

练习可以先停在这里，之后用 `/dopsx:apply <name>` 接着做；也可以先往下看验收会怎么报告。
```

Keep narration light—don't over-explain every sentence of the document.

After the items you can do are done:

```
## 能做的都做完了

- [x] 第 1 项
- [x] 第 2 项
- [ ] 第 3 项（等待【客户】）

接下来验收。
```

---

## Phase 10: Verify

**EXPLAIN:**
```
## 验收

验收检查做好的文书和计划是否一致：清单做完没有、正式版齐不齐、各文件里的名称、住址、日期是否一致、有没有漏填、要求的出处是否太旧、要求库里有没有混进个人信息。
```

**DO:** Run the `/dopsx:verify` checks for this job and show the report in its format (概要, CRITICAL / WARNING / SUGGESTION, 等待中, 需要人工核对).

**SHOW:** Explain the report in plain words: what must be fixed before archiving, what is only a suggestion, and what is waiting for someone else.

**PAUSE** - Wait for user acknowledgment. If the user wants to fix a CRITICAL issue now, fix it (following `/dopsx:apply` for documents or `/dopsx:update` for the plan) and verify again.

---

## Phase 11: Archive

**EXPLAIN:**
```
## 归档

工作全部完成后归档：计划搬进归档文件夹，做好的正式版留在 documents 里，并多一份「过程记录」写明计划去了哪里。核对过的要求汇总进要求库，下次同类工作可以直接用。
```

**DO:** Follow the `/dopsx:archive` rules. If any open item remains (unticked items, 【客户】 or 【官方】 waits, unanswered 待确认), do not archive. Show the refusal and explain:

```
现在还不能归档：[未决事项]。

这正是归档前的检查在起作用：没办完的事不会被悄悄归档。等 [谁] 完成 [什么] 后，勾选对应的项目，再运行 `/dopsx:archive <name>`。
```

Otherwise archive it (requirement library sync, move the job to the archive, write `过程记录.md`) and show:

```
已归档到：`openspec/changes/archive/<target-name>/`
成果物：`documents/<name>/`（原地保留，已写入 过程记录.md）
要求库：[已汇总的类别 ／ 本工作没有需要汇总的要求]

这件工作的来龙去脉都留下了记录：文书在 documents 里，计划和过程材料在归档里。
```

---

## Phase 12: Recap & Next Steps

```
## 完成了！

你刚刚走完了一遍 dopsx 的完整流程：

1. **探讨** - 先把事情想清楚
2. **建立工作** - 准备好放计划和成果物的地方
3. **提案** - 为什么做、做什么
4. **要求** - 必须满足什么（带出处）
5. **做法** - 怎么做
6. **清单** - 谁做什么、怎样算完成
7. **执行** - 做出文书，轮到别人时等待
8. **验收** - 检查一致性和遗漏
9. **归档** - 留下记录，要求进库

大大小小的文书工作都可以按这个节奏来。

---

## 指令一览

**常用：**

| 指令 | 做什么 |
|------|--------|
| `/dopsx:propose` | 建立工作，并一次写完四份计划 |
| `/dopsx:explore` | 探讨：动手前或做到一半时把事情想清楚 |
| `/dopsx:apply` | 按清单制作文书，轮到客户或官方时停下来等 |
| `/dopsx:verify` | 验收做好的文书 |
| `/dopsx:archive` | 归档一件已完成的工作 |

**其他：**

| 指令 | 做什么 |
|------|--------|
| `/dopsx:new` | 只建立工作，计划一份一份地写 |
| `/dopsx:continue` | 写下一份计划文件 |
| `/dopsx:ff` | 建立工作并一次写完计划（推进优先） |
| `/dopsx:update` | 中途修改计划，保持四份一致 |
| `/dopsx:sync` | 不归档，先把要求汇总进要求库 |
| `/dopsx:bulk-archive` | 一次归档多件已完成的工作 |
| `/dopsx:onboard` | 再走一遍这个入门教程 |

---

## 接下来

找一件你真正要做的文书，试试 `/dopsx:propose`。节奏你已经掌握了！
```

---

## Graceful Exit Handling

### User wants to stop mid-way

If the user says they need to stop, want to pause, or seem disengaged:

```
没问题！这件工作已经保存好了，位置是 `openspec status --change "<name>" --json` 显示的工作文件夹。

以后接着做：
- `/dopsx:continue <name>` - 接着写计划文件
- `/dopsx:apply <name>` - 开始或接着制作文书（清单写好之后）

什么都不会丢，随时回来。
```

Exit gracefully without pressure.

### User just wants command reference

If the user says they just want to see the commands or skip the tutorial:

```
## dopsx 指令速查

**常用：**

| 指令 | 做什么 |
|------|--------|
| `/dopsx:propose <工作名>` | 建立工作，并一次写完四份计划 |
| `/dopsx:explore` | 探讨（不制作文书） |
| `/dopsx:apply <工作名>` | 制作文书 |
| `/dopsx:verify <工作名>` | 验收 |
| `/dopsx:archive <工作名>` | 归档 |

**其他：**

| 指令 | 做什么 |
|------|--------|
| `/dopsx:new <工作名>` | 只建立工作 |
| `/dopsx:continue <工作名>` | 写下一份计划文件 |
| `/dopsx:ff <工作名>` | 一次写完计划 |
| `/dopsx:update <工作名>` | 修改计划 |
| `/dopsx:sync <工作名>` | 先把要求汇总进要求库 |
| `/dopsx:bulk-archive` | 批量归档 |

试试用 `/dopsx:propose` 开始第一件工作。
```

Exit gracefully.

---

## Guardrails

- **Follow the EXPLAIN → DO → SHOW → PAUSE pattern** at key transitions (after explore, after the proposal draft, after the checklist, after verification, after archive)
- **Keep narration light** while producing documents—teach without lecturing
- **Don't skip phases** even if the job is small—the goal is teaching the workflow (skip Phase 6 only when specs is skipped, and say why)
- **Pause for acknowledgment** at marked points, but don't over-pause
- **Handle exits gracefully**—never pressure the user to continue
- **Use real document tasks**—pick from the user's own materials or their own request; don't invent a fictional example
- **Follow the dopsx rules in every phase**: requirements need sources and checked dates; checklist items need owner markers; never tick a 【客户】 or 【官方】 item without evidence; never change the original materials; archive only through the `/dopsx:archive` checks
- **Keep personal information out of the requirement library, and credentials out of every file**
- **Adjust scope gently**—guide toward smaller tasks but respect user choice
