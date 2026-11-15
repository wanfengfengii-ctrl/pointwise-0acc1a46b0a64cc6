# 湖表在线备份发布审查

- 审查对象：`backup.py`、`reclaim.py`、`lake.py`；重放入口 `lab.exercise`、`checks.py`。
- 审查范围：正常完成、受控复制失败、`backup.selected/catalog/before_file/after_file` 检查点上的提交与回收交错。
  不覆盖任意进程崩溃恢复。
- 证据：`review_cases.json` 的 17 个用例全部由原 `lab.exercise` 重放；`review.json` 是
  `python3 evidence_gate.py record` 的真实输出，`verify` 会重新重放并逐字节比对。
- 判断依据：独立期望行由批次覆盖语义直接计算（`lab.py:31`，不调用 `Lake.read`）；实际行来自
  `backup.restore` 返回的 Lake（`lab.py:79-81`）。版本一致性同时比较 `selected_version`、
  回执 `declared_version` 与副本 `catalog_version`。
- **发布结论：不核准（NO-GO）。** 现有实现可以在 `complete.json` 存在、全部文件摘要通过的情况下
  恢复出错误版本或缺行；回收还会在正常交错中删除当前版本仍引用的对象。

## 1. 期望语义（独立于实现）

一次成功备份必须同时满足：

| 编号 | 不变量 | 独立判定 |
| --- | --- | --- |
| I1 | 副本目录 `head` = 回执 `version` = 选择时声明版本 | `catalog_version == declared_version == selected_version` |
| I2 | 回执/副本对象集合 = 声明版本完整对象集合 | `actual_rows` 能读出且行集等于期望 |
| I3 | 恢复的默认读 = 声明版本完整行集 | `actual_rows == expected_rows` |
| I4 | 提交与回收继续推进且不改变本次备份结果 | 交错事件出现在允许检查点，备份仍满足 I1-I3 |
| I5 | 载荷复制不跨目录写事务 | 复制前后无长写事务 |
| I6 | 受控失败无可接受完成标记，保护释放，残留责任明确 | `complete == false` 且 `restore` 抛错 |

I2/I3 不能由"每个文件摘要正确"推出：摘要只证明对象内容未变，不证明回执覆盖的是同一个版本。

## 2. 真实重放结果（当前实现）

`review.json` 中每个用例均含 `expected_rows`、`actual_rows`、`complete`、`backup_error`、
`restore_error`、`catalog_version`、`declared_version` 与完整事件序列。下表"期望"为独立计算的行集，
"实际"为恢复后 `read()` 的真实输出；`None` 表示 `restore` 未产生可读 Lake。

