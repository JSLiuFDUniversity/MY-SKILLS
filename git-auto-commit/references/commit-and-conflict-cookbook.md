# 提交信息模板、冲突分析范例与故障处置

## 1. 提交信息模板

### 1.1 通用骨架

```
<type>(<scope>): <subject>        ← summary，≤72 字符，无句号

<body>                            ← description，详细说明
- 改动要点一
- 改动要点二
- 影响范围与注意事项

<footer>                          ← 可选
BREAKING CHANGE: <说明>
Refs: #<issue>
```

### 1.2 按改动类型的范例

**新增分析脚本**

```
feat(analysis): 新增样本量估算脚本 power_analysis.py

- 支持两独立样本 t 检验与卡方检验的效能/样本量互算
- 参数改为从 config.yaml 读取，替换原先散落在脚本里的硬编码值
- 结果写入 results/power/ 并附参数快照，便于复核
- 独立入口，不影响现有 analyze.py 流程

Refs: #12
```

**修正数据处理缺陷**

```
fix(data): 修正缺失值编码导致的均值偏移

- 原清洗脚本把 -999 当作有效数值参与计算，使 3 个变量的均值偏低
- 现统一将 -999/-888 识别为缺失并按变量类型处理（连续变量中位数、分类变量单独编码）
- 受影响图表 fig2、fig3 已重跑并更新
- 结论方向未变，但描述统计数值有变化，需同步更新稿件正文

Refs: #27
```

**文档更新**

```
docs(readme): 补充环境安装与数据获取步骤

- 新增 conda 环境导出文件的使用说明
- 补充原始数据申请途径与放置路径约定
- 修正旧版命令示例中已失效的参数
```

**重构**

```
refactor(analysis): 抽出重复的预处理逻辑为公共模块

- clean.py 与 clean_validation.py 中重复的 7 个步骤合并为 preprocess.py
- 两个入口改为调用公共模块，行为保持不变
- 已用同一输入比对重构前后输出，结果逐行一致

BREAKING CHANGE: 依赖 clean() 函数签名的外部脚本需改用 preprocess.clean()
```

**依赖升级**

```
build(deps): 升级 pandas 至 2.2 并适配 API 变更

- 替换已弃用的 append/iteritems 调用
- 修正因 dtype 推断变化导致的 2 处类型错误
- 全量分析流程已跑通，输出与升级前一致
```

### 1.3 质量自检

提交前逐条核对：

- [ ] summary 能让人不看正文就明白这次改了什么？
- [ ] summary 是否避免了 `update`、`修改`、`fix bug`、`完善功能` 这类零信息词？
- [ ] description 是否说明了**为什么**，而不只是**改了哪些文件**？
- [ ] 是否写清了影响范围（谁需要重跑、谁需要改代码、结论是否变化）？
- [ ] 语言与项目近期提交风格一致？
- [ ] 有没有夹带无关文件（编辑器配置、本地数据、临时产物）？

## 2. 冲突分析范例

### 2.1 输出格式

