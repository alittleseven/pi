# pi × article-system 自进化写作 Harness：实现 Spec（M1+M2）

> 版本：v1.2（2026-09-20 定稿）· 状态：**定稿**——R1/R2 两轮独立子代理审阅（4+5 major）全部修订，R3 验证复审通过（20/20 已修、0 blocker/0 major/4 minor 已修），审阅记录见同目录 [复审记录](pi-自进化写作Harness-实现Spec-复审记录.md)
> 依据：[主方案](pi-自进化写作Harness设计方案.md) v1.2、[答疑与记忆设计](pi-自进化写作Harness-答疑与记忆设计.md) v1.2、[替代假设与砍单分析](pi-自进化写作Harness-替代假设与砍单分析.md) v1.2（下称"替代分析"，同目录）
> 范围：**M1 对话模式**（article-system 仓零代码流程实现）+ **M2 跑批模式**（harness/ 代码：adapter + 状态机）。M3~M5（毕业管道细则、月度回看例程）与 OV 桥不在本 spec，只定接口边界。
> 仓库分工：设计文档与本 spec 在 pi 仓 `feat/self-evolving-writing-harness` 分支 `docs/harness/`；实现落点在 article-system 仓（`.harness/` 契约、命令修订、`harness/` 代码）。

---

## 1. `.harness/` 文件契约（M1 建骨架即按此，M2 同构读取）

### 1.1 harness.config.json

```json
{
  "version": 1,
  "minorToMajorThreshold": 5,
  "maxReviewRounds": 5,
  "graduationCountThreshold": 2,
  "shadowLookbackArticles": 3,
  "shadowFalsePositiveMax": 0.5,
  "ovQueryTimeoutSeconds": 10,
  "spawnTimeoutSeconds": 900
}
```

| 字段 | 类型 | 默认 | 含义 | 主方案出处 |
| --- | --- | --- | --- | --- |
| version | number | 1 | 契约版本，结构变更时 +1 | — |
| minorToMajorThreshold | number | 5 | minor 去重累计折算 1 个 major 的阈值 | §4.3 规则 6 |
| maxReviewRounds | number | 5 | 复审轮软上限（R5） | §4.3 规则 5 |
| graduationCountThreshold | number | 2 | 同模式标签毕业候选阈值 | §5.3 |
| shadowLookbackArticles | number | 3 | 影子验证回看篇数 | §5.3 |
| shadowFalsePositiveMax | number | 0.5 | 影子验证误报率打回线 | §5.3 |
| ovQueryTimeoutSeconds | number | 10 | OV 查询超时预算（M3+ 用，本 spec 仅预留） | 答疑 §五.2 |
| spawnTimeoutSeconds | number | 900 | M2 单次子代理调用超时 | 本 spec §4.2 |

规则：config 只存可调参数；**流程规则文本（命令文件）里把关键数字写死并注明"与 config 同步"**，对话模式不读 JSON 计算。

### 1.2 spec/review-spec.md（审阅清单，单一权威源）

条目格式（四元组 + 机核标注）：`[视角-轮次] [默认严重度] 检查描述 | 模式标签 | [机核化:脚本名（可选）]`

- 视角-轮次 ∈ `修辞-R1` / `事实-R2` / `精读-R2`（B/C 类翻译检查归入精读标记，常规文跳过）；
- 默认严重度 ∈ blocker/major/minor，审稿人可上调不可下调；
- 机核化条目由脚本判定，审稿人只复核不重查。机核脚本盘点：`check_quotes.py` **已有**；`check_basics.py`（合并 YAML 头/行宽/图片路径/重复 n-gram/段长/违禁词断言）**M1 新建**；精读机核自 `scripts/bak/check-jingdu-sea.py` 改造，按篇配置快照路径；**机核脚本须有测试**：每个 check 脚本每项断言至少一个正例 + 一个反例（正例可用真实成稿，反例构造片段），随脚本入库并纳入提交前必跑——与 §4.7 对 M2 TS 代码的测试要求对称（机核是唯一被允许「替代人工重查」的环节，未测试的检查器会让假阴性静默通过；2026-09-21 起 article-system `tests/` 已按此执行）；
- 事实类条目（review-final A 类）只查写手自述文字，不查译文（原文件口径，保留）。

