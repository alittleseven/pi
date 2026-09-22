# pi × article-system 自进化写作 Harness：实现 Spec（M1+M2）

> 版本：v1.7（2026-09-22）· 状态：**定稿 + M1/M2 完成 + 用量账本**（v1.6 及之前见下方变更行）
> v1.7 变更（2026-09-22）：M2 增设用量账本（§4.2）——adapter 聚合 message_end 事件的 usage 随 SpawnResult 返回（ok 分支必有、失败分支尽力保留部分值）；SpawnRequest 增可选 `progress` 回调输出轮级实时进度（JSON 模式事件按轮推送，无逐 token 流，轮间可估 tok/s）；run.ts 逐 spawn 打用量行、终态写选题目录 `运行台账.json`（calls 明细 + 汇总，append 追加，损坏改名保留）。运行台账属运行侧账本，不计入 §4.7 模式同构产物集；**cost 暂未入账（UsageInfo 无 cost 字段）**，token 为准、人民币口径走用量快照换算。挂账：writeUsageLedger 无单测，M3 校准时抽纯函数补测。
> v1.6 变更（2026-09-22）：M2 全量完成，§4.7 全项 PASS 附证据（article-system 仓 harness/test/e2e/README.md）——端到端四场景（干净稿 DONE / blocker 三周期 / kill+resume 零重跑 / ESCALATED 隔离挂起）+ 模式同构；单测 48/48；审阅闭环四轮（parse 4 minor、状态机 1 major+6 minor、run.ts 1 blocker+1 major+8 minor、验收 4 观察全处置并 R2 维持「M2 完成」）。同批钉死：§4.3 补 DONE 挂账 minor 计数口径（原始 finding 计数，跨轮不去重；S7 入池收窄 M3 校准）。
> v1.5 变更（2026-09-22）：M2 全量开工的契约前置修订（§6 纪律，先 spec 后代码）——①§4.3 COLLECT 前置初始机核（审稿人无 bash，机核红按标注严重度计入发现清单、来源 R{n}-机核；机核记录.md 落选题目录）；②§4.3 FIX 采纳清单分节语义（首周期无节头与对话模式同构，后续周期追加「## 修复清单-R{n}」节）；③§4.5 续跑状态一律从盘面重建（不设状态文件，含 FIX 后崩溃从 CHECKS 重入）；④§5 精读类跑批边界（--type B 显式报错，M2.5 另立契约）；⑤§4.3 norm(位置) 解读钉死（序号保留进键 `段N|锚文本`，跨段同锚不同键）；⑥§4.5 补重建语义挂账（fixer 摘要/历史机核红不重放，checks-stuck 预算崩溃恢复后从零重计）。
> v1.4 变更（2026-09-22）：M2 spike 完成，§4.6 五项全勾附证据；§4.2 SpawnRequest 增补可选 `extensions?` 字段（留接口变更记录行）+ 实现要点⑤新发现「`--tools` 全局白名单连扩展工具一起过滤」。实现落点 article-system 仓：`harness/adapter/{types,pi,pi.test}.ts`、`harness/spike.ts`、`.pi/agents/reviewer.md`。
> v1.3 变更（2026-09-21）：§4.1 就位判据修正——原「`npx pi --version` 返回码 0」经实测为**假判据**（npx 解析到 npm 上同名无关包 `pi@2.0.5`，输出 `3` 且返回 0，pi 未装也「通过」），改为四条状态判据（包名真值 / engines 满足 / 可 spawn 版本自洽 / provider `status=ready`），并补两条 fail-open 说明（`auth check` 的 not_ready 也返回 0；pi 无运行时 Node 版本守卫）。动机：判据 fail-open 会让后续里程碑失去「能否开工」的真实信号。
> v1.3 变更（2026-09-21，续）：§4.2 补 provider 配置传递契约（`PI_CODING_AGENT_DIR` 注入仓内配置目录 + `models.json` 写 `$VAR` 引用，密钥走 env 不入库）；§4.1 补就位状态（四条判据全过，M2 可开工）。
> v1.3 变更（2026-09-22）：§4.2 补扩展面契约——引入 `pi-web-access@0.30.0`（零配置可用，无需搜索 key），接线走 adapter spawn argv 的 `--extension`，**只给事实-R2 挂、修辞-R1 不挂**；§4.4 reviewer 工具面同步。记录负向发现：`settings.extensions` 键在 CLI 面未生效（agentDir 与项目 settings 两处实测），故不用声明式。同批评估 6 个插件均未引入（理由见 §4.2）。
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

条目格式（三列表 + 机核化后缀）：`| 检查描述 | 默认严重度 | 模式标签 ·[机核化:脚本名（可选）] |`——机核化标注以 `·[机核化:…]` 后缀并入标签格（实现从简，M2 解析按后缀识别；此为 2026-09-21 按落地实现回改，原「机核标注独立列」的写法与实现不符）。

