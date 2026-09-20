# article-system 现状审阅报告

> 审阅日期：2026-09-20 · 审阅对象：`D:\lgq\ai-official-account\article-system`（分支 `main`，HEAD `d010008`）
> 审阅人：Claude Code / `deepseek-flash`。**W7 无强模型档**，每条结论附可复跑命令与输出。
> 立场：本报告以 [自进化写作 Harness 文档审阅报告](2026-09-20-自进化写作Harness文档审阅报告.md) 所审的设计为参照系，核对**落地是否忠实于设计**、**现状是否健康**。两者互为正反面。
> 严重度口径：**blocker** = 已有明确故障路径或文档承诺的机制不存在；**major** = 会持续产生偏差或阻塞下一步；**minor** = 整洁度/文档滞后。

---

## 0. 总体判断

1. **M1 落地是成功的，验收 5/5 全过。** `.harness/` 骨架完整、命令文件彻底「引用化」、首篇真实文章走完 R1→R2→修复→R3 全循环并留下合规产物（§2）。这不是「文档写完了」，是「机制真的跑起来了」。
2. **实现有一处比 pi 的设计文档更正确**——`review-spec.md` 把「违禁词」拆成「机核粗筛 + 人工终判」两层，而 pi 的 `实现Spec §1.2` 仍是单行且过度声明机核覆盖。**实现反哺了 spec**，但 spec 未回改（见报告一 §3.3）。
3. **一个 blocker 级缺口**：精读类「翻译命中」是 review-spec 里标注为 **blocker** 且标为**机核化**的条目，但它指向的脚本是 `scripts/bak/check-jingdu-sea.py`——**docstring 自称「一次性脚本，用完归档」，路径硬编码到另一篇文章，无任何参数**。下一篇精读开工时，这条门禁无脚本可跑（§4.1）。
4. **一个系统性风险**：整套设计的确定性支柱是「机核脚本 = L3 评测层」，但这批脚本**一个测试都没有**，仓库也**无 CI、无依赖声明**（§4.2/§4.8/§4.9）。
5. **毕业管道已有现成首个候选，却没有执行者**：首篇真实运行暴露了 `check_basics.py` 段长检查的假阴性（按 markdown 行数查不出手机端超行），教训已入 `pending.md` 且同标签计数**已达毕业阈值 2**——但 M3 未建，无人生成提案（§4.5）。

---

## 1. 审阅范围与方法

只读审阅，未修改任何文件。方法：

1. **对照验收**：拿 `实现Spec §3.4` 的 M1 验收清单逐项要证据。
2. **对照设计**：拿 `主方案 §3.2/§4/§5/§6` 的每一条要求核对落地形态。
3. **产物合规性**：按 `review-spec.md` 附录的 5 条格式校验规则逐份核对首篇的 `复审-R1/R2/R3.md`。
4. **健康检查**：文档滞后、脚本可运行性、依赖、测试、git 卫生。

---

## 2. Harness M1 落地验收（对照 `实现Spec §3.4`，5/5 通过）

| 验收项 | 判定 | 证据 |
| --- | --- | --- |
| 用现行 slash 编排真实写一篇（A 或 B 类），R 循环按 spec 走通 | ✅ | A 类《喊「慢一点」的 AI 巨头，被自己的用户告了》，`materials/topics/2026-09-20-AI治理十天/` 下 `复审-R1.md`(9181B) / `复审-R2.md`(4334B) / `复审-R3.md`(2620B) / `采纳清单.md`(5439B) / `复审记录.md`(3261B) 均在盘 |
| 至少经历一次「发现 blocker/major → 修稿 → +1 复审」 | ✅ | R2 报 2 个 blocker（引语平台无来源 + 违禁词「最」）→ 两轮 fixer 共 15 条采纳（12+3）→ R3 干净（blocker 0 / major 0 / minor 1） |
| `metrics.md` 首行且格式合法；`pending.md` 出现本篇发现 | ✅ | `.harness/metrics.md` L12 唯一数据行，11 字段 `\|` 分隔齐备；`.harness/lessons/pending.md` 8 条，全部来源「喊慢一点」 |
| 复审报告通过 §2.1 格式校验 | ✅ | R1 为 7 列管道表 11 行 + 「正面发现」8 条 + 结论 `通过（blocker: 0, major: 0, minor: 11）`；R2 7 列 2 行；R3 7 列 1 行。模式标签全部落在 `pattern-tags.md` 词表内（含 `其他`） |
| git 提交（article-system 仓，main 分支纪律照旧） | ✅ | `d010008`（56 files）；`.harness/` 全部产物已入库 |

