# DSH 编排机制对标 Claude Code — C7

[English](dsh-编排机制对标-claude-c7.md) | 中文

> **Tag**: C7(编排机制的设计决策对标——增量自 [C6](dsh-能力复盘与对标-c6.md))
>
> **日期**: 2026-08-16
>
> **范围**: Claude Code 官方文档(code.claude.com/docs/en,2026-08 现行)的编排机制设计决策 × DSH harness 现状(path:line 证据)。C6 做能力盘点,本稿做机制契约——上下文怎么流转、约束在哪一层强制。所有 Claude Code 断言附官方 URL,所有 DSH 断言附代码位置。

## 1. 方法与演进信号

方法:Claude Code 侧抓取官方文档八页(sub-agents、hooks、hooks-guide、permissions、permission-modes、skills、memory、output-styles),tools 页与官方仓库 issue 辅证;DSH 侧代码抽查,并继承 C6 已结论的项不重复裁决。

演进信号(对标必须以现行页为准,早期二手资料已失效):Task 工具改称 Agent;自定义 slash commands 并入 skills(官方原文 "Custom commands have been merged into skills");权限档从 4 种扩到 6 种;hook 事件从早期 9 个扩到 31 个。

## 2. 七维对标矩阵

| 维度 | Claude Code 设计决策 | DSH 现状 | 判定 |
|---|---|---|---|
| plan mode | 权限档之一:编辑硬阻断、只读探索、研究委托 Plan 子代理、ExitPlanMode 审批 | 正交软约束:session log 记状态、prompt 段落劝导、工具零硬禁、plan-review intent 审批 | 落后(约束硬度) |
| subagents | 用户可写的 markdown 角色定义 + description 驱动自动委派 + 内置只读 Explore/Plan + 回传安全扫描 | 7 provider 执行面(spawn/fork/inprocess/acp/claude-code/codex/dsh-sdk)+ 后台持久 + 跨 agent 消息 | 互有胜负:执行面超越,角色定义面缺失 |
| hooks | 声明式 shell hook:31 事件、5 种 handler、全 JSON 协议(updatedInput/systemMessage/continue)、多层 settings 合并热加载 | CC 兼容子集:7 事件、command only、协议子集、进程级一次读 | 落后(协议与发现面) |
| skills | 渐进披露 + allowed-tools 预批准 + context:fork + compaction 后重附 | 渐进披露 + 多级发现 + `/name` 手势 | 持平 |
| slash commands | 已并入 skills:markdown 命令与 SKILL.md 等价,同名 skill 优先 | 27 内置 + 双注册表 + skill `/name` 手势 | 持平(CC 的合并证伪了独立 markdown 命令差距) |
| 记忆 | 四级 CLAUDE.md 拼接 + @import 4 跳 + auto memory 默认开(索引限额加载) | walk-up AGENTS.md 层级 + 嵌套投影 + 自动记忆(子串检索) | 持平偏前 |
| 权限 | 6 档模式 + deny→ask→allow 三层规则 + "always allow" 写回 settings.local.json + circuit breaker | OS 级沙箱三档(Seatbelt/bwrap/Landlock)× 一次性审批 | 沙箱超越,规则面落后(深化 C6 H2) |

## 3. 逐维契约细节

### 3.1 plan mode

