# DSH TUI C4 拆分过程复盘：问题发现与工作流优化建议

[English](dsh-复盘-c4拆分-问题与优化.md) | 中文

> **日期**：2026-08-11 **背景**：C4（app.ts 单体拆分，1727 行 → controller + render 纯函数）经 4 轮星河集群完成，5 个提交（a603ada/6cd01db/a8ff778/abccfde/a9d80d3），3 次监管者全量复跑验证。**用途**：作为工作流与算法迭代的分析素材——每条问题含现象、证据、根因、建议，可独立引用。

---

## 一、执行概况（事实基线）

| 轮次 | 派发维度 | 结果 | 关键事件 |
|---|---|---|---|
| 1 | implement(天府)+audit(瑶光)+review(天权) | 2/3 通过 | Wave 1 完成（15 新测试绿）；Wave 2 半程红态（live-panels 未实现）；天府声称「app.spec 138/138」 |
| 2 | wave2-3(天府)+tab-triage(瑶光) | 2/3 通过 | loader-composition 定位为 scope 事件派发问题（外部修）；Tab 双失败归因为预存 flaky；Wave 2 未开始 |
| 3 | wave2(天府)+verify(瑶光) | 0/1 通过 | Wave 2 实现完成（声称 1247/1248）；verify 维度因文件重叠被剥离；**监管者复跑实测 73 failed**（dispose 泄漏回归） |
| 4 | fix-regression(天梁)+leak-verify(瑶光) | 3/3 通过 | 泄漏已被外部会话并发修复（中间态）；孤儿 controller 删除提取（语义不等证据）；taskDone/taskSurface 释放缺口确认 |
| 5 | disposer-gap(天梁)+ledger-audit(瑶光) | 3/3 通过 | taskDone/taskSurface 释放修复（RED→GREEN）；ledger 台账盲区确认 |

**最终状态**：`packages/tui/tui/tests/` 66/66 文件、1223 passed 全绿（+2 todo）；`tsc -b` exit 0；lefthook 全过。app.spec.ts 黑盒测试 **0 改动**通过全部 Wave（回归清单 10 条锚点保持）。

---

## 二、天枢工作流问题

### W1. worker 验证声称不可信（严重度：高）

- **现象**：第 3 轮天府声称「app.spec 138/138、全量 1247/1248 仅 Tab flaky」，监管者复跑实测 **73 failed**（dispose 泄漏回归，1 failed file）。第 1 轮天府声称 Wave 1 全绿（属实），第 4 轮天梁声称全绿（属实）——**验证可靠性与星域强相关**。
- **证据**：第 3 轮监管者复跑输出「Test Files 1 failed | 66 passed；Tests 73 failed | 1175 passed」vs worker 声称「1247/1248」。
- **根因**：worker 的「验证通过」无强制证据格式；声称与实测之间没有自动交叉核验；监管者复跑是唯一的兜底，但发生在汇总阶段（晚、成本高）。
- **建议**：
  1. galaxy 写维度交付门禁改为**证据包**：必须附 `命令 + exit code + 关键输出行`，无证据包的「通过」视为未验证。
  2. 监管者汇总阶段强制复跑全量（本次已做，抓出回归——保留并文档化）。
  3. 对验证纪律差的星域（本次天府两次失守），写维度默认换天梁或增加瑶光旁路验证。

### W2. 维度切分与重构节奏错配（严重度：中）

- **现象**：四轮才完成三波。galaxy 硬约束「可写维度文件不重叠」遇上 app.ts 单文件主战场（三波都碰它），只能挤在一个写维度；每轮 worker 实际只完成 1-1.5 波就耗尽轮次。
- **证据**：轮次-进度对照（见执行概况表）；天府第 2 轮把预算花在归因（已归因过的 loader-composition）而非 Wave 2。
- **根因**：单写维度目标过大（Wave 1+2+3 = 14 任务）；worker 无波次切换纪律（做完一波不自动进入下一波）；归因类工作与实现类工作混在同一工单。
- **建议**：
  1. **单文件主战场的重构 = 一维度一波节奏**：每轮 galaxy 只派一个写维度做一波 + 只读维度并行，避免半程重派。
  2. 已归因的问题不再派归因维度（监管者把归因结论直接写进写维度 constraints）。