初始检查项（从 review-final.md 与 polish 阶段 A/D、AGENTS.md 文风规则与硬性禁令迁移，迁移后原文件只留引用）：

| 条目 | 严重度 | 模式标签 | 来源 |
| --- | --- | --- | --- |
| [修辞-R1] 开头 3 行内钩子（痛点提问/反常识结论/场景故事三选一） | minor | 钩子缺失 | AGENTS.md 文风规则 2 |
| [修辞-R1] 金句 2~3 个、用引用块标出 | minor | 金句数量 | 文风规则 3 |
| [修辞-R1] 引用块单块 ≤300 字（非空白字符），超限按句界分框 | major | 引用块超限 | 文风规则 3 + review-final E [机核化:check_quotes.py] |
| [修辞-R1] 单段 ≤4 行 | minor | 段落长度 | 文风规则 1 |
| [修辞-R1] AI 味 16 模式逐段扫描，聚类命中，豁免标注 | minor | AI味#N | polish 阶段 A |
| [修辞-R1] 统计特征异常：句长波动过平、段落均匀度机械 | minor | 统计特征 | polish 阶段 A/D |
| [修辞-R1] 小标题为短观点句，无「一、二、三」 | minor | 小标题风格 | 文风规则 4 |
| [修辞-R1] 结尾互动引导（留言问题 + 在看/关注） | minor | 互动引导 | 文风规则 5 |
| [修辞-R1] 话题标签行：3~5 内容标签 + 品牌标签殿后，单个 ≤10 字 | minor | 话题标签 | 文风规则 6 |
| [修辞-R1] 文末二维码图行存在 | minor | 二维码 | 文风规则 8 |
| [修辞-R1] 注释用 `^ ` 小字语法；内部项目/系统一律匿名 | minor | 注释格式 | 文风规则 9 |
| [修辞-R1] 代码片段输入规范：空格缩进无 tab、行宽 ≤70、一语句一行、缩进统一、嵌套 ≤3 | minor | 代码规范 | 代码片段规范（行宽部分机核可查） |
| [修辞-R1] 错别字 | major | 错别字 | polish 阶段 D |
| [事实-R2] 不编造数据/案例/名人名言；引用须核实并注明来源；无来源信息不入文 | blocker | 编造与无源引用 | 硬性禁令 1 + 素材纪律 |
| [事实-R2] 相对时间词逐个对照成稿日期与实际日期 | blocker | 相对时间词 | review-final A1 |
| [事实-R2] 跨文章引用（篇名/发布状态/系列序号/金句原文）对 output/ 与推送记录核实 | blocker | 跨文章引用 | review-final A2 |
| [事实-R2] 本机事实（账本数字/项目名/工具行为）对记忆或数据源 | blocker | 本机事实 | review-final A3 |
| [事实-R2] 归因句冒号后内容存在且同义；内部指向指对 | major | 归因句 | review-final A4 |
| [事实-R2] 编号递进不重复不跳号；交叉引用指向正确；时间线用绝对日期验算 | major | 编号一致性 | review-final D |
| [事实-R2] 违禁词：绝对化用语/标题党词/政治·医疗·投资内容 | blocker | 违禁词 | 硬性禁令 2~4 [机核化:check_basics.py] |
| [事实-R2] 机核体检全绿：YAML 头齐全（digest ≤120）、围栏块配对、代码行宽、图片路径存在、重复 n-gram、段长 | major | 机核体检 | review-final E [机核化:check_basics.py] |
| [精读-R2] 覆盖完整性：承诺数量=清点数；直译跟随；图直译逐项一致；小节号与原文对上无漏节 | major | 覆盖缺口 | review-final B |
| [精读-R2] 围栏块与英文表格对快照逐行命中，未命中 0 | blocker | 翻译命中 | review-final E [机核化:精读机核脚本] |
| [精读-R2] 最高级不弱化（most 类必须译「最」） | blocker | 翻译弱化 | review-final C1 |
| [精读-R2] 方向句不译反 | major | 方向译反 | review-final C2 |
| [精读-R2] 数字逐个对原文；残留英文清零（术语除外） | blocker | 翻译数字 | review-final C3 |
| [精读-R2] 术语一致（harness 不译等，先列术语表再全文扫描） | major | 术语漂移 | review-final C4 |

