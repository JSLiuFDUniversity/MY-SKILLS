# 工程文件分类细则、审计 schema 与边界案例

## 1. 常见工程文件清单（按科研工作流）

以下清单用于快速判定，不是穷举。命中即视为工程文件，除非它是证据链。

### 文献与文档处理

- PDF 抽取脚本（`extract_*.py`）、抽取出的纯文本/JSON 中间件
- 版面分析或 OCR 产生的逐页图片、裁剪图
- 参考文献去重脚本、临时 BibTeX 合并文件
- 文档格式转换的中间版本（`.docx` → `.md` → `.tex` 的中间态）
- 下载的 PDF 原文件：**不是工程文件**，除非用户明确说可删；视为用户输入

### 数据处理

- 清洗脚本、一次性 ETL 脚本
- 中间宽表/长表、临时 parquet/csv、join 用的键表
- 统计检验的探索性 notebook
- 环境锁定文件（仅当为本次任务临时建立的隔离环境时）

### 图表与产出

- 草稿图（`fig1_draft.png`、`fig1_v2.png` 之外的迭代版本）
- 图表渲染的中间 SVG、字体子集
- LaTeX 编译产物：`.aux` `.log` `.out` `.bbl` `.synctex.gz` `.fls` `.fdb_latexmk`、临时 `.pdf`
- Markdown → HTML/PDF 的中间件

### 系统与环境

- `__pycache__/`、`.pytest_cache/`、`.ipynb_checkpoints/`、`.mypy_cache/`、`.ruff_cache/`
- `.Rhistory`、`.Rproj.user/`
- 临时虚拟环境、`node_modules/`（若为本次任务安装）
- 下载到本地的模型权重/依赖包缓存（若可重新下载）
- 日志文件、`nohup.out`、调试输出重定向文件
- 临时目录 `tmp/`、`.scratch/`、`out_tmp/`

## 2. 证据链判定

满足以下任一条即为证据链，**保留**：

1. 交付物的结论依赖它，且没有别的途径重算。
2. 它是最终图的唯一数据来源（绘图用的 tidy 数据表）。
3. 它是分析脚本的中枢（被其他脚本 `source` / `import`），删了会破坏流水线。
4. 用户或合作者可能在审稿、复核、答复质疑时索要它。
5. 生成它依赖不可重复的资源（一次性的检索时点、会变化的外部 API 快照、昂贵计算）。

典型易误删项：

- 从 PDF 抽出的表格 CSV —— 若正文汇总基于它，就是证据链。
- 随机种子、抽样得到的子集清单 —— 删了无法复现同一结果。
- 检索式与检索日期记录（PRISMA 类综述必需）。
- 手工修订过的中间表：含人工判读内容，不可由脚本重建。

## 3. 审计记录完整示例

首条记录（文件刚创建）：

```json
{"ts":"2026-02-14T09:05:11+08:00","session":"pdf-extract-2026-02-14","task":"从 3 篇 PDF 提取方法学段落并汇总为 methods.md","created":[{"path":".workspace-log/","purpose":"建立审计留痕目录（自豁免，不再逐条记录）"},{"path":"scripts/extract_pdf.py","purpose":"pypdf 批量抽取正文并定位 Methods section","status":"deleted"},{"path":"tmp/pages/*.png","purpose":"版面分析逐页中间图","status":"deleted"},{"path":"out/methods.md","purpose":"三篇文献方法学段落汇总，本次主要交付物","status":"delivered"}],"deleted":[{"path":"tmp/pages/*.png","purpose":"版面分析中间图，已用毕"},{"path":"scripts/extract_pdf.py","purpose":"一次性抽取脚本，抽取结果已固化在 out/methods.md"}],"modified":[{"path":".gitignore","purpose":"追加 .workspace-log/ 忽略规则，避免审计文件进入版本控制"}],"notes":"用户要求保留原始 PDF；tmp/ 已清空并移除；抽取脚本未保留，若后续需重跑请告知"}
```

同一次会话的第二条记录（追加，不覆盖第一条）：

```json
{"ts":"2026-02-14T11:48:02+08:00","session":"pdf-extract-2026-02-14","task":"根据用户反馈补入第 4 篇文献并更新 out/methods.md","created":[{"path":"tmp/paper4.txt","purpose":"第 4 篇 PDF 的纯文本中间件","status":"deleted"},{"path":"out/methods.md.bak","purpose":"更新前的备份","status":"deleted"}],"deleted":[{"path":"tmp/paper4.txt","purpose":"中间文本，内容已并入交付物"},{"path":"out/methods.md.bak","purpose":"备份，已确认更新无误"}],"modified":[{"path":"out/methods.md","purpose":"新增第 4 篇方法学段落并调整小节顺序"}],"notes":"未新增交付物文件；本次无保留的过程产物"}
```

## 4. 记录一致性自检

写完记录后做一次机械核对，避免"记录说删了、文件还在"：

```
# 应输出为空或全部为有意保留的临时目录
ls -la
find . -name "__pycache__" -o -name ".ipynb_checkpoints" -o -name "*.tmp" 2>/dev/null

# 审计文件行数应等于本次会话的变更次数累计
wc -l .workspace-log/audit-log.jsonl

# 校验每一行都是合法 JSON
python -c "import json,sys;[json.loads(l) for l in open('.workspace-log/audit-log.jsonl',encoding='utf-8') if l.strip()];print('ok')"
```

## 5. 边界案例与处置

### 5.1 一次性脚本该不该留

默认删。但脚本本身就是可复用工具时，保留并登记为**交付物**，放进 `scripts/`（而非临时目录），因为下次同类任务能直接复用。判定问题：「下周做同类分析，我会想再跑它吗？」会 → 留；不会 → 删。

### 5.2 用户中途反悔，要求找回已删文件

工程文件已删除且不可恢复时，如实说明，并给出重建路径（脚本可重写、中间件可由原始输入重算）。不要假装文件还在，也不要用缓存里内容不同的版本冒充。

### 5.3 工作区里原本就有脏文件

只清理**本次任务产生**的文件。任务开始前就存在的散乱文件不是本次责任，登记在 `notes` 里提示用户即可；是否整体整理属于项目结构整理工作，另行处理。

### 5.4 同名目录嵌套多个 git 仓库

子仓库内的删除操作要单独确认其工作区状态，避免把未提交的改动一并丢掉。递归删除前先 `git -C <path> status --porcelain` 检查。

### 5.5 大文件

删除大体积中间件（GB 级）前先确认它不是唯一的本地副本；若是从网络获取的，记录来源 URL 以便重取，再删。

### 5.6 软链接与硬链接

删除链接本身不等于删除目标。先 `ls -l` 看清指向，避免误删工作区外的数据。

### 5.7 只读文件系统或沙箱受限

无法删除时不要假装清场完成。明确报告"X 个文件因权限受限未能删除"，并写入 `notes`。

### 5.8 并发会话

多会话同时改同一工作区时，追加写仍安全（每行独立 JSON）。避免用"读整个文件 → 改数组 → 写回"的方式，那会覆盖其他会话的记录。
