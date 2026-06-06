---
name: agent-cli-design
description: >-
  设计与编写命令行工具（CLI），尤其是给 LLM agent 驱动的 CLI —— 选对命令模型、输出形态、
  护栏、文件架构。当用户在设计或编写 CLI、新增或重构 subcommand/verb、选 CLI 框架或库、
  设计 flags / 寻址 / 批量操作、构建一个会被 agent 调用的工具，或重构一个长烂了的 CLI
  （上帝文件、纠缠的命令 handler、复制粘贴的 op 构造）时使用。触发词："写/设计一个 CLI"、
  "加个命令"、"CLI 架构"、"agent 友好的工具"、"命令行工具"、"做个我的 agent 能用的工具"、
  "梳理我的 CLI 结构" 等 —— 即使用户没说"CLI"这个词、但明显在搭一个终端/命令工具，也应触发。
  目前 TypeScript（commander）有完整参考，Go / Rust / Python 参考待补。
---

# agent-cli-design

## 核心论点

**现代 CLI 的首要使用者是 LLM agent，不是人。** 人运行一条命令、用眼睛读屏幕、用一个不按
token 计费的脑子决定下一步。而 agent 运行你的命令后，*输出直接回灌进一个有限且昂贵的
context window*，并自动驱动下一步决策。仅这一点就重排了所有设计优先级：

> 按 **一次就对（one-shot correctness）、token 经济、批量友好、失败可恢复、可审计** 排序 ——
> 漂亮的 help 文本和聪明的交互式提问几乎一文不值；干净的结构化结果、以及一条「告诉 agent
> 接下来该干什么」的错误，才是全部。

人当然也读这工具（而且好的 agent-facing 设计往往也是好的 human-facing 设计），但 agent
是热路径。为它而设计。

本 skill 分两半：
- **原则（本文件）** —— 语言无关。*怎么思考* 命令模型、输出、护栏、可组合性、架构。
- **语言参考** —— 具体技术栈 + 可直接复制的构件。按用户的栈读对应那份：
  [`references/typescript.md`](references/typescript.md)（commander，最深），以及
  `references/{go,rust,python}.md`（待填的骨架）。

真要动手时：先读下面的原则，再打开语言参考、把 harness 抄过来，别重造。

---

## 第 1 部分 —— 输出即产品

agent 永远看不到你的代码；它看到的是你的输出和错误。把这两样做对，其余水到渠成。

**1. 地址（address）是通用货币。** 你的工具寻址什么 —— 文件、行、区间、实体 id —— 那个
token 就是从「查询」流向下一次「变更」的东西。所以查询必须 *吐出* 地址、动词必须 *吃* 地址。
最大的罪是把地址埋进人类可读的一行里（`第1集 第3场 第5句："..."`），逼 agent 用脆弱的
`awk` 把它切回来。给地址一个独立字段。

**2. 输出分档，按需付重量。**

| 档 | 形态 | 用于 |
|---|---|---|
| 默认 | 简洁文本（行式、地址单独成列） | **agent 自己的 read→decide 循环** + 人扫一眼；便宜 |
| `--json` | 结构化信封 + `rows[]` 数组（addr/type/… 各为字段） | *程序*（jq / 脚本 / MCP host 的结构化 tool）做确定性解析 —— **不是** agent 喂自己的下一步 |
| `--format addr`（或 `-0`） | 裸的、换行/NUL 分隔的地址流 | 喂 xargs 或批量构造器 —— *只要货币，不让正文再进 context* |

**默认走简洁文本档 —— 而且这正是 agent 自己 read→decide 循环该读的档。这点最反直觉，单独讲清：**
agent 的「消费者」不是 `JSON.parse` 的程序，而是把你的输出读进 context window 的 *LLM*。对 LLM：

- **JSON 更费 token** —— 每行重复键名 + 括号引号；纯文本把键名变隐式，同样信息常省 20–50%。
- **JSON 更费「脑力」** —— 已有基准显示，把同样信息从纯文本换成 JSON 编码会让模型推理 *显著退化*
  （观察到 ~10–15%）；结构化包装对 LLM 是噪声，不是帮助。
- **LLM 本就母语级读自然文本** —— 何必逼它先解析一层再理解。

