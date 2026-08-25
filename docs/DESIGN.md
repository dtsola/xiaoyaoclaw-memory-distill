# xiaoyaoclaw-memory-distill 设计方案

> 状态：待指挥官确认 · 日期：2026-08-25
> 定位：OpenClaw 记忆整理工具（记忆蒸馏器），与 xiaoyaoclaw-workspace-initializer 组成「家 + 内容」两件套。

---

## 1. 项目定位

**一句话：** 将对话上下文蒸馏为结构化记忆文件，解决会话上下文溢出 + 记忆碎片化问题。

**与 initializer 的分工：**

| 项目 | 职责 | 类比 |
|---|---|---|
| xiaoyaoclaw-workspace-initializer | 记忆系统的「家」：目录结构 + WORKSPACE.md 规范 + 配置安全 | 房子 |
| xiaoyaoclaw-memory-distill | 记忆系统的「内容」：蒸馏 + 分类 + 增量沉淀 + 归档 | 管家 |

**继承的体系铁律：**
- 路径一律按 WORKSPACE.md 走（技能路径冲突仲裁原则）
- 配置修改只用 config.patch，禁 config.apply（多 agent 安全）
- 反馈至上：每次蒸馏必须向用户汇报结果

---

## 2. 文件结构

```
xiaoyaoclaw-memory-distill/
├── SKILL.md                    # 技能主体（工作流 Step 1-7）
├── templates/
│   ├── distill-config.json     # 蒸馏配置文件模板（安装时复制到工作区根目录）
│   ├── MEMORY.md               # 长期记忆结构模板（首次建忆的结构底子；无历史日志时兜底建空骨架）
│   └── AGENTS-memory-safety.md # 记忆安全规范（写入 AGENTS.md 用）
├── README.md
├── LICENSE                     # MIT
└── docs/
    └── DESIGN.md               # 本方案文档
```

**记忆文件位置（与日志分离，对齐既有体系）：**

```
工作区根目录/
├── MEMORY.md               ← 长期记忆（核心级信息），与 AGENTS.md/SOUL.md 平级
├── distill-config.json     ← 蒸馏配置（per-agent 独立）
├── memory/
│   ├── YYYY-MM-DD.md       ← 今日/过往日志（日常级 + 临时级）
│   └── archive/            ← 过期归档（可选，按 retentionDays）
```

> ⚠️ MEMORY.md **不在** `memory/` 目录下——memory/ 只放日志。这是既有 AGENTS.md 记忆体系的约定（MEMORY.md = 长期记忆，memory/ = 日常日志），本技能遵守不改变。

---

## 3. SKILL.md 工作流（Step 1-7）

### Step 1: 检测记忆状态
- 检查 `memory/` 目录是否存在 → 缺失则**自动创建**（本技能只创建记忆所需目录，不创建整套工作区规范；需要完整规范时引导 initializer 初始化）
- 检查**工作区根目录** `MEMORY.md` 是否存在：
  - **缺失 → 执行「首次建忆」**（本技能核心价值之一）：扫描 `memory/` 全部历史日志 → 语义提炼核心级信息（身份/协议/项目状态/环境备忘/时间线）→ 按 `templates/MEMORY.md` 结构生成 MEMORY.md 初版 → 报告指挥官确认
  - 若 `memory/` 也没有任何历史日志 → 才用模板建空骨架（标注「待填充」）
- 检查**工作区根目录** `distill-config.json` → 缺失则用模板复制默认配置
- 检查今日 `memory/YYYY-MM-DD.md` 是否存在

### Step 2: 扫描会话历史，语义分级
信息分三级（判断标准：**会话重启后是否还需要**，语义理解，非关键词匹配）：

| 级别 | 判定 | 落盘位置 | 示例 |
|---|---|---|---|
| 核心级 | 重启后仍必须知道 | MEMORY.md | 决策、项目状态、核心知识点、长期偏好、关键联系人 |
| 日常级 | 近期有效/需追踪 | memory/YYYY-MM-DD.md | 任务、待办、当日事件、进展 |
| 临时级 | 一次性 | 只留当日，过期提示归档 | 验证码、临时链接、一次性数据 |

