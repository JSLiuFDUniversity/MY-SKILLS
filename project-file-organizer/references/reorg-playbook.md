# 整理操作手册：风险分级、依赖排查与失败案例

## 1. 文件风险分级

移动前给每个文件定级。等级决定是否需要用户逐项确认。

| 等级 | 特征 | 处置 |
|---|---|---|
| **绿：安全** | 独立文档（`.md`/`.docx`/`.pdf`）、最终图表、无人引用的数据表、按自身位置无关方式调用的脚本 | 可批量移动 |
| **黄：需排查** | 程序文件、配置、被文档引用的资源、名称含路径语义的文件 | 先做依赖排查，方案中列明，确认后移动 |
| **红：默认不动** | 构建配置、入口文件、`.git`/`.gitignore`、隐藏配置、被 `source`/`import` 的模块、CI 配置、留痕目录 | 列入不可动清单，用户逐项明确要求才动 |

黄级判定的实操方法：**在每个候选脚本里搜索它读写过的路径**，再看有没有别的文件引用它。两个方向都干净，才降到绿。

## 2. 依赖排查清单

### 2.1 程序文件

- Python：`import`/`from ... import` 本地模块、`open()`、`Path()`、`pd.read_csv()`、`__file__` 相对拼接、`sys.path` 操作。
- R：`source()`、`read.csv()`、`here()`、`setwd()`、`.Rprofile`。
- Shell：`source`、相对路径参数、`cd` 后接相对路径。
- Notebook：`%run`、`!python`、相对路径读取、内核工作目录假设。**notebook 的相对路径依赖最隐蔽**，因为内核 CWD 常与文件位置无关。
- 通用：绝对路径、`os.chdir()`、符号链接目标。

### 2.2 文档类文件

- Markdown：`![](path)`、`[text](path)`、`include` 指令。
- LaTeX：`\input{}`、`\include{}`、`\includegraphics{}`、`\bibliography{}`、`\lstinputlisting{}`。
- Word/PPT：嵌入对象、链接的外部数据源（这类难以静态排查，需提醒用户人工确认）。
- HTML：`src`、`href`、`<link>`。

### 2.3 工具链与外部约定

- 依赖固定相对布局的工具：Quarto/R Markdown 的 `_site`、Sphinx、MkDocs、Jupyter Book、DVC、Snakemake/Nextflow 的路径配置。
- 期刊投稿系统的上传清单（有时要求特定文件名与位置）。
- 实验室内部共享盘约定、LIMS 导出位置。

### 2.4 版本控制

- 已在 git 中的文件：用 `git mv` 保留历史；移动会体现在 diff 里，可回退。
- 未跟踪文件：`git mv` 不适用，移动后**无法通过 git 恢复**。这是最需要回退清单的一类。
- 被 `.gitignore` 忽略的文件：常在本地很重要（数据、密钥、大文件）。移动时格外小心，且不要因为"git 不管"就当作可随意处置。

## 3. 回退清单格式

非 git 目录或存在未跟踪文件时，生成一份纯文本清单，逐行 `原路径\t新路径`：

```
# reorg-manifest-20260214-1042.txt
# 用途：文件整理回退依据。按此清单反向执行即可还原结构。
# 生成时间：2026-02-14T10:42:00+08:00
docs/notes.md	docs/meeting/notes.md
out/fig1.png	figures/fig1.png
analysis_clean.py	analysis/clean.py
```

反向还原脚本示例（执行前先核对路径，不要盲跑）：

```bash
# 从清单反向移动（第一二列互换）
awk '!/^#/ && NF>=2 {print $2 "\t" $1}' reorg-manifest-20260214-1042.txt \
  | while IFS=$'\t' read -r from to; do mkdir -p "$(dirname "$to")"; mv "$from" "$to"; done
```

## 4. 典型失败案例

### 4.1 移走了 notebook，内核找不到数据

`analysis.ipynb` 里写的是 `pd.read_csv("data.csv")`，与数据同层。移到 `analysis/` 后读取失败。**对策**：notebook 一律列黄级，方案里说明"移动后需要把路径改为 `../data/raw/data.csv`（或改用项目根锚定）"，并作为引用修正项列出。

### 4.2 破坏了 LaTeX 编译

把 `refs.bib` 归入 `references/` 后，`\bibliography{refs}` 失效。**对策**：`.tex` 与 `.bib` 的位置关系属于功能性约定，若移动则必须同步改 `\bibliography{}`，或干脆不动。

### 4.3 打散了流水线目录

把 `Snakefile` 依赖的 `scripts/` 拆分到 `analysis/`，规则里的相对路径全部失效。**对策**：识别工作流引擎后，把其管辖范围整体列为红级不分区。

### 4.4 "整理"变成"删除"

移动过程中顺手删掉了看起来重复的文件（如两个版本的图），事后发现被稿件引用。**对策**：本 skill 不删除；冗余文件只列不删。

### 4.5 大小写重命名在 Windows/macOS 上静默失败

`Data.csv` → `data.csv` 在大小写不敏感的文件系统上可能不生效或产生重复。**对策**：这类重命名用两步法（先改为临时名，再改为目标名），并在 git 中留意 `core.ignorecase`。

### 4.6 隐藏文件被漏掉或误动

复制/移动时通配符 `*` 不含隐藏文件，导致 `.Rprofile` 被漏；或反过来把 `.env`、`.ssh` 之类挪走。**对策**：显式统计隐藏文件，逐个定级，默认不动。

### 4.7 用户已按旧路径写好了文档

合作者的邮件、实验记录本、投稿信里引用了旧路径。**对策**：整理后交付一份"路径变更说明"，供用户转发给协作者。

## 5. 交付回执模板

```
## 整理结果

新建目录：figures/ tables/ docs/meeting/ analysis/
移动文件：23 个（文档 8、图表 6、脚本 5、数据 4）
引用修正：3 处（analysis/clean.py 的读取路径、docs/notes.md 的图片链接、manuscript.tex 的 \includegraphics）

未移动（及原因）：
- Makefile、pyproject.toml —— 构建配置，位置即语义
- run_all.sh —— 入口脚本，被 README 引用
- data/raw/ —— 原始数据整体保持原位
- .workspace-log/ —— 留痕目录，位置固定

回退：tmp/reorg-manifest-20260214-1042.txt（含全部原路径→新路径映射）
未处理：发现 3 组疑似重复文件（fig1_v1/v2、old/ 下草稿），已列出未删除，等你决定
```
