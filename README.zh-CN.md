[English](README.md) | **中文** | [日本語](README.ja.md)

# dopsx：文书工作版的 OpenSpec 指令

dopsx 是一个 Claude Code 插件，用来处理文书工作，例如公司设立、许可申请、合同、请求书、通知函。流程和 OpenSpec 的 `/opsx` 一样：探讨 → 写计划 → 执行 → 验收 → 归档。底层用的是同一个 openspec 程序，只是处理的对象从代码换成了文书。

`/opsx` 是为改代码设计的：执行时改代码，验收时查代码和测试。拿它做文书工作，验收方法对不上，也没有「轮到客户或官方来做」的等待状态。dopsx 补上了这些：

- 清单里每一项都标明由谁来做：【我方】【客户】【官方】。
- 验收检查的是文书本身：成果物是否齐全，名称、住址、日期、金额在各份文书之间是否一致，有没有漏填。
- 核对过的官方要求连同出处和确认日期存进要求库，下一件同类工作可以直接用。

> dopsx 是个人开发的非官方社区扩展，与 OpenSpec 及其维护方（Fission AI）没有从属关系。

## 前提条件

- Claude Code
- Node.js
- openspec 程序 1.12.0（dopsx 在这个版本上测试过）

## 安装、更新、卸载

先安装 openspec 程序：

    npm install -g @fission-ai/openspec@1.12.0

再在终端里安装插件：

    claude plugin marketplace add Emiyashare/dopsx
    claude plugin install dopsx@dopsx

也可以在 Claude Code 会话里依次输入 `/plugin marketplace add Emiyashare/dopsx` 和 `/plugin install dopsx@dopsx`。装好后，新开的会话里就有 `/dopsx:*` 十二个指令。Windows、macOS、Linux（包括通过 SSH 使用的远程主机）都是同样的步骤，不需要运行任何安装脚本。

更新到最新版本（更新后，新开的会话才会用上新版本）：

    claude plugin marketplace update dopsx
    claude plugin update dopsx@dopsx

卸载：

    claude plugin uninstall dopsx@dopsx

卸载只移除插件本身，已经建立的文书工作（`openspec/` 和 `documents/`）不受影响。

## 十二个指令

**常用**

| 指令 | 做什么 |
|---|---|
| `/dopsx:explore` | 探讨：一起把这件文书工作想清楚，不制作文书 |
| `/dopsx:propose` | 建立工作，并一次写完 proposal、specs、design、tasks 四份计划文件 |
| `/dopsx:apply` | 执行：按清单制作文书；轮到客户或官方的项目会停下来等 |
| `/dopsx:verify` | 验收：清单和成果物是否齐全，各文件里的名称、住址、日期是否一致，有没有漏填，出处是否过旧，要求库里有没有个人信息 |
| `/dopsx:archive` | 归档：还有未决事项就不许归档；计划搬进归档，成果物原地保留 |

**分步写计划**

| 指令 | 做什么 |
|---|---|
| `/dopsx:new` | 只建立工作，不写计划，显示第一份计划文件的写法 |
| `/dopsx:continue` | 写下一份计划文件，一次一份，写完可以先看再继续 |
| `/dopsx:ff` | 建立工作并一次写完计划；只在关键信息缺失时才问，其余先按合理假设写进计划 |

**其他**

| 指令 | 做什么 |
|---|---|
| `/dopsx:update` | 中途修改计划，并保持四份计划文件一致 |
| `/dopsx:sync` | 不归档，先把核对过的要求汇总进要求库 |
| `/dopsx:bulk-archive` | 一次归档多件已完成的工作；每件照样检查，有未决事项的不归档；几件工作的同一条要求写法不同时，以确认日期较新的为准 |
| `/dopsx:onboard` | 入门教程：用当前文件夹里一件小的真实文书，带讲解走一遍完整流程 |

也可以不输入指令，直接用平常的话说要做什么，Claude 会选用对应的指令。

一般的顺序：