| 用例 | 期望版本 | 实际 版本/行 | complete | restore 异常 | 归类 |
| --- | --- | --- | --- | --- | --- |
| `case:commit-after-select` | 1 | 目录=2，行=不可读 | true | `FileNotFoundError` | 单因：版本快照分裂 |
| `case:return-after-select-rows-deceive` | 1 | 目录=3，行恰好相同 | true | 无 | **静默错版**（I1 违反被行集掩盖） |
| `case:commit-after-catalog-legit` | 1 | 1 / 行相同 | true | 无 | 合法交错 |
| `case:commit-during-copy-legit` | 1 | 1 / 行相同 | true | 无 | 合法交错 |
| `case:gc-plan-sweep-no-commit-legit` | 1 | 1 / 行相同 | true | 无 | 合法交错（无候选） |
| `case:commit-and-gc-after-select` | 1 | 无回执 | false | `FileNotFoundError` | 组合：提交+回收 |
| `case:commit-and-gc-after-catalog` | 1 | 无回执 | false | `FileNotFoundError` | 组合：提交+回收 |
| `case:commit-and-gc-mid-copy` | 1 | 无回执 | false | `FileNotFoundError` | 组合：复制中回收 |
| `case:commit-and-gc-after-last-file-legit` | 1 | 1 / 行相同 | true | 无 | 合法交错（删除发生在复制完成后） |
| `case:stale-candidate-live-sweep-before-backup` | 3 | 无回执 | false | `FileNotFoundError` | 单因：旧候选被再次引用后仍删 |
| `case:stale-candidate-at-select-masked` | 2 | 目录=3，不可读 | true | `FileNotFoundError` | 组合掩盖：回收错误被版本错误掩盖 |
| `case:plan-return-select-no-sweep-control` | 2 | 目录=3，不可读 | true | `FileNotFoundError` | 对照：不 sweep 也错版（证明 F1 独立） |
| `case:stale-candidate-sweep-at-catalog` | 3 | 无回执 | false | `FileNotFoundError` | 单因：回收裁决不复核 |
| `case:copy-fail-at-0` | 1 | 无回执 | false | `FileNotFoundError` | 受控失败基线 |
| `case:copy-fail-at-1-residual` | 1 | 无回执，目标残留 | false | `FileNotFoundError` | 受控失败+残留 |
| `case:copy-fail-masks-gc-deletion` | 1 | 无回执 | false | `FileNotFoundError` | 组合掩盖：OSError 掩盖旧候选已被删除 |
| `case:gc-before-backup-legit` | 2 | 2 / 行相同 | true | 无 | 合法交错（备份前回收） |

原 `checks.py repro` 的真实输出与此一致：`selected_version=1`、`declared_version=1`、
`catalog_version=2`、`complete=true`、恢复 `FileNotFoundError`，打印
`BACKUP_RESTORE_MISMATCH` 并退出 1。`checks.py verify` 的正常、历史读与复制失败断言保持通过。

## 3. 缺陷与源码定位

### F1 版本选择与目录快照分裂（阻断）

- `backup.py:20-24`：短事务读取 `head` 和该版本 `manifest` 后立即 `c.commit()`（第 24 行）。
- `backup.py:25`：发出 `backup.selected`，此时检查点可发生提交。
- `backup.py:26`：之后才复制 `catalog.db`，副本目录可能已经是新版本。
- `backup.py:41-42`：回执仍写选择时的 `version`，并照常写 `complete.json`。

结果：`case:commit-after-select` 中提交把源推进到版本 2，副本 `head=2`，回执声明 1；对象文件
全部复制成功、摘要全部匹配，`restore` 仍因目录版本 2 引用未被复制的对象而失败。
`case:return-after-select-rows-deceive` 更危险：源推进后行载荷回到初始内容，恢复行集与声明版本
**完全一致**，但副本当前版本是 3。若只比对行内容而不比对版本，会把错版当成功发布。

### F2 备份没有对象保护（阻断）

- `backup.py:29-30`：只从选择结果取对象名和摘要，不登记任何保护。
- `backup.py:31-40`：按文件名逐个 `read_bytes`（第 35 行），复制过程跨越回调。
- `reclaim.py:4-6`：`plan` 只按当前 `head` 计算存活集，完全不知道活动备份。

`case:commit-and-gc-after-select`：`backup.selected` 后提交到版本 2，`plan` 立刻把版本 1 的
`north` 对象列为候选，`sweep` 删除；备份在 `backup.py:35` 抛 `FileNotFoundError`。
违反 I4：正常提交与回收使备份失败。

### F3 回收删除裁决沿用过期候选（阻断，单独即可损坏当前版本）

- `reclaim.py:6`：候选在 `plan` 的快照里确定。
- `reclaim.py:11-21`：`sweep` 对每个候选仅 `BEGIN IMMEDIATE` 后检查 `path.exists()`（第 17 行）
  就 `unlink`（第 18 行），不复核当前 `head` 的存活集，也不看保护。

