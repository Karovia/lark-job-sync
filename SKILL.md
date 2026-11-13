---
name: lark-job-sync
version: 1.1.0
description: "把 3 个对外招聘源表（Web3Job-2025 / 量化对外职位--FinTech社区 / Fintech社区-AI职位）中的岗位同步到内部表《量化内部职位表-2025》。当用户需要：同步岗位/职位、增量更新职位表、检查源表新增职位、岗位去重同步时使用。首次运行会让用户确认目标子表归属（归纳职位归到哪个表）；默认只生成同步预览报告，经用户确认后才真实写入。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# Skill: 岗位同步（3 个源表 → 量化内部职位表）

> **前置条件：** 先阅读 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)（认证）与 [`../lark-sheets/SKILL.md`](../lark-sheets/SKILL.md)（表格操作准则，尤其「读全再写」「真实写回+回读校验」「最小改动」「多步写入合并 +batch-update」）。表格编辑准则全部适用。

## 资源清单（token 已固化，直接使用）

| 角色 | 工作簿 | token | 域名 |
|---|---|---|---|
| 源1 | Web3Job-2025 | `OfrxsVfWShXfrntl606cefeqnBg` | uq1ro7vc0a.feishu.cn |
| 源2 | 量化对外职位--FinTech社区 | `shtcnzAfngrsRRxHlysm20QKA9g` | **c0hd9mw0fp.feishu.cn**（跨租户，勿换域名） |
| 源3 | Fintech社区-AI 职位 | `Q6lYslQIDh0boItCdmKcMpqHnic` | uq1ro7vc0a.feishu.cn |
| **目标** | 量化内部职位表-2025 | `GIDpsafgbhdNA8tMasMcaw5gnGe` | uq1ro7vc0a.feishu.cn |

**目标子表由首次运行的「目标文档确认」决定并持久化**（见 Step 1），历史默认值为 `oQPkx0`（Quant）。该子表结构快照（2026-11-13，行号会随插入位移，执行时必须重新扫描）：

- 列结构：`A 回复快 | B 日期 | C 公司 | D 职位编号 | E 职位名称（分区标签列）| F 地点 | G 薪资 | H JD | I 备注`，J 列及以后为空。
- **多分区纵向堆叠**：凡满足「B 列=『日期』且 C 列=『公司』」的行即分区表头，分区类型由表头行 E 列文本决定。当前有 3 个分区：
  - 分区① 表头 row2，E 列=「股票QR」（数据 row3–135）
  - 分区② 表头 row137，E 列=「期权/期货QR」（数据 row138–147）
  - 分区③ 表头 row149，E 列=「Crypto/固收/ETF/等其他QR」（数据 row150–152）
- 分区①尾部之后（当前 row154 C 列）有 **「以下是暂停职位」标记行：该行及其下方永不写入**。
- 日期格式混用（`10月20日` 与 `2025/5/16`）；编号 L/F/A 系列混用，**存在前导零（如 `L099`），必须按文本保真**；编号最大值约 L1158，但**不要求新编号**——源编号原样带入。
- 各分区内数据按日期倒序排列（最新在分区表头下方第一行）。

**配置文件**：与 SKILL.md 同目录的 `sync-config.json`。存在 → 视为已完成首次确认，直接加载；不存在 → 必须先走 Step 1 的目标文档确认。模板见同目录 `sync-config.example.json`。

## Step 0 认证与结构确认

```bash
lark-cli auth status --json --verify   # 确认 user 身份可用
# 分别对 4 个 token 跑（源2 必须用完整 URL，含 c0hd9mw0fp 域名）：
lark-cli sheets +workbook-info --url "https://uq1ro7vc0a.feishu.cn/sheets/GIDpsafgbhdNA8tMasMcaw5gnGe"
```

任一表 40403/DENY 报权限 → 停止，提示用户换有权限账号或申请授权，**不要**尝试 bot 身份硬闯。

## Step 1 首次运行：目标文档确认（配置存在则跳过，改为校验）

**触发**：SKILL.md 同目录无 `sync-config.json`，或用户说「重新确认目标 / 改目标表 / 换个子表」。

**流程**（归纳职位要归到哪个表，由用户拍板）：

