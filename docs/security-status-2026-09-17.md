# ClawHub 安全检查核查与修复（2026-09-17）

> 执行人：天桐｜指令：指挥官「处理 OpenClaw Memory Distill」
> 命令：`clawhub skill verify xiaoyaoclaw-memory-distill`（对象 v1.0.2）

---

## 1. 结论

`ok:false` / `decision:fail` / 原因码 `security.status_not_clean`；`security.status = suspicious`（confidence high）。
**共 6 条**：aig **2**（T09 warning 明文敏感信息 + T02 note 首跑模板注入行为指令）+ skillspector **4**（SQP-1 / SQP-2 / SQP-3 ×2，均 MEDIUM）。

LLM 判词：*"not malicious, but it deserves review because it can automatically persist private conversation details into workspace memory files."*

> 特点：这个技能**天生就是写持久记忆的**，所以命中点集中在「写什么 / 谁能批准写 / 会不会写进不该写的」。

---

## 2. 命中与修复对照

| 命中 | 位置 | 问题 | 修复 |
|---|---|---|---|
| **aig T09 warning** | `SKILL.md:92-110`、`templates/distill-config.json` | 可能把**临时凭据类敏感信息以明文写进记忆日志** | 新增 **「敏感信息拦截（落盘前必做的扫描）」**章节：给出**必跳过类别表**（访问令牌/API key、口令与私钥文件、会话与 cookie、连接串、支付与证件、第三方隐私）+ 明确做法（命中**整条跳过**、用描述替代原文、示例里的密钥一律替换 `<REDACTED>`、配置里也不得放真实密钥、拿不准就跳过并回报）；`distill-config.json` 的 `sensitivePatterns` 从 6 条扩到 **16 条**（含 `ghp_`/`github_pat_`/`sk-`/`xoxb-`/`AKIA`/`Bearer`/`BEGIN PRIVATE KEY`/`Set-Cookie`/连接串 DSN 等） |
| **aig T02 note** | `templates/MEMORY.md:7-13` | 首跑模板里**内置了一条行为指令**（"反馈至上"），首次建忆时会变成影响未来会话的持久条款（记忆投毒面） | 模板**移除内置行为指令**；顶部明确 **「本文件是用户资产：技能只提议，行为规则由用户决定，不会替你塞持久指令」**；工作协议章节标注「可选，默认留空」；原位置改成一个 HTML 注释，说明"如需写规则请用户自行添加" |
| **SQP-1** MEDIUM | `SKILL.md:29` | 触发词过宽（「整理对话/整理记忆/压缩上下文」随口说也可能命中，而本技能会落盘） | 触发条件改为「明确要求蒸馏/整理**记忆**」；补**不触发**清单（随口「整理一下」、问机制、只是讨论要不要整理）；新增**写前闸门**（先说清将更新哪些文件 + 预计条目数，敏感项跳过并计数；仅探明意图时**只报告不落盘**） |
| **SQP-2** MEDIUM | `SKILL.md:47` | Cron 模式写明「直接写入 + 汇报差异」——**无人确认就改持久记忆文件** | 改为**默认只报告**：`autoWrite: false`（默认）时定时任务只产出「建议写入条目 + 目标文件」；`autoWrite: true` 需**显式开启并向用户明示**；开启后仍受约束：写前先备份（`MEMORY.md` → `memory/archive/MEMORY-YYYYMMDD.md`）、**只写白名单路径**（根 `MEMORY.md` + `memory/*.md` + 归档）、写完汇报差异与跳过项；两种模式都在当日日志留一行「本次蒸馏改了什么」便于回看/回滚 |
| **SQP-3** MEDIUM ×2 | `templates/AGENTS-memory-safety.md:1`、`templates/MEMORY.md:3` | 模板纯中文、未声明语言可选 | 两个模板 + SKILL.md 均补 **「语言可选」** 说明（记忆记录语言跟随用户） |
| 附带（同类口径，预防下轮） | `SKILL.md` | 权限与写入范围未声明 | frontmatter 补 **`allowed-tools`**（Read/Write/Edit/Glob/Grep/Bash）；description 明确**写入范围**（仅工作区内：根 `MEMORY.md` + `memory/YYYY-MM-DD.md` + 归档副本）与「不写密钥/凭据/第三方隐私」「定时默认只报告」 |

**包内容卫生**：新增 `.clawhubignore` 排除 `PROGRESS.md` / `docs/` → 包内 **9 个文件**。

**顺手修的资产问题**：`hero.svg` 移除 4 处注释、副标题改双语；并把低对比度的说明文字 `#6e7681` 提亮为 `#8b949e`（对 `#161B22` 约 5.3:1）—— 用本技能自己的 visual_verify 复检由 **EXIT=1 转 EXIT=0**。

---

## 3. 验证（全部 PASS）

`tmp/md_test.py`（含真跑正则）：

1. **敏感正则有效性**：11 个真实形态凭据样本（GitHub PAT、sk- key、Slack token、AWS AKIA、Bearer、password=、postgres DSN、私钥头、Set-Cookie、session_id）**全部命中**
2. **不误伤**：4 条普通记忆内容（发布节奏决策、项目状态、API 设计讨论、条数统计）**均不误报为凭据**
3. **配置默认值**：`autoWrite=false` ✅、写入白名单存在 ✅、覆盖前归档开启 ✅、`autoClean` 仍为 false ✅
4. **模板口径**：`MEMORY.md` 不再含内置行为指令 ✅、声明为用户资产 ✅、语言可选 ✅；`AGENTS-memory-safety.md` 语言可选 + 建议文本 ✅
5. **SKILL.md 结构**：`allowed-tools` ✅、敏感信息拦截章节（含必跳过类别）✅、触发/不触发示例 ✅、写前闸门 ✅、cron 默认只报告 + autoWrite ✅、白名单/备份 ✅、语言可选 ✅、只归档不删除 ✅

附加：`hero.svg` 渲染 46 KB PNG 且对比度检查 **EXIT=0**；发布包预览 **9 个文件**。

---

## 4. 待办

- [ ] 发 **v1.0.3** → 等扫描 → 复扫核对 6 条
- [ ] GitHub 推送（代理 22307 未监听 + 直连超时）

## 5. 原始证据

- `docs/evidence/verify-v1.0.2-2026-09-17.json`
