# Hermes Agent 记忆功能：实现与对比

分析日期：2026-10-04（Asia/Shanghai）  
代码基线：`eb7e8620324b32424c06218f6a28094df2e921f8`  
仓库：[NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent/tree/eb7e8620324b32424c06218f6a28094df2e921f8)  
原理图：[打开 Archify 交互图](hermes-memory.html) · [图的 JSON 源文件](hermes-memory.architecture.json)

## 1. 核心结论

Hermes 的记忆是模型外部的持久化与上下文编排机制，默认路径不需要 embedding、向量数据库或模型微调。它把信息分配到不同载体：**少量跨任务事实进入长期笔记；历史细节留在会话数据库；可复用做法进入 Skills；更丰富的自动抽取与语义检索由可选 provider 提供。**

最值得关注的是以下组合：

1. **长期笔记既常驻又有界。** `MEMORY.md` 与 `USER.md` 启动时注入 system prompt，默认字符预算分别为 2200 和 1375。
2. **持久化状态与提示词快照分开。** 新记忆可以立即落盘，当前会话的 system prompt 快照保持冻结；新会话或允许刷新提示词的压缩边界再读取更新。
3. **动态召回不重写历史前缀。** 外部检索结果追加到当前回合的模型输入，并保存 `api_content`，后续请求重放相同字节。
4. **“记住事实”和“学会做事”分别处理。** 任务流程与该类任务的偏好、纠错主要写入对应 Skills，避免把所有经验堆进常驻笔记。

这些是源码显示的设计特点，不代表已经证明它比其他 agent 的召回率或任务成功率更高。本文没有进行效果基准测试。[内置实现][s-store]、[输入重放][s-wire]、[复盘策略][s-lessons]

## 2. 分层与存储

下表是对实现的分析分类，并非代码中的统一“记忆类型”枚举。

| 层次 | 存什么 | 载体与范围 | 何时进入模型上下文 |
|---|---|---|---|
| 当前会话工作上下文 | 正在进行的对话、工具调用、任务状态 | Agent 的消息列表；SessionDB 支持恢复 | 当前回合消息历史；过长时由 ContextEngine 压缩或选择 |
| 精简长期事实 | 用户身份、跨任务偏好、稳定环境与约定 | `$HERMES_HOME/memories/MEMORY.md`、`USER.md` | 启动加载的 system prompt 快照 |
| 历史会话记忆 | 以前具体说过、做过什么 | `$HERMES_HOME/state.db`，SQLite 消息与检索索引 | 模型按需调用 `session_search` |
| 程序性经验 | 某类任务的操作流程、有效方法、纠错 | Skills 的 `SKILL.md` 与支持文件 | 默认提供技能索引，再由 `skill_view` 加载相关内容；配置可预加载技能 |
| 可选增强层 | 更多事实、用户建模、结构化或语义检索 | `memory.provider` 指定的插件及其后端 | provider 的静态提示块、每回合 prefetch，或显式工具调用 |

`HERMES_HOME` 在这里表示当前 profile 解析出的 home，不是固定的默认用户目录。内置记忆的路径在调用时通过 `get_hermes_home()` 解析。[路径解析][s-path]、[初始化][s-init]、[技能索引][s-skills-index]

## 3. 内置长期记忆如何实现

### 3.1 启动读取与提示词注入

生产链路是：

```text
agent/agent_init.py::_init_memory
  → 读取 memory 配置及两份 store 的开关
  → 构建 MemoryStore
  → MemoryStore.load_from_disk()
      → 读取 MEMORY.md / USER.md
      → 按完整分隔符 "\n§\n" 拆成条目，去重
      → 扫描威胁模式，生成 _system_prompt_snapshot
  → agent/system_prompt.py::_memory_parts
      → format_for_system_prompt("memory" / "user")
  → 构建并缓存会话的 system prompt
```

每个笔记块包含名称、使用比例与字符计数。多行内容也可以作为一个条目；裸 `§` 字符不等于完整分隔符。预算按字符计算，包含条目分隔符的长度，不应把字符数换算为所有语言、所有模型都相同的 token 数。

加载时检测到威胁模式的条目，在提示词快照中替换为 `[BLOCKED: …]`，原始条目仍保留在 live entries 中供查看与移除。这是模式扫描，不是对记忆真实性或所有 prompt injection 的完整证明。[加载与格式][s-load]、[提示词组装][s-prompt]