1. `+workbook-info` 读目标工作簿，列出**全部可见子表**（sheet_id、名称、行列数），向用户展示。
2. 快速扫描三个源表：对每个参与同步的源子表，给出「类别（表头 B 列名 / 子表名）+ 岗位数 + 2–3 个样例职位名」。
3. 产出**路由确认表**给用户确认，形如：

   | 源子表（类别） | 岗位数 | 建议目标子表 | 分区 |
   |---|---|---|---|
   | 源1 产品经理 | 3 | Quant (oQPkx0) | 股票QR |
   | 源2 新职位quant (L系列) | 162 | Quant (oQPkx0) | 按关键词分 3 分区 |
   | 源3 AI-技术 | 10 | Quant (oQPkx0)？或 AI相关 (bfMfLb) | — |
   | … |

   默认建议：**全部 → Quant (oQPkx0)**（多分区表内再按 Step 4 关键词分类）。用户可逐项改指其他子表（如 AI 职位 → `bfMfLb AI相关`、职能类 → `6pw5d8 职能`、应届类 → `K5PdW3 应届quant`）。有 AskUserQuestion 类交互工具就用结构化提问，否则文本提问列出选项。
4. 用户确认后，把结果写入 `sync-config.json`（结构见 `sync-config.example.json`），记录 `confirmed_at` 与目标 sheet_id/sheet_name、每条路由（源子表或关键词 → 目标子表 + 分区）。
5. **配置已存在**的后续运行：只做轻校验——`+workbook-info` 确认配置里的 `target.sheet_id` 仍存在且未改名；子表被删/改名 → 停下来重新走本步确认，禁止静默改写到别的子表。

**硬规则**：未经 Step 1 确认，后续任何步骤不得启动；用户一次确认只约束本次配置，改配置需重新确认。

## Step 2 读源：抽取岗位行

对每个源子表执行（一次性并行读，source2 用其完整域名 URL）：

```bash
lark-cli sheets +csv-get --url "<源URL>" --sheet-id "<sheet_id>" --range "A1:<末列><行数上限>"
```

抽取规则：

1. **跳过的子表**：源2 的 `Oz8nE6 表格使用说明`、`rmCcnY 猎头招聘`（纯宣传无数据）。其余子表全部参与。子表清单变化时以 `+workbook-info` 实时结果为准，按「表头行定位规则」自适应，不硬编码。
2. **表头行定位**：源1/源2 前 1–2 行是宣传行（含"求职+微信"“Linkedin”等），源3 第 1 行是宣传行；**表头行 = A 列值为「职位编号」或「编号」的行**。源2 的 `Ops619 香港` 表头在第 1 行。
3. **按表头名映射列**（列位置在各子表不固定，禁止按列号硬编码）：
   | 目标字段 | 源表头（任一匹配） |
   |---|---|
   | 职位编号 | `职位编号`、`编号` |
   | 职位名称 | `职位`、`岗位名称`；源1 无此列头时取第 2 列（列头是类别名，如"产品经理"） |
   | 地点 | `地点` |
   | 薪资 | `薪资`、`薪酬`、`薪资范围` |
   | JD | `JD`、`jd` |
   | 备注 | `备注`、`备注（人选看）`、`备注（内部看）` |
   | 附加要求 | `必备要求（学历/经验）` → 追加到 JD 末尾（`\n【必备要求】…`），不单列 |
   | 公司 | 仅源2 `Ops619` 有 `公司介绍` 列：若其文本可明确识别公司名则填入，否则留空 |
4. **跳过的行**：空行；分组标签行（仅 A 列有值且其余全空，如源2 `WRR9Xe` 的「初级-中级」）；编号与职位名称都为空的行。编号空但职位名称非空的行（如源2 `EDENt2` 首行数据）是有效岗位，照常纳入，编号留空。
5. 用 `annotated_csv` 的 `[row=N]` 前缀定位，先确认真实末行（`current_region`），禁止只探前几行。**解析陷阱**：`[row=N] ` 前缀是 csv.reader 解出记录的**第一个单元格的一部分**（如 `'[row=4] F116'`），逐格处理前必须先 `re.sub(r'^\[row=\d+\]\s?','',cell0)` 剥离，否则表头定位（`职位编号`）会全部失配——已踩过坑。
6. `+csv-get` 的 `--range` 是**必填 flag**，缺失直接报 `required flag(s) "range" not set`；范围按 `+workbook-info` 返回的行列数上限给（宁大勿小）。
7. 源内去重：同编号+同职位名称的重复行只保留一条；**同编号但职位名称不同 = 两个岗位，都保留**（源2 `Ops619` 存在此情况）。

