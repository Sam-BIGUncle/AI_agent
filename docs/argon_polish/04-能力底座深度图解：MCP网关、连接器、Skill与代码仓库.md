# 04 能力底座深度图解：MCP网关、连接器、Skill与代码仓库

在 [03-核心机制深度图解](file:///Users/xuye/360/oversea/oversea_AI/docs/argon_polish/03-核心机制深度图解：运行时、上下文与多模态.md) 中，我们拆解了运行时引擎是如何运转的。但一个空转的引擎解决不了任何业务问题——**决定 Agent 到底能不能查对线上问题的，是它手里的“工具（Tools）”、“排查说明书（Skills）”和“代码目录（Code Workspace）”**。

本文档将原草稿中《04-连接器接入规范》《05-MCP网关与Skill管理》《08-代码仓库管理》三篇专题整合，用图解法讲清能力层的四大支柱：
1. **MCP 网关五层架构**：如何保证网关核心“零业务代码”，同时守住权限、脱敏与审计大门？
2. **提示词与 Skill 管理法则**：哪句话该写在哪里？如何按需加载才不会把上下文撑爆？
3. **连接器（Connector）双模式演进**：如何用一套目录结构同时支撑一期的“平台主动接入”和二期的“业务自助接入”？
4. **代码仓库三层管理机制**：如何在不给 Agent 任何 Git 权限的前提下，让它读到与线上运行版本完全一致的多仓库代码？

---

## 1. 为什么工具和 Skill 多了会乱？三条治乱铁律

很多团队做 Agent，刚开始三五个工具跑得很顺；过了两个月加到 40 个工具、20 篇提示词，系统突然开始**选错工具、引用已废弃接口、网页端和本地插件表现不一致**。
为了从第一天起杜绝这种混乱，平台确立了**“单一事实来源（Single Source of Truth）”**架构：

```mermaid
flowchart TB
  subgraph REPO["📂 1. 唯一源文件：连接器目录 (connectors/)"]
    C_PLAT["connectors/_platform/<br/>通用日志 · 监控 · 发布"]
    C_MEM["connectors/membership/<br/>用户 · 会员 · 权益"]
    C_PAY["connectors/payment/<br/>(二期新业务接入)"]
  end

  subgraph CI["⚙️ 2. 唯一变更通道：发布流水线 (Release Pipeline)"]
    CHECK["规范与安全检查<br/>命名空间 · 必填env · 依赖校验 · 防泄题"] --> EVAL["自动运行评估门禁<br/>成功率不降 · 越权拦截100%"]
  end

  SNAP["📸 3. 注册表快照 (Registry Snapshot v1.0.x)<br/>带版本号的不可变清单：工具 · Skill · 知识 · 权限映射"]

  GW["🛡️ MCP 网关实例"]
  RT["🐍 Agent 运行时"]
  PLUG["🔌 本地 Claude Code 插件包"]

  C_PLAT & C_MEM & C_PAY --> CHECK
  EVAL -- "达标生成" --> SNAP
  SNAP ==>|"加载工具路由与脱敏规则"| GW
  SNAP ==>|"挂载对应版本 Skill 与知识"| RT
  SNAP ==>|"打包分发"| PLUG
```

> [!IMPORTANT]
> **治乱的三条铁律（从第一天起遵守）**：
> 1. **一切资产必归属于连接器**：不存在“游离”的工具或 Skill。每个连接器（如 `membership`）都有明确的负责人（Owner）。
> 2. **注册表快照是唯一来源**：MCP 网关暴露什么工具、运行时加载什么 Skill、本地插件打包什么配置，全部由同一份带版本号的注册表产出。
> 3. **严禁提供“后台直接改提示词/改工具”的散装入口**：所有修改必须改仓库里的文件，走评估验证后发布。出了问题一秒钟就能切回上一个快照版本。

---

## 2. MCP 网关设计：五层管道与工具设计精髓

### 2.1 网关内部五层架构（“零业务代码”原则）

MCP 网关用 Go 语言实现，对外暴露一个统一的 MCP 地址。其内部严格划分为五层，任何一次工具调用都像过安检一样依次穿过这五层：

```mermaid
flowchart TB
  REQ(["📥 收到运行时 MCP 工具调用请求<br/>Header: X-Agent-Identity-Token + X-Agent-Turn-ID"])

  subgraph L1["第 1 层：接入层 (Ingress Layer)"]
    I1["解析 MCP Streamable HTTP 协议<br/>验证平台签名的身份令牌，提取发起人 user_id / role"]
  end

  subgraph L2["第 2 层：策略层 (Policy Layer)"]
    P1["1. 授权裁决：发起人是否有权访问目标 app + env？<br/>2. 参数强校验：env 是否必填？时间范围是否合规？<br/>3. 频控与配额：按用户 + 工具限流，防死循环打挂日志库"]
  end

  subgraph L3["第 3 层：路由层 (Router Layer)"]
    R1["查注册表：将 tool_name 映射到所属连接器与适配器<br/>（🚫 严禁出现 if service == 'member' 等硬编码分支！）"]
  end

  subgraph L4["第 4 层：适配层 (Adapter Layer) —— 唯一接触数据源的地方"]
    direction LR
    A1["内置只读适配器 (一期)<br/>ClickHouse / MySQL从库 / Redis / 发布"]
    A2["声明式 HTTP 适配器 (二期)<br/>由 YAML 配置直接转 HTTP 请求"]
    A3["外部 MCP Server 代理 (二期)<br/>转发并注入签名身份令牌"]
    A4["评估回放适配器 (Eval)<br/>命中快照直接返回冻结数据"]
  end

  subgraph L5["第 5 层：出口层 (Egress Layer)"]
    E1["1. 大小与行数硬截断（附 total_count 与缩小范围提示）<br/>2. 按连接器规则脱敏（手机号/邮箱掩码，密码/Token直接丢弃）<br/>3. 写入 AUDIT_LOGS 审计表（谁、哪个任务、调了什么、耗时）"]
  end

  RESP(["📤 返回标准三态结果 (ok / empty / error)"])

  REQ --> L1 --> L2 --> L3 --> L4 --> L5 --> RESP
```

**如何检验你的网关代码写得合不合格？**
看一条标准：**当你要新增一个工具时，第 1 层（接入）、第 2 层（策略）、第 3 层（路由）、第 5 层（出口）的 Go 代码是否一行都不用改？** 如果只需要在第 4 层（适配层）加一个函数并在配置文件里注册，说明架构是干净的。

### 2.2 一期四大类工具设计与安全护栏

在一期（M1~M2），我们围绕三个基础服务提供以下四类只读工具：

| 工具类别 | 一期实现形态 | 强制安全护栏（由网关代码强制，不靠模型自觉） | 二期演进方向 |
| :--- | :--- | :--- | :--- |
| **1. 日志查询 (`ClickHouse`)** | `logs_search` / `logs_count_by_error`（自用阶段可支持受控只读 SQL） | • 使用专用只读账号<br/>• 强制校验查询包含时间范围（如跨度 ≤ 24h）<br/>• 强制附加 `LIMIT`（默认 50 行）与最大执行时间（如 5s） | 全部收敛为语义化参数工具 + 按 `trace_id` 聚合全链路日志 |
| **2. 业务库查询 (`MySQL 从库`)** | `mysql_readonly_query` 及高频汇总工具 `member_diagnose_user` | • **物理隔离：只连只读从库（Read Replica）**<br/>• AST/语法级拦截：仅允许 `SELECT`，强制带索引条件或 `LIMIT`<br/>• 敏感列在数据库视图层或网关出口层直接脱敏 | 将高频排查 SQL 全部固化为像 `member_diagnose_user` 一样的链路级工具 |
| **3. 缓存读取 (`Redis`)** | `redis_get_user_cache` | • **从第一天起就严禁开放任意 Redis 命令！**<br/>• **严禁 `KEYS *` 或全量 `SCAN`**<br/>• 只提供按业务语义封装的读取（传入 `user_id`，内部拼好固定 Key 前缀去 `GET/HGETALL`） | 按连接器声明的 Key 模板自动生成只读缓存工具 |
| **4. 发布记录 (`CD 系统`)** | `deploy_list_recent` | • 只读 Token 访问发布系统 API<br/>• 必填 `service` 与 `env`，返回最近发布时间、版本号、Commit 与变更人 | 增加 `deploy_get_current_version`（含灰度比例）与版本间 `deploy_diff` |

### 2.3 工具命名空间规范与动态加载（防上下文膨胀）

- **命名格式**：`<连接器前缀>_<受限动词>_<对象>`。
- **受限动词表（只允许这 6 个动词）**：
  - `get`：按唯一标识精确取一条（如 `order_get`）
  - `list`：按条件列多条，带分页（如 `deploy_list_recent`）
  - `search`：按关键字、时间段模糊搜索（如 `logs_search`）
  - `count`：按维度统计数量或分布，先看分布再查明细（如 `logs_count_by_error`）
  - `diagnose`：面向一个业务问题，一次性汇总多段状态（如 `member_diagnose_user`）
  - `diff`：对比两个版本或两个时间段（如 `deploy_diff`）
- **工具数量增长后的按需过滤（二期）**：
  一期工具总数少于 15 个，直接全量放入上下文；二期当接入多个业务、工具破百时，网关只向当前会话返回 **“提问人有权限 ∩（当前目标应用 + 依赖应用 + 平台通用）”** 的工具子集，并额外提供一个始终可见的 `tools_search` 工具供跨业务按需申请加载。

---

## 3. 提示词工程与 Skill 管理：内容归位与三层按需加载

> **为什么 Agent 答得好不好，80% 取决于你写的文字（提示词与 Skill）？**
> 因为循环和网关只是管道，真正教会模型“你们公司的会员业务长什么样、遇到未到账先查哪张表、看到什么状态才算查清”的，全靠连接器里的文字。

### 3.1 黄金法则：五类信息的“内容归位表”（每条信息只写在一处！）

写任何说明文字前，先对照下表判断它该放哪，**严禁把同一段话在工具说明、Skill 和常驻说明里到处复制粘贴**：

```mermaid
flowchart LR
  INFO["✍️ 你想告诉模型的一条信息"] --> Q1{"每个任务、每个业务<br/>都必须遵守吗？"}
  Q1 -- "是" --> P1["1️⃣ 常驻说明 (CLAUDE.md)<br/>控制在 1~2 页内！<br/>放角色、只读铁律、双层输出格式、全局服务清单"]

  Q1 -- "否" --> Q2{"每次提问都会动态变化吗？<br/>(时间/提问人/线上版本)"}
  Q2 -- "是" --> P2["2️⃣ 任务/轮次注入信息<br/>由 Agent 服务用代码查好拼入<br/>绝不让模型去猜或问用户"]

  Q2 -- "否" --> Q3{"只跟某一个具体工具<br/>怎么调有关吗？"}
  Q3 -- "是" --> P3["3️⃣ 工具说明 (Tool Description)<br/>写明用途、何时调用、参数格式<br/>绝不在此写跨工具排查流程"]

  Q3 -- "否" --> Q4{"是某类问题的排查思路、<br/>多工具配合与判断标准吗？"}
  Q4 -- "是" --> P4["4️⃣ 技能文件 (SKILL.md)<br/>写服务地图、排查分支、停止条件<br/>按需读取，平时只露 1 行描述"]

  Q4 -- "否 (长篇细节)" --> P5["5️⃣ 附属参考 / 知识文档<br/>(references/ 或 knowledge/)<br/>放表结构字典、状态码枚举、接入文档"]
```

### 3.2 Skill 的纵向三层划分与横向三阶段按需加载

#### （1）纵向分三层：避免两个 Skill 抢同一道题
| 层级 | 放在哪里 | 回答什么问题 | 典型例子 |
| :--- | :--- | :--- | :--- |
| **平台层 Skill** | `connectors/_platform/skills/` | 跨所有业务通用的排障方法论 | 《如何判断线上异常是否由近期发布或配置变更引起》 |
| **领域层 Skill** | `connectors/membership/skills/overview/` | 整个业务领域的全局架构与服务地图 | 《用户、会员、权益三服务调用链路、消息关系与核心表总览》 |
| **场景层 Skill** | `connectors/membership/skills/<具体场景>/` | **某一类具体高频问题**的排查指南（**铁律：一类问题只准有一个场景 Skill**） | 《付费后会员或权益未到账排查指南》 |

#### （2）横向分三阶段按需加载：花最少的 Token 办最大的事

```mermaid
sequenceDiagram
  autonumber
  participant RT as 🐍 运行时 (Context Manager)
  participant LLM as 🧠 大模型
  participant FS as 📂 连接器目录 (只读挂载)

  Note over RT,FS: 阶段 1：会话启动时（仅消耗极少 Token，且 100% 命中前缀缓存）
  RT->>FS: 扫描 skills/*/SKILL.md 的 YAML Frontmatter (头部)
  RT->>LLM: 系统提示中仅放入【Skill 名称 + 1~2 行适用场景描述 + 文件路径】

  Note over RT,FS: 阶段 2：收到具体问题后（模型自主判断相关性）
  LLM->>LLM: 用户问“付费后权益没加上”，匹配到 member_entitlement_missing 的描述！
  LLM->>RT: 调用 read_file(".../skills/entitlement-missing/SKILL.md")
  RT->>FS: 读取 SKILL.md 正文（排查思路、4段链路检查点、停止条件）
  FS-->>LLM: 返回 SKILL.md 正文

  Note over RT,FS: 阶段 3：需要核对具体状态码或表字段含义时（深水区按需读取）
  LLM->>RT: 正文提示“详细状态码见 references/schema-and-codes.md”，调用 read_file 读取
  RT->>FS: 读取附属字典文件
  FS-->>LLM: 返回表结构与状态码明细
```

> [!TIP]
> **如何衡量你的 Skill 写得好不好？看两个指标**：
> 1. **Skill 命中率（Hit Rate）**：在应该用该 Skill 的评估题中，模型实际去读了 `SKILL.md` 正文的比例。**如果命中率低，说明头部的 `description`（适用场景描述）写得太含糊！**
> 2. **加载后成功率（Post-Load Success Rate）**：模型读了 `SKILL.md` 正文后，最终查出正确根因的比例。**如果读了还是答错，说明正文里的排查思路、判断标准或引用的工具写得有问题！**

---

## 4. 连接器（Connector）双模式演进：从“平台包办”到“业务自助”

在一期，我们只覆盖用户、会员、权益三个基础服务；到了二期，支付、商品、营销等十几个业务团队都想接入。怎么做到**一期快、二期不返工**？

答案是：**从第一天起，平台自己写的三个基础服务接入，也严格按 `connectors/membership/` 标准连接器目录来写！**

```mermaid
flowchart LR
  subgraph MODE1["模式 A：平台主动接入（一期主力 + 长期通用系统）"]
    M1_WHO["👨‍💻 平台团队编写并维护<br/>• _platform (日志/监控/发布)<br/>• membership (试点三个基础服务)"]
  end

  subgraph MODE2["模式 B：业务自助接入（二期从第 2 个业务开始）"]
    M2_WHO["👩‍💻 各业务团队按模板编写<br/>• 提交 connector.yaml + 声明式工具 + Skill + 评估案例"]
  end

  subgraph MODE3["模式 C：直接复用已有 MCP Server"]
    M3_WHO["🔌 开源或内部现成 MCP 服务<br/>• 补齐清单、Skill 与评估案例"]
  end

  SPEC["📐 同一份连接器目录规范<br/>(清单 + 工具 + Skill + 知识 + 案例)"]
  GATE["🚧 同一套自动化准入门槛<br/>(格式校验 + 安全检查 + 评估集跑分达标)"]
  HANDOVER["🤝 无感移交机制：<br/>一期由平台代写的 membership 连接器，<br/>后续只需修改 connector.yaml 中的 owner 即可平滑移交业务方维护！"]

  MODE1 & MODE2 & MODE3 --> SPEC --> GATE --> HANDOVER
```

---

## 5. 代码仓库管理：让 Agent 安全、精准地读懂线上代码

“某某业务逻辑是什么”答案在代码里；“为什么报这个错”也需要顺着日志里的文件名和行号去看代码。
但代码管理有三个棘手问题：
1. **安全问题**：绝不能把 Git 私钥给 Agent，也不能让 Agent 在目录里随便跑 `git checkout` 把别的任务搞乱。
2. **版本不一致问题**：Git 主干（`main`）上的最新代码可能刚提交还没发版，线上实际跑的是昨天发布的 `v1.9.0`。看错代码版本就会给出荒谬的结论。
3. **跨仓库与公共库问题**：会员业务跨 3 个仓库，还依赖一个公共库，怎么放到一起？

### 5.1 三层代码目录架构（镜像层 → 不可变版本层 → 任务工作区）

我们设计了一套极其轻巧的**三层目录结构**（一期先实现简化的脚本拉取 + 任务目录组装，二期接 CD 系统按线上 Commit 精确检出）：

```mermaid
flowchart TB
  GIT_REMOTE[("🦊 远程 Git 仓库<br/>(GitLab / GitHub)")]
  CD_SYS[("🚀 发布系统 (CD)<br/>查询某时刻线上运行的 Commit")]

  subgraph PLATFORM_ZONE["🛡️ 平台受信任区 (repo-sync 服务 / 脚本 —— 持有只读 Deploy Key)"]
    direction TB
    L1_MIRROR["第 1 层：Git 裸镜像层 (Bare Mirrors)<br/>`/data/git-mirrors/<repo>.git`<br/>• 仅含 git 对象数据，不检出工作文件<br/>• 通过 Webhook / 定时 / 按需 git fetch 持续同步"]

    L2_VERSION["第 2 层：不可变版本缓存层 (Immutable Version Cache)<br/>`/data/repo-versions/<repo>@<commit_sha>/`<br/>• 按 commit 命名，git worktree 检出后设为**永久只读**<br/>• 同一个 commit 内容永不改变，可被成百上千个并发任务安全共享！"]

    L3_WORKSPACE["第 3 层：单任务代码工作区 (Per-Task Workspace)<br/>`/workspace/task-<turn_id>/code/`<br/>• **零拷贝秒级创建**：仅创建指向第 2 层对应版本目录的只读符号链接 (Symlink) / 挂载<br/>• 自动生成 `INDEX.md`（写明每个目录对应哪个服务、哪个版本、是否与线上一致）"]

    L1_MIRROR -->|"git worktree add (按需检出)"| L2_VERSION
    CD_SYS -. "决定检出哪个 commit<br/>(一期默认主干或手动指定)" .-> L2_VERSION
    L2_VERSION -->|"创建只读链接 + 生成 INDEX.md"| L3_WORKSPACE
  end

  subgraph ISOLATED_ZONE["🔒 运行时隔离区 (Agent 进程 —— 零 Git 凭据，无网络)"]
    AGENT["🐍 Agent 运行时<br/>仅通过内置 `read_file` / `grep` / `glob` 读取 `/workspace/task-<turn_id>/code/`"]
  end

  GIT_REMOTE -->|"SSH Deploy Key (只读)"| L1_MIRROR
  L3_WORKSPACE ==>|"容器只读挂载 (ro)"| AGENT
```

### 5.2 每个业务仓库必备的“导游图”：`docs/agent-overview.md`

如果把一个几万行代码的陌生仓库直接扔给 Agent，它得调十几次 `glob` 和 `grep` 才能摸清门道，既慢又费钱。
因此，我们在**每个业务代码仓库内部**放一份跟代码一起维护的 `docs/agent-overview.md`（约 1~2 页）：

| 必填章节 | 写清楚什么内容 | 为什么放在业务仓库里而不是平台上？ |
| :--- | :--- | :--- |
| **1. 服务定位与入口** | 这个仓库对应线上哪个服务？HTTP 路由入口、RPC 入口、定时任务入口分别在哪个文件？ | 代码重构或新增接口时，研发在同一个 PR 里顺手更新说明，**保证说明与对应 Commit 的代码永远 100% 同步，绝不脱节** |
| **2. 核心目录结构** | `internal/handler`、`internal/service`、`internal/dao` 等关键目录各管什么？ | 让 Agent 一步定位到目标模块，少走弯路 |
| **3. 对外接口与消息消费** | 暴露了哪些核心接口？消费了哪些 MQ Topic（消费者代码在哪个文件）？ | 排查“上游发了消息但本服务没处理”时直接命中代码位置 |
| **4. 下游依赖与发出的消息** | 调用了哪些下游服务？往外发了哪些 MQ 消息？配置开关在哪读？ | 支撑跨服务链路顺藤摸瓜 |

### 5.3 只读部署密钥（Deploy Key）安全配置 4 步法

在一期准备代码目录（学习单元 10）时，**绝对不要把你个人的 SSH 私钥（`~/.ssh/id_rsa`）拷到服务器上**（个人私钥有写权限且人员变动会失效）。标准做法只需 4 步：

```bash
# 1. 在平台受信任区生成专用的只读 Ed25519 密钥对（无密码）
ssh-keygen -t ed25519 -C "oversea-agent-readonly" -f /etc/agent-platform/keys/membership_deploy_key -N ""

# 2. 将生成的公钥 (.pub) 内容交给仓库管理员，在 GitLab/GitHub 仓库设置中添加为【Deploy Key（不勾选写权限）】
cat /etc/agent-platform/keys/membership_deploy_key.pub

# 3. 固定 Git 服务器 Host Fingerprint 到 known_hosts（严禁使用 StrictHostKeyChecking=no）
ssh-keyscan -t ed25519 git.example.com >> /etc/agent-platform/keys/known_hosts

# 4. 验证只读拉取权限（私钥仅平台区 repo-sync 脚本可读，运行时容器绝不挂载此目录）
GIT_SSH_COMMAND='ssh -i /etc/agent-platform/keys/membership_deploy_key -o IdentitiesOnly=yes -o UserKnownHostsFile=/etc/agent-platform/keys/known_hosts' \
  git ls-remote git@git.example.com:oversea/member-service.git
```

---

搭建好了运行引擎（03）与能力底座（04），我们如何科学地衡量它聪不聪明、安不安全，并让它每周持续变强？请继续阅读：**[05-评估驱动开发（EDD）与安全防护体系.md](file:///Users/xuye/360/oversea/oversea_AI/docs/argon_polish/05-评估驱动开发（EDD）与安全防护体系.md)**。