所以：**agent 的「发现 → 决策 → 下一条命令」全程读简洁文本档；`--json` 只留给确定性 *程序***
**（jq 管道、脚本、MCP host 的结构化 tool）。** 把「机器 = 吃 JSON」当默认，是把「程序消费者」的
前提错套到「LLM 消费者」身上 —— 这是 agent-facing CLI 最常见、最隐蔽的设计错误之一（连"机器就该
输出 JSON"的肌肉记忆都会把你带过去）。

**但简洁文本 ≠ 一团散文。** 它必须 *行式、字段稳定*：地址单独成列、一行一条，让 LLM 不靠脆弱
解析就能逐条抬起地址（呼应原则 1）。同时别走反面 —— 逼 agent 把一墙正文重灌进 context 只为捞一个
它早算好的 id。简洁文本要同时满足：**人/LLM 直觉可读、地址机器可抬、正文按需付费**。

**3.「一条命令覆盖一片」省的是 *context*，不是调用次数。** 这是批量操作对 agent 重要的最深
理由。N 条单点命令的循环会产生 N 份结果回显 *堆进 context window* —— N 份几乎一样的输出。
一条带 selector 的命令把它压成一份命中摘要。你省的不是 5 秒，是 50 份 context。

**4. 错误即下一条命令。** 出错时，错误体里应当装着 *修法*，而不只是诊断：
- 命中歧义 → 回候选地址 + 消歧的 flag
- 无命中 → 回当前有效的区间/范围
- 缺/多参数 → 回那条改好的调用
- 前置条件未满足 → 回该先跑什么

agent 读完错误一回合就自纠，而不是三次试探性调用。把 `nextSteps` 焊进你的错误类型
（见参考里的 `CliError`）。

**5. 护栏住在工具里，不住在文档里。** agent 会 *自信地宣称成功* ——「我已经把所有标记清掉
了！」—— 不管它有没有发生。你只写在 help 字符串或注释里的东西，都会被跳过。在代码里强制：
宽写之前回显 blast radius、默认拒绝歧义命中、让 `--dry-run` 与真跑共用完全相同的匹配逻辑、
用 **语义化退出码**（见下）让脚本能按结果分支。

---

## 第 2 部分 —— 命令模型：`selector × mutation`

多数 CLI 攒了一堆零散动词。有威力、好学的形态是一个 **笛卡尔积**：一套「选哪些」的语言
（selector），正交地喂给每个「做什么」的动词（mutation）。N 动词 × M selector 变成 N + M
要学的东西，而不是 N × M。

**选择轴**（选哪些），按威力粗排：
- 单点 —— 一个地址
- 区间 —— `a..b`（用 `..`；它在 Rust/Ruby/Kotlin 里就是闭区间）
- 多区间 —— 逗号连接 `a..b,c..d`（用 `,`；它读起来就是列表）
- 谓词 —— 按属性筛（`--type X --owner Y`，或 `--where k=v`）
- 模式 —— 对内容做 regex / glob
- 全部 —— *显式* 的「一切」（`--all` 或 `--in '*'`）。**绝不**让空 selector 静默等于「全部」
  —— 那正是 agent 漏个 flag 就把整份文档清空的方式。让全本范围是个刻意的 token。

**变更轴**（做什么）：正文编辑 / 属性设置 / 状态与关系 / 实体（改名/合并/删除） / 结构
（插入/移动/拆分）。

**让它成立的两条规则：**
- **同一套选择语言既读又写。** agent 用 `list --in a..b --type X` 发现，用
  `set-type --in a..b --type X <value>` 变更 —— *完全相同* 的选择 flag。选择语法学一次、两边用。
  这是整个模型里杠杆最高的一步。
- **结构性动词留单点。** 插/删/移会让后面所有元素的下标漂移，所以预先展开的 selector 会在
  sweep 中途指错。要诚实：在结构性动词上拒绝 selector，改走事务 surface（第 3 部分）。别假装
  一个会损坏数据的能力。

---

## 第 3 部分 —— 可组合性：三个 surface，一个引擎

agent 爱管道。管道适合 *发现* 和 *变换*，对 *写入* 是 **范畴错误**。

**为什么管道串写是错的：** 一份有引用完整性的文档是 *共享可变状态*。`edit A | edit B` 是两个
进程各自独立 read→mutate→write 文件；管道里流的是报告、不是文档；并发写还会 last-write-wins
互相覆盖。写不是流水线 stage，是对同一份状态的顺序覆写。

**正确的形态是 `discover → plan → apply`**，按 *谁来写* 分成三个 *authoring surface*，全部编译
到同一个事务 op 引擎：