### 3.2 模型通过 `memory` 工具写入

工具支持 `add`、`replace`、`remove`，目标为 `memory` 或 `user`；也支持一次提交 `operations` 批量变更。没有单独的 `read` 动作，常驻内容从启动快照获得。

```python
# 示例：模型工具调用，不是可以直接执行的独立 Python 程序
memory(target="user", action="add", content="用户希望跨任务的解释使用中文。")

memory(target="memory", operations=[
    {"action": "replace", "old_text": "旧环境约定", "content": "完整的新环境约定"},
    {"action": "add", "content": "新的、跨任务稳定事实"},
])
```

`old_text` 是条目定位条件：完整条目精确匹配优先，否则使用唯一子串匹配。`replace` 替换的是**整个条目**，不能把它理解为字符串局部补丁。

实际写入采用“加文件锁 → 重读磁盘 → 验证与修改 → 原子写文件”的路径。批量操作检查最终总长度，全部通过后一次提交；任一操作不合法则不写入。容量不足时返回错误和整理提示，由模型合并、缩短或移除条目，存储层不自动调用 LLM 压缩，也不默默淘汰旧条目。[工具契约][s-schema]、[写入事务][s-mutate]、[批量语义][s-batch]

此外，写路径会拒绝读取失败后覆盖现有文件；部分编辑路径遇到不能往返解析的外部漂移会保存备份并拒绝覆盖。连续整理失败有每回合上限，避免记忆维护循环耗尽用户任务的预算。原子写与文件锁解决存储竞争，不会使两个不同 Agent 在语义上自动协调共享笔记。[存储实现][s-store]

### 3.3 为什么“保存了”却没出现在当前 system prompt

`MemoryStore` 同时维护两个视图：

| 视图 | 用途 | 工具写入后 |
|---|---|---|
| `memory_entries` / `user_entries` 与磁盘文件 | 当前可修改、可持久化的条目 | 更新 |
| `_system_prompt_snapshot` | 当前会话提示词使用的已加载版本 | 保持原样 |

`format_for_system_prompt()` 返回冻结快照。写入成功后模型可以根据本回合上下文继续使用刚讨论的事实，但不能据此认为缓存的 system prompt 已经同步更新。

新会话会读取更新；上下文压缩后 `invalidate_system_prompt()` 清除缓存并重新加载文件，让重建提示词吸收更新。压缩本来就改变上下文前缀，因此是项目允许的刷新边界。[冻结快照][s-snapshot]、[压缩边界刷新][s-invalidate]

恢复同一个历史会话与开启新会话也不能混为一谈：重启进程未必意味着逻辑会话结束，尤其是 gateway 的连续聊天。判断是否记住，应检查工具是否真正成功提交、是否待审批、是否同一 profile，以及实际会话边界，不能只看模型说了“我记住了”。

## 4. 历史会话记忆：`session_search`

消息持久化路径把新增消息批量提交给 `SessionDB.append_messages_batch()`。历史数据库与精简笔记互相补充：笔记不必保存完整任务日志，细节可以回到历史原文检索。[消息落盘][s-persist]

当前 `session_search` 根据参数自动选择四种形状：

| 输入 | 行为 |
|---|---|
| `query` | DISCOVERY：关键词检索，按会话 lineage 去重；默认只完整展开最相关结果 |
| `session_id` + `around_message_id` | SCROLL：读取命中消息附近的窗口 |
| 只有 `session_id` | READ：读取会话或其头尾部分，控制消息体大小 |
| 没有查询条件 | BROWSE：浏览近期会话 |

检索使用 SQLite FTS5，默认 relevance 排序基于 BM25；也支持时间与角色过滤。当前实现为中文提供 CJK bigram 扩展路径，在条件不满足时使用 trigram 或 LIKE 回退；英文零命中也存在部分宽松重试。因此不能把“FTS5 的默认 AND 语义”当作所有查询的最终行为。[工具实现][s-search]、[检索路由][s-search-routing]

这个工具**不额外调用 LLM 生成摘要**，返回的主体是实际数据库消息、片段与元数据。它也不是 embedding 语义检索。当前策略还会：

- 隐藏 `subagent`、`tool`、`kanban` 等辅助来源；将 `cron` 命中降到交互会话之后。
- 按 lineage 去重，避免压缩延续导致重复结果。
- 避免重复检索还在当前上下文里的消息，但允许已离开 live context 的压缩归档与旧边界会话重新被发现。
- 控制展开量，提供锚点供后续读取，而不是一次返回全部历史。