说明：机核化条目（引用块超限、违禁词、机核体检、翻译命中）由 CHECKS 门禁脚本判定，审稿人只复核不重查；机核红 = 该条目按标注严重度计入发现清单，来源标 `R{n}-机核`。

R1 视角 = 取全部 `修辞-R1` 条目；R2 视角 = 全部 `事实-R2`（精读类加 `精读-R2`）；加审轮按主方案 §4.3 规则 3 拼装。

### 1.3 spec/pattern-tags.md（模式标签受控词表）

初始词表 = §1.2 表内全部标签 + `其他`。匹配规则：全等匹配，唯 `AI味#N` 按前缀 `AI味#` + 编号 1~16 匹配（N 取 1~16，越界按 `其他`）。规则：新增标签只能经毕业拍板进入（M3 起）；发现清单里出现词表外标签时解析器归一化为 `其他` 并在 metrics 备注列记原文。

### 1.4 spec/writer-prompt.md（写手提示词补充）

章节结构：①当前生效版本号与日期；②历届教训摘要（从 lessons/adopted.md 当前生效条目提炼，≤20 行）；③文风要点提醒（指向 AGENTS.md，不复制正文）。S3 开工由编排方拼进 writer prompt；文件头部版本号在拍板修订时 +1。

### 1.5 lessons/pending.md 与 adopted.md（条目格式）

```
pending 条目：- YYYY-MM-DD | <模式标签> | <一句话教训> | 来源: <篇名>#R<n>-<ID>
adopted 条目：- <生效日期> | <模式标签> | <教训> | 来源: <篇名>#R<n>-<ID> | 采纳: <拍板方式> | 落点: <spec/review-spec或writer-prompt@版本> | 状态: 生效|退役
```

append-only；毕业时从 pending 移入 adopted 并删 pending 行（移动留 archive 不做，pending 行删除即毕业，历史在 git）。

### 1.6 metrics.md（指标账本）

每篇一行，字段分隔 ` | `：`日期 | 标题 | 类型(A/B) | R轮数 | blocker数 | major数 | minor数 | 高频模式标签(≤3) | 机核拦截次数 | 规格版本 | 备注`。标题等自由文本写入前清洗：`|`→`/`、换行→空格。append-only。

### 1.7 pending-decisions.md 与 archive/

pending-decisions 条目：`- <时间> | <挂起阶段> | <待拍板问题> | <上下文文件路径>`，拍板后移入 archive/（带结果）。M1 对话模式一般用不到（交互式拍板），跑批模式必需。

---

## 2. 产物模板契约（解析依赖，格式偏离按机核失败处理）

### 2.1 复审-R{n}.md 模板

```markdown
# 复审-R{n}：<视角名>审阅报告
审阅对象：<文章路径> · 对应计划轮次：<S5/R加审>
## 发现清单
| ID | 类别 | 严重度 | 位置 | 描述 | 模式标签 | 建议动作 |
| --- | --- | --- | --- | --- | --- | --- |
| R{n}-01 | … | … | … | … | … | … |
## 正面发现
- （可选，无严重度，每行一条：金句/结构/开头钩子）
## 结论
通过 / 不通过（blocker: <x>, major: <y>, minor: <z>）
```

格式校验规则（编排方执行）：①发现清单为 markdown 管道表，恰 7 列（单元格写入前清洗：`|`→`/`、换行→空格）；②严重度 ∈ {blocker, major, minor}；③ID 格式 `R<n>-两位序号` 且全表唯一；④模式标签 ∈ 词表，词表外归一化 `其他`；⑤结论行三元计数（blocker/major/minor）均与清单实际行数一致。任一违反 → 整份报告退回该审稿代理重出（附违规清单），**最多重出 1 次**，仍失败升主会话（对话模式）或写 pending-decisions（跑批）。

### 2.2 采纳清单（fixer 输入）

逐条：`- [ ] <ID> <严重度> <位置>：<修复要求>`，尾部一行机核要求：`修完重跑：<脚本清单>`。fixer 只改清单内条目，禁止顺手优化（主方案 §3.3）。

### 2.3 任务计划.md 计划表

阶段表新增「R 轮次」列：值 `R1+R2` / `R3(修F1)` / …；判据列统一写「§4.3 终态达成」口径；中断续跑规则不变（从第一个非 done 阶段接着派）。

---

## 3. M1：对话模式实现（article-system 仓改动清单）