- 视角-轮次 ∈ `修辞-R1` / `事实-R2` / `精读-R2`（B/C 类翻译检查归入精读标记，常规文跳过）；
- 默认严重度 ∈ blocker/major/minor，审稿人可上调不可下调；
- 机核化条目由脚本判定，审稿人只复核不重查。机核脚本盘点（2026-09-21 按落地现状更新）：`check_quotes.py`（引用块 ≤300 字）**已有**；`check_basics.py`（YAML 头/围栏配对/代码行宽/图片路径/重复 n-gram/段长行数+字数 ≤110/违禁词粗筛断言）**M1 新建**，段长字数断言为毕业管道首例（提案 `.harness/proposals/2026-09-21-段落长度机核化.md`）；精读机核 **已落地** `scripts/check_jingdu.py`（`--article`/`--snapshot` 参数化，自 `scripts/bak/check-jingdu-sea.py` 提升，按篇配快照路径）；**机核脚本须有测试**：每个 check 脚本每项断言至少一个反例 + 一个正例（反例构造片段，正例可为共享的全绿最小稿或真实成稿），随脚本入库并纳入提交前必跑——与 §4.7 对 M2 TS 代码的测试要求对称（机核是唯一被允许「替代人工重查」的环节，未测试的检查器会让假阴性静默通过；2026-09-21 起 article-system `tests/` 已按此执行）；
- 事实类条目（review-final A 类）只查写手自述文字，不查译文（原文件口径，保留）。

初始检查项（从 review-final.md 与 polish 阶段 A/D、AGENTS.md 文风规则与硬性禁令迁移，迁移后原文件只留引用）：

| 条目 | 严重度 | 模式标签 | 来源 |
| --- | --- | --- | --- |
| [修辞-R1] 开头 3 行内钩子（痛点提问/反常识结论/场景故事三选一） | minor | 钩子缺失 | AGENTS.md 文风规则 2 |
| [修辞-R1] 金句 2~3 个、用引用块标出 | minor | 金句数量 | 文风规则 3 |
| [修辞-R1] 引用块单块 ≤300 字（非空白字符），超限按句界分框 | major | 引用块超限 | 文风规则 3 + review-final E [机核化:check_quotes.py] |
| [修辞-R1] 单段 ≤4 行且 ≤110 字（手机端 22~23 字/行折算） | minor | 段落长度 | 文风规则 1 |
| [修辞-R1] AI 味 16 模式逐段扫描，聚类命中，豁免标注 | minor | AI味#N | polish 阶段 A |
| [修辞-R1] 统计特征异常：句长波动过平、段落均匀度机械 | minor | 统计特征 | polish 阶段 A/D |
| [修辞-R1] 小标题为短观点句，无「一、二、三」 | minor | 小标题风格 | 文风规则 4 |
| [修辞-R1] 结尾互动引导（留言问题 + 在看/关注） | minor | 互动引导 | 文风规则 5 |
| [修辞-R1] 话题标签行：3~5 内容标签 + 品牌标签殿后，单个 ≤10 字 | minor | 话题标签 | 文风规则 6 |
| [修辞-R1] 文末二维码图行存在 | minor | 二维码 | 文风规则 8 |
| [修辞-R1] 注释用 `^ ` 小字语法；内部项目/系统一律匿名 | minor | 注释格式 | 文风规则 9 |
| [修辞-R1] 代码片段输入规范：空格缩进无 tab、一语句一行、缩进统一、嵌套 ≤3 | minor | 代码规范 | 代码片段规范（行宽子项删归 R2 机核体检统一查，避免「只复核不重查」边界模糊） |
| [修辞-R1] 错别字 | major | 错别字 | polish 阶段 D |
| [事实-R2] 不编造数据/案例/名人名言；引用须核实并注明来源；无来源信息不入文 | blocker | 编造与无源引用 | 硬性禁令 1 + 素材纪律 |
| [事实-R2] 相对时间词逐个对照成稿日期与实际日期 | blocker | 相对时间词 | review-final A1 |
| [事实-R2] 跨文章引用（篇名/发布状态/系列序号/金句原文）对 output/ 与推送记录核实 | blocker | 跨文章引用 | review-final A2 |
| [事实-R2] 本机事实（账本数字/项目名/工具行为）对记忆或数据源 | blocker | 本机事实 | review-final A3 |
| [事实-R2] 归因句冒号后内容存在且同义；内部指向指对 | major | 归因句 | review-final A4 |
| [事实-R2] 编号递进不重复不跳号；交叉引用指向正确；时间线用绝对日期验算 | major | 编号一致性 | review-final D |
| [事实-R2] 违禁词机核粗筛：100%/稳赚/必涨等确定性词表（清单见 check_basics.py） | blocker | 违禁词 | 硬性禁令 2~4 [机核化:check_basics.py] |
| [事实-R2] 违禁词人工终判：绝对化用语（「最/第一」类需语境判断）、政治·医疗·投资内容 | blocker | 违禁词 | 硬性禁令 2~4 |
| [事实-R2] 机核体检全绿：YAML 头齐全（digest ≤120）、围栏块配对、代码行宽、图片路径存在、重复 n-gram、段长（行数与字数） | major | 机核体检 | review-final E [机核化:check_basics.py] |
| [精读-R2] 覆盖完整性：承诺数量=清点数；直译跟随；图直译逐项一致；小节号与原文对上无漏节 | major | 覆盖缺口 | review-final B |
| [精读-R2] 围栏块与英文表格对快照逐行命中，未命中 0 | blocker | 翻译命中 | review-final E [机核化:check_jingdu.py（--article <文章> --snapshot <快照>）] |
| [精读-R2] 最高级不弱化（most 类必须译「最」） | blocker | 翻译弱化 | review-final C1 |
| [精读-R2] 方向句不译反 | major | 方向译反 | review-final C2 |
| [精读-R2] 数字逐个对原文；残留英文清零（术语除外） | blocker | 翻译数字 | review-final C3 |
| [精读-R2] 术语一致（harness 不译等，先列术语表再全文扫描） | major | 术语漂移 | review-final C4 |