| Surface | 作者 | 何时 |
|---|---|---|
| **selector-verb**（`set --in a..b …`） | agent，交互 | 一片同构改动（80% 场景） |
| **verb-script**（`do -` / `do edits.txt`） | agent，手写 | 一组各不相同的异构改动 |
| **machine JSON**（`patch -`） | 程序 / jq | 管道里的结构化变换 |

**verb-script 的洞察（这点最多人漏）：** 批量应当 *就是 agent 平时敲的那套动词*，一行一条命令
—— 而不是另一套要 agent 翻译过去的 JSON op schema。把 `replace ep#3 --from x --to ""` 逼成
`{"op":"content.replace","at":"ep#3","from":"x","to":""}` 既是翻译税、*又是* 分叉陷阱（两套
编码会漂移；见坑 #4）。让 agent 写它会敲的行；用同一个 tokenizer 解析；编译到同一批 op。
JSON 留给 jq 管道的 *机器* 接口、从 stdin（`-`）读，而不是给人/agent 手写。

**事务 = 持久化边界。** 把所有 op 应用到一份内存副本；任一失败就 *不保存* —— 仅这一点就给了
全成或全不成的原子性，不需要回滚引擎。末尾再对内存里的 *将成之态* 重新校验（见坑 #3）。

---

## 第 4 部分 —— 护栏默认值（面对一个会宣称成功的 actor）

- **多点写默认 dry-run。** 一旦写操作选中超过一个东西（区间 / 谓词 / `--all`），就默认 dry-run、
  回显 blast radius（数量 + 样例地址 + would-skip），要 `--apply` 才落盘。一条好记的规则：
  *「选中不止一个 → 必须确认」*。单个显式地址直接写（blast radius 低）。
- **replace 默认字面量、regex opt-in、多命中拒绝。** 字面量 `--from` 在一个项里命中两次就是
  歧义 —— 默认拒绝、要么 `--all`、要么写更长更唯一的串。对应 `sed -F` / 不带 `-i`。
- **语义化退出码** 让 agent 能 `&&` / 分支：
  `0` = 有改动/成功 · `1` = 无命中、什么都没做（grep 风格） · `2` = 写了但校验需修复 ·
  `>2` = 致命/用法错。agent 就能链 `discover && apply` 并信任退出码。
- **幂等。** 重跑一次清理（"strip the `[xxx]` prefixes"）必须无害、第二次报 "0 changed"。agent
  会重试，为此而设计。
- **一套选择语法，校验一次。** 区间端点同 kind、禁止跨层裸数字（`3..7` 有歧义 —— 强制
  `ep_3..ep_7`）、reversed 区间自动 swap（id 单调，无歧义、比报错更对 agent 友好）。

---

## 第 5 部分 —— 架构（让它扛得住生长）

CLI 烂成上帝文件时，agent *和* 维护者都付代价。下面的结构让 5 个命令的玩具和 100 个命令的怪兽
一样好导航。

- **单向分层：** `bin`（入口 shim）→ `cli`（只解析/路由）→ `usecases`（编排）→ `domain`
  （纯、无框架的核心）→ `infra`（IO 适配）。依赖只指向 *下*；用环检测（如 `madge --circular`）
  在 CI 守住。永远无环。
- **薄命令壳。** 一个命令 handler 解析 flag、校验、*委派*、格式化结果。业务逻辑住在
  usecases/domain，绝不内联进 handler。（gh、kubectl、Cobra 官方指南都这么做。）
- **一动词一文件。** `commands/<verb>.ts`，共享 helper 进 `lib/`。这是主流（oclif、Cobra）且有
  道理：每个命令都好找、互相隔离。
- **deep-import，别用 barrel。** *内部* 结构就 import 具体模块（`commands/foo.js`），别用
  re-export 的 `index` 门面 —— barrel 招循环依赖、糊 go-to-definition。（gh/kubectl 直接 import
  子包。）barrel 只在你刻意要一个稳定的 *对外* API 面时才值。
- **用 Command/registry 取代大 if/else 派发器。** 一个 1000 行的
  `if (op==='a'){…} else if (op==='b'){…}` 是经典上帝函数。改成 **dispatch map**：
  `Map<opKind, handler>`、一个 op（或一组）一文件、一个查表即调的小 driver。（这个重构有名字：
  *Replace Conditional Dispatcher with Command*。）参考里有具体写法。