- 一次写好计划：`/dopsx:propose` → `/dopsx:apply`（可能分几次，中间等客户或官方）→ `/dopsx:verify` → `/dopsx:archive`
- 一份一份写、边写边看：`/dopsx:new` → `/dopsx:continue`（重复到四份计划写完）→ `/dopsx:apply` → `/dopsx:verify` → `/dopsx:archive`
- 想先聊清楚再开始，用 `/dopsx:explore`；第一次用，可以先跑一遍 `/dopsx:onboard`

propose、new、continue、ff 的区别：

| 指令 | 建立工作 | 写计划 | 遇到不清楚的地方 |
|---|---|---|---|
| propose | 是 | 一次写完四份 | 影响做哪些文书、给谁、期限的问题，先问清楚再写 |
| new | 是 | 不写 | — |
| continue | 否（接着已有的工作写） | 一次写一份 | 不清楚就先问 |
| ff | 是 | 一次写完四份 | 只在关键信息缺失时问，其余先假设并写进计划 |

四份计划文件写什么：

| 文件 | 内容 |
|---|---|
| proposal | 做哪些文书、给谁、期限、成果物清单、待确认事项 |
| specs | 文书必须满足的要求（官方规定、客户要求、格式要求），每条写明出处和确认日期 |
| design | 做法：用哪个官方样本、格式、签名方式、提交方式、文件命名、预期的空白、待确认问题 |
| tasks | 清单：每项标明负责人和完成判断 |

没有需要核对的要求的简单文书（例如一张常规请求书），specs 会自动跳过。

## 目录约定

在哪个文件夹开始第一件工作，建立工作的指令（`/dopsx:propose`、`/dopsx:new`、`/dopsx:ff`）就在那个文件夹里自动生成下面的结构，不需要先做别的：

```
<开始的文件夹>
+-- openspec
|   +-- config.yaml          schema: doc-driven
|   +-- schemas/doc-driven   文书版流程定义（从插件复制过来）
|   +-- specs                要求库
|   +-- changes
|       +-- <工作名>          计划、草稿、过程材料
|       +-- archive          归档后的计划
+-- documents
    +-- <工作名>              正式版成果物，归档后也留在原地
```

- `openspec/schemas/doc-driven`：文书版流程定义。建立文件夹时从插件复制一份，所以这个文件夹拷到别的电脑上也能接着用；进行中的工作一直用开工时的那一版。
- `openspec/specs`：要求库，核对过的要求都汇总在这里，下一件同类工作直接用
- `openspec/changes/<工作名>`：这件工作的计划文件、草稿、过程材料
- `openspec/changes/archive`：归档后的计划
- `documents/<工作名>`：正式版成果物，归档后也留在原地，并多一份 `过程记录.md`，写明计划归档到了哪里

补充：

- 在子文件夹里开始时，会自动沿用上层已有的 `openspec`，不会重复建立。
- 要求库跟着 `openspec` 文件夹走：在同一个文件夹里做的工作共用一个要求库；在不同文件夹开始，要求库就各自分开。
- 如果要把正式版另存一份到客户的公司文件夹，把「复制到公司文件夹」写成清单的最后一项。
- `/dopsx:apply` 不会改动你原来的材料（`documents` 和 `openspec` 以外的文件）：要改的文件先复制一份再改。确实要直接改原文件时，会先问你。
- 如果这个文件夹已经是写代码用的 OpenSpec 项目，建立文书工作之前会先问你。

## 工作名的规则

openspec 只接受小写英文字母、数字和短横线。日文、中文、大写字母、下划线都会被拒绝。

- 可以用：`sample-company-setup`、`2026-office-move`
- `documents` 下的文件夹名和工作名完全一样
- 文书本身的文件名可以用日文或中文，例如 `documents/sample-company-setup/就任承諾書.docx`

## 负责人标记

清单每一项写成：`- [ ] 1.1 【我方】事项；完成判断：…`

- 【我方】：我们来做，`/dopsx:apply` 会执行
- 【客户】：客户本人来做，例如提供资料、签名、到场
- 【官方】：等官公署、银行等处理，例如受理、审查、发证