**命令定义「引用化」执行到位**（这是设计 §4.4「单一权威源」的核心要求）：

- `write-article.md:87` 明写「检查项唯一权威源是 `.harness/spec/review-spec.md`」；§3 六条循环规则 + 采纳清单格式 + S7 盘面自检四项（L64）齐备；
- `review-final.md` / `polish.md` 均改为「读 review-spec 取对应视角条目」，`polish.md:55` 明写「本文件只保留工序步骤，不复述检查项」；
- **三份文件内无任何内联检查项残留**（这是最容易做成两处漂移的地方，做对了）。

**首篇 R1 报告的质量值得单独记一笔**：它引用历史数据做对照（「近 6 篇旧稿同类段落超限数为 0~1，本稿 2 处属偏高」）、给出统计量（全篇 51 句、句长标准差 26.3 字、变异系数 0.71）、并明确区分豁免（数据罗列豁免、来源标注重复豁免）。这说明 ReviewSpec 条目 + R1 prompt 在真实运行中确实产出了「有依据的判断」，而非泛泛挑刺。

---

## 3. 实现优于设计之处（建议反哺 spec）

| 项 | pi 的 `实现Spec v1.2 §1.2` | 落地 `review-spec.md v1.0` | 评价 |
| --- | --- | --- | --- |
| 违禁词 | **1 行**，标注 `[机核化:check_basics.py]` | **拆 2 行**：L40「机核粗筛：100%/稳赚/必涨等确定性词表（清单见 check_basics.py）」标机核化 + L41「违禁内容人工终判：绝对化用语（「最/第一」类需语境判断）」不标 | **实现对**。`check_basics.py:22` 的 `BANNED` 只有 8 词，不含绝对化用语；spec 的单行标注会让「最」这类 blocker 因「审稿人只复核不重查」而落空 |

**旁证**：`复审记录.md:34` 记机核在 5 个时点各跑一次、**全部全绿、拦截次数 0**；而 R2 人工抓到了违禁词「最」。若按 spec 的单行标注读，会误判为「机核漏报」；按落地实现读，则完全自洽——**分层是对的**。建议把这条反哺回 pi 的 spec（见报告一 §3.3）。

---

## 4. 问题清单

### 4.1 【blocker】精读类 blocker 级机核门禁**没有可调用的实现**

`review-spec.md:49` 把「围栏块与英文表格对快照逐行命中，未命中 0」定为 **blocker** 且标注 `·[机核化:精读机核脚本（自 scripts/bak/check-jingdu-sea.py 改造，按篇配快照路径）]`。`write-article.md:51` 要求精读类每轮必跑「`check_quotes.py` + `check_basics.py --jingdu` + **精读机核**」。

但该脚本的**实际状态**：

```python
# scripts/bak/check-jingdu-sea.py 前 8 行
"""精读稿机核：围栏块与英文表格逐行对照原文快照。一次性脚本，用完归档。"""
ART  = r"output/2026-09-19-精读自进化智能体-开发者指南.md"          # 硬编码
SNAP = r"materials/topics/2026-09-19-SelfEvolvingAgents精读/原文快照.txt"  # 硬编码
```
```
$ grep -n "argparse\|sys.argv" scripts/bak/check-jingdu-sea.py   → 无输出
```

四条事实叠加成一条明确故障路径：
1. 脚本在 `bak/`（留档区），不在 `scripts/` 一级；
2. docstring 自称「**一次性脚本，用完归档**」——它从未被设计为可复用；
3. `ART`/`SNAP` 硬编码到**另一篇（09-19）**文章，与下一篇精读无关；
4. **零参数化**——不能 `python check-jingdu-sea.py <文章>` 那样调用。

