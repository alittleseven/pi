# 复审-R2：技术视角审阅报告（设计方案 v1.0）

审阅人：独立子代理（技术视角，全新上下文）· 日期 2026-09-20
审阅对象：pi-自进化写作Harness设计方案.md v1.0
事实核对底稿：pi packages/coding-agent src/core/sdk.ts、src/core/extensions/types.ts、examples/extensions/subagent/、examples/sdk/

## pi 事实核对结果

sdk.ts 的 CreateAgentSessionOptions 确有 tools/excludeTools/customTools/resourceLoader/scopedModels/noTools，§3.1 说法成立；extensions/types.ts 的 registerCommand(L1334)/registerTool(L1325)/agent_start|end(L1300)/turn_start|end(L1305)/session_shutdown(L1283) 均在；subagent 示例确为独立 pi 子进程（spawn + `--mode json -p --no-session`），agents/*.md frontmatter（name/description/tools/model，tools 经 `--tools` 下发），并行上限 8/并发 4，中止 SIGTERM→SIGKILL 传播，agent 缺 model 时继承宿主模型——均如方案所述。05-tools.ts 与 12-full-control.ts 证实 M2（createAgentSession + tools 白名单 + customTools 经扩展注册）可行。§7 轮数封顶算术（R1+R2+三次修复复审=R5）自洽。

## 发现清单

| ID | 严重度 | 位置 | 问题 | 建议 |
| --- | --- | --- | --- | --- |
| A1 | major | §3.1/§3.3/T2 | 「审稿人无写权限」被定为硬约束，但 reviewer 工具面含 bash，bash 可写文件；pi 示例 reviewer.md 自认工具权限非完全可强制 | reviewer 改纯 read-only，E 类机核由宿主跑完把结果喂给 reviewer；或明示是软约束 |
| A2 | major | §4.2/§3.1 | pi 子进程 reviewer 无 write 工具，则 复审-R{n}.md 由谁落盘未定义；对话模式子代理可自行写文件、跑批模式不能，两模式产物不同构 | 统一规定两模式都由编排方持久化 reviewer 返回文本 |
| A3 | major | §4.3 vs §6/§9 | 终态判据自相矛盾（与 R1-F1 同源独立命中）：修稿后 R3 一轮干净即收敛还是需再一轮？循环控制器无法实现 | 统一为「修复后 +1 复审干净即收敛」，删改图注 |
| B1 | minor | §3.1 | .pi/agents/ 项目级代理默认不加载：subagent 示例默认 agentScope:"user"，项目级需显式 "project"/"both"，非信任仓还会弹确认 | 方案注明 scope 配置项 |
| B2 | minor | §1.2/§7 | 子代理以 --no-session 运行，R 轮子进程无 JSONL 留痕，「会话持久化供复盘」仅适用主会话 | 注明此边界，审计完全依赖盘面 |
| B3 | minor | §1.2/§3.1 | 「Agent 工具等价独立进程」措辞不准：pi 无内置 Agent 工具（subagent 是示例扩展）；ZCode Agent 子代理亦非独立 pi 进程 | 措辞改「上下文隔离等价」，代理定义两模式各自维护 |
| B4 | minor | §7 | 引用「⑤ 回看退役」但 §5.2 只定义 ①–④ | 改为「§5.3 回看退役」 |
| B5 | minor | §4.3 | 流程图 R1→R2 串行，规则1 说可并行派发 | 图上加并行注记 |
| B6 | minor | §8 | metrics.md 以 `|` 分隔且标题为自由文本，未定义转义规则，标题含 `|` 即破行 | 规定标题清洗（禁 `|`/换行） |

## 结论

blocker 0 + major 3 → **通过（附条件）**：A1~A3 均为一处措辞或机制澄清级修改，不动五层架构，修订后按 §9 流程 +1 复审即可定稿。pi 能力引用无虚构：未发现方案声称了源码中不存在的能力。

## 采纳情况（主会话记）

- A1 采纳：reviewer 工具面改 read/grep/ls 纯只读（不下发 write/edit/bash，工具注册层强制），E 类机核由编排方跑完喂结果文件；§3.3 措辞同步。
- A2 采纳：§3.3 与 §4.3 明确「审稿人返回报告文本，编排方负责落盘」，两模式同一落盘契约。
- A3 采纳（与 F1 同源）：终态统一，见 R1 采纳记录。
- B1~B6 全部采纳（agentScope 注明、--no-session 审计边界、等价措辞、退役改引、并行注记、metrics 清洗规则）。
