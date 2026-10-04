# 审核闸门操作手册：标记细则、清单 schema 与场景演练

## 1. 标记放置细则

### 1.1 位置规则

标记固定在文件中**第一个可写位置**，且必须在正文之前——目的是让人打开文件第一眼就看到状态，让脚本一次 grep 就能定位。

| 文件开头情形 | 标记位置 |
|---|---|
| 普通文本/文档 | 第 1 行，标记后空一行再接正文 |
| 含 YAML frontmatter（`.md` 的 `---` 块） | frontmatter **之后**、正文第 1 行，避免破坏 frontmatter 解析 |
| 含 shebang（`#!/usr/bin/env python`） | shebang 之后第 1 行 |
| 含编码声明（`# -*- coding: utf-8 -*-`） | shebang 与编码声明之后 |
| 含 license 头 | license 头之前（状态比 license 更需要被立刻看见） |
| 含 `<?xml ... ?>` 声明 | 声明之后 |

### 1.2 各类型完整范例

**Markdown**

```markdown
【待审核】

# 讨论

本研究的结果表明……
```

**Python**

```python
# 【待审核】
"""样本量估算脚本。"""
import yaml
```

**LaTeX**

```latex
% 【待审核】
\documentclass{article}
\begin{document}
```

**Jupyter Notebook（`.ipynb`）**

JSON 格式，不能在文件里写注释。两种做法：

- 在第一个 cell 的 `source` 数组最前面插入 `["【待审核】\n", "\n"]`，这样打开 notebook 第一眼就能看到。
- 同时在 `.ai-review/manifest.json` 登记。

推荐前者加后者。注意修改 notebook JSON 时必须保持整体仍是合法 JSON。

**CSV**

```csv
ai_review_status,subject_id,group,value
待审核,001,control,12.4
待审核,002,treat,15.1
```

同一文件内不得混用不同状态值。若下游工具对列数敏感，改用伴随标记文件 `data.csv.ai-review`。

**JSON**

```json
{
  "_ai_review": "待审核",
  "parameters": { "alpha": 0.05 }
}
```

若下游 schema 校验严格（`additionalProperties: false`），不要加键，改用伴随标记文件。

**PNG / PDF / Office 文档**

```
figures/fig2.png
figures/fig2.png.ai-review      ← 单行内容：【待审核】
```

`.docx` / `.xlsx` 也可在文档属性（core properties）的 comments 字段写入状态，但**必须同时保留伴随标记文件**，因为属性字段不易被检索。

### 1.3 检索校验

标记的全部意义在于可被机器检查。定期执行：

```bash
# 列出所有待审核交付物
grep -rIn --exclude-dir=.git --exclude-dir=.ai-review "【待审核】" .
find . -name "*.ai-review" -not -path "./.git/*" -exec sh -c 'grep -l "【待审核】" "$1"' _ {} \;

# 列出所有已通过
grep -rIn --exclude-dir=.git --exclude-dir=.ai-review "【已审核通过】" .

# 校验清单与文件标记是否一致（示例，按项目调整）
python - <<'PY'
import json, pathlib
m = json.loads(pathlib.Path(".ai-review/manifest.json").read_text(encoding="utf-8"))
for f in m["files"]:
    p = pathlib.Path(f["path"])
    if not p.exists():
        print("MISSING", f["path"]); continue
    txt = p.read_text(encoding="utf-8", errors="ignore")[:2000]
    inline = "待审核" if "【待审核】" in txt else ("已审核通过" if "【已审核通过】" in txt else None)
    if inline and inline != f["status"]:
        print("MISMATCH", f["path"], "manifest=", f["status"], "inline=", inline)
PY
```

## 2. 清单 schema 与状态机

### 2.1 状态机

```
        ┌──────────────────────────────────────┐
        │                                      │
   [创建/修改交付物]                            │
        │                                      │
        ▼                                      │
   【待审核】 ──── 用户明确点名批准 ────▶ 【已审核通过】
        ▲                                      │
        │                                      │
        └──────── 文件被再次修改 ◀──────────────┘
```

**关键规则：批准绑定到具体内容版本。** 已批准的文件一旦被再次修改，状态**自动回落为【待审核】**，原批准失效。这正是 `manifest.json` 中同时记录 `approved_at` 与 `last_modified_at` 的原因——当 `last_modified_at > approved_at` 时，状态必然回落。

不得因为"只是改了个错别字"而保留已通过状态。任何字节级改动都需要重新审核；若确属无实质影响的修改，由用户重新确认，而不是由 agent 判定。