它通常是模型主动调用的工具；并不意味着每个用户问题都会自动搜索全部历史。[发现与去重][s-discovery]

## 5. 可选外部记忆：统一接口，插件决定算法

### 5.1 默认路径与扩展路径的区别

默认配置为 `memory.provider: ""`，内置笔记可独立工作。选中外部 provider 后，`_init_memory()` 加载插件、检查 `is_available()`、加入 `MemoryManager`，并调用 `initialize_all()`。

这里有一个容易被类注释误导的细节：虽然 `MemoryManager` 的说明写着 “Builtin provider … plus … external”，**当前正常初始化里内置 `MemoryStore` 单独绑定在 `agent._memory_store`，并没有先把一个 builtin provider 加入 manager。** 不应据此画出每次内置写入都经过 `MemoryManager` 的错误链路。外部 provider 可以与内置笔记并存，但两份内置 store 也可以独立配置关闭。[真实初始化][s-init]、[开关配置][s-flags]

manager 最多接受一个外部 provider。内置随仓库存在的实现为 `openviking`、`mem0`、`holographic`、`retaindb`、`byterover`；Honcho、Hindsight、Supermemory 等以目录或 catalog 插件方式接入，不能把它们都描述为当前核心树中的实现。

发现顺序是 bundled → 当前 home 的用户插件 → 显式启用的 project 插件 → Python entry points；先出现的同名实现优先。接口是 `MemoryProvider` ABC，关键点包括静态提示块、召回、回合同步、会话边界、压缩前处理和显式工具。[发现机制][s-loader]、[ABC 契约][s-provider]

### 5.2 一个回合的召回与同步

```text
回合开始
  → on_turn_start(当前作者等信息)
  → 非 trivial 输入：prefetch_all(当前问题)
  → provider 返回相关记忆
  → <memory-context> 包装、去重与超大内容处理
  → 添加到本回合用户消息的 API 内容
  → 模型与工具循环
完整回复交付后
  → sync_all(原始用户输入, 最终回复, 可选完整 messages)
  → queue_prefetch_all(为后续回合准备)
```

prefetch 并非对所有后端都“零等待”：manager 对外部调用施加超时，具体插件还可能使用缓存或短等待。比如 Mem0 会先启动后台检索、尝试消费当前 query 的结果，慢后端本回合可以跳过注入，显式搜索工具仍然可用。[回合入口][s-turn-start]、[有界召回][s-prefetch]、[Mem0 路径][s-mem0]

动态召回不重新写 system prompt，而是放进当前用户消息的模型发送副本。字符串消息保留干净的用户内容与 `api_content` sidecar；之后重放使用已发送的 sidecar，避免下一回合重新检索把旧消息字节改掉。多模态消息则走文本 part 的路径。`codex_app_server`、MoA 等专用 API 模式有分支，不能把字符串 sidecar 视为全部传输模式的唯一实现。[上下文拼接][s-compose]、[发送与重放][s-wire]、[回合分支][s-turn-branches]

### 5.3 一致性与边界

manager 用单 worker 后台队列串行执行回合同步与预取，减少主回复等待；中断回合不进行正常 turn sync，避免把未完成输出当作持久事实。`commit_session_boundary_async()` 将旧会话 `on_session_end` 和新会话 `on_session_switch` 放在同一任务中，保证先抽取旧会话、再重绑定，CLI `/new` 有实际调用点。[同步入口][s-sync]、[边界队列][s-boundary]、[CLI 调用][s-cli-boundary]

关闭时队列有界排空，超时会记录放弃的写任务与预取任务。排空的是 manager 的工作队列；provider 可能还有自己的线程、服务端处理或最终一致性，所以不能据此声称整个外部存储系统获得跨进程强一致或绝不丢失写入。

内置 `memory` 工具成功提交的变更，通过 `notify_memory_tool_write()` 通知外部 provider；错误和待审批结果不镜像。通知包含 provenance 和被替换/移除的完整旧条目，但插件是否实现更新、删除、去重，仍由插件决定。例如当前 Holographic 的 `on_memory_write()` 只处理新增；所以这条桥接不能当作双向一致复制协议。[桥接调用][s-bridge]、[镜像条件][s-mirror]、[Holographic 实现][s-holographic]

### 5.4 压缩前记忆与 ContextEngine