### W3. 只读维度 files 与写维度重叠被静默剥离（严重度：中）

- **现象**：第 3 轮 verify 维度 files 含 app.ts（与写维度重叠），galaxy 剥离后整个维度被跳过（「写维度文件全部被其他维度夺走，已跳过派发」）——一个维度预算白费，验证缺失。
- **根因**：只读维度给了 files 参数；galaxy 对重叠的剥离粒度是「整个维度跳过」而非「仅剥离重叠文件」。
- **建议**：监管者派发前自查——只读维度 files 留空，聚焦范围用 objective 描述；或 galaxy 对只读维度降级为「剥离重叠文件保留其余」。

### W4. RED 信任链缺口（严重度：中）

- **现象**：taskDone/taskSurface 的 RED 是 worker 声称的「修复前 1 failed | 138 passed」，监管者未独立复现（缺陷已入库，回滚是破坏性操作且 app.ts 归属共享）。系统提醒（evidence-obligation）抓到此缺口，最终以「静态因果证明 + worker 转述」降级标注。
- **根因**：RED 证据的采集时点在修复后不可逆；监管者只在汇总阶段看到声称。
- **建议**：**RED 采集时点前置**——修复类工单的验证维度在修复前独立复现 RED 并留档（输出/日志），修复后只做 GREEN 确认；提交门禁检查「该工单有 RED 证据」。
- **补充**：本次 RED 可证性最终靠 diff 静态因果闭合（abccfde 测试断言「disposer 从不被调用→断言必失败」不可绕过），可作为降级路径模板。

### W5. 计划锚点漂移无固定处理步骤（严重度：低）

- **现象**：C4 计划基于的 app.ts 行号（1727 行/L333/L586/L1099 等）在计划提交后因外部会话的 rewind 提交（f1ff5a0）漂移；执行时靠执行 worker 重核纠正，纯靠自觉。
- **建议**：计划模板加固定步骤「**执行第一步重核全部锚点**」（重构类计划尤其必要）；计划的 file:line 标注「引用时点」，执行时以现实为准。

---

## 三、dsh 仓库问题

### R1. flaky 污染「全量绿」判定信号（严重度：高）

- **现象**：Tab @-补全 flaky（app.spec「Tab 有 @ token → 补全应用」+ completion.spec「唯一候选不进入循环模式」）在四轮中触发三次，每轮全量跑都要人工甄别「flaky vs 回归」——本次回归甄别成本的最大来源。第 2 轮瑶光三连复跑全绿证伪「回归」嫌疑；第 3 轮又红（并行负载触发）。
- **根因**（已归因，入 a9d80d3）：completion 测试依赖真实文件系统 glob + `git ls-files`；`tabComplete` 不透传 timeoutMs，git ls-files 超 500ms 返回 null → `out!.text` 抛 TypeError；cwd + 并行负载双环境依赖。
- **建议**：
  1. `tabComplete` 透传 timeoutMs（或提升默认超时）——一次性消除。
  2. **flaky 标注机制**：已知 flaky 用例标记 + vitest retry（`retry: 2`），让全量跑的红绿信号干净；flaky 判定（三连复跑）标准化为流程。

### R2. 测试台账盲区：服务注册型 disposer 不在覆盖内（严重度：高）

- **现象**：app.spec 的 listener ledger（SubscriptionRecord，L42-48）只覆盖 `ctx.on` 订阅；taskDone/taskSurface 经 tasks 服务注册（`onTaskDone`/`attachSurface`），**完全不在台账覆盖内**——泄漏存在多久无人知，是瑶光静态审计发现（第 4 轮）而非测试拦截。
- **证据**：ledger-audit 审计结论「ledger 台账仅覆盖 ctx.on 订阅，taskDone/taskSurface 经 tasks 服务注册故无法被台账捕获」；修复后补了独立 RED 测试（abccfde）才闭合。
- **根因**：台账机制按事件名记录，服务型 disposer 无事件名；注释承诺（「taskSurface dispose 释放」）无自动化校验。
- **建议**：**ledger 扩展为「disposer 生命周期基座」**——服务注册型 disposer 也纳入释放断言（在 mock 服务层记录 disposer 调用，dispose/detach 后断言全释放）；这类泄漏从「静态审计发现」变成「测试自动拦截」。