## Step 3 读目标：建立去重基线

按配置的 `target.sheet_id` 读取：

```bash
lark-cli sheets +csv-get --url "https://uq1ro7vc0a.feishu.cn/sheets/GIDpsafgbhdNA8tMasMcaw5gnGe" --sheet-id "<配置的sheet_id>" --range "A1:I<行数上限>"
```

- 覆盖到真实末行；识别所有分区表头行、各分区已有编号集合（D 列，strip 后比较，保留原文本）与「以下是暂停职位」标记行。
- 记录每个分区表头行号（后续插入点）。
- 若目标子表结构与「资源清单」快照差异巨大（无分区表头、列结构不同），停下来向用户说明并确认后再继续，禁止套用旧结构硬写。

## Step 4 分类（决定落入目标子表的哪个分区）

先查配置 `routes`：命中源子表专属路由的，直接采用其 `target_sheet_id` + 分区；未命中的按关键词规则（关键词对职位名称+JD 前 200 字匹配，优先级从上到下）：

| 目标分区 | 关键词（不区分大小写） |
|---|---|
| ② 期权/期货QR | 期权、期货、CTA、衍生品、可转债、FICC、商品、股指 |
| ③ Crypto/固收/ETF/等其他QR | crypto、web3、web 3、固收、债券、利率、ETF、数字货币、稳定币、交易所 |
| ① 股票QR（默认兜底） | 其余全部（含股票、量化研究、AI、开发、产品、职能等） |

配置把某类岗位路由到了**非 Quant 的其他子表**时，该子表若也是分区结构，同样先扫描分区表头再落位；若是单表结构，找到其表头行后在表头下方按日期倒序插入。

## Step 5 去重与变更检测（只报告，不覆盖）

**去重范围是配置的目标子表**：

- **重复**：编号非空且已在目标 D 列集合中 → 跳过；编号为空时用「公司+职位名称」归一化（去空格/大小写）匹配，命中 → 跳过。
- **变更候选**：编号+职位名称命中，但地点/薪资/JD 与目标现有值不同 → 计入差异报告，**默认不改写目标行**；用户明确要求更新时，只更新有差异的具体单元格（用 `+cells-set` 精确到格），JD/编号列禁止整块覆盖。
- 若配置把岗位路由到了多个不同目标子表，逐子表分别建基线、分别去重。

## Step 6 预览报告（写入闸门，必做）

生成同步预览（对话内文本即可）并**等待用户确认**，包含：

```
路由：源子表 → 目标子表（按 Step 1 配置）
新增 N 条：Quant 分区① a 条（编号：…）、分区② b 条、分区③ c 条；<其他目标子表> x 条
跳过重复 M 条（编号列表）
变更候选 K 条（编号 + 差异字段摘要）
插入位置：各目标子表分区表头行号
```

用户未确认前，**禁止任何写调用**。用户要求先看明细时，逐条列出（编号/公司/职位名称/地点/薪资）。

## Step 7 写入（仅在用户确认后）

对每个有新增的目标子表+分区组合，**从最靠下的分区往最靠上的倒序执行插入**（避免先插的分区使后面分区表头行号位移；跨子表互不影响顺序）。可先给每个写调用加 `--dry-run` 预览请求。

1. **插行 + 写值合并为一次原子 `+batch-update`**（operations 元素结构为 `{shortcut, input}`，`input` 是该 shortcut 的 flag 展平 JSON，sheet 定位在 input 里给 `sheet_id`，url 只在顶层给一次）：