`ContextEngine` 负责会话上下文预算、选择与压缩，`MemoryProvider` 负责持久知识和召回，二者是不同扩展接口。

压缩前 `_pre_compress_memory_context()` 调用 provider 的 `on_pre_compress()`，把返回的知识带入摘要提示。常规情况下是 best effort；**如果启用了 checkpoint-required 模式**，需要兼容 checkpoint API v2 的活跃 provider 成功完成检查点，否则拒绝进行这次压缩。不是所有插件都实现了该能力，也不表示默认压缩都会持久化一个外部检查点。[上下文接口][s-context-engine]、[压缩契约][s-precompress]

## 6. 自动学习：后台复盘与 Skills

内置 memory nudge 默认每 10 个用户回合触发一次检查，要求 memory 工具可用且 store 存在。回复结束时如果有完整输出、未中断、未禁用后台复盘，并且记忆或技能阈值触发，才启动复盘。[nudge][s-nudge]、[回复后触发][s-finalizer]

复盘 fork 在同模型路径复用父会话的缓存 system prompt；它共享内置 store，但禁用自身会话持久化、不结束父会话，并跳过外部 provider，避免把复盘指令写入用户历史或外部记忆空间。它使用隔离的消息副本，而不是在前台对话中插入一条“请复盘”的合成用户消息。[fork 隔离][s-review-fork]

复盘策略把跨任务身份事实与任务方法分开。比如“用户需要中文沟通”可以进入通用用户笔记；“用户要求此类架构图采用某种图例、验证流程与输出格式”应进入该类任务的 skill。技能内容按相关性加载，缓解常驻笔记的预算压力。技能复盘也有受保护内容与先读后改约束，不是可以任意重写所有技能。[经验归属][s-lessons]

当前实现还将无人工参与的复盘中的 `replace` / `remove` 暂存为待审批变更，避免后台整理直接删除旧笔记；新增仍受容量、内容检查和通用写审批配置约束。前台和显式复盘的策略有所区别，不能简化为“所有 memory 操作都必须审批”，默认 `memory.write_approval` 为 `false`。[后台破坏性操作约束][s-review-gate]、[默认配置][s-defaults]

## 7. Profile 与用户身份：实际范围

内置笔记的隔离单位是 profile home。同一个 profile 下的多个会话会读取相同两份文件；它们不是天然的逐聊天用户存储。这让个人 Agent 能在 CLI 与不同入口之间共享稳定事实，也要求多用户部署明确选择身份映射与记忆范围。

外部 provider 初始化获得 `hermes_home`、platform、profile 身份以及可用的 gateway 用户/聊天信息；每回合还可收到当次作者信息。最终如何隔离由 provider 实现，例如 Mem0 的配置用户 ID 优先于 gateway ID，而查询按 `user_id` 检索，允许跨 channel 召回。

`session_search` 也存在显式指定另一 profile 并以只读方式打开数据库的路径。因此“默认按 profile 分开”不应被夸大为所有工具都绝对禁止跨 profile 读取的安全边界。[身份初始化][s-identities]、[Mem0 身份][s-mem0-identity]、[显式跨 profile 检索][s-cross-profile]

## 8. 与其他 Agent / 框架相比

对比依据为 2026-10-04 查阅的官方文档。LangGraph 是构建 agent 的框架，其组件可被开发者组合成类似 Hermes 的策略；不能拿框架的裸默认配置当作所有基于它的 agent。