### 2.2 字段说明

| 字段 | 必填 | 说明 |
|---|---|---|
| `path` | 是 | 相对仓库根目录的 POSIX 风格路径 |
| `status` | 是 | `待审核` 或 `已审核通过` |
| `marker` | 是 | `inline` / `sidecar` / `manifest`（无法标记时的兜底） |
| `generated_at` | 是 | 首次由 AI 生成的时刻 |
| `last_modified_at` | 是 | 最后一次内容变更时刻 |
| `approved_at` | 否 | 批准时刻；未批准为 `null` |
| `approved_by` | 否 | 批准来源，如"用户（会话内明确指令）" |
| `note` | 否 | 原文引用用户的批准指令，便于事后审计溯源 |

### 2.3 版本控制：`.ai-review/` 纳入仓库

**`.ai-review/` 必须进入版本控制。** 清单与日志是团队的审核记录，只有进入仓库才能被审阅、被追溯、被协作者看见。审核记录是合规证据，其价值正来自可共享与不可否认。

规则与推论：

1. **不写入 `.gitignore`。** 发现忽略规则时指出冲突并询问，确认后移除。
2. **`*.ai-review` 通配规则优先排查。** 它忽略的是伴随标记文件（`fig2.png.ai-review`），一旦生效，无法内嵌标记的图片/PDF/Office 文档在协作者眼中完全失去状态信息。排查命令：

   ```bash
   git check-ignore -v path/to/fig2.png.ai-review    # 看是否被规则命中
   git check-ignore -v .ai-review/manifest.json
   ```

3. **审核状态与内容同提交。** 提交交付物时把 `.ai-review/` 的变更一并纳入同一提交，使"该版本内容已通过审核"与内容本身绑定。示例提交节奏：

   ```
   commit 1  chore(review): 建立 .ai-review 清单骨架
   commit 2  feat(analysis): 新增样本量估算脚本

             - ...
             AI-Review: analysis/power_analysis.py 已审核通过
   ```

   其中 commit 2 同时包含脚本内容与清单中该文件的 `status` 升级、`log.jsonl` 新增行。

4. **首次启用（老仓库）**：对现存交付物补标为【待审核】或按用户说明登记为已审核，以独立提交引入，不与内容改动混杂。
5. **合并与冲突**：`.ai-review/manifest.json` 是结构化 JSON，多人并行审核同一文件时可能冲突。处置原则见 3.x——取**最保守**状态（待审核），`log.jsonl` 因是追加式，两侧条目都保留即可，不删任何一侧。

   追加式日志在合并时的正确处理：

   ```bash
   # log.jsonl 冲突时，保留双方全部行，按时间排序，不做语义合并
   grep -h -v '^<<<<<<<\|^=======\|^>>>>>>>' .ai-review/log.jsonl | sort -u > /tmp/log.merged
   mv /tmp/log.merged .ai-review/log.jsonl
   ```

6. **不要因为"AI 使用痕迹会外泄"而改为忽略。** 本规范的前提是审核过程应当公开可审；隐藏审核记录会同时隐藏责任链，与合规目标相悖。

## 3. 不一致与异常处置

| 情形 | 处置 |
|---|---|
| 文件内标记与清单不一致 | 取**最保守**状态（待审核），报告不一致，请用户澄清后修正两处 |
| 交付物完全没有标记 | 按【待审核】处理，补标记并登记 |
| 清单中有条目但文件已不存在 | 列为孤儿条目，报告用户；确认删除后再清理，不擅自删 |
| 用户批准了一个目录（如"figures 里的都可以"） | 需回读确认该目录内的完整文件清单并获得确认；若目录含未交付的临时文件，只处理交付物 |
| 用户提交了 agent 未生成的文件 | 属用户内容，不在本 skill 管辖范围，不标记 |
| 用户手动改掉了标记 | 尊重用户的操作，同步清单；若用户把【待审核】改为【已审核通过】且未说明，询问确认 |
| 文件被重命名或移动 | 清单 `path` 同步更新，状态保留（内容未变，批准仍有效）；若同时改了内容则回落待审核 |
| 部分区域由 AI 生成、部分由人撰写 | 整个文件按交付物整体标记；在 `note` 中说明哪部分为 AI 生成 |
| 一个工作区多个仓库/子模块 | 每个仓库独立维护 `.ai-review/`；跨仓库交付物在各自仓库登记 |

