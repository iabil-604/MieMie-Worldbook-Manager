# Chat Context Integration Test Plan / Results

关联 [Issue #3](https://github.com/SheepSheepLab/MieMie-Worldbook-Manager/issues/3)；结论、源码引用和产品待定项见 [Compatibility Research](CHAT_CONTEXT_COMPATIBILITY.md)。

执行日期：2026-09-28。

## 1. 环境和证据范围

| 项目 | 记录 |
| --- | --- |
| SillyTavern | 1.19.0，`7e8663cd9c184a550b37238218bdd32c6efc68e9` |
| Tavern Helper | 4.11.1，`830ebc84dc5aa74991f996bfdab910787ed7efff` |
| 运行环境 | Windows，Node.js 24.13.0，Chromium 154；独立回环服务；顶层扩展上下文 |
| 测试数据 | 专用合成角色 `ISSUE3_RESEARCH_ONLY`，短文本状态样本、长文本样本、旧格式 JSONL、单成员群聊 |
| 执行方式 | 原创临时诊断扩展构造样本；调用真实原生/Helper 接口；浏览器读取诊断面板及原生 DOM |
| 隔离 | 从独立数据集启动；没有导入或访问用户既有聊天；没有模型服务配置；聊天操作仅针对合成样本 |

这里的“真实宿主验证”指实际 SillyTavern 和 Helper 正在运行，输入数据为合成内容。T13 的缺字段补充探针直接调整合成内存对象，因此单独标注为结构探针。T10 则确实经过原生 JSONL 导入和聊天加载。

测试准备和写入是明确的 fixture 操作；读取不改变聊天的断言只覆盖指定读取步骤，不是声称整个测试流程没有写入。

## 2. 前置条件与可复现样本

1. 使用上述固定版本，建立全新、可丢弃的专用测试实例和数据集，确认默认资源加载完成。
2. 加载对应版本 Helper，确认 `getTavernVersion()` 和 `getTavernHelperVersion()`。使用顶层扩展上下文，避免把尚未验收的 iframe 代理混入基线。
3. 建立专用角色，greeting 为 `FIXTURE opening`。所有写入辅助步骤先检查当前角色确为该合成角色，群聊测试另外检查专用群聊身份。
4. 在该实例选择未连接的 Kobold 作为连接类型；本测试不调用模型生成。首次运行曾受默认 Horde 预设影响，详见 §5。
5. 用 Message IDs 设置显示原生编号；读取数组长度、`message_id`、DOM `mesid` 和 `.mesIDDisplay` 进行交叉核对。设置关闭时编号文本仍可能存在于 DOM，但不可称其为当前可见 UI。

初始消息序列由 greeting 加以下 5 条组成，可通过 Helper `createChatMessages(..., {refresh: 'all'})` 添加：

```json
[
  {"role":"user","message":"FIXTURE U1"},
  {"role":"assistant","message":"FIXTURE A1"},
  {"role":"system","message":"FIXTURE N1"},
  {"role":"assistant","message":"FIXTURE H1","is_hidden":true},
  {"role":"assistant","message":"FIXTURE A2"}
]
```

此时预期原 index 为 0–5；role 依次为 assistant、user、assistant、system、assistant、assistant；只有 index 4 的 `is_hidden` 为 true。该 system fixture 由 Helper 创建为 narrator，用于验证旁白语义，不代表所有核心 system/internal 消息。

读取探针使用：

```js
const context = SillyTavern.getContext();
const helper = TavernHelper;
const messageIds = rows => rows.map(row => row.message_id);
```

聊天切换后重新获取 context；不要把上面一次返回的 `chatId`、`characterId` 等值当作会自动更新的上下文对象。

## 3. 已执行的 Integration Test

“已观察”表示用例成功暴露实际行为，不等于该行为符合产品需求。失败及复测都保留在下表和 §5。

| ID | 操作 / 核对点 | 本次实际结果 | 证据等级 |
| --- | --- | --- | --- |
| T01 Opening | 新角色打开后比较数组、Helper、DOM | 1 条 greeting；index / Helper ID / DOM 均为 0，编号文本 `#0` | R |
| T02 Normal history / latest | 添加 §2 的 5 条消息，读 `0-999` 和 latest | 共 6 条，ID 0–5，角色和隐藏标记与 fixture 一致；latest 为 5 | R |
| T03 Inclusive range / bounds | 依次读 `2-5`、`80-90`、`5-2`、`-1`、`invalid` | 分别为 `[2,3,4,5]`、`[5]`、`[2,3,4,5]`、`[5]`、`[]`；越界夹取与类型注释不一致 | R |
| T04 Detached readonly result | 记录 chat/metadata/DOM，读取副本并改副本的 message、extra | 再次比较原 chat、metadata、DOM，保持相同 | R |
| T05 Saved single chat | 显式保存 fixture；从 `getChatHistoryBrief('current')` 找当前描述符，读详情 | 6 条；第一项为 greeting，单聊 header 已移除 | R |
| T06 Native hide | 原生 `/hide 2`，读 `hide_state: 'unhidden'` | 数组和 DOM 仍有 6 条；index 2 变隐藏；可见投影 ID `[0,1,3,5]`；latest 仍为 5 | R |
| T07 Swipe / selected edit | 原生 `/addswipe switch=true`；读当前与全部候选；Helper 编辑激活候选 | 首次未切换，复测通过：index 4 不变，2 个候选，选中 1，当前正文是 alternate；编辑后候选 1 正文更新，role 仍 assistant | R，含失败复测 |
| T08 Native delete | 在 6 条样本中调用原生 `deleteMessage(1)` 并保存 | 总数 5；旧 index 2 成为 1；Helper ID 与 DOM 重新为 0–4 | R |
| T09 Native branch | 对删除后的聊天执行 `/branch-create 2` | 新聊天只有前 3 条，ID 0–2；chatId 和 integrity 均改变 | R |
| T10 Native legacy import | 原生 JSONL 导入两条均缺少 `is_system` 的消息，再打开 | 数组/Helper/DOM 为 0、1；DOM 标记未隐藏；原字段仍缺失；`unhidden` 却返回空；聊天 integrity 已补齐 | R |
| T11 Group file / current | 保存 header + 1 条消息，Helper 读群聊文件；再建立专用群聊打开同一文件 | 文件详情 2 项且第一项为 header；当前数组/Helper 仅 1 条，index 0，正文 `FIXTURE GROUP` | R |
| T12 Large current chat | 建立 500、2000 条长文本，读全范围与末尾 50 条 | latest 分别 499、1999；末尾范围均 50 条；DOM 都只渲染 100 条；读前后数组未变；启用编号后末条可见为 `#1999` | R |
| T13 Empty / missing flags | 清空专用 fixture，读取；再添加一条并删除其 `is_system` | 空数组时 Helper latest **为 0**，范围返回 `[]`；缺标记探针默认返回 `[0]`、unhidden 返回 `[]` | R + 合成结构探针 |
| T14 Saved large / missing file | 保存 2000 条并重复读详情；读取确定不存在的合成文件名 | 返回 2000 条；当前 chat/metadata/DOM 和保存结果保持相同；不存在文件的键保留、值为 `[]` | R |

结果说明：

- T01/T02 的编号文本检查开始时以 DOM 为依据；T12 另开启 Message IDs，并确认末条编号实际可见且为 `#1999`。
- T02 在正常、稳定状态下比较 latest。T13 证明“Helper latest 为 0”本身不能区分空聊天和确有第 0 条消息。
- T07 的通过记录使用分支中的新 assistant fixture；index 4 是该复测位置，不是最初六条样本中的隐藏消息。两次失败记录没有被删掉或改写成通过。
- T08 的原生删除要求目标消息在 DOM 中。本次短聊天满足此条件；这不是只读 Adapter 应调用的接口。
- T11 的存储样本先经真实群聊保存端点写入，再经原生群聊打开。验证了单成员群聊读取，未验证多成员生成。
- T13 为避免额外渲染使用 Helper 的 `refresh: 'none'` 清空。空数组读数来自实时状态；没有把尚未刷新的旧 DOM 当作空聊天 UI 证明。
- 只读断言比较的是可序列化 chat/metadata、相关 DOM 结构与保存接口返回结果；不是完整进程内存或所有磁盘文件的字节审计。

## 4. 复现细节

### 普通范围、隐藏、删除、分支

在 §2 的六条序列上执行 T01–T06。先保存，再读已保存详情；保存属于 setup。

T04 记录读取前的 `JSON.stringify({chat, metadata, domRows})`，其中 DOM 行包含 `mesid`、编号文本和 `is_system` 属性；改变读取副本的字段后重新记录并比较。不要为了测试深拷贝去改原消息。

T06 用原生 hide 命令修改 index 2。执行 T08 时删除 index 1；检查 `FIXTURE A1` 迁到 index 1、DOM 编号重排。T09 以当前 index 2 创建分支，比较前缀内容、长度及聊天身份。

### Swipe 和 edit

在专用聊天追加 assistant `FIXTURE SWIPE clean original`。执行：

```text
/addswipe switch=true FIXTURE SWIPE clean alternate
```

操作完成后读取相同 index 的默认投影和 `include_swipes: true` 投影；检查 `swipe_id === 1`、候选数为 2、当前正文为 alternate。再用 `setChatMessages([{message_id: index, message: 'FIXTURE SWIPE clean edited'}], {refresh: 'all'})` 编辑，检查激活候选正文和原位置。这个写接口仅用来验证编辑行为，不能出现在产品后台读取链中。

### 导入旧格式

使用原生导入入口导入以下合成 JSONL，再打开导入结果。首行是旧式 header，后两行刻意缺少 `is_system`，也不带持久消息 ID：

```jsonl
{"user_name":"User","character_name":"ISSUE3_RESEARCH_ONLY"}
{"name":"ISSUE3_RESEARCH_ONLY","is_user":false,"mes":"FIXTURE IMPORT opening","extra":{}}
{"name":"User","is_user":true,"mes":"FIXTURE IMPORT user","extra":{}}
```

本次通过原生 `importCharacterChat(FormData, {refresh: false})` 执行：文件字段为 `avatar`，另含 `file_type: 'jsonl'`、合成角色的 `avatar_url`、`character_name`、`user_name`。随后使用返回的文件标识调用 `openCharacterChat`。这些是测试导入动作，不是聊天读取实现。

比较默认读取、`hide_state: 'unhidden'`、原字段和 DOM 隐藏标记。这里只确认上述具体旧形态；不能由此推广为所有历史来源格式已验收。

### 群聊文件

为专用群聊保存以下数组，使用合成文件标识 `ISSUE3_SYNTHETIC_GROUP_FILE`：

```json
[
  {"chat_metadata":{"integrity":"issue3-synthetic-integrity"}},
  {"name":"Fixture","is_user":false,"is_system":false,"mes":"FIXTURE GROUP","extra":{}}
]
```

调用 `getChatHistoryDetail([{file_name: 'ISSUE3_SYNTHETIC_GROUP_FILE'}], true)`；然后把此文件关联到只含测试角色的专用群聊，通过原生群聊加载器打开。对照保存详情的 2 项与当前数组的 1 条消息。不要把 header 判为第 0 条剧情。

### 空聊天

仅在专用 fixture 中用 `deleteChatMessages` 清空；立即记录数组长度、`getLastMessageId()` 与 `getChatMessages('0-1')`。当前 core 可能在打开新的未 tainted 聊天时自动加入 greeting，因此必须确认真的测到了长度 0，不能只用“新建聊天”替代判空实验。

## 5. 失败、环境修正和复测

初次测试实例的默认资源目录复制出现 `EIO / Access denied`，默认内容未完整建立，界面只提供 GUI preset。此时连接类型是 AI Horde。

- 首次 `/addswipe switch=true` 增加了第 2 个候选，但选中索引仍为 0；依赖“已选中候选 1”的编辑断言也未通过。
- 单独重试并等待 10 秒，选中状态仍未改变，超时。不能将该失败解释成普通异步延迟。
- 查到核心 Swipe 会执行 Horde 预设检查；`koboldhorde` 与 `gui` 组合会提前返回。
- 仅为独立实例补齐缺失的默认文件，未覆盖已有文件；选择未连接的 Kobold 后复测。候选切换和候选编辑都通过，见 T07。

宿主业务源码未打补丁，未配置模型凭据。报告保留首轮失败，成功结果仅对应修正后的环境条件。

## 6. Performance 方法和实测

每条新增长消息的正文由下式构造，交替 user/assistant；已有少量状态测试消息保留在前缀：

```js
const text = 'Synthetic long-message research text. '.repeat(120);
const message = `FIXTURE LARGE ${i} ${text}`;
```

使用 `createChatMessages` 扩充到总数 500，再扩充到 2000，刷新完毕后测量。正文约 4.4 KB；带有候选/元数据的数组序列化字节数另外计算。时间字段和前缀状态会影响准确字节数，不要求复现得到逐字节相同的聊天。

同步测量函数为原创研究片段，只测 `fn()` 的调用耗时：

```js
function sample(fn) {
  for (let i = 0; i < 3; i++) fn();
  const samples = [];
  for (let i = 0; i < 20; i++) {
    const start = performance.now();
    fn();
    samples.push(performance.now() - start);
  }
  samples.sort((a, b) => a - b);
  return {
    median: (samples[9] + samples[10]) / 2,
    p95: samples[18],
    max: samples[19],
  };
}
```

分别测 `getChatMessages('0-' + latest)` 与 `getChatMessages((count - 50) + '-' + latest)`。未包含造数、渲染、序列化、tokenization 或网络。

| 规模 | `JSON.stringify(chat)` UTF-8 字节 | DOM 条数 | 全量 median / p95 / max (ms) | 末尾 50 条 median / p95 / max (ms) |
| --- | ---: | ---: | --- | --- |
| 500 | 2,533,158 | 100 | 1.9 / 2.4 / 2.6 | 0.3 / 0.5 / 0.5 |
| 2000 | 9,733,098 | 100 | 6.4 / 7.3 / 7.5 | 0.2 / 0.3 / 0.3 |

保存 2000 条后，从当前角色摘要选中**当前聊天**描述符，详情读取预热 1 次、计时 5 次：min 132.7 ms、median 135.3 ms、max 160.2 ms，均返回 2000 条。5 次样本不足以报告有意义的 p95。

所有读取中没有发现当前消息变化。本次没有 long-task、内存峰值或 UI 帧率测量；未测更多候选、巨型变量对象、媒体或多文件并发；不据此声称完整 AIRP 工作流无卡顿。

## 7. 后续 Integration Test Plan

| 待验场景 | 方法 | 接受条件 |
| --- | --- | --- |
| 产品楼层语义 | 维护者先选择原生 #N / 一基计数，覆盖 0、1、200、负数、小数、超上界 | 显示、输入和读取范围一致；错误不被夹取成另一段剧情 |
| 空结果与 latest | 空聊天、仅隐藏消息、末尾为内部消息、无活动会话 | 分清不存在、空、被过滤、未就绪；不误报第 0 条 |
| 模型 regenerate | 配置获准的测试模型服务，在独立聊天执行 regenerate、成功/取消/失败 | 记录真实数组变化；读取不混入未完成生成；不能默认等于追加 Swipe |
| 流式与并发切换 | 生成中读取、Swipe 中读取、跨分段编辑/删除、切换角色/群聊/分支 | 返回未就绪或丢弃失效快照，不拼接多个会话/修订 |
| 多成员群聊 | 多 greeting、连续非用户回复、多成员重新生成 | 每条原生消息计一位置；保留正确发言者；header 不计楼 |
| 历史来源兼容 | 不同年份真实格式的脱敏最小样本、第三方导出、损坏/缺 header | 明确支持/拒绝，不能无条件移除第一条剧情 |
| 工具/扩展/多模态 | 工具调用链、small-system、展示替代文本、媒体、未知 extra | 内部内容不无声进入剧情 Prompt；未知字段不因读取被改写 |
| 错误路径 | 受控地模拟请求非 2xx、JSON 解析失败、拒绝、重复描述符 | 错误不伪装为空聊天；保存读取不能伪装成实时快照 |
| 长期规模 | 更多消息、更长正文、更多候选/变量；记录内存、long tasks 和分段一致性 | 预算可见、无静默截断；界面保持可操作；阈值由产品验收决定 |
| 最终业务只读性 | 对工作区完整请求链和最终 Prompt 做检查 | 不写正式消息、不切换正式聊天、不自动写世界书；仅必要正文进入已批准的模型请求 |

以上待验项没有标记为本次通过。尤其模型生成、iframe 代理、移动端和完整工作区尚无实测结论。

## 8. 提交前 Self Review

- 基线固定到稳定 release 与 Helper commit；源码证据和实测分开记录。
- Compatibility Map、楼层方案、范围边界和维护者待定项完整列出。
- Opening、普通多轮、范围、hide、delete、Swipe、branch、旧 JSONL 导入、群聊读取、空聊天和大聊天均有实际结果。
- 失败及环境修正已披露；性能样本和测量边界已写明。
- 测试使用独立合成数据；提交仅含研究文档和复现片段，不含用户聊天、原始环境日志、凭据、设备标识或安装位置。
- 没有正式 Adapter、Worldbook 修改、其他业务实现或第三方源码副本。
- 仓库当前没有约定构建/CI 命令；本次验证为真实宿主实验、文档/引用核对、差异与隐私扫描，不声称业务测试套件已通过。