而 `review-spec` 的规则是「机核化条目由脚本判定，**审稿人只复核不重查**」。两者相乘：**下一篇精读开工时，一条 blocker 级检查既无脚本可跑、审稿人又被规则告知不必重查**。

**建议**（下一篇精读开工前必须完成）：把 `check-jingdu-sea.py` 提升为 `scripts/check_jingdu.py`，`argparse` 接收 `--article` 与 `--snapshot`（或从任务计划读快照路径），并在 `review-spec.md` L49 的标注里写上真实可跑的命令。这是 `实现Spec §1.2` 已经承诺的「改造」，只是还没做。

---

### 4.2 【major】机核脚本自身没有任何测试

整套设计把「确定性」压在机核脚本上（L3 评测层、R 轮入场券、出口①的落点）。而：

```
$ find . -name "test_*" -o -name "*_test.py" -o -name "pytest.ini" -o -type d -name tests
   → 仅 scripts/bak/test-quote-split.py（一次性验证脚本，在 bak/ 留档）
```

`check_basics.py`（161 行、7 项断言）、`check_quotes.py`、以及即将改造的精读机核——**一个测试都没有**。对照 `实现Spec §4.7` 对 M2 明确要求「单元测试：状态机 next() 全转移覆盖；parse.ts 对 §2.1 五条校验规则各一个正/反例」——**同一份 spec 对 python 机核脚本没有任何测试要求**，这是不对称的。

**为什么这条要紧**：机核是唯一被允许「替代人工重查」的环节（review-spec：「审稿人只复核不重查」）。一个未测试的检查器 + 一条「不必重查」的规则 = 假阴性会静默通过。首篇已经暴露了一个真实假阴性（见 §4.5）。

**建议**：给每个 `check_*.py` 建最小测试（每项断言一个正例 + 一个反例，用 `output/` 里真实文章做正例、构造片段做反例），纳入提交前必跑；并把「机核脚本须有测试」写进 `实现Spec`（当前只对 M2 的 TS 代码有要求）。

---

### 4.3 【major】`README.md` 滞后 8 天，入口文档与现状脱节

`README.md` mtime **2026-09-12**，早于 harness 落地（2026-09-20）8 天。实测缺口：

| 位置 | 现状 | 缺 |
| --- | --- | --- |
| 目录结构树（L31-52） | 无 `.harness/`、无 `images/` | 整个记忆层目录未列 |
| `scripts/` 列表 | 列 5 个 | 缺 `check_basics.py`、`check_quotes.py`、`article_stats.py`、`make_cover_dark.py` |
| 斜杠命令表（L54-58） | 只列 3 个：`/write-article`、`/collect-materials`、`/push-draft` | **缺 `/polish`、`/review-final`、`/company-intel`、`/company-intel-status`**（实测 `grep -n harness\|check_basics README.md` 零命中） |

**注意**：`AGENTS.md` 不含审阅循环**不是**缺陷——`主方案 §6` 明确决定「AGENTS.md 不动正文规则本身；文末加一行指向 `.harness/spec/`」，且 L160 已照做。README 滞后则是**无人决定的漂移**。

**建议**：README 补 `.harness/` 目录说明、4 个命令、4 个脚本，并把「文档随命令改动同步」列为 `write-article` S7 盘面自检的第 5 项（现有 4 项不含文档）。

---

### 4.4 【major】双平台镜像的自检规则**不可满足**

`AGENTS.md:12` 要求改 `.zcode/commands/` 后同步 `.workbuddy/commands/` 并 `diff -q` 自检。实测：7 个命令文件**内容实质一致**，但 `diff -q` **永远非空**，原因有二——

1. **行尾差异**：4 个 workbuddy 文件是 CRLF，zcode 侧是 LF；
2. **标记行差异**：workbuddy 侧多一行 `> platform: workbuddy  # 由 .zcode/commands/ 自动镜像（保持与 zcode 版本同步）`。

