# 复审记录：实现 Spec（v1.0 → v1.2 定稿）

审阅方式：R1（迁移忠实度/一致性视角）与 R2（可实施性/技术正确性视角）并行独立子代理审阅，修订后 R3 验证轮（+1，因有 major 已修）确认，均不采信采纳自报。

## R1 发现与采纳（0 blocker + 4 major + 5 minor）

| ID | 严重度 | 问题 | 采纳 |
| --- | --- | --- | --- |
| F1 | major | 覆盖完整性判 blocker、引用块超限判 minor，与主方案 §4.2 口径（均 major）相反 | 对齐：覆盖缺口=major；引用块超限单列=major 机核化 |
| F2 | major | review-final E 机核项迁移不全（围栏配对/行宽/图片路径/n-gram/段长/精读逐行命中），配合"原文件只留引用"会静默丢失 | 补「机核体检」「翻译命中」两条 + 机核化标注；脚本盘点写实名 |
| F3 | major | polish 统计特征（句长波动/段落均匀度）未迁移且无处落 | 补 R1 条目「统计特征异常」 |
| F4 | major | M2 范围相对主方案 §10 静默收窄（S3 腿无验收、createAgentSession 宿主被取代未注明） | §5 显式收窄声明：M2 仅 R 循环跑批，S 腿归 M2.5+；宿主取代为有意取舍并注明理由 |
| F5 | minor | 硬性禁令 1（编造与无源引用）未迁移 | 补 R2 blocker 条目 |
| F6 | minor | 零散遗漏（小节号/推送记录/自述限定/^注释语法/代码规范/错别字） | 逐项补条目或并条 |
| F7 | minor | M1 修改清单漏 AGENTS.md 文末指向行 | 已补 |
| F8 | minor | 机核脚本仅泛名未盘点 | 写实名：check_quotes.py（已有）/check_basics.py（M1 新建）/精读机核（bak 改造） |
| F9 | minor | agentScope 表述与主方案 §3.1 有出入 | 注明前提（跑批不走扩展才成立） |

## R2 发现与采纳（0 blocker + 5 major + 6 minor）

已核实为准确：--mode json -p --no-session（index.ts:300）、--tools/-t 逗号分隔（args.ts:137）、官方 reviewer 白名单含 bash、agentScope both、批宿主等价性、getFinalOutput L170-180。

| ID | 严重度 | 问题 | 采纳 |
| --- | --- | --- | --- |
| F1 | major | frontmatter 漏必填 description（agents.ts:90 缺失即静默跳过代理） | §4.4 表补 description 必填列 |
| F2 | major | round 初值/触顶语义未定义 | 写死：初值 2、REVIEW 先自增、round≥maxReviewRounds 触顶 |
| F3 | major | COLLECT 半完成无幂等规定 | 「文件在盘且 §2.1 校验通过才跳过」；半完成只补缺失轮 |
| F4 | major | minor 去重键无算法定义、跨轮计数、折算清账未定 | norm 伪代码（剥标号+空白折叠+20 字锚文本）、同键跨轮计 1、折算后来源移出挂账 |
| F5 | major | 「产物 diff 仅时间戳差异」验收不可达成 | 改为可判定：文件集合相同+结构同构+过校验，正文允许差异 |
| F6 | minor | -m 短型不存在 | 改 --model 长型；--tools 注明逗号拼接 |
| F7 | minor | system prompt 实为 --append-system-prompt 追加非替换 | §4.2② 照 index.ts:334-338 机制改述 |
| F8 | minor | 复审报告单元格 \| 未清洗；结论行仅二元计数 | 补清洗规则；⑤ 扩三元计数 |
| F9 | minor | runChecks checks 来源未绑定；fixer text 未定义 | checks=ReviewSpec 机核化标注提取；fixer text=改动摘要仅记日志 |
| F10 | minor | REVIEW 双视角单次 spawn 拼装规则未定义 | 补 A 节全量/B 节回归双节拼装 |
| F11 | minor | 「JSON 事件结构是核心未知数」高估 | spike 改为以 index.ts getFinalOutput 为参照实现 |

## R3 验证轮（+1）

逐条核验 20/20 已修、0 改坏：§1.2 扩充后 27 行无重复、标签与词表自洽；§4.3 轮次约定/幂等/折算与 §4.5/§4.7 自洽；§4.2 与 §4.4 分工无矛盾；§5 收窄显式且与 §4.7 一致。抽查 agents.ts:90、args.ts、index.ts L338/L170-180 与 spec 引用相符。新发现 4 minor：WAIT_DECISION 触发点在 M2 范围悬空（已注明为 M2.5+ 预留）；§6 未豁免 §5 有意偏离（已补）；AI味#N 匹配规则未定义（已补前缀+编号规则）；spec 自身审阅报告未落盘（即本文件）。均已修。

## 结论

**通过，v1.2 定稿**（0 blocker / 0 major / 4 minor 已修）。M1 骨架（§1~§3）与 M2 spike（§4）可按文直接开工。