同摘要载荷再次引用时对象名不变（`lake.py:76-81`，`lake.py:79` 已存在则不写），所以旧候选可能
在 `plan` 与 `sweep` 之间重新变成当前版本的对象。`case:stale-candidate-live-sweep-before-backup`
（`initial→change→plan→return→sweep`）中，`return` 重新引用摘要 `2c32…` 的对象，`sweep` 仍删除它；
随后备份在选择版本 3 后读取该对象失败。该缺陷不需备份即可复现：直接
`plan → commit(return) → sweep` 后，当前版本 `read()` 抛 `FileNotFoundError`。
`case:stale-candidate-sweep-at-catalog` 证明在备份检查点上同样触发。

### F4 恢复校验不足（阻断，掩盖 F1）

- `backup.py:54`：只要求 `complete.json` 存在。
- `backup.py:58-62`：逐文件核对摘要，但不核 `catalog.db` 的 `head` 是否等于 `receipt["version"]`，
  也不核回执对象集合是否覆盖该版本完整清单。

因此 F1 产生的错版副本可以"摘要全部通过"并返回一个可用的 Lake（`return-after-select-rows-deceive`）。
摘要正确不能替代 I1/I2。

### F5 受控失败的保护释放与残留责任未定义（需明确）

- `backup.py:9-10`：目标目录和 `data/` 先创建。
- `backup.py:33-34`：`fail_at` 在复制前抛 `OSError`。
- `backup.py:45-49`：异常路径只追加 `backup.failed` 并写 `events.json`，不删除任何残留，也不存在
  可释放的保护。

实测 `fail_at=0` 残留 `catalog.db`、空 `data/`、`events.json`；`fail_at=1` 还残留第一个已复制对象。
`complete.json` 不存在，`restore` 拒绝，这一部分满足 I6 的"拒绝恢复"。但保护释放（修复 F2 后必须有）
和残缺目录清理没有责任声明。`case:copy-fail-masks-gc-deletion` 还显示：若失败前已有提交+回收，
`OSError` 会先于"对象已丢失"出现，使一次真实的回收数据损失被当作普通复制失败处理——清理残留时
不能把它当成单纯 I/O 故障。

### F6 原文档承诺失实（已修订）

`docs/backup.md` 原文称"目录一致+逐文件摘要即代表声明版本完整可恢复""备份不需要参与回收协议"。
两条均被上表反驳。修订稿改为 I1-I6 与删除时刻裁决顺序，并标注发布阻断。

## 4. 合法交错、单因与组合掩盖的区分

- **合法交错**（备份仍满足 I1-I3）：`commit-after-catalog-legit`、`commit-during-copy-legit`、
  `commit-and-gc-after-last-file-legit`、`gc-before-backup-legit`、`gc-plan-sweep-no-commit-legit`。
  其中 `commit-and-gc-after-last-file-legit` 说明：回收并非一律非法，只有删除仍在复制或仍被保护的
  对象才违反 I4。提交在 `backup.catalog` 之后发生时目录已固定，因此本身合法（`commit-after-catalog-legit`）；
  同样的提交发生在 `backup.selected` 之后却因 F1 失败（`commit-after-select`）。**提交时点**是单因变量。
- **单因**：`commit-after-select`（只有提交，无回收）已足以缺行/错版；`stale-candidate-live-sweep-before-backup`
  与 `stale-candidate-sweep-at-catalog` 只有过期候选回收问题。
- **对照**：`plan-return-select-no-sweep-control` 不执行 `sweep`，仍得到目录 3、回执 2 的错版，
  证明该失败由 F1 单独造成，不是 F3 的连带效应。
- **组合掩盖**：
  - `stale-candidate-at-select-masked`：F3 删除对象与 F1 错版同时存在，最终表现为 `FileNotFoundError`，
    容易被误判为"只是复制时机问题"；对照组证明版本错误独立存在。
  - `commit-and-gc-after-select/catalog/mid-copy`：F2 的失败以缺文件形式出现，但从
    `backup-events` 可看到删除发生在前；若只看异常类型会归错因。
  - `copy-fail-masks-gc-deletion`：受控 `OSError` 掩盖了先前回收已删除仍需复制的对象。