```
$ diff -q（CRLF 归一化后）
IDENTICAL: polish.md / review-final.md / write-article.md
REAL-DIFF: collect-materials.md / company-intel.md / company-intel-status.md / push-draft.md
   → 4 处 REAL-DIFF 的唯一差异均为上述 platform 标记行
```

即：**纪律要求一个按字面永远失败的自检**，执行者只能靠「我知道这是假阳性」绕过——这类「已知会误报的检查」最终会被忽略，包括它本该抓到的真差异。

另：`.workbuddy/commands/SYNC_NOTE.md` 的「已镜像命令清单」只列 **6 个**，**缺 `review-final.md`**；该文件自身 mtime 为 2026-09-07，未随 09-19 新增命令更新。SYNC_NOTE 第 3 条建议的 `diff -q .zcode/commands/ .workbuddy/commands/` 同样恒非空。

**建议**：把自检改成「忽略 CRLF 与 `^> platform:` 行后比对」（一条 `diff <(grep -v '^> platform:' a | tr -d '\r') <(...)` 即可），并把该命令写进 AGENTS.md 与 SYNC_NOTE；同时补全 SYNC_NOTE 的命令清单。

---

### 4.5 【major】毕业管道已有现成首个候选，却无执行者

首篇真实运行**暴露了一个机核假阴性**，而它正好是出口①的完美候选：

- `check_basics.py` 的段长检查按 **markdown 行数**计，`复审-R1.md:26` 记「段长按 markdown 行计 **0 超长**」——机核全绿；
- 但 R1 人工发现 R1-01：行17 约 126 字、行31 约 119 字，按手机端 22~23 字/行折算约 5 行，**超出「单段 ≤4 行」**；
- 教训已入 `pending.md`：「机核按 markdown 行数查不出手机端超行：按 22~23 字/行折算，单段 **110 字以上**即超 4 行」。

**这条教训是可确定性化的**（纯字数阈值，无歧义）→ 完全符合出口①「判定可确定性化（正则/计数）」的条件。且 `pending.md` 里 `段落长度` 标签**已有 2 条**，达到 `graduationCountThreshold = 2`。

**但**：`.harness/lessons/adopted.md` 为空（「（暂无条目）」）、`pending-decisions.md` 为空、`archive/` 只有 `.gitkeep`——**毕业评估从未触发过一次**，因为宿主侧（M3）未建。

**建议**：这条不需要等 M3 全建——手工走一遍 `主方案 §5.3` 五步（起草提案 → 影子验证回看 3 篇 → 拍板 → 版本 +1 → 记账），既补上 `check_basics.py` 的段长断言，也**实测一次毕业管道本身**。这是 M3 开工前最有价值的验证。

---

### 4.6 【major】「修订轮」已在进行中，与 `metrics.md` 契约冲突

首篇文章的 `任务计划.md` 有一处**未提交**的「修订轮」（+16 行），由用户 2026-09-20 指令触发（「再审阅一遍文章，然后结合国内外拿用户数据的案例加入进来讨论」），计划自行排了 S8 案例调研 → S9 案例入文 → **R4 复审** → S10 修复到终态（R5 按需），四阶段全部 `pending`。

- 计划本身**做得对**（复用 R 循环、复审走新进程、每轮机核全绿、R5 按需）；
- 但它**是规格外行为**：没有 ReviewSpec 条目定义修订轮的审阅范围，没有规则说明 minor 折算是否重新计数、R 轮预算是否与软上限 R5 共用；
- **并且与 `实现Spec §1.6`「每篇一行 + append-only」直接冲突**：同一篇文章现在走 R1/R2/R3 再走 R4/R5——更新原行违反 append-only，追加第二行违反「每篇一行」。

详见 [报告一 §3.5](2026-09-20-自进化写作Harness文档审阅报告.md)。这是**当下正在发生**的状态，不是理论风险。

---

### 4.7 【minor】未提交改动三处（收尾卫生）

