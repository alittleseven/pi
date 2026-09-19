# 复审记录：替代假设与砍单分析（v1.0 → v1.2 定稿）

审阅方式：R1（与主方案/答疑一致性视角）与 R2（事实核查/技术可行性视角）并行独立子代理审阅，修订后 R3 验证轮（+1，因有 major 已修）确认，均不采信采纳自报。

## R1 发现与采纳（0 blocker + 1 major + 5 minor）

| ID | 严重度 | 问题 | 采纳 |
| --- | --- | --- | --- |
| F1 | major | §3.2 跑批宿主行把宿主形态（createAgentSession，D1 拍板）与子代理 spawn（--mode json -p）混为一谈 | 改两段式：宿主=createAgentSession；子代理=spawn --mode json -p --no-session |
| F2 | minor | 事件钩子出处应为答疑 §二.4（可靠性优化），主方案机制本是 S7 显式步骤 | 已改并补「钩子本来就不是机制」 |
| F3 | minor | .harness/ 枚举漏 pattern-tags/archive；出口①归属未交代 | 补全；出口①（平台无关 python 断言）显式归不用砍 |
| F4 | minor | 模型分级同现 3.2/3.3 两档不互斥 | 3.2 加「worst case 归入 §3.3 第 3 条」互斥指针 |
| F5 | minor | 会话持久化未归类 | 补「不入三档，随引擎走」（依主方案 §7 可静默放弃定性） |
| F6 | minor | 「实 favorable」编辑残留、「程序化驱整条」漏字 | 已修 |

## R2 发现与采纳（0 blocker + 2 major + 7 minor）

已核实为准确：--tools（index.ts L307）、独立子进程（L300）、createAgentSession（sdk.ts L173）、.workbuddy/commands 与 .codebuddy 存在、机核喂果/编排方落盘与两原语职责划分对得上、OV/两问回答到位。

| ID | 严重度 | 问题 | 采纳 |
| --- | --- | --- | --- |
| J1 | major | ZCode 两处【验】与 SKILL.md 冲突：subagent_model 是 run 级非逐代理；工作流子代理无逐代理工具面（只读靠提示词）——「补齐三层」不成立 | 降级为【引·SKILL.md】；结论改「只补齐跑批宿主一层（带保留）」 |
| J2 | major | 「可直接当 R 循环状态机用/整体替代 run.ts」夸大：逐 run 用户审批 + world.run 命令集提交时冻结，不满足 M2 无人值守 | 三处收敛为「可表达控制流，不宜作无人值守实现」；§四触发条件 4 同步修正 |
| m1 | minor | .codebuddy 仅 memory/ 无命令面，「直接证明」偏强 | 收敛为 WorkBuddy 间接证据 |
| m2 | minor | CC headless 为记忆级事实，标注档位不一致 | 拆【验·实测】/【验·记忆】两档并全文统一 |
| m3 | minor | 「实 favorable」编辑残留 | 并入 F6 |
| m4 | minor | Codex 结论确定语气但依据为【引】；串行退化需写明隔离纪律前提 | 挂【引】+「若属实」+ 补 §4.3 规则 7 纪律（只喂发现清单不传报告） |
| m5 | minor | 同 F1 | 并入 F1 |
| m6 | minor | 2~3 天/平台无分解依据 | 标粗估 + 一行分解（adapter 1d/迁移 0.5d/验证 1d） |
| m7 | minor | spawnAgent 签名缺 model 与错误/超时契约 | §3.2 与 §五 补全：model? 参数（分级降级挂点）+ { text } \| { error } |

## R3 验证轮（+1）

逐条核验 13 项全部已修、0 改坏；抽查 SKILL.md 确认 J1/J2 收敛表述与明文相符（「Every subagent has the same tools…no per-subagent tool profile or model choice」、subagent_model run 级）。ZCode 降级后 OpenCode「损失最小」/Codex「最不划算」排序仍成立，无需改排序；三档砍单清单完整成立。新发现 2 minor：矩阵行 6 ZCode/CC「闭源」格漏事实标注（已补【验·记忆】）；行 3 与结论的轻度张力（已有限定语，非矛盾）。

## 结论

**通过，v1.2 定稿**（0 blocker / 0 major / 2 minor 已修）。核心结论保持：主方案无「换底座必须砍」的不可让步项；真要砍的只有事件钩子、运行时级自改、细粒度模型分级三样锦上添花项。
