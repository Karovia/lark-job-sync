# lark-job-sync 使用说明

> 仓库：https://github.com/Karovia/lark-job-sync ｜ zip 直链：https://raw.githubusercontent.com/Karovia/lark-job-sync/main/lark-job-sync.zip

把 3 个对外招聘源表的岗位，同步到内部飞书表格《量化内部职位表-2025》。

- 源1 **Web3Job-2025**（编号 F 系列）
- 源2 **量化对外职位--FinTech社区**（编号 L 系列，跨租户）
- 源3 **Fintech社区-AI 职位**（编号 A 系列）
- 目标 **量化内部职位表-2025**（首次运行时由你确认岗位归到哪个子表）

## 它做什么

1. 读取 3 个源表的全部岗位（自动跳过宣传行、说明页、分组标签行）
2. 按编号去重（源编号原样保留，含前导零文本保真），识别新增 / 重复 / 已有岗位的变更
3. 先生成**同步预览报告**给你确认，确认后才写入目标表
4. 写入后回读校验（行数、编号保真、分区结构、JD 换行完整性）

**安全设计：源表只读；写入前必须两道确认（首次目标确认 + 每次同步预览确认）；目标表「暂停职位」以下永不写入。**

## 环境要求

- [lark-cli](https://github.com/larksuite/lark-cli)（飞书命令行）已安装
- 飞书应用已开通表格权限，且你登录的账号对 4 个表都有访问权限

```bash
# 检查登录态（需要 user 身份，含 sheets 读+写 scope）
lark-cli auth status --json --verify

# 缺权限时按需授权（split-flow，扫码完成）
lark-cli auth login --scope "sheets:spreadsheet:read sheets:spreadsheet:write_only sheets:spreadsheet.meta:read sheets:spreadsheet.meta:write_only" --no-wait --json
```

## 安装

**方式 A：zip 解压（推荐）**

```bash
curl -L -o /tmp/lark-job-sync.zip "https://raw.githubusercontent.com/Karovia/lark-job-sync/main/lark-job-sync.zip"
unzip -o /tmp/lark-job-sync.zip -d ~/.zcode/skills/
# 校验：ls ~/.zcode/skills/lark-job-sync/SKILL.md 存在即可
```

其他 agent 目录同理：把 `lark-job-sync/` 整个文件夹放进你的 skills 目录（如 `~/.agents/skills/`、`~/.claude/skills/`）。

**方式 B：手动复制** —— 把 `SKILL.md`、`README.md`、`sync-config.example.json` 三个文件放到 skills 目录下的 `lark-job-sync/` 文件夹。

## 首次运行：目标文档确认（重要）

第一次运行时，Skill 会：

1. 列出目标工作簿的全部子表（Quant / PM / AI相关 / 职能 / 应届quant / 新职位-技术……）
2. 扫描 3 个源表，给出「源子表（类别）→ 建议目标子表」的路由确认表
3. **由你确认**每类岗位归到哪个子表（默认建议全部进 Quant 子表，表内再按期权期货 / Crypto固收ETF / 股票QR 三个分区自动分类）
4. 确认结果保存到 `sync-config.json`，之后每次运行直接使用，不再重复询问

想改归属时，对 agent 说「**重新确认目标**」即可重跑这个流程。

## 日常使用

对任意接入了本 Skill 的 agent 说：

- 「**同步岗位**」/「**跑一下岗位同步**」→ 全流程：读源 → 去重 → 预览报告 → （你确认）→ 写入 → 回读校验
- 「**检查源表有没有新职位**」→ 只读模式：只输出新增/变更候选报告，不写任何表
- 「**重新确认目标**」→ 重跑首次确认流程
- 「先看明细再决定」→ 预览报告会逐条列出编号/公司/职位/地点/薪资

## 配置文件

`sync-config.json`（首次确认后自动生成，模板见 `sync-config.example.json`）：

| 字段 | 说明 |
|---|---|
| `target.sheet_id` | 默认写入的目标子表 |
| `routes[]` | 路由规则：按源子表或关键词，把岗位指到不同目标子表/分区 |
| `fallback` | 未命中任何规则时的兜底落位 |
| `confirmed_at` | 确认时间（下次运行会校验目标子表仍存在） |

## 常见问题

- **报权限错误（40403/DENY）**：当前账号没有某个表的访问权，换有权限的账号重新 `auth login`，或在飞书里申请该文档权限。
- **源2 读不到**：它在另一个租户（`c0hd9mw0fp.feishu.cn`），必须用完整原始 URL 访问，Skill 已内置。
- **写入被拒绝（exit 10）**：lark-cli 的高风险确认协议，agent 会向你展示操作详情，你同意后才会加 `--yes` 重试。
- **想换目标表**：说「重新确认目标」，或直接删除 `sync-config.json` 后重跑。