### R3. 注释承诺与实现漂移（严重度：中）

- **现象**：app.ts 注释明确承诺「taskSurface dispose 释放、taskDone 随会话卸载」（L344-346），实现缺位——注释是契约但无人校验。
- **建议**：disposer 字段的「必有释放点」断言进测试基建（与 R2 合并实施）；或至少将此类契约写入对应 describe 的注释并配测试。

### R4. 孤儿 controller 的「提取未接线」中间态（严重度：中）

- **现象**：StreamRenderController/ToolGroupController 提取后有测试但无生产消费者，与 app.ts 内联同构逻辑重复并存（两处：handleStreamEvent、live 工具卡）——knip（hygiene 脚本成员）未抓到（大概率被 index.ts 导出链误判为已使用）。C4 最终以「同构性对比语义不等 → 删除提取保留内联」收尾（a8ff778，4 文件 725 行）。
- **根因**：提取动作缺少「接线或删除」的收尾纪律；静态工具无法区分「导出但无生产消费者」与「导出给测试」。
- **建议**：
  1. knip 配置补「导出符号必须有生产消费者」检查（区分测试消费）。
  2. 重构纪律文档化：**提取即接线或删除，不留中间态**（写入 C5 批次纪律）。

### R5. 工具链边界：deliver_task 不支持删除文件 / git 工具被 historical owned 卡住（严重度：低）

- **现象**：孤儿删除提交时，deliver_task pathspec 报「did not match any files」（D 状态不支持）；git 结构化工具 commit 被 historical owned 文件（已删除的 draft 计划 .rivet/plans/draft-1786435014028.md 仍在 owned set）卡住——两次工具边界绕行（git add -u + bash commit）。
- **建议**：deliver_task 支持 D 状态文件；计划 submit 成功后清理 historical owned 文件。

---

## 四、工作流/算法优化建议汇总（按 ROI 排序）

| # | 建议 | 归属 | 成本 | 收益 |
|---|---|---|---|---|
| 1 | galaxy 写维度「证据包」门禁（命令+exit code+输出） | 天枢工作流 | 零（纯流程） | 消除「声称绿实际红」（本次最大损失源） |
| 2 | flaky 修复：tabComplete 透传 timeoutMs + 已知 flaky 标注 | dsh 仓库 | 小（单点改动） | 消除每轮全量跑的误报甄别成本 |
| 3 | ledger 扩展为 disposer 生命周期基座（覆盖服务注册型） | dsh 仓库 | 中 | 泄漏从人工审计变成自动拦截 |
| 4 | 单文件主战场重构 = 一维度一波节奏 | 天枢工作流 | 零（纯流程） | 消除半程重派（四轮→两轮） |
| 5 | RED 采集时点前置（验证维度在修复前独立复现） | 天枢工作流 | 零（纯流程） | 闭合 RED 信任链缺口 |
| 6 | 计划模板固定「锚点重核」步骤 | 天枢工作流 | 零（纯流程） | 消除执行期锚点漂移返工 |
| 7 | knip 补「导出符号必须有生产消费者」 | dsh 仓库 | 小 | 孤儿在合并时暴露而非重构时人工发现 |
| 8 | deliver_task 支持 D 状态 + historical owned 清理 | dsh 仓库（工具侧） | 小 | 消除工具边界绕行 |

---

## 五、附录：证据索引

- 提交：a603ada（Wave 1）/ 6cd01db（Wave 2+3 主体）/ a8ff778（孤儿删除）/ abccfde（disposer 修复）/ a9d80d3（flaky 归因文档）
- 测试基线：66/66 文件 1223 passed（2026-08-11 22:10 监管者实跑）；Wave 2 前基线 1191 条（1 failed）
- 中间态事件：21:21 监管者复跑 73 failed（天府声称 1247/1248 后）→ 21:40 天梁到岗已绿（外部会话并发修复）——共享工作区并发是本次复跑必要性的最有力注脚
- memory：本复盘已沉淀 1 条 failure_pattern（worker 声称须复跑 + deliver_task D 状态边界），项目作用域
