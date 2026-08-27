# AI Security Radar Data — Agent 操作规范

本仓库是 AI Security Radar 的数据同步仓库，通过 GitHub 连接外网（抓取+LLM加工）和内网（展示+Bot推送）。
代码仓库在 `../ai-security-radar`，数据仓库在这里。

## 仓库结构

```
ai-security-radar-data/
├── manifest.json          ← 索引文件，记录所有可用日报/周报/快照
├── daily/
│   └── {YYYY-MM-DD}.json   ← 每日日报完整 JSON（含 markdown 字段）
├── weekly/
│   └── {YYYY-Www}.json      ← 每周周报 JSON（含 markdown 字段）
├── snapshot/
│   └── latest.json         ← 最新条目快照（覆盖写入，10-15 条）
└── assets/
    └── {type}_{date}.png   ← 日报/周报 hero 图片
```

## Git 配置

- **Remote**: `git@ssh.github.com:zk1te/ai-security-radar-data.git`
- **认证方式**: Deploy Key（`~/.ssh/radar_data_deploy`，ed25519，不过期）
- **SSH**: 直接连接 `ssh.github.com:443`（22 端口被封，走 443），密钥 `~/.ssh/radar_data_deploy`
- **core.sshCommand**: `ssh -i ~/.ssh/radar_data_deploy -p 443 -o IdentitiesOnly=yes`（已写入 git config local，push 时自动生效）
- **Git 用户**: `AI Security Radar <radar-data@local>`（仅 commit 署名，不影响认证）

## 标准推送流程（外网侧 Agent 使用）

每次 worker 完成一轮抓取+LLM加工后，执行以下步骤：

```bash
cd D:\NewsInsight\ai-security-radar-data   # 或服务器上的对应路径

# 1. 写入数据文件（由导出脚本生成）
#    - daily/{date}.json（如有新日报）
#    - snapshot/latest.json（每次都覆盖更新）
#    - assets/{type}_{date}.png（如有新图片）
#    - weekly/{week}.json（周一才有）
#    - manifest.json（每次更新索引）

# 2. 检查是否有变更
git status --short

# 3. 如果没有变更，跳过推送（避免空提交）
#    如果 git status --short 输出为空，说明没有新数据，直接退出

# 4. 暂存所有变更
git add -A

# 5. 提交（commit message 格式：类型 + 日期/说明）
git commit -m "data: daily 2026-08-24 + snapshot update"

# 6. 推送
git push origin main
```

### Commit Message 规范

格式：`data: {类型} {日期/说明}`

| 场景 | 示例 |
|------|------|
| 日报+快照 | `data: daily 2026-08-24 + snapshot update` |
| 仅快照 | `data: snapshot update 2026-08-25 10:00` |
| 周报 | `data: weekly 2026-W34` |
| 初始结构 | `init: data-export repository structure` |

### 推送频率

- **快照**：每轮 worker 循环结束后（约 5-10 分钟间隔），仅当有新条目被加工时推送
- **日报**：日报生成后（北京时间 00:00-00:35）推送一次
- **周报**：周一凌晨生成后推送一次
- **空数据不推送**：如果 `git status --short` 为空，跳过整个 push 流程

## 标准拉取流程（内网侧 Agent 使用）

内网通过定时 `git pull` 同步数据，然后从本地 JSON 读取：

```bash
cd /path/to/ai-security-radar-data

# 1. 拉取最新数据（内网只读，不会有冲突）
git pull origin main

# 2. 读取数据
#    - 先读 manifest.json 了解有哪些可用数据
#    - 按需读取 daily/{date}.json 或 snapshot/latest.json
```

### 拉取频率

- **定时拉取**：每 5 分钟一次（cron 或定时任务），失败静默重试
- **私聊触发**：用户私聊 Bot 时不触发 git pull，直接读本地 JSON（数据延迟 ≤5 分钟可接受）

## 数据文件格式

### manifest.json

```json
{
  "updated_at": "2026-08-25T10:00:00+08:00",
  "daily_reports": ["2026-08-24", "2026-08-25"],
  "weekly_reports": ["2026-W34"],
  "snapshot": {
    "latest": "2026-08-25T10:00:00+08:00",
    "item_count": 12
  }
}
```

### daily/{date}.json

复用代码仓库 `DailyReport.json_content` 结构，补充 `markdown` 字段。包含：cover_title、lead_takeaway、digest_points、action_items、top_items、also_worth_attention、groups、papers、metrics、today_observation、markdown。

### snapshot/latest.json

最近 48 小时内 `include_status != excluded` 且 `score >= 60` 的条目，按分数降序取前 10-15 条。每次覆盖写入。

### weekly/{week}.json

聚合 7 天日报的 top_items，去重排序取前 10 条，含 LLM 生成的简短趋势概述。

## 重要规则

1. **外网是唯一写入方**，内网只读 pull，不会有 git 冲突。
2. **不要在数据仓库里放代码**，只放 JSON 和图片。代码在 `../ai-security-radar`。
3. **不要 force push**，数据追加为主，覆盖只针对 `snapshot/latest.json` 和 `manifest.json`。
4. **图片用 PNG 格式**（SVG 兼容性差），放在 `assets/` 目录。
5. **JSON 文件用 UTF-8 无 BOM**，避免解析问题。
6. **路径用正斜杠**（`daily/2026-08-24.json`），跨平台兼容。
7. **生产环境部署时**，路径和 SSH config 需在服务器上重新配置，参考代码仓库的 `DECISION_LOG.md` 第十章。

## 排障

| 问题 | 解决方案 |
|------|---------|
| `Host key verification failed` | `ssh -o StrictHostKeyChecking=accept-new -T git@ssh.github.com` |
| `Connection closed by x.x.x.x port 22` | 确认 SSH config 用 `ssh.github.com:443` 而非 `github.com:22` |
| `Permission denied (publickey)` | 确认 `~/.ssh/radar_data_deploy` 存在且 SSH config 中 `IdentityFile` 指向它 |
| `fatal: detected dubious ownership` | `git config --global --add safe.directory <仓库路径>` |
| push 成功但 PowerShell 报错 | git 的进度信息走 stderr，PowerShell 会误报，实际已成功，用 `git log` 验证 |