| 对象 | 官方公开机制 | 相对 Hermes 的区别 |
|---|---|---|
| Claude Code | `CLAUDE.md` 持久指令；自动记忆为每仓库的 `MEMORY.md` 索引及主题文件，启动加载索引前 200 行或 25KB | 两者都有文件型记忆与 Skills。Hermes 更突出个人 profile 的两份精简笔记、内置历史检索及可替换 provider；不能声称文件记忆或经验写入 Skills 是 Hermes 独有。[官方说明](https://code.claude.com/docs/en/memory) |
| Letta | 可自编辑、常驻上下文的 memory blocks；归档记忆提供外部检索 | 两者都把少量重要事实常驻并支持工具维护。Letta 把 memory block 作为可共享、可绑定的抽象；Hermes 的本地笔记明确区分 live 状态与冻结快照，外部增强遵循 provider 接口。[Memory blocks](https://docs.letta.com/v1-sdk/memory/memory-blocks)、[Archival memory](https://docs.letta.com/v1-sdk/memory/archival-memory) |
| LangGraph | thread state + checkpointer 保存短期上下文；namespace + key 的 Store 保存跨线程数据，可选语义检索 | Hermes 提供现成的保存、复盘与调用策略；LangGraph 提供可组合持久化原语。开发者仍可在 LangGraph 上实现快照、Skills 或异步抽取，不存在这种能力的框架性缺失。[Memory overview](https://docs.langchain.com/oss/python/concepts/memory)、[Memory guide](https://docs.langchain.com/oss/python/langgraph/add-memory) |

**根据以上代码与公开文档作出的判断：Hermes 的特殊之处主要是工程取舍的组合，而非某个全新记忆算法。** 它把缓存成本作为会话设计约束，把小型常驻事实、可追溯历史和程序性经验分开，再在插件边缘接入复杂记忆服务。

## 9. 优势、代价与不能推断的结论

| 实现取舍 | 实际收益 | 代价或限制 |
|---|---|---|
| 小型文件笔记，启动全量注入 | 默认少依赖、可读可编辑；关键事实不用等检索命中 | 容量小，需要整理；中文字符与 token 成本不等价 |
| 会话快照冻结 | 避免每次笔记更新改变缓存前缀 | 长会话会持有旧快照，持久化成功与提示词可见性存在时间差 |
| FTS5 查真实历史 | 无额外 LLM 摘要调用，可核对原文 | 主要是词面检索，不能保证找出语义等价的不同表述 |
| 回复后后台学习与同步 | 减少前台等待，避免复盘污染主会话 | 额外模型或服务成本；关闭、超时和 provider 内部队列影响完整性 |
| 任务经验进入 Skills | 经验能带步骤、脚本和模板，只在相关任务加载 | 经验质量依赖模型判断、维护规则与技能发现 |
| 外部 provider 插件化 | 可以切换本地结构化存储或远端抽取与召回服务 | 后端的数据范围、隐私、身份映射、时延与一致性各不相同；最多一个活跃外部 provider |

持久记忆不等于模型权重学习；模型说“已保存”不等于工具真正提交；检索到旧内容不等于内容仍然有效；支持学习循环也不等于效果一定持续提升。这些结论需要保存记录、任务评测及时间/冲突处理测试来证明。

## 10. 图、证据与验证范围

图类型为 Archify `architecture`，按组件职责展示调用与 I/O 请求；每个节点带有固定提交的源码引用。虚线是可选 provider 路径，返回内容沿原调用路径回到 Agent。图下方说明卡补充冻结快照、动态召回、会话边界及身份范围。

本次只新增 `design/` 分析与图形产物，没有修改运行时，也未启动真实记忆后端或运行 Python 测试。已有测试仅作为契约交叉核对，例如冻结快照的 `tests/tools/test_memory_tool.py`，以及 FIFO 同步、边界顺序的 `tests/agent/test_memory_async_sync.py`、`test_memory_boundary_commit.py`；本文不把阅读测试算作测试通过。

图已通过 Archify 的 `validate`、`deliver`、严格产物 `check` 和真实 Chrome `browser-check`，无诊断。另生成 1440×900 与 2048×1320 的浅色/深色截图，并人工查看代表性浅色和深色截图；截图捕获本身与人工检查分开记录。

- [最终自动校验摘要](hermes-memory.finalize-summary.json)
- [完整生成回执](hermes-memory.finalize.json)
- [规范与产物校验绑定](hermes-memory.delivery.json)
- [浏览器校验回执](hermes-memory.browser-check.json)
- [截图联系表](visual-check/hermes-memory.visual-check.html)
- [截图检查回执](visual-check/hermes-memory.visual-check.json)

在当前机器、从仓库根目录重新生成的命令为：

```powershell
$archifyRoot = 'C:/Users/leeking/.agents/skills/archify'
node "$archifyRoot/bin/archify.mjs" finalize architecture `
  design/hermes-memory.architecture.json design/hermes-memory.html `
  --repo-root F:/github/hermes-agent --quality showcase --json
```

其他机器需替换 Archify 安装目录与仓库路径。修改 JSON 后若已有浏览器证据，应按 Archify 交付契约使用新的 `--out-dir` 保存复验回执。

### 阅读源码时的版本差异

当前 `website/docs/user-guide/features/memory.md` 的部分示例仍建议保存“已完成任务日志”与“技能技巧”，但当前 `MEMORY_SCHEMA` 明确要求任务记录用 `session_search`、可复用流程用 Skills。`memory-providers.md` 对“后台、不阻塞”的概述也需要结合实际有界等待理解。本文涉及这些细节时以当前生产调用路径与工具契约为准；没有顺手修改已有文档。

### 源码索引（固定到分析基线）

| 入口 / 模块 | 重点符号与职责 |
|---|---|
| [agent/agent_init.py][s-init] | `_init_memory`：加载 store 与选中的 provider |
| [tools/memory_tool_store.py][s-store] | `MemoryStore`：冻结快照、条目预算、文件锁与原子写 |
| [tools/memory_tool.py][s-schema] | `MEMORY_SCHEMA`：模型实际收到的保存规则与参数 |
| [agent/system_prompt.py][s-prompt] | `_memory_parts`、`invalidate_system_prompt`：注入与压缩后刷新 |
| [tools/session_search_tool.py][s-search] | `session_search`、`_discover`：真实历史按需召回 |
| [hermes_state_search.py][s-search-routing] | `_search_messages_impl`、`_search_cjk`：FTS5 与中文回退 |
| [agent/memory_provider.py][s-provider] | `MemoryProvider`：插件生命周期契约 |
| [agent/memory_manager.py][s-prefetch] | `prefetch_all`、`sync_all`、`commit_session_boundary_async`：召回与队列 |
| [agent/turn_context.py][s-wire] | `build_api_messages`：保存并重放 API 内容，保持缓存前缀 |
| [agent/turn_finalizer.py][s-finalizer] | 完整回合同步与复盘触发 |
| [agent/background_review.py][s-review-fork] | 复盘 fork：缓存复用、隔离与经验整理 |
| [agent/conversation_compression.py][s-precompress] | `_pre_compress_memory_context`：压缩前知识与检查点 |

[s-path]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/memory_tool.py#L38-L40
[s-init]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/agent_init.py#L1323-L1402
[s-identities]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/agent_init.py#L1285-L1319
[s-store]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/memory_tool_store.py
[s-load]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/memory_tool_store.py#L133-L164
[s-prompt]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/system_prompt.py#L516-L541
[s-schema]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/memory_tool.py#L327-L399
[s-mutate]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/memory_tool_store.py#L245-L361
[s-batch]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/memory_tool_store.py#L397-L461
[s-snapshot]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/memory_tool_store.py#L463-L466
[s-invalidate]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/system_prompt.py#L812-L830
[s-persist]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/session_persistence.py#L290-L300
[s-search]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/session_search_tool.py
[s-search-routing]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/hermes_state_search.py#L1088-L1218
[s-discovery]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/session_search_tool.py#L354-L431
[s-cross-profile]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/session_search_tool.py#L434-L444
[s-flags]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/memory_tool.py#L266-L284
[s-defaults]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/hermes_cli/config_defaults.py#L1314-L1329
[s-loader]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/plugins/memory/__init__.py#L1-L8
[s-provider]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/memory_provider.py#L84-L206
[s-turn-start]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/turn_context.py#L889-L917
[s-turn-branches]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/turn_context.py#L1168-L1189
[s-compose]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/turn_context.py#L106-L129
[s-wire]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/turn_context.py#L1220-L1289
[s-prefetch]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/memory_manager.py#L447-L501
[s-sync]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/run_agent.py#L923-L954
[s-boundary]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/memory_manager.py#L673-L713
[s-cli-boundary]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/hermes_cli/cli_session_mixin.py#L582-L586
[s-bridge]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/inline_tool_executors.py#L142-L161
[s-mirror]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/memory_manager.py#L798-L858
[s-mem0]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/plugins/memory/mem0/__init__.py#L291-L323
[s-mem0-identity]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/plugins/memory/mem0/__init__.py#L227-L256
[s-holographic]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/plugins/memory/holographic/__init__.py#L189-L201
[s-context-engine]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/context_engine.py#L48-L143
[s-precompress]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/conversation_compression.py#L2967-L3004
[s-nudge]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/turn_context.py#L745-L754
[s-finalizer]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/turn_finalizer.py#L746-L769
[s-review-fork]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/background_review.py#L943-L1038
[s-review-gate]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/tools/memory_tool.py#L166-L203
[s-lessons]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/background_review.py#L560-L587
[s-skills-index]: https://github.com/NousResearch/hermes-agent/blob/eb7e8620324b32424c06218f6a28094df2e921f8/agent/system_prompt.py#L301-L323