- **op 构造单一真相源。** 如果一个 CLI handler 和一个 script translator 都在造「同一个 op」，
  它们 *一定* 会漂移 —— 一边丢了互斥校验、一边字段默认值不同。抽一个共享 `buildXTemplate()`
  两边都调。（见坑 #4 —— 这是这里最贵的一课。）

---

## 第 6 部分 —— 选框架

为该语言挑 **最主流、最低开销** 的库，除非你真需要更重框架的特性，否则别上。

> **对 agent-facing CLI，startup latency 是真实成本。** agent *每次调用* 都 spawn 一次二进制，
> 常常几十次。一个在 15–25 ms 基线上多加 50–120 ms 冷启动（插件发现、manifest 加载）的框架，
> 是在 *每一次* 调用上交这笔税。对交互式人类工具它隐形；对 agent 循环它是可测量的拖累。挑精瘦的。

- **TypeScript** → `commander`（事实标准，每周数亿下载）。除非你需要插件生态，否则别上
  `oclif` —— 它约 5–7× 的启动开销。想要框架的结构又不要重量，`citty`（UnJS）是精瘦的现代选择。
  完整细节 + harness：[`references/typescript.md`](references/typescript.md)。
- **Go** → `cobra`（kubectl/gh/docker 标准）。`references/go.md`。
- **Rust** → `clap`（derive API）。`references/rust.md`。
- **Python** → `typer`（或 `click`）。`references/python.md`。

上面的 *原则* 四种语言完全一致；只有 harness 不同。

---

## 一个完整示例：「跨整份文档清掉一个标记」

一个真实、反复出现的 agent 任务：「我很多文件开头有 `[draft]` 之类的标记，全帮我去掉。」它把
整个 skill 浓缩成一例。

**坏路径**（朴素 CLI 逼你走的）：grep 找出来，然后对每个命中循环跑单点字面量 replace，每条的
括号内容都不同、要 agent 读回来原样传 → N 条命令、N 份 context dump、没有预演。

**好路径**（本 skill 搭出来的）：
```bash
tool replace --in '*' --regex --from '^\s*\[[^\]]*\]\s*' --to ''            # dry-run："matched 312, would change 88"
tool replace --in '*' --regex --from '^\s*\[[^\]]*\]\s*' --to '' --apply    # 落地
```
一条命令。全本范围显式（`--in '*'`）。regex 是 opt-in。默认 dry-run 带 blast radius、只在
`--apply` 时落盘、幂等（重跑 → "0 changed"）、吐一个干净的计数 —— 不是 88 份回显。*那* 才是
agent-facing CLI。

---

## 坑（每一条都付过一个真 bug）

1. **引号陷阱。** verb-script 的 shell 式 tokenizer 若 *剥掉* 引号，就再也分不清引号值 `"--x"`
   和真 flag `--x`。要逐 token 记录「是否有字符来自引号/转义」，并把引号 token 当作永不是 flag。
   否则像 `--content "-- Act I --"` 这样的内容会被当成 flag 而被拒。
2. **布尔 `--flag=false` 陷阱。** `--force=false` 必须变成 *false*，而不是真值字符串 `"false"`。
   把 `=` 形式的布尔 flag 解析成布尔。
3. **dry-run 校验错了状态。** dry-run 改的是 *内存* 副本、不保存。若你的校验回去读 *磁盘* 文件，
   它校验的是改之前的状态、报一个误导的 "would pass"。要校验内存里的将成之态。
4. **一个 op 两套真相源。** CLI verb handler 和 verb-script translator 都造「同一个 op」一定会
   分叉 —— 丢互斥校验、字段默认不同（如单 speaker 的 delivery 默认值不一致）。抽一个共享 template
   builder 两边都调。
5. **全局 flag 预扫吞掉了某 flag 的值。** 若你在命令运行前扫 *整个* argv 来预解析 `--format` /
   `--json`，那么 `--from --format` 会让扫描偷走 `--format`。只在下一个 token 不是 flag 时才把它
   当值，并支持 `=` 形式。
6. **假装结构性批量。** 让 selector 驱动 insert/delete/move 会随下标漂移而静默指错。拒绝它，走
   事务 surface。

---

## 如何把它装成可用 skill

本项目在 `~/Projects/skills/skills/agent-cli-design/`。要让它成为活跃 skill，symlink 或拷进你的
skills 目录：
`ln -s ~/Projects/skills/skills/agent-cli-design ~/.claude/skills/agent-cli-design`
（或用 skill-creator 的 `package_skill.py` 打包）。
