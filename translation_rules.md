# Jovida Trace → 用户可视化：规则转译手册

> 目标：用户激活一个 Desire 后会等 3-4 分钟。期间 Jovida agent 在跑 `project_scaffold` + `jov` + `project_data_update` 三段 trace。本手册定义**纯规则**（无需 LLM 二次转译）的渲染映射，把这段过程展示给用户，缓解等待焦虑。
>
> 同时给出"直接看结果"模式所需的提取规则。

---

## 0. 输入：trace 文件结构速览

每一个 Desire 至少产生两个 trace 入口文件：

```
~/.jovida/debugger/trace/oss-dev/user_<uid>/
├── thread_project_scaffold/traces/<date>/project_scaffold_goal_<id>_<ulid>.json    # 1. 脚手架
└── thread_default/traces/<date>/<root_ulid>.json                                    # 2. 主线 dispatcher
    └── <root_ulid>_<dispatched_ulid>.json                                           #    └── 派生：project_data_update
```

### 顶层 schema
```jsonc
{
  "trace_id": "ULID",
  "agent_name": "project_scaffold | jov | project_data_update",
  "started_at": "ISO+8",
  "finished_at": "ISO+8",
  "duration_ms": 81873,
  "usage": { "input_tokens", "output_tokens", "cached_tokens", "llm_calls", "tool_calls" },
  "spans": [ ... ],            // 主要内容
  "path":  ["main","tool_executor","main",...],
  "edges": [ {from,to,step,source} ],
  "step_count": 5
}
```

### span 三种类型

| type | name 例 | 关键子字段 |
|---|---|---|
| `llm` | `main` | `llm.input.blocks[]`、`llm.output.blocks[]`（`of_thinking` / `of_text` / `of_function_call`）、`usage`、`latency_ms`、`system_prompt`、`pin_prompt` |
| `tool` | `read_goal_core` / `read` / `list` / `write` / `image_gen` / `image_search` / `remind` / `validate_project_data` | `tool.tool_name`、`tool.args`、`tool.result` |
| `dispatch` | `project_data_update` | `dispatch.agent_name`、`dispatch.trace_ref`（指向子 trace 文件） |

---

## 1. 总体策略：把 trace 折成"用户视角的舞台"

trace 里有大量 LLM call + tool call。**不要原样展示**，而是把它们折叠成"用户能感知的大步骤"，每个步骤具备：

```ts
type Stage = {
  group:      "scaffold" | "dispatch" | "content";
  icon:       string;          // emoji
  label:      string;          // 主标题，动词为主：理解 / 设计 / 生成 / 写入
  sublabel:   string;          // 补一句"这一步在干嘛"
  duration_ms:number;          // 真实耗时
  payload:    StagePayload;    // 展示用的结构化内容
}
```

### 1.1 关键优化：把 LLM 拆成 **思考 / 输出** 两段

Gemini 的 trace 在 `llm.latency_ms` 里给出了 `first_nonthought_chunk` —— 这是模型"停止思考、开始流式输出 function_call / text"的时间点。这意味着每个 LLM span 天然可以切成两段：

```
api_call = first_nonthought_chunk (= 思考耗时)  +  output_phase (= 流式吐 tool_call 的耗时)
```

**实测两个 demo 的几个大头：**

| 步骤 | 总耗时 | 思考 | 输出 |
|---|---|---|---|
| 设计项目结构（Handan） | 71.8s | 37.3s | 34.5s |
| 设计项目结构（戒烟） | 60.9s | 21.4s | 39.6s |
| Jovida 主线接管（戒烟） | 76.1s | 20.2s | 55.8s |
| 组装卡片内容（Handan） | 35.5s | 31.4s | 4.1s |

**好处**：用户看到"它在想 → 它在写"两段不同的视觉张力（紫色思考流 vs 橘色 tool_call 列表），而不是一个 70s 的空脉冲。

**规则**：当 `first_nonthought_chunk ≥ 3000ms` 时，拆成两个 Stage：