```
$ git status --porcelain
 M .gitignore
 M "materials/topics/2026-09-20-AI治理十天/任务计划.md"
?? .opencode/
```

- `.gitignore` +4 行（`semantica.log`、`.codebuddy/memory/2026-09-11.md`、2 个 `.omo/run-continuation/ses_*.json`）——**逐文件枚举会话 id**，属打补丁式维护；`.omo/run-continuation/` 整目录忽略更稳。
- `任务计划.md` 的修订轮（见 §4.6）——**在途工作**，宜随 S8 起步一并提交，勿长期悬空。
- `.opencode/` 整个目录未跟踪且未忽略（内 4 文件，2 个被其内部 `.gitignore` 忽略；`openviking-config.json` 裸奔）。

### 4.8 【minor】无依赖声明

无 `requirements.txt` / `pyproject.toml` / `setup.py`。**Pillow 是真实依赖但未声明**（`make_cover.py:35`、`make_illustration.py:55`、`make_cover_dark.py` 均 `from PIL import`），仅在 `make_cover.py` docstring 末尾口头写「需要 Pillow：pip install pillow」。其余 6 个脚本纯标准库。建议补一份最小 `requirements.txt`。

### 4.9 【minor】无 CI、无测试入口

无 `.github/`，全仓无任何 `.yml`/`.yaml`。考虑到机核脚本是「提交前必跑」的纪律，建议加一个最小 CI 或 `Makefile`/`test.sh` 把「机核全绿」变成一条命令，而不是靠每次手敲。

### 4.10 【minor】其余整洁度问题（合并列表）

| # | 问题 | 证据 |
| --- | --- | --- |
| 1 | `make_cover_dark.py`（深色封面，文字**居中**）与 `AGENTS.md` 视觉规范「文字一律靠左」冲突，且已无引用 | 仅在 `.workbuddy/memory/2026-09-07.md:110` 被提及为历史产物 |
| 2 | `article_stats.py` 无任何文档引用 | `grep -rln "article_stats" --include=*.md .` 零命中 |
| 3 | `scripts/check_quotes.py` **缺 shebang**（9 个一级脚本中唯一） | 首行为 `# -*- coding: utf-8 -*-` |
| 4 | `.gitignore` 冗余：L8 `output/*.html` 已覆盖 L16 `output/*.preview.html` | 两行并存，注释只挂在 L8 |
| 5 | 孤儿预览文件：`output/2026-09-12-四个模型槽位填错不报错-OpenViking配置踩坑记.preview.html` 对应成稿已改名，预览未同步 | 命名与现存成稿不符 |
| 6 | 仓库根有 0 字节空文件 `semantica.log` | `ls -la` = 0 字节，且刚被加进 `.gitignore` |
| 7 | `write-article.md:14` 写「每阶段**六行**」，实际计划表为 8 列（阶段/执行者/目标/输入/产物/完成判据/R 轮次/状态） | 与 `任务计划.md:13` 表头不符 |
| 8 | 5 份 `scripts/bak/` 封面脚本硬编码 `FONT_PATH = "C:/Windows/Fonts/msyhbd.ttc"` | 仅在 bak/ 留档，风险低，可不处理 |
| 9 | `harness.config.json` 的 `shadowLookbackArticles` / `shadowFalsePositiveMax` / `ovQueryTimeoutSeconds` / `spawnTimeoutSeconds` 在仓库任何命令定义中无引用 | 属 M2/M3 预留字段，**符合 spec**，仅备注 |

---

## 5. 值得保留的亮点

1. **M1 一次到位**：骨架、命令改造、机核脚本、首篇实跑、产物合规、入库——`实现Spec §3.4` 的五项验收全过，且首篇是**真实发布文章**而非玩具稿。
2. **「引用化」改造彻底**：三份命令文件无一处内联检查项，单一权威源真的做到了单一。
3. **首篇 R1 报告质量高**：引用历史对照、给出统计量、明确豁免边界——证明 ReviewSpec + R1 prompt 的组合在真实运行中产出有依据的判断。
4. **「正面发现」栏真的被填了**（8 条：开头钩子、结构、金句候选、注释语法、话题标签、互动引导）——进化层出口④有料可吃，不是空栏。
5. **实现反哺 spec**（§3）：违禁词分层是实现在实践中做对、spec 需要跟上的例子。
6. **产物落点区分清晰**：成稿在 `output/`，过程产物（任务计划/复审-R\*/采纳清单/原文快照）在 `materials/topics/<选题>/`——审阅与成稿分离，符合「交接靠盘」的设计。