## 5. 最小兼容修复建议（本轮不实施）

只改 `backup.py` 与 `reclaim.py`，保留 `Lake` 公开接口、`lab.exercise` 交错点、事件名/顺序与全部原断言。
以下为建议，不是当前输出；`checks.py repro` 应继续作为缺陷基线存在。

1. **先快照目录再声明版本（F1）**：把 `clone_catalog` 移到 `backup.selected` 之前；从复制出的
   `catalog.db` 读取 `head` 与 `manifest`，以此作为 `version`、`entries` 和回执内容。声明版本与
   目录来自同一份副本，`backup.selected` 的 `version` 字段取自该副本。不得依赖事后比对或让备份失败。
   目录快照与对象清单读取用只读连接完成，不跨载荷复制持有写事务（I5）。
2. **登记/释放对象保护（F2）**：在目录库新增 `pins(token, object)`（或等价旁路注册表），备份在复制
   第一个载荷前以短事务写入 `token→objects`，在 `finally` 中删除该 token 的全部行。成功、`OSError`、
   摘要不符、目标写失败都执行；释放失败不得吞掉原始异常。回收侧 `plan`/`sweep` 读取活动 pin。
3. **删除时刻复核（F3）**：`sweep` 在 `BEGIN IMMEDIATE` 内按当前 `head` 重算存活集，并排除活动 pin，
   仍存活/受保护则跳过；只删"不存在 → 不存活 → 未保护"的文件。`collect` 仍为 `sweep(plan(...))` 的组合。
4. **恢复校验（F4）**：`restore` 在复制目录后断言副本 `head == receipt["version"]`；断言回执对象集合
   等于该版本 `manifest` 对象集合；再逐对象验摘要并读取行集。失败抛异常，不以"文件齐全"放行。
5. **失败契约（F5）**：文档写明 `restore` 对无 `complete.json` 一律拒绝；`backup` 负责在 `finally`
   释放保护；残缺目标目录由调用方在保留 `events.json` 后清理；`complete.json` 用临时文件+原子替换。
   保护记录若因进程崩溃残留，由运维按 token 清理（超出本题验证范围）。
6. **不改变语义的部分**：提交继续按摘要复用文件名（`lake.py:76-81`），不因再次引用换名；备份期间不暂停
   提交/回收；不引入远程存储、增量或通用崩溃恢复。

## 6. 发布意见

- 当前实现 **不核准发布**：`complete.json` 不能作为"声明版本可恢复"的证据；正常提交+回收即可使备份
  失败，过期候选回收还会损坏当前版本。
- 发布门槛：(a) `case:commit-after-select`、`case:return-after-select-rows-deceive` 不再出现
  `complete=true` 且版本/行不一致；(b) `case:commit-and-gc-*` 不再失败；(c) `case:stale-candidate-*`
  删除被拒绝且当前版本可读；(d) `case:copy-fail-*` 保持无完成标记并明确残留责任；
  (e) 合法交错用例继续通过。
- 复审证据必须来自 `evidence_gate.py verify` 的重新重放与 `review.json` 比对，不能引用建议方案或
  `samples.json` 之外的推断。

## 7. 证据与运行

- `python3 checks.py verify`：原回归，含正常提交、历史读、静止备份、`fail_at=1` 拒绝恢复。
- `python3 checks.py repro`：原入口，输出 `BACKUP_RESTORE_MISMATCH`，退出 1。
- `python3 evidence_gate.py record`：按 `review_cases.json` 重放并写 `review.json`。
- `python3 evidence_gate.py verify`：重跑回归 + 重放 + 源码定位 + 报告引用 + 保存结果比对。
- `python3 evidence_gate.py demo`：打印真实证据。
- `.pointwise-runtime.json` 的 `verify`/`demo` 已接到上述 evidence_gate 入口；`setup` 保持为空。