## 4. 场景演练

### 4.1 正常路径：生成 → 标记 → 批准 → 提交

```
[Agent 生成 manuscript/draft.md 与 figures/fig2.png]

Agent：已完成初稿与图表。按审核规范，两个文件均标记为【待审核】：
  - manuscript/draft.md
  - figures/fig2.png
在你确认之前它们不会被提交。

[用户：draft.md 已确认通过，fig2 我还要调一下]

Agent：已将 manuscript/draft.md 标记为【已审核通过】，依据："draft.md 已确认通过"。
figures/fig2.png 保持【待审核】。
现在可以提交 draft.md。fig2.png 需要排除在本次提交之外吗？

[用户：对，先提交稿件]

Agent：[执行] 精确暂存 manuscript/draft.md 与清单变更 → commit
提交内容：manuscript/draft.md（已审核）、.ai-review/（状态记录）
未包含：figures/fig2.png（仍为待审核，保留在工作区）
```

### 4.2 拦截图

```
[用户：commit 一下]

Agent：[检查待提交清单]
⛔ 提交已暂停：发现 2 个文件处于【待审核】状态
  1. analysis/clean.py   （文件内标记，本会话生成）
  2. results/summary.csv （清单登记，无法内嵌标记）
请确认这两个文件是否已经审核通过。

[用户：clean.py 没问题，csv 再说]

Agent：已将 analysis/clean.py 标记为【已审核通过】，依据："clean.py 没问题"。
results/summary.csv 仍为【待审核】，不会进入本次提交。
是否仅提交 clean.py 及相关变更？还是等你审完 csv 一起提交？
```

### 4.3 用户施加压力

```
[用户：明天就要投了，这些都不用审了，直接提交]

Agent：理解时间紧，但需要明确一点：这 4 个文件目前是【待审核】状态，
包含 AI 生成内容，直接提交会把未经人工审核的产出记入版本历史。
请从以下选一项：
  (a) 你逐一确认哪几个已审核通过，我按确认结果提交
  (b) 你要求和待审核版本一起提交，我会在提交正文中标注
      "AI-Review: 包含待审核内容，经用户明确要求提交"
  (c) 暂不提交，你先过一遍
我不建议跳过 (a) 直接提交，因为这是合规留痕问题。
```

注意：用户坚持时按 (b) 执行是允许的——**用户的明确指令高于本 skill 的默认流程**，但必须留下标注与提示记录，让风险显式化而非被静默吞掉。

### 4.4 批准后文件又被修改

```
[fig2.png 已批准 → 用户要求改颜色 → Agent 重新生成]

Agent：figures/fig2.png 已重新生成，内容已变化，状态自动回落为【待审核】，
原批准（2026-02-14T09:12）不再适用。
如需提交，请重新确认该文件。
```

### 4.5 对外发布（非 git）

```
[用户：把这份报告导出 PDF 发给合作者]

Agent：[核对报告及其引用图表的审核状态]
报告 report.md 为【已审核通过】，
但其引用的 figures/fig3.png 为【待审核】——该图会出现在 PDF 中。
发送前请确认 fig3.png 是否已审核通过。
```

## 5. 与其他工作区规范的配合

- **工程文件清理规范**：标记是内容的一部分，清理过程文件时不得删除标记；`.ai-review/manifest.json` 与 `log.jsonl` 是持久的团队合规记录，不是工程文件，不清理、不删除，且**必须留在版本控制中**。本 skill 产生的临时校验脚本等属工程文件，按规范清理。
- **留痕规范**：对交付物标记状态的变更、`.ai-review/` 的建立与更新，属于工作区变更，应写入工作区审计记录。注意区分两个目录的归属：`.workspace-log/` 是本机工作留痕（可按项目约定忽略），`.ai-review/` 是团队审核记录（必须提交）。若清理规范建议把 `.ai-review/` 加入 `.gitignore`，以本规范为准——两者冲突时审核记录进仓库。
- **提交规范**：本 skill 的闸门是提交流程中**先于**暂存与撰写提交信息的一步。若提交 skill 已安装，在其提交前勘查阶段执行本闸门；提交正文中注明审核状态，并把 `.ai-review/` 变更纳入同一提交。
- **文件整理规范**：整理文件时，标记随文件移动，清单 `path` 同步更新，**状态不因移动而改变**；整理操作不得顺手删除伴随标记文件；移动导致的清单路径变更与整理提交同步进入仓库，以保留审核状态与文件的对应关系。