```
## 冲突分析

范围：2 个文件，共 4 处冲突
分支：feature/power-analysis → main
双方提交：
  ours   (main)          3f2a1b0 更新数据清洗规则
  theirs (feature)       9c4d7e2 新增样本量估算

### 文件 1：analysis/clean.py（3 处）

冲突块 1（第 42–51 行）
- ours：把缺失值阈值从 0.2 收紧到 0.1，并新增一行剔除逻辑
- theirs：在相邻位置新增了变量重命名映射
- 判读：两处改动目的不同、互不排斥，属于**并行新增**而非真冲突
- 建议：两侧都保留，重命名映射在前、阈值判断在后，避免映射未生效就做缺失判定

冲突块 2（第 88–95 行）
- ours：函数 clean_df 改名为 preprocess_df
- theirs：在 clean_df 内部新增了参数 strict
- 判读：**改名与改体相遇**。theirs 的新参数有实际价值，但基于旧函数名
- 建议：采用新函数名 preprocess_df，并把 strict 参数并入其签名；同步更新 theirs 分支中对该函数的 2 处调用

冲突块 3（第 130 行）
- ours：删除了一段已废弃的向后兼容代码
- theirs：修改了同一段代码中的一处拼写
- 判读：删除侧更合理（代码已确认无用），theirs 的拼写修正无意义
- 建议：接受删除

### 文件 2：config.yaml（1 处）

冲突块 1
- ours：新增 data.missing_threshold: 0.1
- theirs：新增 power.alpha: 0.05
- 判读：不同小节的新增，纯粹的位置冲突
- 建议：两个键都保留

### 风险与验证
- 采纳建议后，theirs 分支中 2 处对 clean_df 的调用必须同步改名，否则运行时报错
- 验证方式：解决后运行 pytest tests/ 与 python analysis/run_all.py，比对 results/ 输出
- 若验证不通过可回退：git rebase --abort（或 git merge --abort）

是否按上述建议处理？
  (a) 全部按建议解决
  (b) 只解决 config.yaml，clean.py 保留冲突标记由你处理
  (c) 放弃本次集成，回到集成前状态
  (d) 你先自己改，改完告诉我继续
```

### 2.2 冲突类型的判读要点

| 类型 | 特征 | 常规处置 |
|---|---|---|
| 并行新增 | 两侧在不同位置加内容 | 都保留，注意顺序依赖 |
| 同名改写 | 同一逻辑两种实现 | 择优或合并，说明取舍理由 |
| 改名 vs 改体 | 一侧重命名、一侧改内容 | 采用新名并把内容改动并入 |
| 删除 vs 修改 | 一侧删除、一侧改同一段 | 判断该代码是否真的还有用，倾向确认删除 |
| 移动 vs 修改 | 文件被移动/重命名同时被改 | 用 `git log --follow` 追踪，把改动应用到新路径 |
| 配置冲突 | 同一配置文件不同键 | 通常都保留 |
| 锁文件/生成物 | package-lock、编译产物 | 不手工合并，重新生成 |

### 2.3 冲突排查命令

```bash
git status                      # 冲突文件清单
git diff --name-only --diff-filter=U   # 仅未合并文件
git log --oneline --left-right HEAD...MERGE_HEAD    # 双方提交对比
git log --merge -p <file>       # 该文件的冲突相关提交
git diff                        # 查看冲突块内容
git checkout --ours <file>      # 取我方（需理解语义后再用）
git checkout --theirs <file>    # 取对方
git merge --abort / git rebase --abort   # 放弃集成
```

## 3. 常见故障处置

| 现象 | 原因 | 处置 |
|---|---|---|
| `Author identity unknown` | 未配置 user.name/email | 询问用户偏好后配置，或仅对本仓库设置 |
| push 被拒 non-fast-forward | 远端有新提交 | `git fetch` 后 rebase 或 merge；出现冲突则走冲突流程 |
| pre-commit hook 失败 | 格式、lint、测试未过 | 修复问题后重提；不用 `--no-verify` 绕过 |
| 提交了不该提交的文件 | 误 add | 未推送时可用 `git reset --soft HEAD~1` 撤回提交后重新提交；已推送则用新的 revert 提交，不改写历史 |
| 提示文件过大被远端拒绝 | 大文件进了历史 | 报告用户，说明需要 LFS 或历史重写，不擅自操作 |
| 分支无上游 | 首次推送 | `git push -u origin <branch>` |
| detached HEAD | 处于游离状态 | 报告并询问是否新建分支保存改动 |
| 大小写重命名未生效 | 文件系统不区分大小写 | 两步重命名，再提交 |

## 4. 原子提交建议

一个提交只做一件事。若本次改动混杂了功能新增与格式整理，主动建议拆成两个提交，并说明拆分方式（例如先提交格式化，再提交功能改动，便于日后二分定位）。用户要求一并提交时按其要求执行，但可在提交正文中分节说明。