说明：机核化条目（引用块超限、违禁词、机核体检、翻译命中）由 CHECKS 门禁脚本判定，审稿人只复核不重查；机核红 = 该条目按标注严重度计入发现清单，来源标 `R{n}-机核`。违禁词为「机核粗筛 + 人工终判」双层：脚本只查确定性词表，语境类违禁（绝对化用语等）必须由审稿人终判——机核全绿不等于违禁词条目通过。

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

每篇一行，字段分隔 ` | `：`日期 | 标题 | 类型(A/B) | R轮数 | blocker数 | major数 | minor数 | 高频模式标签(≤3) | 机核拦截次数 | 规格版本 | 备注`。标题等自由文本写入前清洗：`|`→`/`、换行→空格。**记法（2026-09-21 校准）**：行不删除、不另起一行；终态后修订轮（主方案 §4.5）**就地更新原行**——R 轮数记累计值，备注列记「含修订轮 R(n)-R(m)」及当篇最大同标签 minor 去重键数（供折算阈值再校准）。原「append-only」表述按此收窄：首篇实践（修订轮 R4，R5 按需未触发）即按就地更新执行。

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

**落盘路径（2026-09-21 钉死）**：`复审-R{n}.md` 落 `<选题调研包>/`（`materials/topics/<选题>/`，与任务计划.md、采纳清单.md 同目录），路径由任务计划给出——M2 的 run.ts/parse.ts 按该确定性路径读回。

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