```bash
lark-cli sheets +batch-update --url "https://uq1ro7vc0a.feishu.cn/sheets/GIDpsafgbhdNA8tMasMcaw5gnGe" --operations - <<'EOF'
[
  {"shortcut":"+dim-insert","input":{"sheet_id":"<目标sheet_id>","position":<表头行号+1>,"count":N,"inherit_style":"after"}},
  {"shortcut":"+csv-put","input":{"sheet_id":"<目标sheet_id>","start_cell":"A<表头行号+1>","csv":"<新增块CSV>"}}
]
EOF
```

   - `+dim-insert --position <表头行号+1>` 表示**插在该行之前**，即恰好落在分区表头与首条数据之间；`inherit_style:"after"` 使新行样式继承**下方数据行**（不要用 `before`，会继承到表头样式）。
   - `dim-insert` 会继承值/公式/边框但**不继承行高**：批内追加一条 `{"shortcut":"+rows-resize","input":{"sheet_id":"<目标sheet_id>","rows":[{"start_index":…,"end_index":…,"row_height":<相邻数据行高>}]}}`（行高先经 `+sheet-info --include row_heights` 读取）。

2. **写值方式**：新增块用 `+csv-put --start-cell <新块左上角>` 整块平铺（含换行/特殊字符的 CSV 走 stdin），或用 `+cells-set`（复合 JSON 走 stdin）逐格精确写。硬性要求：
   - **编号列、日期列按文本写入**，严禁被数值化（`L099`、`12.10` 类文本一数值化就永久失真）；JD 含换行必须走 stdin，不走命令行内联。
   - `A 回复快` 留空；`B 日期` = 执行当天，格式 `M月D日`（如 `11月13日`）；`C 公司` 源里没有就留空（禁止从 JD 猜测填入，Ops619 公司介绍列除外）；`E 职位名称`、`F 地点`、`G 薪资`、`H JD`、`I 备注` 按 Step 2 映射填充。
   - 只写新增块矩形区域，**不触碰**其它任何单元格；整批写入合并为单次 `+batch-update`（原子性）。

## Step 8 回读校验（必做，不通过不算完成）

```bash
lark-cli sheets +csv-get --url "…/sheets/GIDpsafgbhdNA8tMasMcaw5gnGe" --sheet-id "<目标sheet_id>" --range "A1:I<新末行>"
```

逐条 assert：

1. 各目标子表各分区新增块行数 = 预告的 a/b/c…，且首行内容与预告一致；
2. 编号前导零完好（抽查 `L099` 类旧值未被改动 + 新增编号原文一致）；
3. 全部分区表头行仍在（行号整体位移 = 该分区上方累计插入数），分区顺序与 E 列标签不变；
4. 「以下是暂停职位」标记行内容未变，且其下方无任何新写入；
5. 抽查 1 条新增行的 JD 换行完整（未被截断/压平）。

全部通过后向用户汇报：新增/跳过/变更候选计数 + 各目标子表插入位置。可用 `+changeset-get --start-revision <写入前版本>` 复核改动范围。

## 权限

| 操作 | scope |
|---|---|
| 读 4 个工作簿 | `sheets:spreadsheet:read`、`sheets:spreadsheet.meta:read` |
| 写目标表 | `sheets:spreadsheet:write_only`、`sheets:spreadsheet.meta:write_only` |

登录：`lark-cli auth login --scope "sheets:spreadsheet:read sheets:spreadsheet:write_only sheets:spreadsheet.meta:read sheets:spreadsheet.meta:write_only" --no-wait --json`（split-flow，见 lark-shared）。

## 红线（违反任意一条即终止并报告）

1. 三个源表**只读**，永不写入。
2. 目标工作簿内，「以下是暂停职位」标记行及以下永不写入；不删除/改名/隐藏/移动任何子表。
3. 不整块覆盖目标已有行；变更检测只出报告，更新须用户逐条确认且只动差异格。
4. **Step 1 目标文档确认是总开关**：无 `sync-config.json` 不得进入同步流程；确认结果落盘前不得发起任何写调用。
5. 未经用户确认不发起任何写调用（Step 6 是硬闸门）；lark-cli exit 10 确认协议照 lark-shared 处理，禁止默认加 `--yes`。
6. 编号/日期文本保真；JD 禁止压平换行。