### Step 3: 生成蒸馏报告 + 安全检查
- 结构化输出：决策 x 条 / 任务 x 条 / 知识点 x 条 / 临时 x 条
- **敏感信息检测**：匹配 `sensitivePatterns`（ghp_、sk-、api_key、password、token 等模式）→ 标记敏感，**默认跳过不写入**，报告里提示「检测到敏感信息 x 条，已跳过」
- 写入策略分两种模式：
  - **手动触发**：先出报告 → 指挥官确认 → 写入
  - **Cron 触发**：直接写入 + 完成后汇报差异（delivery announce），敏感信息一律跳过

### Step 4: 增量写入 + 去重
- 写 MEMORY.md 前先读已有内容 → **查重**：只追加新条目，重复条目合并（防 30 次蒸馏 = 30 遍重复）
- 按主题组织写入 MEMORY.md（项目/决策/教训/偏好/联系人），不按日期堆砌
- 追加今日蒸馏记录到 `memory/YYYY-MM-DD.md`（带时间戳 + 来源会话）
- 更新根目录 `distill-config.json` 的 `lastDistill` 时间戳

### Step 5: 生成完成报告
```
📊 记忆蒸馏完成
✅ 提取：决策 x | 任务 x | 知识点 x | 临时 x
📝 写入：MEMORY.md +x 条（去重合并 y 条）| daily +x 条
⚠️ 敏感信息 x 条已跳过
💡 上下文若已满，可 /reset（不会影响记忆文件）
```

### Step 6: 定期提炼（Memory Maintenance，补齐 AGENTS.md 缺失环节）
- 触发：每 N 次蒸馏（配置 `maintainEvery`）或手动「提炼记忆」
- 扫描近 7 天 daily 文件 → 识别值得长期保留的内容 → 增量合并进 MEMORY.md
- daily 文件保持 raw log 性质，提炼后不删除（历史留痕）

### Step 7: 过期归档（默认关闭）
- `retentionDays` 到期 → **提示归档**：移动到 `memory/archive/YYYY/`（或按 WORKSPACE.md 约定）
- **只归档不删除**；删除必须指挥官明确确认
- 默认 `autoClean: false`

---

## 4. 配置设计（templates/distill-config.json）

**配置文件是什么：** `distill-config.json` 是蒸馏工具的**运行参数文件**，安装时复制到**工作区根目录**（与 MEMORY.md 平级，每个 agent 工作区一份，互不干扰）。它让用户不用改 SKILL.md 就能调整蒸馏行为。

**每个字段干什么用：**

| 字段 | 默认值 | 用途 |
|---|---|---|
| `retentionDays` | 90 | 日志保留天数，超过提示归档 |
| `autoClean` | false | 是否自动清理（**永远建议 false**，记忆是永久资产） |
| `maintainEvery` | 7 | 每 N 次蒸馏触发一次「定期提炼」（Step 6） |
| `sensitivePatterns` | ghp_/sk-/password 等 | 敏感信息检测规则，命中即跳过不落盘，可扩展 |
| `schedule` | 0 22 * * * | 蒸馏时间参考记录（实际 cron 在 OpenClaw 配置里，这里仅存档） |
| `lastDistill` | null | 上次蒸馏时间戳（状态记录，cron 触发时防同一天重复蒸馏） |

> 注：原设计中的 `autoReset` 字段已移除（2026-08-25 指挥官确认：用不到，直接删）。蒸馏不触发 /reset 是行为承诺，不设配置项。

```json
{
  "retentionDays": 90,
  "autoClean": false,
  "maintainEvery": 7,
  "categories": ["core", "daily", "temporary"],
  "sensitivePatterns": ["ghp_[A-Za-z0-9]", "sk-[A-Za-z0-9]", "api[_-]?key", "password", "secret", "token"],
  "schedule": "0 22 * * *",
  "lastDistill": null
}
```

**设计原则：**
- 蒸馏不触发 /reset——重置由用户自行决定，只在报告里提示
- `autoClean` 默认 false——清理需人工确认
- 配置是**运行时参数**，不是日志；模板在技能 templates/ 下，实际配置在工作区根目录

---

## 5. 安全红线（写入 AGENTS-memory-safety.md，追加到 AGENTS.md）

1. 蒸馏**不删除**任何记忆文件，只提取和整理
2. 敏感信息（token/密码/密钥）默认跳过，报告提示，不落盘
3. 归档 ≠ 删除；删除必须指挥官确认
4. 不自动 /reset；不自动清理
5. 不改 openclaw.json；记忆路径按 WORKSPACE.md 走
6. MEMORY.md 含个人上下文 → 只在主会话加载（继承 AGENTS.md 既有规则）

---

## 6. 触发方式