```ts
makeLLMStages(span, label) → Stage[]
  if thinking >= 3000ms:
    [{ icon:"🧠", label:`思考·${label}`, duration_ms: thinking,
       payload: { type:"phase_think", thinking } },
     { icon:"📤", label:`输出·${label}`, duration_ms: api - thinking,
       payload: { type:"phase_emit", tool_calls:[{name, args_preview}], text } }]
  else:
    [{ icon: ..., label, duration_ms: api, payload:{type:"phase_quick", ...} }]
```

`payload.tool_calls[i].args_preview` 是按 tool 类型做的极简摘要（write → 文件路径；image_gen → prompt 截断；dispatch → 子 agent 名）。

### 1.2 过程回放里 phase_think / phase_emit 的渲染

- **phase_think**：紫色脉冲 bullet，详情区是"思考流" —— 按 `**xxx**` 切成 ▸ 小段，依次 fade-in（每段间隔 = `realDur / 段数 * 0.85`），最后一段亮起后这一 Stage 就标记 done。视觉给人"它在边想边把要点列出来"的感觉。
- **phase_emit**：橘色脉冲 bullet，详情区是 tool_call 列表（最多 10+ 行）。每行 `▸ <tool_name> <args_preview>` 依次从左侧滑入。等所有行都亮起，说明模型把"决策"全部下达完毕。下一步通常就是对应的 tool 执行（往往 < 100ms 一闪而过 ✓）。
- **phase_quick**（思考 < 3s 的小 LLM）：一次性展示思考摘要 + tool_call 列表，不做时间动画。

### 1.3 回放控制

过程回放区顶部固定：
- 进度条 + 剩余秒数（按 SPEED 实时换算）
- 速度切换：`1×` / `4×` / `16×`
- **`↻ 重播`** 按钮：随时从头播一遍（清掉所有 timeout + 重置 `.in` class）
- **`跳到结果 →`**：把所有 step 标 done、所有 frag 标 in，然后切到"最终成品"模式

---

## 2. 折叠规则：trace → Stage[]

按 trace 类型顺序合并：

### 2.1 `project_scaffold` trace（约 70-80s，5 steps）

| span 选择 | → Stage |
|---|---|
| 第 1 个 `llm/main`（step 1）| **「理解你的目标」** payload = `{ thinking: 截取 of_thinking 前 600 字 }` |
| `tool/read_goal_core` | **「读取你的 Desire」** payload = JSON 化的 goal 内容（重点：`desire_title`、`desire_description`、`qa_snapshot`） |
| `tool/read`(meta-schema) + `tool/list`(`/projects`) | **「加载项目模板规范」** payload = `{ existing_projects: list 返回值里的目录名 }` |
| 第 2 个 `llm/main`（step 3，最长，60-72s）| **「设计项目结构」** payload = `{ thinking: 取前 1500 字 }` |
| 全部 `tool/write`（紧跟着的一串） | **「搭建文件脚手架」** payload = `{ writes: [{file_path, content, duration_ms}] }` —— 渲染成目录树 |
| 最后 `llm/main`（含 `of_text`） | **「脚手架完成」** payload = `{ text: of_text 内容 }` |

### 2.2 `jov` dispatcher trace（约 15-150s，3 steps）

| span 选择 | → Stage |
|---|---|
| 第 1 个 `llm/main`（含 dispatch + remind 的 function_call） | **「Jovida 主线接管」** payload = `{ thinking }` |
| `tool/remind`（如有） | **「创建主动提醒」** payload = `{ title, due_at, result_id }` |
| `dispatch/project_data_update` | **不直接渲染**，跳到子 trace |

### 2.3 `project_data_update` 子 trace（约 60-80s，11 steps）

