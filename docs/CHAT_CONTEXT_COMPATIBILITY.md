# SillyTavern AIRP Chat Context Compatibility

Issue: [#3 — SillyTavern AIRP Chat Context Compatibility](https://github.com/SheepSheepLab/MieMie-Worldbook-Manager/issues/3)

研究日期：2026-09-28。范围为聊天读取兼容性、真实宿主验证和 Phase 4 的设计建议。

## 结论摘要

- 当前聊天的数组索引、Tavern Helper `message_id`、原生消息编号 `#N` 使用同一套 **从 0 开始的当前位置**。它们不是长期稳定的消息 ID；删除会使后续编号前移，分支会切换聊天身份。
- 当前聊天优先读取 `SillyTavern.getContext().chat`，或经参数校验后调用 `TavernHelper.getChatMessages`。`getChatHistoryDetail` 读取的是**已保存的整份聊天文件**，不是当前聊天的范围接口。
- 实测发现三个会改变读取结果的差异：Helper 对越界范围进行夹取；旧消息缺少 `is_system` 时会被 `hide_state: 'unhidden'` 漏掉；群聊文件详情保留文件头，而原生当前群聊已经移除文件头。
- Swipe 切换和编辑保留消息位置；只把当前激活正文作为该楼的剧情。隐藏消息仍占数组位置，筛选后应保留原编号。
- 2000 条合成长文本消息的全量 Helper 读取中位数为 6.4 ms，末尾 50 条为 0.2 ms。文件详情读取中位数为 135.3 ms。这里只能说明本次样本和环境，不能据此承诺性能上限。
- **Needs Maintainer Decision**：产品中的“第 N 楼”采用原生 `#N` 还是一基序号；是否包含对模型隐藏的消息；结束楼层显示最后物理消息还是最后可用剧情。本文给出推荐方案，不替产品确定这些语义。

## 1. Research Baseline

| 项目 | 固定基线 |
| --- | --- |
| MieMie 官方 main | `93c3ace6da3d8aafe952e948267f084af1acd013` |
| SillyTavern 稳定 release | [1.19.0](https://github.com/SillyTavern/SillyTavern/releases/tag/1.19.0)，commit `7e8663cd9c184a550b37238218bdd32c6efc68e9` |
| Tavern Helper | [N0VI028/JS-Slash-Runner](https://github.com/N0VI028/JS-Slash-Runner)，4.11.1，commit `830ebc84dc5aa74991f996bfdab910787ed7efff` |
| 真实验证环境 | 独立 SillyTavern 实例；Windows；Node.js 24.13.0；Chromium 154；Helper 顶层扩展接口 |
| 数据 | 专用合成角色、聊天、群聊文件和旧格式 JSONL；未加载既有用户聊天；未连接模型服务 |
| 代码状态 | 宿主与 Helper 业务源码未修改；临时诊断扩展负责构造样本、调用原生操作及呈现读数 |

下文所有源码链接固定到上述 commit。没有把 staging 或未来版本的行为纳入稳定版结论。原始日志、机器标识、安装位置和用户聊天不属于提交内容。

证据标签：

- **S — Source-confirmed**：固定版本源码确认。
- **R — Real-environment verified**：真实宿主执行；输入可以是合成数据，宿主和 Helper 不是 mock。
- **D — Design recommendation**：后续设计建议，尚未实现。
- **U — Unverified**：未完成对应端到端验证。

测试编号、复现步骤、测量方法和失败记录见 [Integration Test Plan / Results](CHAT_CONTEXT_INTEGRATION_TESTS.md)。

需求对应：[PRODUCT_PLAN](PRODUCT_PLAN.md) §3 要求先研究真实兼容性；§15 是起始楼层到当前结束楼层；§22、§42–43 要求工作区和正式聊天隔离；§45 的接口名是研究入口；§51 Phase 4 留待后续实现；Golden Path 11、15、16 分别涉及聊天隔离、范围截取和世界书写入确认。

## 2. SillyTavern Chat Data Model

### 2.1 当前数组与存储文件

**S + R（T01、T05、T11）**：`getContext().chat` 是当前聊天的可变数组引用。`chatMetadata` 是聊天级状态，独立于消息数组。单聊保存为 JSONL：第一行是 header，后续行是消息；加载时先移除 header。当前版本的群聊保存同样包含 header，原生群聊加载器在第一行存在 `chat_metadata` 时移除它。[上下文入口][st-context]、[单聊保存][st-storage]、[单聊加载][st-load]、[群聊加载][st-group-load]、[群聊保存][st-group-save]

文件行号不能直接当消息索引。不同读取入口还可能返回“带头文件数组”和“不带头当前数组”两种结构。

### 2.2 核心字段及使用边界

核心类型中的字段大多为可选；扩展也可附加字段。下表是该版本核心字段的语义，不是允许把整个对象放进 Prompt 的白名单。[核心类型][st-types]

| 字段 | 语义与稳定性 | Context 用途 |
| --- | --- | --- |
| `mes` | 当前正文；编辑、Swipe、生成可改变 | 已完成消息的正文来源；按纯文本处理 |
| `name` | 当时记录的显示名，可编辑 | 发言者标记；群聊保留，不能作唯一身份 |
| `is_user` | 用户消息标记 | 区分用户和非用户；不能单独识别 narrator/internal |
| `is_system` | 对模型隐藏的消息标记；旧文件可能缺失 | 单独的可见性维度，不能直接转换为 LLM 的 system role |
| `extra.type` | 包括 narrator 等消息种类 | narrator 可能是剧情；内部类型需单独识别 |
| `swipe_id` / `swipes` | 当前候选索引 / 同一条消息的多个正文 | 默认只使用当前正文；不把候选重复计楼 |
| `swipe_info` | 各候选的时间及 `extra` | 用于一致性检查；不是新消息 |
| `send_date`, `gen_started`, `gen_finished` | 时间，可缺失且格式可变 | 可选溯源信息；不适合唯一键或剧情排序 |
| `title`, `force_avatar`, `original_avatar` | UI、头像及来源显示 | 不作为剧情正文，也不作长期消息 ID |
| `extra.api`, `model`, `gen_id`, `token_count` | 生成和计量信息；`gen_id` 不是每条消息都具有的唯一 ID | 仅必要诊断；默认不进入 Prompt |
| `extra.bias`, `memory`, `reasoning_display_text`, `display_text` | 提示偏置、扩展状态、推理或展示替代文本 | 不自动当剧情；不读取渲染 HTML 代替 `mes` |
| `extra.tool_invocations`, `isSmallSys`, `uses_system_ui` | 工具/内部消息和 UI 状态 | 默认不直接拼入剧情；需明确专门支持策略 |
| `extra.files`, `media`, `media_index`, `media_display`, `inline_image` | 附件与媒体；还兼容旧 `file/image/video/image_swipes` 等字段 | 本研究不提取多模态剧情；不得自动传播附件地址 |
| `extra.swipeable`, `overswipe_behavior` | 交互行为控制 | 不属于剧情 |
| `extra[IGNORE_SYMBOL]` | 核心跳过 Prompt 处理的临时标记 | 不推断为持久楼层信息；普通 JSON 也不能完整表达 Symbol |
| `variables` 及其他扩展字段 | Helper 等扩展附加的变量，不属于核心必需字段 | 不把扩展状态作为消息正文 |
| header 的 `chat_metadata` | `integrity`, `tainted`, `scenario`, `persona` 等聊天状态 | 从消息中分离；`integrity` 是聊天级标记 |

**S + R（T02、T06、T10）**：Helper 的 role 由 `extra.type` 和 `is_user` 推导：narrator 映射为 `system`，普通非用户映射为 `assistant`。`is_system: true` 则对应 `is_hidden`，与 role 是两条独立轴。异常组合“narrator 且 `is_user` 为真”甚至返回声明之外的 `unknown`。因此不能用 `role !== 'system'` 排除内部数据，否则会丢弃旁白剧情。[Helper 投影][th-messages]

### 2.3 消息标识与顺序

**S + R（T01、T06–T08、T12）**：数组顺序就是当前消息顺序，原生 DOM 的 `mesid` 和编号文本 `#N` 使用数组位置。编号显示可由 Message IDs 设置关闭；关闭显示不会产生另一套编号。分页渲染可以让 DOM 只含数组的末尾一部分。[渲染编号][st-render]、[重编号][st-renumber]、[显示设置][st-id-setting]、[分页渲染][st-pagination]

没有发现核心为所有消息强制提供的、跨删除/导入/分支仍稳定的消息 UUID。`message_id`、`mesid`、数组 index 都是位置；`swipe_id` 是消息内部的位置；`chat_metadata.integrity` 是聊天级标记。扩展自行写入的 ID 不构成原生通用保证。

## 3. Tavern Helper / Native API

### 3.1 入口对照

| 入口 | 实际语义 | 范围、返回和副作用边界 |
| --- | --- | --- |
| `SillyTavern.getContext()` | 当前聊天及会话身份、元数据、事件和原生方法 | 同步；`chat` 是原对象引用。只读消费者自行投影，禁止就地排序、删除或补字段。[源码][st-context] |
| `TavernHelper.getChatMessages(range, options?)` | 当前已载入聊天的范围投影 | 同步，闭区间，升序，返回深拷贝；默认包含隐藏和所有角色；不访问历史文件。[源码][th-messages] |
| `TavernHelper.getLastMessageId()` | `Number` 转换 `{{lastMessageId}}` 宏 | 不是简单的“最后有效剧情楼层”。必须单独处理空聊天和生成状态。[源码][th-latest] |
| `TavernHelper.getChatHistoryBrief('current')` | 当前角色的已保存聊天摘要列表 | 异步；不是当前聊天消息列表，也不是通用群聊入口。[源码][th-history-entry] |
| `TavernHelper.getChatHistoryDetail(descriptors, isGroupChat = false)` | 按描述符读取已保存文件 | 异步；结果按 `file_name` 为键；一次读取整文件；没有消息范围或分页参数。[源码][th-history] |
| 核心 `/api/chats/get`、`/api/chats/group/get` | 读取保存的 JSONL | 后端读取整文件、解析所有行；返回可含 header 的数组。[源码][st-file-read] |

本次实际调用的读取接口是以上六类（原生后端也由 Helper 间接调用）。`createChatMessages`、`setChatMessages`、`deleteChatMessages`、原生 hide/delete/branch/import 和测试保存仅用于合成样本准备及操作验证，**不属于未来只读 Adapter 的读取链**。

### 3.2 `getChatMessages` 的真实边界

**S + R（T03、T04、T10、T13）**：[实现][th-messages] 接受整数或 `start-end`；端点为 inclusive，负数相对末尾，端点夹到有效范围后排序。6 条消息时：

| 输入 | 实际返回 `message_id` |
| --- | --- |
| `'2-5'` | `[2, 3, 4, 5]` |
| `'80-90'` | `[5]` |
| `'5-2'` | `[2, 3, 4, 5]` |
| `-1` | `[5]` |
| `'invalid'` | `[]` |

[公开类型说明][th-declarations] 声称完全越界会返回空数组，和该固定实现、实测不一致。产品应先验证 `Number.isSafeInteger`、非负、顺序和上界，不依赖 Helper 的夹取/排序替用户纠错。接口会先做宏替换；产品传入已验证整数构成的范围，不透传任意输入作为宏。

`hide_state: 'unhidden'` 使用严格布尔比较。真实导入的两条旧消息均缺少 `is_system`：默认读取返回 `[0,1]`，`unhidden` 返回 `[]`；原生 DOM 却标为未隐藏。建议读取 `hide_state: 'all'`，在独立 View 中把“缺失”作为旧格式未隐藏处理，显式 `true` 才隐藏；非法类型应报告异常而非修改原对象。此建议不表示核心已经做了相同归一化。

默认 `include_swipes: false` 的 `message` 来自当前 `mes`，仍附带兼容字段 `swipe_id/swipes/swipes_data`，并非只复制当前正文。`include_swipes: true` 返回候选数组；不能假定还存在 `message` 字段。`extra` 从当前 `swipe_info` 项派生，可能是含 `extra` 的包装对象，不能假定等于原生 `message.extra`。需要原生类型判断时优先使用原生消息字段。[源码][th-messages]

无显式消息分页或每次条数上限；同步遍历请求范围、整理候选和变量后深拷贝。只读验证中，修改返回副本未改变当前数组、聊天元数据或 DOM。

### 3.3 `getChatHistoryDetail` 的文件语义和错误

**S + R（T05、T11、T14）**：传入摘要描述符数组；单聊描述符包括 `file_name` 以及角色名/头像定位字段。应从摘要获取真实描述符，不靠拆分文件名猜当前角色。群聊参数为 `true`，`file_name` 传聊天标识的 stem；后端自己追加扩展名。[请求构造][th-history-request]

- 单聊返回前无条件 `shift()`；现代格式的 header 被移除。
- 群聊不 `shift()`。本次保存的“header + 1 条消息”得到 2 项；打开同一群聊后，当前原生数组和 `getChatMessages` 都只有 1 条消息。这是读取入口差异。
- 读的是保存版本，可能落后于编辑、流式生成或防抖保存中的内存状态。不能无声地作为“当前最新”的后备来源。
- 实测缺失单聊文件在结果中保留该文件键，值为 `[]`。源码还会跳过非 2xx、非数组响应；网络失败或 JSON 解析失败则可使整批 Promise 拒绝。不能把缺键、空数组、失败和空聊天都报告为成功无剧情。
- 实现对描述符 `Promise.all` 并发读文件，没有内建并发上限或消息分页。按文件名整理输入不保证结果对象的全局时间顺序；重复键也不适合作独立会话列表。
- 已存在聊天的读取未观察到聊天内容或 UI 变化。核心单聊读取路由在目标目录不存在时可能创建目录，因此这里的“只读”指不写聊天正文，不等于所有后端情形都没有文件系统副作用。[Helper 文件读取][th-history]、[核心读取][st-file-read]

不要用 `getChat()`、`openCharacterChat()` 或 `reloadCurrentChat()` 实现后台读取：这些是切换/重载宿主状态的操作，加载空聊天还可能建立 greeting。后续工作区应直接获取当前只读快照。

## 4. AIRP Floor Semantics

### 4.1 已确认的原生规则

**S + R**：

| 情形 | 原生位置规则 | 测试 |
| --- | --- | --- |
| 单角色 greeting | 第一条 assistant，index `0`，编号 `#0`，参与 API 返回 | T01 |
| 后续用户、助手 | 本次样本分别为 `#1`、`#2`；每条消息占一个位置，不是每轮一楼 | T02 |
| narrator / 对模型隐藏消息 | 同样占数组位置；过滤后不应重新编号 | T02、T06 |
| 删除 | 被删项移除；后面的 index、Helper ID、DOM 编号一起前移 | T08 |
| Swipe / edit | 当前消息位置保留；正文及候选信息可变 | T07 |
| 分支 | 截取到选中消息的前缀，保存为独立聊天，带新的 integrity 和父聊天关系 | T09 |
| 旧格式导入 | 本次两条消息保持 index `0,1`；补了聊天 integrity，但没有补 `is_system` | T10 |
| 群聊 | 当前数组排除 header；多个成员 greeting 可以各占一条，不能普遍假设仅有一个 opening | T11；多成员 greeting 为 S |

### 4.2 Needs Maintainer Decision

以下方案会让同一个数字读到不同正文，不能静默互换：

| 方案 | 用户输入 `200` 对应什么 | 优点 | 风险 |
| --- | --- | --- | --- |
| **A：原生消息编号（推荐）** | 当前聊天 index `200`，与启用编号显示后的 `#200` 对齐 | 易于核对；hidden/narrator 不改坐标；允许 `0` | 用户可能把“第一楼”理解为 1；删除后位置仍会改变 |
| B：一基消息序号 | index `199`；opening 是第 1 条 | 符合部分自然计数习惯 | 与原生 `#N` 差 1，UI 必须明确同时显示映射 |
| C：仅剧情消息重新计数 | 过滤后的第 200 条，其原 index 取决于策略 | 可显示连续剧情序号 | 改 hidden/过滤规则就重排；难以和原生 UI 对照；不推荐作为输入坐标 |

推荐维护者选择 A，并用“起始消息编号（与酒馆 `#N` 一致）”消除歧义。若选择 B，应集中做一次 `N - 1` 转换并明确显示原生编号。任何方案都不是长期稳定的 Floor ID。

另请维护者确认：

1. **隐藏策略**：推荐默认排除显式 `is_system: true`，保留位置；需要读取已隐藏剧情时提供明确选项。不能自动把隐藏内容重新提交给 AI。
2. **latest 的显示语义**：推荐以快照中的最后物理编号标明范围，同时报告筛选后最后可用编号。若只显示最后可用编号，应让用户看见过滤原因。
3. **narrator 与内部消息**：推荐保留旁白，单独排除已识别的内部/工具/UI 消息；未知类型报告数量并保留待支持状态，不能简单删掉所有 Helper `system`。
4. **已保存的起点**：删除、分支或会话切换后，起点应失效并要求重新核对，不能把旧数字无条件移用到新快照。

计划书 §39 的“楼层 Entry / 隐藏旧楼层”涉及待确认的 AIRP 世界书条目结构，不是这里的原生聊天隐藏标记。本研究不定义两者的转换规则，也未使用真实 AIRP 变量 Entry 样本。

## 5. `[start → latest]` 推荐读取契约

以下为 **D**，以方案 A 为前提，供后续 Phase 4 实现；本文未实现正式 Adapter。

1. 确认当前单聊/群聊身份和已载入状态；没有活动聊天与活动空聊天分别返回明确状态。
2. 在生成、Swipe、导入或聊天切换结束后取快照。记录会话身份及读取期间的修订状态；分段/异步工作前后再比较身份，变化则取消或重读。
3. 空数组没有 latest。非空且状态稳定时，`latestPhysical = chat.length - 1`。不要仅凭 Helper `getLastMessageId()` 判断是否存在第 0 条：T13 实测空数组时也返回 0；核心宏还会跳过未完成的 Swipe。[核心 latest][st-latest]、[Helper latest][th-latest]
4. 严格验证 `start`：必须是安全整数。方案 A 的 `0` 合法，负数拒绝；方案 B 才要求 `start >= 1`。`start > latestPhysical` 返回明确越界状态，不调用 Helper 让其夹取到最后一条。
5. 读取闭区间 `[start, latestPhysical]`，按原数组升序。使用原生字段作最小投影，或调用 `getChatMessages` 的 `role: 'all', hide_state: 'all'` 后核对结果。不要先过滤再解释用户的起点。
6. 在 View 中执行已批准的 hidden/narrator/internal 策略，保留原 `sourceIndex`。空结果附过滤计数，不能误称聊天不存在。删除没有 tombstone 可读，只能针对当前快照解释位置。
7. 默认只采用当前激活的 `mes`。若候选状态与正文不一致、候选仍在生成，等待完成或返回未就绪；不把所有候选串成多楼剧情。
8. 返回独立 **Context View**，最少包括快照身份、起止坐标、筛选计数，以及每条记录的 `sourceIndex`、发言者、剧情角色、正文、隐藏状态、可选 `swipeId`。保留快照级修订信息，防止仅看长度漏掉编辑或 Swipe。
9. 发给 AI 的白名单仅为必要的发言者/旁白标签、正文和可选引用楼层。会话文件标识、integrity、原始 `extra/data/variables`、模型配置、附件位置、DOM/HTML、推理展示及所有未识别字段不自动进入 Prompt。
10. 读取全过程不调用消息写入、保存、生成、隐藏、切换聊天或世界书写入接口。缓存失效需覆盖 edit/delete/swipe/hidden/chat switch/branch，不能只监听新增消息。

原生数组本来就已全部在内存；范围读取只能减少后续遍历、投影和复制，不能让宿主省去已经载入的聊天。跨事件循环分段时，保留一次范围定义并在变化后放弃结果，避免拼接多个修订状态。

## 6. 特殊状态与 Known Limitations

| 状态 | 证据与边界 | 后续处理建议 |
| --- | --- | --- |
| Edit | R：Helper 更新当前激活候选，index/role 保持；S：原生编辑使用当前位置。[编辑源码][st-edit] | 同编号不代表相同正文；编辑应使内容缓存失效 |
| Delete | R：普通删除、编号前移；S：核心可连带移除前面的隐藏工具调用消息。[删除源码][st-delete] | 不假设一次操作总是只减一条；重取快照 |
| Swipe | R：2 个候选、切至候选 1、当前正文改变、编号不变 | 保留候选选择作为修订依据；旧正文不要重复纳入 |
| Regenerate | S：`Generate('regenerate')` 在非用户末尾先移除旧消息，再走生成；Swipe 路径有独立的候选选择/追加逻辑。[生成源码][st-regen]、[Swipe 源码][st-swipe] | **U：未连接模型服务，未实测生成/流式/取消/失败或群聊 regenerate。**不能承诺总条数或楼层在整个生成期间恒定，也不能把 regenerate 都称为追加 Swipe |
| Opening | R：greeting 进入 index 0；S：alternative greetings 位于同一消息的候选结构。[开场源码][st-opening] | 不固定减去一楼；群聊 greeting 数量另行核对 |
| Legacy | R：真实导入缺标记 JSONL，Helper 隐藏筛选差异；S：导入器还支持多种格式和 Chub 展平。[导入源码][st-import] | 只验证了一个合成 JSONL 形态；其他来源和损坏文件不能声称兼容 |
| Branch | R：前缀 3 条、独立 chatId/integrity；S：metadata `main_chat`、父消息 `extra.branches` 记录关联。[分支源码][st-branch] | 分支不继承原聊天的长期位置身份；新快照重新确认范围 |
| Saved history | R：单聊移头，群聊保留头，当前群聊移头 | 不混用数组下标；基于结构识别 header，不能所有格式无条件减一 |
| Tools / extensions / media | S：原生字段和处理路径；U：未覆盖工具往返、所有第三方扩展及多模态解释 | 默认不传播内部数据；需要后续专门样本和验收 |

首次 Swipe 测试没有切换成功。诊断确认隔离实例在默认资源复制失败后处于 `koboldhorde + gui preset`，核心在 Swipe 过程中也执行 Horde 预设检查并提前返回。[检查位置][st-swipe-guard]、[检查条件][st-horde-guard] 补齐缺失默认资源并把测试连接类型设为未连接的 Kobold 后，真实 `/addswipe switch=true` 和当前候选编辑通过。这不是等待更久就会成功的情形，也不作为 Helper 不兼容结论。

其他验证边界：仅一个 core release 与一个 Helper commit；仅桌面 Chromium 的顶层扩展接口；未验证 Helper iframe 代理、移动端、Tauri、任意主题/扩展组合或历史版本。缺文件返回已实测，网络拒绝/HTTP 错误传播主要依据源码。未对未保存编辑与文件详情落差做独立端到端实验。

## 7. Large Chat / Performance

**R（T12、T14）**：每条新增长文本使用固定合成短句重复 120 次，约 4.4 KB 正文；聊天前缀保留少量特殊状态样本。字节数为当时 `JSON.stringify(chat)` 的 UTF-8 长度，不是网络响应体、文件大小或 JS 堆内存。

| 当前消息数 | 序列化数组字节 | DOM 消息数 | Helper 全范围 median / p95 / max (ms) | 末尾 50 条 median / p95 / max (ms) |
| --- | ---: | ---: | --- | --- |
| 500 | 2,533,158 | 100 | 1.9 / 2.4 / 2.6 | 0.3 / 0.5 / 0.5 |
| 2000 | 9,733,098 | 100 | 6.4 / 7.3 / 7.5 | 0.2 / 0.3 / 0.3 |

每组先预热 3 次、测量 20 次同步调用。median 为中间两项均值，p95 取排序后第 19 项；计时不含造数、DOM 刷新、网络、JSON 字节计算、Prompt 拼接或 tokenization。短调用受计时分辨率和抖动影响，0.2 与 0.3 ms 不应解读为随规模增大而加速。

2000 条保存后，`getChatHistoryDetail` 先预热 1 次、测量 5 次完整读取：132.7–160.2 ms，中位数 135.3 ms；返回 2000 条。包含本机回环 HTTP、服务端读/解析及客户端 JSON 解码，不能直接当作浏览器主线程连续阻塞时间。读取前后当前数组/metadata/DOM 和再次读取的保存结果相同。

**S**：文件端点一次读入全部 JSONL；Helper 当前范围 API 同步投影并深拷贝；没有发现上述读取入口中的显式消息条数上限或分页协议。服务端 `500mb` JSON/urlencoded **请求体**限制是写入/上传方向的配置，不能宣传成历史读取的响应大小上限。[文件读取][st-file-read]、[请求解析][st-body-limit]

**D**：

- 当前聊天按需求范围取数据，先校验再读，避免每次操作回读所有历史文件。
- 首选只复制选中正文和必要映射字段；大量候选、变量、工具对象会增加 Helper 投影的成本。
- 提供消息数/正文长度预检，明确展示预算或分段计划，禁止静默截断剧情。
- 全量 Prompt 拼接、tokenization、频繁重复读取和多文件并发需要独立性能测试。若分段，增加修订检查并让出主线程。
- 本次没有测量内存峰值、long tasks、移动设备表现或最大可用聊天规模；没有依据给出普遍“无卡顿”结论。

## 8. Chat Context Compatibility Map

全表版本均为 §1 固定基线；测试编号链接到配套测试文档。

| 产品需求 | 原生 / Helper 实际依据 | 真实测试 | 限制 | Phase 4 建议 |
| --- | --- | --- | --- | --- |
| 当前 Chat history | `getContext().chat` / `getChatMessages`；[上下文][st-context] | T01–T05 | 原数组可变；文件可能较旧 | 当前快照最小投影 |
| Message identity | index / `mesid` / `message_id`；[重编号][st-renumber] | T07–T09 | 没有通用持久消息 ID | 会话 + 修订 + 原 index |
| Floor mapping | greeting #0，每消息一位置；[渲染][st-render] | T01、T02、T12 | 产品自然语言仍有歧义 | 维护者选择 A/B；不静默 +1 |
| latest | 数组长度、核心宏、Helper 数字转换；[宏][st-latest] | T02、T13 | 空数组/生成/hidden 策略 | 先判空和未就绪；分开 physical/eligible |
| range | 闭区间、负数、夹取、端点排序；[Helper][th-messages] | T03 | 越界与类型注释不一致 | 严格校验数字与边界 |
| Swipe | `mes/swipe_id/swipes/swipe_info`；[切换][st-swipe] | T07 | 生成候选未测 | 当前激活正文；修订检查 |
| Delete | splice + DOM 重编号；[删除][st-delete] | T08 | 工具消息可连带删除 | 旧位置失效并重取 |
| Hidden | `is_system` 与 role 分离；[Helper][th-messages] | T06、T10 | 缺标记被 unhidden 漏掉 | View 归一化；过滤后保留原编号 |
| Narrator / special | `extra.type`、工具及 UI 字段；[类型][st-types] | T02；工具为 S/U | Helper system 不等于内部消息 | 保留旁白；显式过滤策略 |
| Branch | 新聊天、integrity、父子引用；[分支][st-branch] | T09 | 数字相同可能是另一时间线 | 会话切换后重确认 |
| Saved/group history | header + 消息；[原生][st-group-load] / [Helper][th-history] | T05、T11、T14 | 群聊文件头差异；无范围参数 | 文件层与当前层分开 |
| Large Chat | 整文件读取 vs 同步范围投影；[文件][st-file-read] / [Helper][th-messages] | T12、T14 | 单设备合成样本；未测极限 | 取必要范围，限定并发和后处理预算 |

## 9. 后续验收与维护者待办

1. 先决定 §4.2 的楼层坐标、hidden 和 latest 语义，再实现 UI 与 Adapter；PR 关联 Issue #3，保留 **Needs Maintainer Decision**。
2. 实现后重跑 [T01–T14 和待验清单](CHAT_CONTEXT_INTEGRATION_TESTS.md)，并把生成、流式中途状态、取消、网络错误及多个扩展组合纳入验收。
3. 验证整个工作区读取链没有消息/世界书写入；对最终 Prompt 做独立白名单检查。本研究的只读调用验证不代替尚未实现业务链的验收。
4. 每次升级 core/Helper 都检查源码差异与关键回归，尤其是范围夹取、空聊天 latest、群聊 header 和旧隐藏标记。

本贡献只新增研究文档及文档化测试步骤，不依赖未合并的 Worldbook Adapter，不包含正式 AIRP Adapter、Sync、Prompt 或世界书写入实现。未引入第三方源码、构建依赖或二进制。

## 源码索引

链接固定版本；引用的是公开实现，不代表把第三方实现纳入本项目。

- 数据与身份：[类型][st-types]、[上下文][st-context]、[渲染][st-render]、[重编号][st-renumber]。
- 保存与加载：[单聊保存][st-storage]、[单聊加载][st-load]、[群聊加载][st-group-load]、[群聊保存][st-group-save]、[单聊文件端点][st-file-read]、[群聊文件端点][st-group-read]。
- 特殊操作：[开场][st-opening]、[删除][st-delete]、[编辑][st-edit]、[Swipe][st-swipe]、[regenerate][st-regen]、[导入][st-import]、[分支][st-branch]。
- Helper：[消息范围实现][th-messages]、[公开声明][th-declarations]、[latest][th-latest]、[文件请求][th-history-request]、[文件详情][th-history]、[入口][th-history-entry]。

[st-types]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/global.d.ts#L45-L128
[st-context]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/scripts/st-context.js#L115-L171
[st-storage]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L7395-L7447
[st-load]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L7634-L7669
[st-group-load]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/scripts/group-chats.js#L255-L309
[st-group-save]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/scripts/group-chats.js#L626-L641
[st-render]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L2551-L2657
[st-renumber]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L9467-L9473
[st-id-setting]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/scripts/power-user.js#L495-L498
[st-pagination]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L1433-L1452
[st-latest]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/scripts/macros.js#L330-L357
[st-file-read]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/src/endpoints/chats.js#L580-L617
[st-group-read]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/src/endpoints/chats.js#L872-L880
[st-opening]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L7684-L7738
[st-delete]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L1614-L1699
[st-edit]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L8240-L8254
[st-regen]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L4390-L4412
[st-swipe]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L10337-L10424
[st-swipe-guard]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/script.js#L10326-L10335
[st-horde-guard]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/scripts/horde.js#L359-L366
[st-import]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/src/endpoints/chats.js#L771-L870
[st-branch]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/public/scripts/bookmarks.js#L167-L244
[st-body-limit]: https://github.com/SillyTavern/SillyTavern/blob/7e8663cd9c184a550b37238218bdd32c6efc68e9/src/server-main.js#L110-L111
[th-messages]: https://github.com/N0VI028/JS-Slash-Runner/blob/830ebc84dc5aa74991f996bfdab910787ed7efff/src/function/chat_message.ts#L49-L165
[th-declarations]: https://github.com/N0VI028/JS-Slash-Runner/blob/830ebc84dc5aa74991f996bfdab910787ed7efff/@types/function/chat_message.d.ts#L21-L50
[th-latest]: https://github.com/N0VI028/JS-Slash-Runner/blob/830ebc84dc5aa74991f996bfdab910787ed7efff/src/function/util.ts#L9-L15
[th-history-request]: https://github.com/N0VI028/JS-Slash-Runner/blob/830ebc84dc5aa74991f996bfdab910787ed7efff/src/function/raw_character.ts#L6-L68
[th-history]: https://github.com/N0VI028/JS-Slash-Runner/blob/830ebc84dc5aa74991f996bfdab910787ed7efff/src/function/raw_character.ts#L100-L128
[th-history-entry]: https://github.com/N0VI028/JS-Slash-Runner/blob/830ebc84dc5aa74991f996bfdab910787ed7efff/src/function/raw_character.ts#L216-L241