| 方式 | 说明 |
|---|---|
| 手动 | 触发词：「蒸馏记忆」「整理对话」「压缩上下文」「整理记忆」 |
| Cron（推荐） | OpenClaw cron 配置示例：每天 22:00，`systemEvent` 文本**自包含上下文**（含触发指令 + 报告要求），delivery announce |
| HEARTBEAT | ⚠️ 注明：默认心跳关闭时不生效；需先启用心跳，并在 HEARTBEAT.md 加检查项（按对话量动态触发） |

---

## 7. README 结构（对齐 initializer 风格）

hero + 中英引言 + 徽章（license/MIT）→ 为什么需要它（上下文溢出、记忆碎片化、重复膨胀）→ 特性 → 安装（ClawHub + GitHub）→ 使用（三步）→ 🚀 快速上手（三步图文）→ 与其他方案的区别（**vs systiger/memory-distill 对比表**）→ 目录结构 → License → 🛠️ 定制广告 → 小遥Claw → 作者 → 交流群二维码

---

## 7.5 用户快速上手设计（README「🚀 快速上手」蓝本）

**三步，5 分钟，站在用户角度：**

### Step 1: 安装
```bash
clawhub install xiaoyaoclaw-memory-distill
```
或手动：把 `SKILL.md` + `templates/` 放进 skills 目录。

### Step 2: 一句话触发首次蒸馏
对 agent 说：

> 蒸馏记忆

agent 自动完成：检测 `memory/` 与 `MEMORY.md`（**缺失则扫描历史日志「首次建忆」**，不用空骨架）→ 扫描本次会话 → 语义分级 → 出蒸馏报告（含敏感信息提示）→ 确认后写入 → 汇报结果。

### Step 3: 验收 + 开启自动蒸馏（可选）
- 打开工作区验收：根目录 `MEMORY.md`（核心记忆）+ `distill-config.json`（蒸馏配置），`memory/` 下 `YYYY-MM-DD.md`（今日日志）
- 想要每天自动蒸馏，对 agent 说：

> 配置每天 22:00 自动执行记忆蒸馏

- 上下文快满时：先「蒸馏记忆」再 `/reset`，记忆不丢

### 日常使用习惯（最佳实践，写进 README）
| 场景 | 动作 |
|---|---|
| 会话结束 / 上下文快满 | 手动说「蒸馏记忆」，再 /reset |
| 长期使用 | 配置 cron 每日 22:00 自动蒸馏 |
| 每周沉淀 | 说「提炼记忆」→ 近 7 天 daily 精华合并进 MEMORY.md |
| 敏感对话 | 蒸馏报告自动跳过敏感信息，无需手动处理 |

---

## 8. 开发与发布计划

1. 建项目目录 `projects/xiaoyaoclaw-memory-distill/`（长期项目，放 projects/）
2. 写 SKILL.md（工作流 Step 1-7）
3. 写 templates/ 三个模板
4. 写 README.md + LICENSE(MIT)
5. git init（分支 main）→ GitHub 建仓 `dtsola/xiaoyaoclaw-memory-distill` → push（代理 22307）
6. GitHub About 设置（中英 description + topics）
7. 同步全局技能 `state/skills/xiaoyaoclaw-memory-distill/`（当前工作区即可用）
8. initializer 互链：README「姊妹项目」+ SKILL.md 引用块，commit push
9. 工作区实测：手动触发一次蒸馏，验证流程
10. memory/ 日志记录

**发布安全（吸取 tracker 教训）：**
- push 前 `git rev-parse --git-dir` 确认仓库边界，`git status --short` 确认 staged 范围
- 无 BOM JSON 写 GitHub API 请求体
- PATCH 后 GET 确认生效

---

## 9. 风险与注意点

- **Cron 蒸馏的确认缺口**：定时任务无法等人工确认 → 设计为「直接写入 + 汇报差异」，敏感信息一律跳过兜底
- **去重质量依赖 LLM 判断**：查重是语义级，可能出现误合并 → 报告里展示合并项，指挥官可纠正
- **多 agent 共享 memory/**：蒸馏可能读到其他 agent 的 daily 记录 → 默认只处理「当前会话 + 当前 agent 的 daily 文件」，跨 agent 整理需明确指令
- **不替代 initializer**：目录缺失时引导初始化，不自行造轮子

---

*待确认项：① 本方案整体是否 OK；② 多 agent 共享 memory 的处理策略是否认可；③ 确认后按 §8 计划开发。*