| span 选择 | → Stage |
|---|---|
| 第 1 个 `llm/main`（step 1） | **「规划要生成的内容」** payload = `{ thinking }` |
| `tool/read_goal_core`（step 2） | **「重新拉取 Desire 全貌」**（极短，可作为子提示） |
| 第 2 个 `llm/main`（step 3） | **「构思视觉与素材」** |
| 一串 `tool/read` 各 section 的 `schema.json` | **「复核每个 Section 的 schema」** payload = 读取的 schema 文件路径 |
| 所有 `tool/image_gen` | **「生成主题图标」** payload = `{ items: [{prompt, url, duration_ms}] }` |
| 所有 `tool/image_search` | **「搜真实场景图片」** payload = `{ items: [{query, images:[{caption,url}], duration_ms}] }` |
| 中间的大 `llm/main`（>20s） | **「组装卡片内容」** payload = `{ thinking }` |
| 所有 `tool/write` | **「写入卡片到项目」** payload = `{ writes }` —— 解析每个 section.json 的 cards 数组，做精致预览 |
| `tool/validate_project_data` | **「Schema 校验」** payload = `{ result }` |
| 最后的 `llm/main` | **「全部完成」** payload = `{ text }` |

---

## 3. 单个步骤的"payload → UI"渲染规则

### 3.1 `llm_understand / llm_design / llm_plan / llm_visual / llm_compose`（任何带 thinking 的 LLM 步）

```
[卡片 · 思考流]
  ▸ 标题：阶段名
  ▸ 一行灰字：「Jovida 正在思考」
  ▸ 把 thinking 文本按 `**xxx**` 切成小段，每段当作一个 sub-bullet 展示（Gemini 的 thinking 输出本身就带 **小标题** 结构）
  ▸ 处理：把 "I'm" / "I am" 等第一人称替换成"它"或省略，保留小标题
```

### 3.2 `goal_core`（读取 Desire）

```
[卡片 · Desire 速览]
  ▸ 标题：Desire #<id> · <desire_title>
  ▸ desire_description 当副标题
  ▸ qa_snapshot 平铺成 Q/A pill 列表
        Q: 这次旅行你最想去哪里？
        A: 邯郸
     • answer.text 优先；为空时拼 answer.labels 用 · 连接
  ▸ desire_query 折叠在 "原始任务定义" 里（默认收起）
```

### 3.3 `template_load`

```
[卡片 · 模板装载]
  ▸ 一行：「加载项目模板契约 (meta-schema.json)」
  ▸ 已有项目列表（来自 list /projects）：
        📁 ai_founder_feed/
        📁 ai_infra_investment/
        📁 skincare_routine/
  ▸ 一行结论：「未发现同 desire 的现有项目，将新建」
```

### 3.4 `scaffold_writes`（脚手架写入）

把所有 `file_path` 拼成一棵目录树。**核心：渲染成架构图。**

```
📁 /projects/handan_weekend_trip/
├── 📄 meta.json             ▸ 项目元数据：desire_id, goal_name
├── 📄 PROJECT.md            ▸ 项目定位、路由规则、维护指南
└── 📁 data/
    ├── 📁 daily_actions/       Daily Actions 区块
    │   ├── 📄 schema.json      schema 契约
    │   └── 📄 section.json     空 cards: []
    ├── 📁 itinerary/           Itinerary
    │   ├── schema.json
    │   └── section.json
    ├── 📁 budget_breakdown/    Budget Breakdown
    │   └── ...
    └── 📁 pre_departure_checklist/  Pre-departure Checklist
        └── ...
```

每个 `section.json` 解析出 `title` 字段，作为区块名标注在右侧。

过程模式下：让节点一个一个亮起（用 `duration_ms` 实际耗时按比例分布），每亮一个就在右侧滚出对应的 section 名字。

### 3.5 `image_gens`（生成主题图标）

```
[卡片 · 主题图标]
  ▸ 标题：生成 N 张 3D isometric icon
  ▸ N 个并排 thumbnail（圆角 12px）
      • 上方真实图片（如果 url 可加载）
      • 下方一段灰字：从 prompt 里抽出主体名词（"car keys and a small suitcase" → "车钥匙 / 行李箱"）
        ▸ 抽取规则：取 prompt 第一句中 "icon of " 后到 "," 之间的英文短语，做朴素翻译；翻译失败就直接显示英文短语
```

过程模式：图片占位时显示一个脉冲灰块，到耗时点后切换成实际 URL（带 fade-in 0.3s）。

### 3.6 `image_searches`

```
[卡片 · 搜真实图片]
  ▸ 一行：搜索词 "邯郸皮渣 高清 美食"
  ▸ 横向 scroll 5 张候选图
  ▸ 每张图下面是一句 caption（从 Image search result 里提取）
```

