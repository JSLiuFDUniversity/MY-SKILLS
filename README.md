# MY-SKILLS

个人日常使用的科研与工程场景 Agent Skills 集合。每个 skill 独立成目录，遵循 Claude Skills 规范：`SKILL.md` 承载触发条件与核心流程，`references/` 承载细则、模板与示例，按需加载。

## Skill 列表

| Skill | 用途 | 触发方式 |
|---|---|---|
| `workspace-hygiene-and-changelog` | 工作区整洁与留痕。把过程文件按工程文件/交付物/证据链三分类，交付前清除一次性产物，并在 `.workspace-log/audit-log.jsonl` 追加变更记录 | **常态生效**，只要存在于项目文件中即自动执行 |
| `project-file-organizer` | 项目文件结构整理。把平铺散落的文件按用途归类，重点处理文档类与不依赖相对路径引用的程序文件 | **用户显式调用**，且给出方案后必须经确认才动手 |
| `git-auto-commit` | Git 提交流程自动化。自动执行 commit，summary 与 description 双轨填写；冲突先分析再询问，push 前必问 | 用户提出 commit / 提交 / 推送时 |
| `ai-content-review-gate` | AI 内容审核闸门。为交付物标记【待审核】/【已审核通过】，未经明确批准的内容不得提交 | **常态生效**（git 仓库内） |

## 目录结构

```
<skill-name>/
├── SKILL.md                      # 主文件：YAML frontmatter + 触发条件 + 核心流程
└── references/                   # 细则、模板、示例、边界案例
    └── <topic>-playbook.md
```

`SKILL.md` 顶部为 YAML frontmatter，仅含 `name` 与 `description` 两个键。`description` 写明用途与触发条件（含触发关键词），这是 Agent 判断是否启用该 skill 的唯一依据。

## 四个 skill 的协同关系

这四个 skill 在设计上互补，可同时安装：

- **`workspace-hygiene-and-changelog`** 管「工作区里留下什么」——它决定过程文件的生命周期，是所有文件操作类任务的基础规范。
- **`project-file-organizer`** 管「工作区怎么组织」——它只在用户要求时介入，且必须先出方案再动手；整理动作本身受前者约束（需留痕、不删文件）。
- **`git-auto-commit`** 管「改动怎么进版本历史」——它在提交前需要知道哪些文件是可交付的。
- **`ai-content-review-gate`** 是前者的**闸门**：`git-auto-commit` 执行提交前先经过审核闸门，未通过审核的交付物不得进入提交。三者的留痕边界已明确划分：
  - `.workspace-log/` 是本机工作留痕，可按项目约定忽略；
  - `.ai-review/` 是团队审核记录，**必须进入版本控制**以便团队审阅。

## 关键约束（跨 skill）

- **不编造可验证事实。** 文献、DOI、数据集版本、试剂货号等一旦幻觉即为撤稿级事故，找不到就明确说找不到。
- **整洁让位于可复现性。** 删除任何文件前先判断「删掉后能否从留存的东西重新得到」；不能，即为证据链，保留。
- **未标记即待审核。** 审核状态默认保守，不因改动小、催促、权限授予而降级。
- **不可绕过的闸门。** 未经用户明确点名的文件不升级审核状态；用户坚持提交待审核内容时，风险必须在提交信息中显式留痕，而非被静默吞掉。
- **决策权归属。** 涉及 push、冲突解决、文件重排的决定一律留给用户；Agent 负责分析与建议，不代替决策。

## 安装

将 skill 目录整体复制到目标 Agent 运行时的 skills 目录即可，例如：

```bash
git clone https://github.com/JSLiuFDUniversity/MY-SKILLS.git
cp -r MY-SKILLS/<skill-name> ~/.claude/skills/
```

或以 submodule 形式并入具体项目：

```bash
git submodule add https://github.com/JSLiuFDUniversity/MY-SKILLS.git .skills
```

## 说明

规范依据 Claude Skills（`SKILL.md` + YAML frontmatter 的渐进披露结构）。若投放到其它 Agent 运行时，主要差异在 frontmatter 字段与是否需要注册清单，内容主体可复用。