客户或官方的项目还没完成时，`/dopsx:apply` 会停下来，告诉你在等谁、等什么。对方完成后告诉 Claude，确认过凭证才会勾选。还有未勾选的项目，`/dopsx:archive` 和 `/dopsx:bulk-archive` 都不会归档。

## 个人信息边界

- 要求库（`openspec/specs`）只写规定本身、出处和确认日期，不写任何客户的姓名、住址、生日、证件号、账户等。
- 账号、密码、暗证番号不写进任何计划文件，清单里只写「由客户本人操作」或凭证存放的位置。
- 客户的姓名、住址可以出现在工作文件夹里的计划文件中。这些文件夹请只放在自己的电脑或私有的存储里，不要推送到公开的 Git 仓库。
- 验收和归档时会检查要求库里有没有混入个人信息；有的话不会汇总，也不会归档。

## 语言

计划文件的模板、填写说明和负责人标记（【我方】【客户】【官方】）目前是中文。Claude 会用你说话的语言和你交流，计划的内容也用你的语言填写，但模板里的标题和负责人标记保持中文。

## 只使用流程定义

不用 Claude Code、只想在自己的 OpenSpec 项目里使用 doc-driven 流程定义时，按 OpenSpec 社区流程定义的惯例：把本仓库的 `schemas/doc-driven/` 复制到项目的 `openspec/schemas/doc-driven/`，再把 `openspec/config.yaml` 里的 `schema` 改成 `doc-driven`（或者建立工作时加上 `--schema doc-driven`）。

确认方法：

    openspec schema which doc-driven
    openspec schema validate doc-driven

这样得到的只是流程定义（四份计划文件的模板和填写说明）。等待客户或官方、文书验收、归档前的检查这些行为都在 `/dopsx:*` 指令里，需要 Claude Code 插件。

## 专业资格

有些文书依法只能由有资格的专业人士（例如日本的行政书士、司法书士、税理士、社会保险劳务士）代为制作或提交。dopsx 只是整理计划、制作文书的工具，不判断谁有资格办理某类文书，也不提供法律意见。请在使用前自行确认，需要时交给有资格的专业人士办理。

## 维护

**修改流程定义**：`schemas/doc-driven/` 只改填写说明和模板里的文字。不要改 proposal、specs、design、tasks 这四个计划文件的名字，也不要改它们的先后关系。各个文件夹里的旧版副本，就是靠这一点才能继续配合新版指令使用。

**让已有的文件夹用上新模板（可选）**：用新版本的 `schemas/doc-driven/` 覆盖那个文件夹的 `openspec/schemas/doc-driven/`。四个计划文件的名字和先后关系不变，所以进行中的工作不受影响。

**升级 openspec 之后的检查步骤**：

1. 升级 openspec：`npm install -g @fission-ai/openspec@latest`
2. 在一个空的临时文件夹里，用一件虚构的小文书跑一遍：`/dopsx:propose` → `/dopsx:apply` → `/dopsx:verify` → `/dopsx:archive`，中间包含一次客户等待，确认没有报错、归档后要求库里有这次的要求
3. 对照新版 OpenSpec `/opsx` 的工作流，看有没有值得搬到 dopsx 的改进；新版如果多了新的工作流，也在 `skills/` 里补一份对应的正文，保持一一对应
4. 把三份说明文档和各技能 `compatibility` 里测试过的 openspec 版本改成新的版本号

**发布新版本**：修改 `skills/` 或 `schemas/` 后，把 `.claude-plugin/plugin.json`、`.claude-plugin/marketplace.json` 和各技能 `metadata` 里的版本号一起调高，运行 `claude plugin validate . --strict`，三份说明文档在同一次提交里一起更新。

## 许可与来源

- 许可：MIT，见 [LICENSE](LICENSE)。
- 各指令的正文照 OpenSpec（https://github.com/Fission-AI/OpenSpec ，MIT 许可）的 `/opsx` 工作流改写，LICENSE 里保留了 OpenSpec 的版权声明。
- dopsx 是非官方的社区扩展，与 OpenSpec 及其维护方没有从属关系。