### 3.7 `content_writes`（最终卡片写入）

**重点：把 section.json 解析后渲染成最终卡片预览。**

```
对每个 section.json：
  - title              → Section 大标题
  - definition         → Section 副标题
  - cards[]            → 按 card.type 分发渲染
```

#### card.type 分发表

| card.type | 渲染规则 |
|---|---|
| `daily_action` | 头图 (`image_url`) + 标题 (`title`) + 「action_type」chip（todo/read/chat/i_did_it）+ 欢迎语 (`welcome_text`) + 子 cards 列表 |
| `todo_card` | ☐ 复选框 + 标题 + 副标题（无 image） |
| `text_card` | 卡片：粗体 title + 段落 desc |
| `photo_card` | 圆角图 + 一行图说 (`desc`) |
| `composable_card` | 把 `parts[]` 按顺序竖直堆叠：photo_card 在上、text_card 在下 |
| `source_card` | 列表：每个 source 一行 `[link]` 文本 |

过程模式：写卡片时按耗时累积，先骨架（标题），再图片，再文案。

### 3.8 `validate`

```
[小条 · 通过]
  ▸ 绿色 ✓ + 「Schema 校验通过」
  ▸ 失败时（result 包含 errors）展开红色错误列表（本次没踩到）
```

### 3.9 `remind`

```
[小条 · 提醒已加到日历]
  ▸ ⏰ + remind.args.title
  ▸ 「将在 <due_at 友好格式> 触发」
```

### 3.10 `llm_final`

```
[完成卡片]
  ▸ 大字：🎉 完成
  ▸ 之前 llm 的 of_text 内容（清单形式：写入的文件列表）
  ▸ CTA：「跳到结果」按钮
```

---

## 4. 单一视图：过程回放 + 最终成品同页

> 不再做模式切换。"看结果"和"看过程"本就是同一份产物的两种粒度，统一在一个页面里即可。

页面结构（从上到下）：

```
[Sticky 顶部]
  ▸ 进度条 + 当前 stage 标题 + 剩余秒数
  ▸ 速度按钮: 1× / 4× / 16×
  ▸ ↻ 重播
  ▸ 跳到结果 ↓       (→ 滚动到底部 deliver 区，不切视图)

[时间线 timeline]
  按 Stage[] 顺序，思考/输出已按 §1.1 拆分
  每个 stage 渲染规则 3.x，sub-anim 走 §1.2 的节拍

[分割线]  ─── FINAL · 最终成品 ───

[deliver 区]
  ▸ Hero: Desire 标题 + active_thought（"✓ 已生成"角标）
  ▸ 各 section + cards 按 §3.7 渲染
  ▸ ⏰ 提醒 pill / 元信息脚注
```

行为：
1. 页面加载 → 自动 `startProcess()` 开始 4× 回放
2. 每一步播放完，下一步立即接上（无空隙）
3. 全部播放完：`pcur="✓ 全部完成"`，平滑滚动到 deliver 区
4. **`↻ 重播`** 任意时刻按下：清掉所有 timeout + 重置 `.in` class + 滚回顶部，从头再来
5. **`跳到结果 ↓`** 任意时刻按下：把所有 step 标 done、所有 frag/emit-row 标 in，平滑滚到 deliver 区
6. 真实节拍：`stage.realDur = max(280, stage.duration_ms / SPEED)`

> 这样的好处：单页两段，向下滚动即看到"成果"，无需切 tab；过程区和成品区视觉上有明显分界（虚线 + FINAL 标记），但在同一份滚动序列里。

---

## 5. 文案与术语映射（中文）

| trace 里出现的英文 | 中文 |
|---|---|
| Daily Actions | 每日行动 |
| Itinerary | 行程安排 |
| Budget Breakdown | 预算明细 |
| Pre-departure Checklist | 出行清单 |
| Craving Support | 烟瘾陪伴 |
| todo_card | 待办 |
| text_card | 说明 |
| photo_card | 图片 |
| composable_card | 组合卡片 |
| action_type=read | 阅读 |
| action_type=todo | 打勾 |
| action_type=chat | 聊一聊 |
| action_type=i_did_it | 打卡完成 |