---

## 6. 建议（按优先级）

| 序 | 动作 | 对应 | 量级 |
| --- | --- | --- | --- |
| 1 | **提升并参数化精读机核脚本** → `scripts/check_jingdu.py --article --snapshot`，回写 review-spec L49 的真实命令 | §4.1 | 半日；**下一篇精读的前置** |
| 2 | 手工走一遍毕业管道：段落长度机核化提案 → 影子验证回看 3 篇 → 拍板 → `check_basics.py` 加段长断言 | §4.5 | 半日；**同时验证 M3 的机制设计** |
| 3 | 给 `check_*.py` 补最小正/反例测试；把「机核脚本须有测试」写进 `实现Spec` | §4.2 | 半日 |
| 4 | 修双平台自检命令（忽略 CRLF + 标记行）；补 SYNC_NOTE 清单 | §4.4 | 十几分钟 |
| 5 | README 补齐 `.harness/` + 4 命令 + 4 脚本；S7 盘面自检加「文档同步」第 5 项 | §4.3 | 半小时 |
| 6 | 把「修订轮」定义与 metrics 记法补进 harness 设计（与报告一 §3.5 同一件事） | §4.6 | 需拍板 |
| 7 | 收尾提交：修订轮计划随 S8 起步提交；`.gitignore` 改目录级忽略；`.opencode/` 决定跟踪或忽略 | §4.7 | 十几分钟 |
| 8 | 整洁度批量处理（§4.10 的 1–7） | §4.10 | 半小时 |

**1–3 建议在下一篇成稿开工前完成**；**2 是投入产出比最高的一项**——它同时补上一个真实假阴性、并首次实测毕业管道。

---

## 7. 未查证 / 存疑

- **`check-jingdu-sea.py` 的判定逻辑正确性**未审（只审了它的可调用性）。若提升为一级脚本，其逐行命中算法需要单独复核。
- **前 26 篇历史成稿未回填 metrics**——属预期（metrics 为 M1 新建），但意味着 §8 的「月度回看三数」在积累到足够篇数前无法产生有效读数。
- **`article_stats.py` 的接口可用性**未验证（依赖微信 datacube 接口，需凭据）。
- **`.opencode/openviking-config.json` 的内容**未读（是否含凭据未查证）。
- 本报告**只读审阅，未修改 article-system 任何文件**，故该仓仍是 `main` 分支 + 3 处未提交改动（§4.7）的原始状态。

---

## 附：本报告的可复跑核验命令

```bash
cd /d/lgq/ai-official-account/article-system

# M1 验收证据
cat .harness/metrics.md
cat .harness/lessons/pending.md
cat .harness/lessons/adopted.md
ls materials/topics/2026-09-20-AI治理十天/
sed -n '1,25p' materials/topics/2026-09-20-AI治理十天/复审-R1.md

# §4.1 精读机核不可调用
head -8 scripts/bak/check-jingdu-sea.py
grep -n "argparse\|sys.argv" scripts/bak/check-jingdu-sea.py

# §4.2 无测试
find . -name "test_*" -o -name "*_test.py" -o -type d -name tests

# §4.3 README 滞后
grep -n "harness\|check_basics" README.md ; sed -n '52,60p' README.md

# §4.4 镜像自检
for f in .zcode/commands/*.md; do
  b=$(basename "$f")
  diff <(tr -d '\r' < "$f") <(tr -d '\r' < ".workbuddy/commands/$b" | grep -v '^> platform:') >/dev/null \
    && echo "IDENTICAL $b" || echo "REAL-DIFF $b"
done

# §4.7 git 状态
git status --porcelain
```