### 3.1 文件清单

新增：`.harness/harness.config.json`、`.harness/spec/review-spec.md`、`.harness/spec/pattern-tags.md`、`.harness/spec/writer-prompt.md`、`.harness/lessons/pending.md`、`.harness/lessons/adopted.md`、`.harness/metrics.md`、`.harness/pending-decisions.md`、`.harness/archive/`（目录占位 .gitkeep）、`scripts/check_basics.py`（§1.2 机核化标注的新建脚本，v1.2 修订补入本清单）。
修改：`.zcode/commands/write-article.md`、`.zcode/commands/review-final.md`、`.zcode/commands/polish.md`、`AGENTS.md`（仅文末加一行指向 `.harness/spec/` 的细化层说明，正文规则不动）；改后按既有纪律同步 `.workbuddy/commands/` 并 `diff -q` 自检。

### 3.2 write-article.md 修订要点

1. S5 阶段行替换为 R1/R2 + 修复循环（文本按主方案 §4.1/§4.3 写死：两轮视角定义、加审触发、**终态=修复后 +1 复审干净即成稿；首轮双净直接成稿**、软上限 R5、minor≥5 折 1 major——数字直接写死并注明"与 harness.config.json 同步"）；
2. 计划表说明加「R 轮次」列；
3. R1/R2 派发 prompt 五要素之外追加：审稿人**不读**写手过程文件与 fixer 说明，只对盘（文章+ReviewSpec+上轮发现清单）；
4. S7 前加**盘面完整性自检**（逐项核对，缺一项补做才许提交）：复审-R1、复审-R2、终轮复审报告在盘且格式校验通过；采纳清单在盘；metrics.md 已追加本篇行；本篇发现已按标签 append 进 pending.md。

### 3.3 review-final.md / polish.md

检查项定义改为引用 review-spec.md 对应条目；命令文件保留：入口、备料步骤（/polish 的 A/B/C/D 工序步骤、/review-final 的流程性说明）。两文件不得再出现与 ReviewSpec 重复的检查项定义。

### 3.4 M1 验收清单

- [ ] 用现行 slash 编排真实写一篇（A 或 B 类），R 循环按本 spec 走通；
- [ ] 至少经历一次「发现 blocker/major → 修稿 → +1 复审」（人为构造或自然出现均可）；
- [ ] metrics.md 出现首行且格式合法；pending.md 出现本篇发现；
- [ ] 复审报告通过 §2.1 格式校验（对话模式主会话人工核）；
- [ ] git 提交（article-system 仓，仅 main 分支纪律照旧）。

---

## 4. M2：跑批模式实现（article-system 仓 `harness/` 目录，TypeScript）

### 4.1 目录与依赖

```
harness/
  run.ts            # 入口：读 config + 计划文件，驱动状态机
  adapter/
    types.ts        # AgentAdapter 接口
    pi.ts           # pi 实现（第一份）
  state-machine.ts  # 审阅循环状态机（纯函数核心，可单测）
  parse.ts          # 复审报告解析与格式校验（§2.1 规则的实现）
```