英文 section 标题保留原文，但在副标题里加中文翻译（如 "Daily Actions · 每日行动"）。

---

## 6. 两个例子的实际折叠结果

### 例 A · Desire 2891「邯郸周末特种兵」（真实总耗时 257.8s）

```
👁️  理解你的目标          4.4s
🎯  读取你的 Desire       0.9s        ← 8 个 Q/A
📐  加载项目模板规范      0.05s       ← 看到 ai_founder_feed / ai_infra_investment / skincare_routine
🧠  设计项目结构          71.8s       ← 主要等待
📁  搭建文件脚手架        0.4s        ← 10 个文件
✅  脚手架完成            4.8s
🧩  Jovida 主线接管       15.0s       ← 决定 remind + dispatch
⏰  创建主动提醒          0.04s       ← 邯郸周末特种兵 | 出发前准备 (chat) @20:00
📋  规划要生成的内容      5.6s
🎯  重新拉取 Desire 全貌  0.6s
🎨  构思视觉与素材        6.0s
📑  复核每个 Section schema  0.01s
🖼️  生成主题图标          18.0s       ← 3 张：car keys+suitcase / Buddha head / city gate
🔍  搜真实场景图片        18.7s       ← 2 组：邯郸皮渣 / 圣旨骨酥鱼
✍️  组装卡片内容          35.5s
💾  写入卡片到项目        0.1s        ← 更新 daily_actions/section.json (3 daily_action) + itinerary/section.json (2 美食 composable) + meta.json (active_thought)
🛡️  Schema 校验           0.2s
🎉  全部完成              4.8s
```

### 例 B · Desire 2894「戒烟行动计划」（真实总耗时 284.4s）

```
👁️  理解你的目标          4.8s
🎯  读取你的 Desire       1.0s        ← 9 个 Q/A
📐  加载项目模板规范      0.07s
🧠  设计项目结构          60.9s
📁  搭建文件脚手架        0.4s        ← 6 个文件（只创建 daily_actions + craving_support 两个 section）
✅  脚手架完成            4.1s
🧩  Jovida 主线接管       76.1s       ← 拉长思考：用户偏好驱动 plan
⏰  创建主动提醒          0.04s       ← 戒烟行动计划 | 下午茶歇防破功提醒 @15:30
📋  规划要生成的内容      5.2s
🎯  重新拉取 Desire 全貌  0.6s
🎨  构思视觉与素材        6.0s
📑  复核每个 Section schema  0.01s
🖼️  生成主题图标          29.7s       ← 5 张：trash bin + crossed cigarette / mint tin + stress ball / chat bubble + wind / glowing lung + leaf / open book + quote
✍️  组装卡片内容          23.5s
💾  写入卡片到项目        0.1s        ← daily_actions (5 daily_action：深呼吸/金句/环境清理/替代品/打卡) + craving_support (1 text_card) + meta.json
🛡️  Schema 校验           0.04s
🎉  全部完成              5.4s
```

---

## 7. 工程实施清单

- [ ] 写一个 `trace_to_stages.ts/py` 函数，输入 3 个 trace 文件路径，按 §2 折叠成 `Stage[]`
- [ ] 写 `Stage` 的渲染组件库（§3）
- [ ] 模式切换 + 速度切换（§4）
- [ ] 文案 i18n（§5）
- [ ] 单元测试：拿 2891 + 2894 两个例子做快照（Stage 数 / duration / payload key）
- [ ] 在 app 侧：用户激活 Desire 之后立刻打开过程模式，trace 边产生边推送（可用 WebSocket 增量 push 新 span，前端持续追加）

> **关键洞察**：真正"久等"的只有 4 个 span — 设计项目结构、Jovida 主线接管思考、组装卡片内容、生成主题图标。这 4 个加起来占了总耗时的 60-70%。过程模式重点把这 4 段做出"它在认真想 / 它在认真画"的视觉张力（如思考流文字 + 图片占位脉冲），别让用户面对一个无声的转圈。