Node ≥22.19.0（对齐 pi `packages/coding-agent` 的 `engines` 声明；`--experimental-strip-types` 需 ≥22.6，低版本用 tsx）、TypeScript strip-only（run.ts 自身零 npm 运行时依赖，只用 node: 内建 + child_process；pi 本体为经包内 bin spawn 的外部可执行依赖，见下方 M2 运行环境前置）。落业务仓的理由见替代分析 §三（强依赖 scripts/*.py 与 output/ 结构）。

**M2 运行环境前置（2026-09-21 校准新增，此前为隐含前置导致 spike 无法起步；同日后置判据修正见下）**：
1. 本机 Node 升级至 ≥22.19.0（2026-09-21 已装 22.19.0 并置为当前版本）；
2. pi 以 npm 依赖安装进 article-system 仓（`@earendil-works/pi-coding-agent`，`npm install --ignore-scripts`），adapter 解析其包内 bin spawn——**不依赖全局安装**（本机无全局 pi 可执行文件）；
3. **就绪验收四条（全过才算就位；判据为「状态为真」，不是「命令跑通」）**：
   - ① 包可解析且是真包：`node -p "require('./node_modules/@earendil-works/pi-coding-agent/package.json').name"` → 输出必须是 `@earendil-works/pi-coding-agent`。**必须用相对路径 require**：该包 `exports` 未导出 `./package.json`，裸子路径 `require('@earendil-works/pi-coding-agent/package.json')` 会被拒；
   - ② engines 满足：包内 `engines.node` 与 `process.version` 比对为真（pi **无运行时版本守卫**，Node 低于门槛时安装与启动都不报错，只能显式断言）；
   - ③ 可 spawn 且版本自洽：`node node_modules/@earendil-works/pi-coding-agent/dist/bundle/cli.js --version` → 输出与 ① 同版本号；
   - ④ provider 就绪：`PI_CODING_AGENT_DIR=<仓内配置目录> node node_modules/@earendil-works/pi-coding-agent/dist/bundle/cli.js auth check --provider <name> --json` → JSON 的 `status` 为 `"ready"`。**必须解析 JSON 的 status，不可看返回码**：`auth check` 在 `not_ready` 时同样返回 0（实测 `{"status":"not_ready","reason":"provider_not_found"}` 退出码 0）。
4. **禁止用 `npx pi --version` 作判据**：npx 会解析到 npm 上同名的**无关包** `pi`（实测自动安装 `pi@2.0.5`、输出 `3`、返回码 0），判据恒真——即使 pi 完全没装也「通过」。确需走 npx 时必须写全包名并加 `--no-install`：`npx --no-install @earendil-works/pi-coding-agent --version`。

**就位状态（2026-09-21 实测，四条全过 → M2 可开工）**：Node v22.19.0 已置为当前版本（nvm，镜像 npmmirror）；article-system 已建 `package.json` 并装入 `@earendil-works/pi-coding-agent@0.86.1`（`npm install --ignore-scripts`，119 包）；判据① `@earendil-works/pi-coding-agent 0.86.1`、② `satisfied=true`、③ spawn 输出 `0.86.1` 与包版本自洽、④ 三条 provider 全部 `status=ready`（不注入 env 时为 `not_ready`，机制承重已证）。另：adapter 依赖的 6 个 flag（`--mode`/`--no-session`/`--tools`/`--append-system-prompt`/`--system-prompt`/`--model`）已在 0.86.1 安装产物上逐个确认存在。provider 传递契约见 §4.2。

### 4.2 agent-adapter 接口（签名来自替代分析 §五，此处定稿）

```ts
interface SpawnRequest {
  definitionPath: string;   // 代理定义文件路径（.pi/agents/*.md）
  prompt: string;           // 已拼装的完整 prompt
  allowedTools: string[];   // 工具白名单（注意：此白名单连扩展工具一起过滤，见实现要点⑤）
  extensions?: string[];    // 扩展入口路径，逐个传 --extension（事实-R2 挂 pi-web-access，修辞-R1 不传）
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

**用量账本（v1.7）**：adapter 聚合事件流各 assistant 消息的 `usage`（input / output / cacheRead / cacheWrite / totalTokens（取末值）/ turns / durationMs（墙钟））随 SpawnResult 返回——ok 分支 `usage` 必有，失败分支保留已收到的部分值（超时/被杀也有账）；`SpawnRequest.progress?` 回调按轮输出实时进度（JSON 模式事件按轮推送、无逐 token 流，轮间估 tok/s）。run.ts 逐 spawn 打用量行、终态把本次运行（calls 明细 + 汇总）append 进选题目录 `运行台账.json`（损坏时改名保留为 `运行台账.corrupt-<ts>.json` 再重建）；该台账属运行侧账本，不计入 §4.7 模式同构产物集。**cost 暂未入账**（UsageInfo 无 cost 字段）：token 数为准，人民币口径走用量快照换算，后续可给 UsageInfo 补可选 cost（models.json 配了 cost 的 provider——deepseek 线已配——才有真实单价基础）。

**接口变更记录**：2026-09-22（M2 spike）SpawnRequest 增补可选 `extensions?: string[]`——扩展挂载是逐次 spawn 的角色差异（R2 挂、R1 不挂），v1.3 字段集无法表达；2026-09-22（v1.7 用量账本）SpawnResult ok 分支增必填 `usage`、失败分支增可选 `usage`（部分值），SpawnRequest 增可选 `progress?` 回调；其余字段与 v1.3 定稿一致。

pi 实现要点：①spawn `pi --mode json -p --no-session --tools <逗号拼接的 allowedTools>`，模型覆盖用长型 `--model <pattern>`（支持 provider/id，无 `-m` 短型）；②definitionPath 的处理**照抄官方 subagent 示例（index.ts:334-338）**：剥离 frontmatter 后的 body 写临时文件，经 `--append-system-prompt` **追加**到 pi 默认系统提示——是追加不是替换，勿用 `--system-prompt`；③从 JSON 事件流截取最终 message 文本（index.ts getFinalOutput L170-180 已有参照实现）；④runChecks 的 `checks` 参数 = 从 review-spec.md 机核化标注提取的脚本清单（常规文过滤精读项），不是手写清单；⑤**`--tools` 白名单作用于全部注册工具（含 `--extension` 扩展工具）**——源码 agent-session.ts `_refreshToolRegistry` 对扩展注册工具与内置工具过同一 allowlist，故事实-R2 的 allowedTools 必须显式含 web_search/fetch_content/source_check/get_search_content 四名，漏传则扩展挂了、工具也不在面（2026-09-22 spike 实测踩中后修正）。错误处理：超时或失败重试 1 次，再失败写 pending-decisions 挂起，不静默跳过。

**provider 配置传递（2026-09-21 定案并实测，契约落点即本节）**：adapter 在 spawn 时向子进程注入 `PI_CODING_AGENT_DIR=<仓内配置目录>`（本仓为 `config/pi/`），pi 由此读取该目录下的 `models.json` / `settings.json`（源码 `config.ts` 的 `ENV_AGENT_DIR`；不设则回落 `~/.pi/agent`）。

- `config/pi/models.json` 的 `apiKey` 写 **`$VAR` 环境变量引用**——pi 原生支持（语法是 `$VAR`，**不是 `${VAR}`**），故**密钥永不落盘、不入库**；实际凭据走本机环境变量（`ARK_AGENT_PLAN_KEY` / `DS_OV_KEY` / `BIGMODEL_API_KEY`，三条线 `authType` 均为 `api_key`）。
- 由此 `config/pi/models.json` 只含 provider 定义与 `$VAR` 引用，**可入库**；`config/pi/auth.json` 与 `config/pi/models-store.json` 属凭据与运行时状态，入 `.gitignore`（为将来 OAuth 类 provider 预留）。
- 实测承重性（2026-09-21）：注入 env → 三条线 `{"status":"ready",…}`；不注入 → `{"status":"not_ready","reason":"provider_not_found"}`。
- 未采用的候选：「全局面 `~/.pi/agent/`」（破坏仓自持与多机可复现）；「仅靠 CLI `--api-key`/`--provider`」（只能传密钥，provider 定义仍须 models.json，而 volc-plan / bigmodel-coding 是自定义 provider，内置覆盖不到）。
- **配置漂移风险（挂账）**：本仓 `config/pi/models.json` 与 a-pi-space `config/models.json` 是两份副本（pi 只读单一 models.json，无法 include）；任一侧调整模型线时须手工同步，同步记录写在本节。
- **env 透传收窄（挂账，M2 跑批上线前处理；2026-09-22 spike R1 审阅 minor）**：adapter spawn 现按 `{...process.env}` 全量透传子进程——R2 形状代理（带 web 工具、接触外部内容）进程可读全部 provider 密钥；`$VAR` 解析实际只需 models.json 里被引用的几条，后续按引用清单收窄为 allowlist 再传。spike 阶段不收窄：仓促收 PATH/系统变量易引入隐蔽破坏。

**扩展面（2026-09-22 引入 pi-web-access，接线已实测）**：事实-R2 需要核一手来源（首篇实践里 R2-01 的修复依据就是「已核 arXiv 摘要原文」），而 pi 核心只有 `read|bash|powershell|edit|write|grep|find|ls` 八个内置工具、无联网面。引入 `pi-web-access@0.30.0` 补这一环。

- **零配置可用**：Exa MCP 零配置搜索 + keyless DuckDuckGo（显式选用）+ Jina Reader 抓取 + 本地 `unpdf` 解析 PDF，**不需要任何搜索 key**；只有要换/加 provider 时才需 `config/pi/web-search.json`（含密钥，已入 `.gitignore`）。
- **接线方式：adapter 在 spawn argv 显式传 `--extension ./node_modules/pi-web-access/dist/index.js`。**
- **负向发现（勿再试声明式）**：`settings.extensions` 键在 **CLI 面实测未生效**——`config/pi/settings.json`（agentDir）与 `<cwd>/.pi/settings.json`（项目）两处都试过，tools 段仍只有内置工具。CLI 面实际生效的是 `--extension`；另有两处目录发现路径（`<cwd>/.pi/extensions/`、`<agentDir>/extensions/`）本仓未采用——走 argv 是为了下一条。
- **分角色控制（正是设计意图）**：**只有事实-R2 传 `--extension`，修辞-R1 不传**。实测两条配置的 tools 段：R1 = `read, grep, ls`；R2 = `read, grep, ls` + `web_search, fetch_content, source_check, get_search_content`。两条**都不含 write/edit/bash**，故 §3.3「审阅者纯只读由工具注册层强制、不靠提示词自觉」在能力层成立。
- **安全边界（引入前置）**：`fetch_content` 带 GitHub 仓库克隆与浏览器 cookie 能力，属写原语/敏感面。上线前须在 `config/pi/web-search.json` 设 `fetchContent.domainPolicy`，把可访问目标限定为 evidence 里出现过的域名（README 口径，schema 实现时确认），并保持 `authFetch` / `allowBrowserCookies` 关闭（默认即关）。
- **同批评估但未引入的插件与理由**：pi-subagents（M2 的 adapter 已做进程级并行 spawn，属重复实现）；pi-undo-redo（仅交互面，无头跑批用不了，改稿回滚靠 git + 采纳清单）；pi-hermes-memory（与 `.harness/lessons` 面重叠，且 better-sqlite3 原生依赖撞 `--ignore-scripts`）；pi-permission-system（peer 封顶 `^0.80.0`，0.86.1 装不上，且只读已由 `--tools` 白名单达成）；pi-mcp-adapter（后置到 M3 的 OV 慢记忆桥）；pi-agent-browser-native（后置到 M2.5 的 S 腿——模板 B「公众号文章用浏览器抓」与实拍截图确有需求，但 M2 只跑 R 循环）。

### 4.3 审阅循环状态机

纯函数核心：`next(state, input) → { state, actions[] }`，副作用（spawn/写盘）由 run.ts 执行——保证可单测、可重放。

```
状态：COLLECT → EVALUATE → (FIX → CHECKS → REVIEW → EVALUATE)* → DONE | ESCALATED | WAIT_DECISION
      （WAIT_DECISION 为 M2.5+ S 腿跑批预留，M2 范围内无触发点；触发时写 pending-decisions.md，
        进程退出，续跑从该阶段恢复）

轮次约定：round 初值 = 2（R1+R2 已审完进入 EVALUATE）；REVIEW 先自增再落盘（首个加审轮即 R3）；修订轮重入（主方案 §4.5）时 maxReviewRounds 的判定基准自修订入口重计（2 个加审轮）——M2 范围内无修订轮触发点，此为预留契约；
触顶判定 round ≥ maxReviewRounds（即第 5 轮复审后仍有 blocking → ESCALATED）。

COLLECT   : 先跑 runChecks 初始机核（常规文=check_quotes.py+check_basics.py，输出追加写 选题目录/机核记录.md），
            再并行 spawnAgent(R1)、spawnAgent(R2)——prompt 附机核记录路径（审稿人无 bash：机核红按标注严重度
            计入发现清单、来源标 R{n}-机核；机核绿项只复核不重查）→ 落盘 复审-R1.md、复审-R2.md → 解析校验（§2.1）
            幂等规则：resume 时逐产物检查「文件在盘且 §2.1 校验通过」才视为完成并跳过；
            COLLECT 半完成（一轮过一轮失）只补缺失/不合格的那一轮，不重跑合格轮。
EVALUATE  : blocking = (任一发现 severity∈{blocker,major}) ∨ (同一模式标签〔「其他」除外〕的去重键数 ≥ minorToMajorThreshold；折算按标签聚类各自计数、不跨标签混算——主方案 §4.3 规则 6 校准后口径)
            blocking 为空 → DONE
            blocking 非空且 round ≥ maxReviewRounds → ESCALATED（写 pending-decisions，附各轮分歧点）
            否则 → FIX
FIX       : spawnAgent(fixer, 采纳清单=blocking 发现按 §2.2 格式) → CHECKS
            采纳清单落盘=选题目录/采纳清单.md：首个修复周期不加分节头（与对话模式产物同构）；
            后续周期在文件尾部追加「## 修复清单-R{n}」节（n=下一加审轮号）——历史节保留，
            供 outstanding 对账与 --resume 重建「已采纳 ID 集」（§4.5）。
CHECKS    : runChecks()（checks 取自 ReviewSpec 机核化标注，常规文过滤精读项）
            全绿 → REVIEW；有红 → 回 FIX 一次（附机核输出），再红 → ESCALATED
REVIEW    : round++; 单次 spawnAgent(reviewer)，prompt 双节拼装——
              A 节（全量）：上轮未通过视角的全部 ReviewSpec 条目；
              B 节（回归）：已通过视角条目 + 修改段落清单，仅检查修改段落；
              附机核结果文件路径（审稿人只复核不重跑机核）。
            落盘 复审-R{round}.md → 解析 → EVALUATE

minor 折算：位置归一化 norm(位置)（2026-09-22 实现钉死：标号剥离但序号保留进键，键形如 `段3|锚文本`/`行42|锚文本`，无序号为纯锚文本；跨段同锚文本不同键，防止折算被低估）+ 空白折叠 + 取前 20 字锚文本；
            去重键 = (模式标签, norm(位置))；同键跨轮合并计 1 次（保留最早 ID——sourceRound 小者优先、同轮比 ID，不依赖喂入顺序；后续 ID 记为别名，**折算时该键全部 finding（代表+别名）随来源移除，foldedFrom 收录全量**）；
            模式标签 =「其他」的条目只挂账、不参与折算（兜底桶聚合互不相关瑕疵，折算语义失真）；
            同一标签的去重键数 ≥ minorToMajorThreshold → 折算 1 个 major（不同标签各自计数、不混算——主方案 §4.3 规则 6 校准后口径），
            ID 记 `M-折算-<序>` 并引用来源 minor 清单；
            折算进采纳清单后，来源 minor 标记「已折算」，从挂账计数移除；修复后剩余 minor 重新计数。
终态判定：加审轮解析后无 blocker/major（含折算）→ DONE（主方案 §4.3 唯一口径）。
            DONE 挂账 minor 计数=各报告原始 finding 计数（跨轮同题不去重，去重仅作用于折算口径）——
            S7 入池是否按去重键收窄，M3 校准（2026-09-22 验收挂账）。
```

### 4.4 四代理定义（`.pi/agents/*.md`，frontmatter 字段按 pi subagent 契约）

| 文件 | name | description（**必填**，缺了 agents.ts:90 静默跳过该代理） | tools | prompt 来源 |
| --- | --- | --- | --- | --- |
| writer.md | writer | 撰写或修复公众号文章成稿 | read, write, edit, bash | 角色与阶段目标 + writer-prompt.md 最新版 + adopted 生效摘要 |
| polisher.md | polisher | 执行 /polish 润色工序 | read, edit, bash | /polish 工序文本 + ReviewSpec R1 条目引用 |
| reviewer.md | reviewer | 按指定范围审阅文章并输出结构化报告 | read, grep, ls（**事实-R2 另加 pi-web-access 四工具** web_search / fetch_content / source_check / get_search_content，经 `--extension` 挂载；修辞-R1 不加——见 §4.2 扩展面） | 宿主按轮次从 ReviewSpec 拼装；明确"报告文本即全部产出，无写权限" |
| fixer.md | fixer | 按采纳清单修复文章 | read, edit, bash | 采纳清单 + 修复边界纪律（禁止清单外改动） |

注意：①官方示例 reviewer 白名单含 bash，这里**必须显式收窄**（答疑 §二.1）；②跑批不经 subagent 扩展、adapter 直接读这四个 .md 剥 frontmatter 取 body，故无需 agentScope——若跑批改走 subagent 扩展路径，则同样需要 `"both"`（项目级加载），无头模式下项目级确认自动跳过（主方案 §3.1）。

### 4.5 挂起与续跑

所有 WAIT_DECISION/ESCALATED 落盘 pending-decisions.md 后进程以非零码退出；重跑 `run.ts --resume <文章>` 从挂起阶段恢复。续跑状态一律从盘面重建（**不设状态文件**）：逐份 复审-R{n}.md「在盘且 §2.1 校验通过」→ 已完成轮次；采纳清单各节 `- [ ] <ID>` 全集 → 已采纳 ID 集；COLLECT 半完成只补缺失轮；崩溃于 FIX 之后、对应加审轮报告落盘之前时，从 CHECKS 重入（机核红会自然打回 FIX）。已知重建语义（2026-09-22 钉死）：fixer 摘要与历史机核红不落盘故不重放，崩溃恢复后 checks-stuck 预算从零重计（接受——执行循环步数防呆兜底）；状态机每轮输入输出全在盘（复审报告/采纳清单/机核输出），进程崩溃重跑不丢轮次。

### 4.6 M2 spike 验收清单（半天，先行）

- [x] **M2 前置就位**：Node ≥22.19.0、pi 可 spawn、provider 就绪（§4.1 运行环境前置第 3 条四条判据全过）——前置不齐不开工；（2026-09-22 spike 开工日复核：0.86.1 / satisfied=true / spawn 输出 0.86.1 / 三 provider 全 ready，四判据仍全过）
- [x] 最终文本截取函数以官方 subagent 示例 index.ts（getFinalOutput L170-180，message_end/tool_result_end 事件）为参照实现，并跑 1 个真实调用验证（有参照非探索）；（`harness/adapter/pi.ts` 的 extractFinalText + parseEventLine，真实调用回文含约定标记 SPIKE-OK）
- [x] adapter 两原语在玩具稿（article-system sandbox 文章）上跑通 spawn → 文本返回；（`harness/spike.ts`：spawnAgent 读 `examples/sample-article.md` 返回概括；runChecks 跑 check_basics.py + check_quotes.py 结构化返回且玩具稿全绿）
- [x] reviewer 只读验证：allowedTools 收窄后代理尝试写文件应失败（能力层而非提示词层）；（R1/R2 两形状各一次：写探针文件均未创建；R1 工具面=read,grep,ls，R2 另有 web 四工具——两条均无 write/edit/bash）
- [x] adapter 接口签名若需变更，变更记录写回本 spec（§6）。（`extensions?` 增补已记 §4.2 变更记录行）

**spike 完成状态（2026-09-22）**：产物 `harness/adapter/{types,pi,pi.test}.ts`、`harness/spike.ts`、`.pi/agents/reviewer.md`（§4.4 四代理中 spike 仅需 reviewer）；验证 `node --experimental-strip-types harness/spike.ts` 六项全过（5 次真实 LLM 调用走 volc-plan 默认线）+ `node --experimental-strip-types --test harness/adapter/pi.test.ts` 15 用例全绿（纯函数、机核退出码映射、桩 cli.js 夹具覆盖 spawnCli 失败映射与 PI_CODING_AGENT_DIR 注入；无 LLM 调用）。执行中新发现一条承重机制：`--tools` 全局白名单连扩展工具一起过滤（已记 §4.2 实现要点⑤）。spike 驱动脚本 `harness/spike.ts` 保留入库，作 M2.5+ 之后的回归冒烟用。

**spike 审阅记录（2026-09-22）**：R1 独立子代理审阅 0 blocker / 0 major / 3 minor——① spawnCli 失败映射与 env 注入无单测（已补桩夹具 5 用例）、② stripFrontmatter 不剥 UTF-8 BOM（已修 + 反例 1 例）、③ 子进程 env 全量透传（挂账 §4.2，M2 上线前收窄）。R2 复审：三项修复全过、15/15 单测实跑绿、无回归，可提交。

### 4.7 M2 完整验收清单

- [x] 单元测试：状态机 next() 全转移覆盖（含 minor 折算、解析失败打回、触顶 ESCALATED）；parse.ts 对 §2.1 五条校验各一个正/反例；
  （2026-09-22：47/47 绿 = parse 15 + 状态机 17 + adapter 15；状态机另覆盖别名移除/双标签折算与续号/错配 no-op/重复喂入守卫）
- [x] 端到端：玩具稿全流程 COLLECT→DONE，产物落盘与对话模式同构（同路径同格式）；
  （harness/test/e2e/topicA：初始机核→R1+R2→FIX（fixer 真实改稿）→CHECKS→R3→DONE 挂账 6 minor）
- [x] 端到端：构造 blocker → FIX → CHECKS → REVIEW → DONE 路径；
  （topicB：植入编造统计，R2-01 blocker 抓获，3 个修复周期（采纳清单分节 R4/R5），编造句修后 grep=0，R5 干净 DONE）
- [x] 中断续跑：kill 后 --resume 不重跑已完成轮次（含 COLLECT 半完成只补缺失轮）；
  （topicC2：R3 in-flight 杀进程树 → 原跑批按预算优雅升级入隔离台账 → resume 重建 phase=FIX → 崩溃恢复从 CHECKS 重入 → R3 重做 → DONE；R1/R2 mtime 未变。topicC 另证 ESCALATED rounds-exhausted 全链路（挂起条目含各轮分歧点））
- [x] 模式同构验证：同一篇文章跑批模式产物与对话模式产物对照——**文件集合相同、路径与模板结构同构、全部通过 §2.1 校验即为通过；报告正文允许差异**（两次独立 LLM 运行文本必然不同，不做逐字 diff）。
  （跑批 R 循环产物集 = {复审-R1..Rn, 采纳清单.md} 与对话模式相同，另有机核记录.md（v1.5 契约：机核输出独立成文件）；全部复审报告经 parse.ts 强制 §2.1 校验）

**M2 完成状态（2026-09-22）**：§4.7 全项 PASS，证据与运行方式见 article-system 仓 `harness/test/e2e/README.md`（夹具在 harness/test/ 下，不入 materials/topics/ 正式语料库、不被 harness_index 扫描）。实现文件：`harness/{run.ts, parse.ts, state-machine.ts, adapter/{types,pi,pi.test}.ts, parse.test.ts, state-machine.test.ts, spike.ts}` + `.pi/agents/{reviewer,fixer,writer,polisher}.md`。审阅闭环：parse R1(4m)→R2 通过；状态机 R1(1 major+6 minor)→R2 通过；run.ts R1(1 blocker+1 major+8 minor)→R2 通过（0/0）。

---

## 5. 明确不做（本 spec 边界）

**M2 范围声明**：本 spec 的 M2 状态机从 COLLECT 起，输入是已完成 S1~S4 的成稿——即 M2 只覆盖 R 循环跑批；S1 素材调研、S3 成稿、S6 封面、S7 提交的跑批编排与主方案 §3.1 的 `run_article` customTool 属后续里程碑（M2.5+，另立 spec）。这是对主方案 §10 M2 验收「跑通 S3→R1→R2 全程」的**显式收窄**：S3 腿暂由对话模式承担。另有一步有意取舍：主方案 §3.1 的 createAgentSession 宿主形态被 adapter 纯 spawn CLI 取代（理由见替代分析 §3.2/§五，等价且少一层依赖）。
**精读类跑批边界（2026-09-22 增补）**：M2 跑批只支持常规文（模板 A）——精读机核 `check_jingdu.py` 是 `--article/--snapshot` 按篇配快照的参数形状，与 runChecks 的「脚本名+文章路径」契约不兼容，且精读条目的 R1 适用域切分依赖任务计划上下文；`--type B` 在 run.ts 显式报错退出，精读跑批随 M2.5 另立契约。
其余不做：毕业管道执行细则与月度回看例程（M3~M5，另立 spec，仅预留 config 字段与 lessons 格式）；OV 慢记忆桥（答疑 §五，挂点在 S1 prompt 与月度回看，M2 不实现）；/push-draft 改动（仅文案级前置条件，随 M1 顺手做）；精读模板 B 的 R1 边界细则已定于主方案 §3.2 映射表，不重复。

## 6. 本 spec 的维护

实现中发现契约缺陷 → 先改本 spec（版本 +1，注明动机）再改代码；adapter 接口签名变更必须在 §4.2 留变更记录行。spec 与主方案冲突时以主方案为纲回改本 spec，唯 §5 已声明的有意收窄与取舍除外。