Node ≥20、TypeScript strip-only（node --experimental-strip-types 或 tsx，零 npm 运行时依赖，只用 node: 内建 + child_process）。落业务仓的理由见替代分析 §三（强依赖 scripts/*.py 与 output/ 结构）。

### 4.2 agent-adapter 接口（签名来自替代分析 §五，此处定稿）

```ts
interface SpawnRequest {
  definitionPath: string;   // 代理定义文件路径（.pi/agents/*.md）
  prompt: string;           // 已拼装的完整 prompt
  allowedTools: string[];   // 工具白名单
  model?: string;           // 模型覆盖（分级降级挂点；缺省继承宿主）
  timeoutMs?: number;       // 缺省 config.spawnTimeoutSeconds*1000
}
interface SpawnResult { ok: true; text: string } | { ok: false; error: string; timedOut: boolean }
interface AgentAdapter {
  spawnAgent(req: SpawnRequest): Promise<SpawnResult>;
  runChecks(articlePath: string, checks: string[]):
    Promise<{ passed: boolean; results: Array<{ check: string; passed: boolean; output: string }> }>;
}
```

契约：spawnAgent 返回代理**最终报告文本**（超时/非零退出/无输出 → ok:false）；落盘由状态机负责（审稿人无写权限）。fixer 的返回 text 定义为改动摘要，仅记运行日志，不落产物。

pi 实现要点：①spawn `pi --mode json -p --no-session --tools <逗号拼接的 allowedTools>`，模型覆盖用长型 `--model <pattern>`（支持 provider/id，无 `-m` 短型）；②definitionPath 的处理**照抄官方 subagent 示例（index.ts:334-338）**：剥离 frontmatter 后的 body 写临时文件，经 `--append-system-prompt` **追加**到 pi 默认系统提示——是追加不是替换，勿用 `--system-prompt`；③从 JSON 事件流截取最终 message 文本（index.ts getFinalOutput L170-180 已有参照实现）；④runChecks 的 `checks` 参数 = 从 review-spec.md 机核化标注提取的脚本清单（常规文过滤精读项），不是手写清单。错误处理：超时或失败重试 1 次，再失败写 pending-decisions 挂起，不静默跳过。

### 4.3 审阅循环状态机

纯函数核心：`next(state, input) → { state, actions[] }`，副作用（spawn/写盘）由 run.ts 执行——保证可单测、可重放。

```
状态：COLLECT → EVALUATE → (FIX → CHECKS → REVIEW → EVALUATE)* → DONE | ESCALATED | WAIT_DECISION
      （WAIT_DECISION 为 M2.5+ S 腿跑批预留，M2 范围内无触发点；触发时写 pending-decisions.md，
        进程退出，续跑从该阶段恢复）

轮次约定：round 初值 = 2（R1+R2 已审完进入 EVALUATE）；REVIEW 先自增再落盘（首个加审轮即 R3）；
触顶判定 round ≥ maxReviewRounds（即第 5 轮复审后仍有 blocking → ESCALATED）。

COLLECT   : 并行 spawnAgent(R1)、spawnAgent(R2) → 落盘 复审-R1.md、复审-R2.md → 解析校验（§2.1）
            幂等规则：resume 时逐产物检查「文件在盘且 §2.1 校验通过」才视为完成并跳过；
            COLLECT 半完成（一轮过一轮失）只补缺失/不合格的那一轮，不重跑合格轮。
EVALUATE  : blocking = (任一发现 severity∈{blocker,major}) ∨ (minor 去重累计 ≥ minorToMajorThreshold)
            blocking 为空 → DONE
            blocking 非空且 round ≥ maxReviewRounds → ESCALATED（写 pending-decisions，附各轮分歧点）
            否则 → FIX
FIX       : spawnAgent(fixer, 采纳清单=blocking 发现按 §2.2 格式) → CHECKS
CHECKS    : runChecks()（checks 取自 ReviewSpec 机核化标注，常规文过滤精读项）
            全绿 → REVIEW；有红 → 回 FIX 一次（附机核输出），再红 → ESCALATED
REVIEW    : round++; 单次 spawnAgent(reviewer)，prompt 双节拼装——
              A 节（全量）：上轮未通过视角的全部 ReviewSpec 条目；
              B 节（回归）：已通过视角条目 + 修改段落清单，仅检查修改段落；
              附机核结果文件路径（审稿人只复核不重跑机核）。
            落盘 复审-R{round}.md → 解析 → EVALUATE

minor 折算：位置归一化 norm(位置) = 剥离「L42/第N段」类标号取段落序号 + 空白折叠 + 取前 20 字锚文本；
            去重键 = (模式标签, norm(位置))；同键跨轮合并计 1 次（保留最早 ID，后续 ID 记为别名）；
            去重累计 ≥ minorToMajorThreshold → 折算 1 个 major，ID 记 `M-折算-<序>` 并引用来源 minor 清单；
            折算进采纳清单后，来源 minor 标记「已折算」，从挂账计数移除；修复后剩余 minor 重新计数。
终态判定：加审轮解析后无 blocker/major（含折算）→ DONE（主方案 §4.3 唯一口径）。
```

### 4.4 四代理定义（`.pi/agents/*.md`，frontmatter 字段按 pi subagent 契约）

| 文件 | name | description（**必填**，缺了 agents.ts:90 静默跳过该代理） | tools | prompt 来源 |
| --- | --- | --- | --- | --- |
| writer.md | writer | 撰写或修复公众号文章成稿 | read, write, edit, bash | 角色与阶段目标 + writer-prompt.md 最新版 + adopted 生效摘要 |
| polisher.md | polisher | 执行 /polish 润色工序 | read, edit, bash | /polish 工序文本 + ReviewSpec R1 条目引用 |
| reviewer.md | reviewer | 按指定范围审阅文章并输出结构化报告 | read, grep, ls | 宿主按轮次从 ReviewSpec 拼装；明确"报告文本即全部产出，无写权限" |
| fixer.md | fixer | 按采纳清单修复文章 | read, edit, bash | 采纳清单 + 修复边界纪律（禁止清单外改动） |

注意：①官方示例 reviewer 白名单含 bash，这里**必须显式收窄**（答疑 §二.1）；②跑批不经 subagent 扩展、adapter 直接读这四个 .md 剥 frontmatter 取 body，故无需 agentScope——若跑批改走 subagent 扩展路径，则同样需要 `"both"`（项目级加载），无头模式下项目级确认自动跳过（主方案 §3.1）。

### 4.5 挂起与续跑

所有 WAIT_DECISION/ESCALATED 落盘 pending-decisions.md 后进程以非零码退出；重跑 `run.ts --resume <文章>` 读任务计划与 pending-decisions 从挂起阶段恢复。状态机每轮输入输出全在盘（复审报告/采纳清单/机核输出），进程崩溃重跑不丢轮次。

### 4.6 M2 spike 验收清单（半天，先行）

- [ ] 最终文本截取函数以官方 subagent 示例 index.ts（getFinalOutput L170-180，message_end/tool_result_end 事件）为参照实现，并跑 1 个真实调用验证（有参照非探索）；
- [ ] adapter 两原语在玩具稿（article-system sandbox 文章）上跑通 spawn → 文本返回；
- [ ] reviewer 只读验证：allowedTools 收窄后代理尝试写文件应失败（能力层而非提示词层）；
- [ ] adapter 接口签名若需变更，变更记录写回本 spec（§6）。

### 4.7 M2 完整验收清单

- [ ] 单元测试：状态机 next() 全转移覆盖（含 minor 折算、解析失败打回、触顶 ESCALATED）；parse.ts 对 §2.1 五条校验规则各一个正/反例；
- [ ] 端到端：玩具稿全流程 COLLECT→DONE，产物落盘与对话模式同构（同路径同格式）；
- [ ] 端到端：构造 blocker → FIX → CHECKS → REVIEW → DONE 路径；
- [ ] 中断续跑：kill 后 --resume 不重跑已完成轮次（含 COLLECT 半完成只补缺失轮）；
- [ ] 模式同构验证：同一篇文章跑批模式产物与对话模式产物对照——**文件集合相同、路径与模板结构同构、全部通过 §2.1 校验即为通过；报告正文允许差异**（两次独立 LLM 运行文本必然不同，不做逐字 diff）。

---

## 5. 明确不做（本 spec 边界）

**M2 范围声明**：本 spec 的 M2 状态机从 COLLECT 起，输入是已完成 S1~S4 的成稿——即 M2 只覆盖 R 循环跑批；S1 素材调研、S3 成稿、S6 封面、S7 提交的跑批编排与主方案 §3.1 的 `run_article` customTool 属后续里程碑（M2.5+，另立 spec）。这是对主方案 §10 M2 验收「跑通 S3→R1→R2 全程」的**显式收窄**：S3 腿暂由对话模式承担。另有一步有意取舍：主方案 §3.1 的 createAgentSession 宿主形态被 adapter 纯 spawn CLI 取代（理由见替代分析 §3.2/§五，等价且少一层依赖）。
其余不做：毕业管道执行细则与月度回看例程（M3~M5，另立 spec，仅预留 config 字段与 lessons 格式）；OV 慢记忆桥（答疑 §五，挂点在 S1 prompt 与月度回看，M2 不实现）；/push-draft 改动（仅文案级前置条件，随 M1 顺手做）；精读模板 B 的 R1 边界细则已定于主方案 §3.2 映射表，不重复。

## 6. 本 spec 的维护

实现中发现契约缺陷 → 先改本 spec（版本 +1，注明动机）再改代码；adapter 接口签名变更必须在 §4.2 留变更记录行。spec 与主方案冲突时以主方案为纲回改本 spec，唯 §5 已声明的有意收窄与取舍除外。
