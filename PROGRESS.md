---
type: project
status: active
progress: 98
created: 2026-08-25
updated: 2026-08-28
docs:
  - path: docs/DESIGN.md
    desc: 设计文档（工作流 7 步 + 配置设计 + 安全红线 + 开发计划 10 步）
  - path: SKILL.md
    desc: 技能主体（语义分级 + 首次建忆 + 增量去重）
  - path: README.md / README.en.md
    desc: 中英双语 README
---

# xiaoyaoclaw-memory-distill（记忆蒸馏）

## 目标 / 背景

两件套第二件：**家（initializer）→ 内容（memory-distill）**。以 ClawHub 技能 systiger/memory-distill 为蓝本，结合 xiaoyaoclaw-workspace-initializer 体系改造为记忆整理工具。

- 定位：将对话蒸馏为结构化记忆——核心 → 根目录 MEMORY.md / 日常 → memory/ 日志 / 临时 → 当日；解决上下文溢出
- 核心能力：首次建忆（MEMORY.md 缺失时从历史日志提炼生成）、增量去重写入防膨胀、敏感信息自动跳过、只归档不删除、多 agent 各自独立处理
- 多 agent 策略（指挥官确认）：每个 agent 独立处理自己的 memory/，跨 agent 需明确指令

## 当前状态

已开发完成并发布（98%）：GitHub + ClawHub 双平台公开 + 全局技能同步 + initializer 互链 + 本工作区首次蒸馏实测通过。剩余：无重大遗留，随生态演进维护。

## 进度日志

- 2026-08-25 17:xx：立项（方案确认）——指挥官选定 systiger/memory-distill 为蓝本，命名 xiaoyaoclaw-memory-distill，只与 initializer 互链（两件套）
- 2026-08-25 18:00-18:35：开发完成 + 发布——GitHub dtsola/xiaoyaoclaw-memory-distill（public/main/MIT/7 topics，commit d19f955 → e859413 → 42bbceb hero 美化）；ClawHub v1.0.0 提交；全局技能同步；initializer 互链（40a212d）；本工作区实测 Step 1-5 全通
- 2026-08-25 18:31+：指挥官调整 3 点已落地——autoReset 配置删除、「指挥官」→「用户」用词替换、分级判定改为「判断三问」（下次会话用户还会问起吗？）
- 2026-08-28：ClawHub 已公开（latest 1.0.2，MIT-0）；六件套 README 互链闭环

## 文档索引

| 文档 | 说明 | 更新 |
|------|------|------|
| docs/DESIGN.md | 设计文档（工作流 7 步 + 配置 + 红线 + 开发计划） | 2026-08-25 |
| SKILL.md | 技能主体（分级 + 首次建忆 + 去重 + 安全） | 2026-08-25 |
| README.md / README.en.md | 中英双语 README | 2026-08-25 |
| templates/distill-config.json | 蒸馏配置模板 | 2026-08-25 |
| templates/MEMORY.md | 长期记忆模板 | 2026-08-25 |
| templates/AGENTS-memory-safety.md | 记忆安全规则模板 | 2026-08-25 |

<!--
使用说明（agent 维护，用户可忽略）：
- status: active | paused | archived
- progress: 0-100，时刻维护（每次更新进度日志时同步调整）
- 进度日志只追加不删除
- 重要文档：移入 docs/ 或记录路径，追加到 docs 数组（机器可读）+ 本表格（人可读）
- 项目完结：status 改 archived + 关键结论记入 MEMORY.md（供 memory-distill 蒸馏）
-->

## 2026-09-17 16:0x ClawHub 安全检查 6 条修复 → v1.0.3（待批）

**核查**：`clawhub skill verify xiaoyaoclaw-memory-distill` → fail / suspicious（conf high）；aig **T09 warning**（明文敏感信息可能落盘）+ **T02 note**（首跑模板注入行为指令）；skillspector 4 条（SQP-1 触发过宽 / SQP-2 cron 无人确认就写 / SQP-3×2 语言）

**修复**
- **敏感信息拦截**：SKILL.md 新增专章（必跳过类别表 + 命中整条跳过 + 描述替代原文 + 示例密钥换 `<REDACTED>`）；`distill-config.json` 的 `sensitivePatterns` 6 → **16 条**（ghp_/github_pat_/sk-/xoxb-/AKIA/Bearer/PRIVATE KEY/Set-Cookie/DSN…）
- **模板去行为指令（T02）**：`templates/MEMORY.md` 移除内置「反馈至上」条款；顶部声明**用户资产**、行为规则由用户决定；工作量协议标「可选，默认留空」
- **触发收紧（SQP-1）**：仅明确要求「蒸馏/整理记忆」才触发 + 不触发清单 + **写前闸门**（先说清改哪些文件、条目数；仅探意图时只报告）
- **cron 默认只报告（SQP-2）**：`autoWrite=false` 默认；开启需显式并明示用户；开启后仍须「覆盖前备份 + 只写白名单（根 MEMORY.md / memory/*.md / archive）+ 汇报差异与跳过项 + 当日日志留痕」
- **权限与写范围声明**：frontmatter 补 `allowed-tools`；description 写明只在工作区内写、不写凭据、定时默认只报告
- **语言（SQP-3×2）**：两个模板 + SKILL.md 补「语言可选」
- 包卫生：`.clawhubignore` 排除 PROGRESS/docs → 包内 9 文件；hero 去 4 处注释 + 双语副标题 + **说明文字提亮修对比度（visual_verify 由 EXIT=1 → EXIT=0）**

**验证**（`tmp/md_test.py` 全 PASS）：11 类凭据样本全命中 ✅ ｜ 4 条普通内容不误伤 ✅ ｜ 配置默认值合规 ✅ ｜ 模板口径合规 ✅ ｜ SKILL.md 八项要点齐全 ✅
**产物**：`docs/security-status-2026-09-17.md` + `docs/evidence/verify-v1.0.2-2026-09-17.json`
**待批**：发 v1.0.3

- **2026-09-17 16:22 指挥官批「发」→ 已提交 ClawHub v1.0.3**（回执 pending security scans）；复查任务 cron ba4b07b\（16:52）→ 落地后核对 6 条

- **2026-09-17 16:3x v1.0.3 复扫：aig 清零 ✅，skillspector 9 条 → 已修复待发 v1.0.4**
  - 性质：几乎全是**文档自相矛盾**（cron 口径一处说默认只报告、另一处仍说直接写）→ SDI-4 / SDI-1×2
  - 修复：cron 表加「唯一权威口径」声明 + Step 3 改写对齐；**全仓一致性自查脚本**（6 文件全过）；`distill-config.json` 纳入声明写入范围（新增「写入范围（白名单）」表）；README 中英补写盘警示 + 触发契约 + 不触发示例；语言中立（切换条 / hero 第三行 / config 注释双语 + language 字段）
  - 验证：规则测试 PASS ｜ 一致性自查 6/6 ｜ hero visual_verify EXIT=0 ｜ 包 9 文件

- **2026-09-17 16:48 指挥官批「发」→ 已提交 ClawHub v1.0.4**（回执 pending security scans）；复查任务 cron \c1d4c4d\（17:20）