Claude Code:plan 是权限档而非功能开关(Shift+Tab 循环、`/plan` 前缀、`--permission-mode plan` 三种进入方式);编辑一律阻断至批准(bypass 会话除外);`useAutoModeDuringPlan` 默认开,shell 命令交 classifier 审批;批准 UI 三选项(批准并切 auto / 批准并逐条审批 / 继续 planning);子代理默认被剥掉 EnterPlanMode。(https://code.claude.com/docs/en/permission-modes)

DSH:plan mode 自述 "independent of sandbox mode and approval policy"(`packages/plan/plan-mode/src/index.ts:5-7`);只读约束仅靠 `plan:policy` system prompt 段落(index.ts:224-232);plan 是 `exit_plan_mode` 工具的字符串参数,无计划文件;审批走 `plan-review` intent(index.ts:331-347)。

差距:Claude Code 把只读放在执行层硬强制,DSH 放在提示层劝导——模型走神或注入即可越过。

### 3.2 subagents

Claude Code:子代理是用户可写的 markdown frontmatter 角色定义(`.claude/agents/*.md`),`description` 专职驱动自动委派;内置 Explore/Plan 只读快省(跳过 CLAUDE.md 与 git status),设计意图是把探索输出挡在主上下文之外;回传摘要做安全扫描(模仿 `<system-reminder>`/`Human:` 的文本转义);context 隔离只带自身 system prompt + delegation message + CLAUDE.md 层级 + git status 快照;并发上限默认 20、嵌套深度默认 3。(https://code.claude.com/docs/en/sub-agents)

DSH:执行面远超——7 provider、fork 继承父会话前缀、后台持久 child、`send_message`/`interrupt_agent` 跨 agent 通信、maxDepth 递归限制(`packages/subagent/tool-subagent/src/index.ts:246-383`、`packages/subagent/subagent/src/depth.ts`)。但子代理是 provider 类型而非角色定义:没有用户可写的 markdown agent 面,没有 description 驱动的自动委派路由,没有内置只读 explore 角色。回传安全扫描一项未查到对应实现,列入存疑(§7)。

### 3.3 hooks(差异最大的一维)

Claude Code:31 个事件点分三种节奏(每会话/每轮/每次工具调用);handler 五种(command/http/mcp_tool/prompt/agent);exit 2 是唯一 JSON 无法推翻的阻塞;`hookSpecificOutput.permissionDecision`(allow/deny/ask/defer)+ `updatedInput` 整体替换入参 + `additionalContext` 注入;`~/.claude/settings.json` 与项目 `.claude/settings.json` 等多层合并、watcher 热加载、deny 跨层不可被 allow 推翻;hook 只能收紧不能放松,command hook 以用户全权限跑、无沙箱,交互会话在 workspace trust 接受前不执行任何 hook。(https://code.claude.com/docs/en/hooks)

DSH:`hooks-claude` 桥跑未修改的 Claude Code command hook——7 事件(`packages/hooks/hooks-claude/src/config.ts:11-19`)、exit-2 block、`additionalContext` 注入、Stop deny 转 steer 强制继续都已对齐;但 `updatedInput` 只警告不生效、`systemMessage` 不透出、项目级 per-session 配置发现是 TODO、进程级加载时一次读。

判定:主干设计同源(兼容 CC 生态是正确决策),差距在协议覆盖度与配置发现面。

### 3.4 skills

Claude Code:常驻上下文只有 name+description(合并 when_to_use 后截 1,536 字符);`allowed-tools` 是"预批准"而非白名单(仅当回合免审批);`context: fork` 在隔离上下文跑;compaction 后为每个 skill 重附最近一次调用(每个截 5,000 token、总预算 25,000)。(https://code.claude.com/docs/en/skills)

DSH:渐进披露(pre-step 发布 `<available_skills>` 目录,`skill` 工具加载全文)+ 四级发现(项目 `.dsh/skills` → customSkillDirs → `~/.dsh/skills` → bundled)+ `/name` 用户手势注入(`packages/skill/tool-skill/src/index.ts:178-205`)+ `disable-model-invocation`/`user-invocable` 字段对齐。差距仅在 allowed-tools 预批准语义与 compaction 重附契约。

### 3.5 slash commands

Claude Code 已把自定义命令并入 skills:`.claude/commands/deploy.md` 与 `.claude/skills/deploy/SKILL.md` 行为一致,同名时 skill 优先;命令名取自文件名/目录名。(https://code.claude.com/docs/en/skills)

DSH 的 skill `/name` 用户手势已等价此通道。**C6 H5(声明式命令元数据)建议关闭**:Claude Code 自己的演进证伪了独立 markdown 命令面的价值——参数化模板的方向是交给 skill,而不是另建命令层。

### 3.6 记忆

Claude Code:四级 CLAUDE.md 按"宽→窄"拼接(非覆盖);`@path` import 递归 4 跳、项目级文件解析到工作目录之外的 import 首次弹审批;auto memory 默认开启,写 `~/.claude/projects/<project>/memory/`,`MEMORY.md` 索引每会话只加载前 200 行或 25KB;官方明示 memory 是上下文不是强制执行层,硬约束必须走 PreToolUse hook 或 permissions.deny。(https://code.claude.com/docs/en/memory)

DSH:walk-up 层级 + 嵌套投影(`packages/context/workspace-context`)+ 自动记忆 store(global/session scope,`packages/memory/memory`);检索是朴素子串匹配。差距:auto memory 的"索引文件 + 限额加载"防膨胀契约可参考。

### 3.7 权限

Claude Code:6 档模式(default/acceptEdits/plan/bypassPermissions/auto/dontAsk);规则三层求值 deny→ask→allow,先匹配者生效,与规则具体度无关;`Tool(specifier)` 语法(`Bash(npm run *)`),Read/Edit 走 gitignore 路径语法;Bash 类批准写回 `.claude/settings.local.json`(复合命令拆条,上限 5 条);裸名 deny 把工具从模型上下文整体移除;任何一级 deny 不可被他级 allow 推翻;授予类配置被 workspace trust 门控。(https://code.claude.com/docs/en/permissions)

DSH:OS 级沙箱三档(read-only/workspace-write/danger-full-access,Seatbelt/bwrap/Landlock 硬强制)× 审批仅 ask/never 两态、单次一次性、无持久规则(C6 H2 已列)。防线哲学:Claude Code 是"规则面 + sandbox 纵深",DSH 是"sandbox 独挑"。

## 4. 两条主线(借鉴的筛选标准)

**上下文经济学**:subagent 隔离、skill 渐进披露、子目录 CLAUDE.md 惰性加载、auto memory 限额——一切设计把信息挡在主上下文之外。DSH 已有同等主线(spark 截断、skill 披露、投影层)。

**双层安全模型**:settings/hook 强制 × CLAUDE.md 劝导,授予类配置一律过 workspace trust。DSH 的双层是"sandbox 强制 × prompt 劝导"——中间缺了规则层。借鉴价值排序以此为准:补规则层 > 补协议覆盖 > 补角色面。

## 5. 借鉴清单(按价值/成本排序)

| # | 项 | Claude Code 机制要点 | DSH 落点 | 成本 |
|---|---|---|---|---|
| C7-1 | plan mode 硬只读 + 计划文件 | 编辑阻断在工具层;计划落盘可复阅 | plan/mode 状态接入 fs/bash 写路径守卫(复用 sandbox read-only 接缝);计划存 session artifact | 中 |
| C7-2 | hooks 协议补齐 | updatedInput 生效、systemMessage 透出 | hook-protocol codec + hooks-claude 桥 | 中 |
| C7-3 | 权限规则持久化(深化 C6 H2) | Tool(specifier) 语法、deny→ask→allow 求值、批准写回 settings.local.json | interaction/user-approval + settings 分层 | 中高 |
| C7-4 | hooks 项目级发现 + 事件扩充 | 项目 settings 合并热加载;Notification/SessionEnd/PreCompact | hooks-claude 配置面 + session 生命周期事件点 | 中 |
| C7-5 | 角色化子代理定义面 | markdown agent + description 委派路由 + 内置只读 explore | subagent 服务 + tool-subagent;发现面可与 skill 同构 | 高 |
| C7-6 | auto memory 索引限额契约 | MEMORY.md 索引 + 200 行/25KB 注入上限 | memory store 的注入段 | 低 |

## 6. 不建议借鉴与 C6 结论更新

- markdown 自定义命令不做:Claude Code 已把命令并入 skills,DSH 的 skill `/name` 手势等价;C6 H5 建议关闭(待 C6 下次修订时标注)。
- 6 档模式循环不学:DSH 的"模式三态 × 沙箱三档"正交结构更清晰;acceptEdits ≈ workspace-write 已有等价。
- matcher 非锚定正则陷阱不学(Claude Code 的 `Edit.*` 会匹配 `NotebookEdit`):DSH 若做 matcher 应直接锚定全串。
- output styles 价值低:DSH 已有 `systemPrompt.section` 装配面。

## 7. 存疑项(未能证实)

- DSH 子代理回传是否有 Claude Code 式安全扫描(`<system-reminder>` 转义)——未查到对应实现,实施 C7-5 前需确认。
- Claude Code Agent 工具完整 schema 与计划文件路径惯例:官方页未写,issue 辅证指向 `~/.claude/plans/`。
- Claude Code `allowed-tools` 与 workspace trust 的关系两个官方页面互相矛盾,以实测为准。
