# 1、那么最开始提到的状态机模块就告...
**问题详情：**
那么最开始提到的状态机模块就告一段落了，三大模块已经实现了一个了，还有两个模块：数据库索引系统（直接构成上下文）以及post learning（微调agent，学习用户个人信息）

---
**回答（Answer）：**

我把后两块按“先能跑、再变强”的思路整理：数据库索引系统负责把题目、答案、步骤、卡片和当前会话装成上下文；post learning 负责把一次次 trace 变成学生画像和 agent 参数，而不是一上来就做真微调。

---
**回答（Answer）：**

很好，接下来这两块其实比状态机更像“系统底座”：
1. **数据库索引系统**：负责把“该给模型什么上下文”拼出来
2. **post learning**：负责把“这次学到了什么”沉淀下来，反过来调 agent

我建议你把它们分别理解成：
- **静态知识底座**
- **动态用户适配层**

而且我先给一个核心判断：

> **post learning 先不要理解成真正的模型微调，而要理解成“画像更新 + 策略更新 + 检索偏置更新”。**

这样你会稳很多，也更容易落地。

---

# 一、数据库索引系统：直接构成上下文

你前面说得很对，这个模块不是普通题库，而是一个**上下文构造系统**。
它的职责不是“把数据存进去”，而是：

> **在某一步 tutoring 发生时，快速找出最该进入 prompt 的那一小撮信息。**

所以这里不能只做“数据库”，要做成 **多级索引 + 上下文打包器**。

---

## 1.1 这个模块的目标

对于任意一次 tutor 轮次，系统都应该能组出一个 `ContextPack`：
JSON
{
  "problem_context": {},
  "solution_context": {},
  "step_context": {},
  "card_context": {},
  "anti_mislead_context": {},
  "student_context": {},
  "runtime_policy_context": {}
}

也就是说，模型每次看到的不是整个数据库，而是一个**当前步专用的小上下文包**。

---

## 1.2 推荐你把索引系统拆成 4 层

### A. 题目层索引

回答：
这是什么题、属于哪个章节、有哪些答案、有哪些分支。

索引字段建议：
- `problem_id`
- `chapter`
- `topic`
- `difficulty`
- `problem_type`
- `source`
- `tags`

这一层主要给：
- 老师选题
- 推荐题
- 章节聚合
- 粗粒度检索

---

### B. 答案步骤层索引

这是最重要的一层。

因为你现在的系统不是“题目 → 讲解”，而是：

> **答案步骤 → 命中 cards → 讲解**

所以每个 `solution_step` 都要可检索。

建议字段：
- `step_id`
- `solution_id`
- `step_order`
- `step_goal`
- `step_text`
- `step_type`
- `branch_id`
- `parent_step_id`
- `chapter_path`

其中 `step_type` 是关键，比如：
- `trig_identity_expansion`
- `substitution`
- `quadratic_rooting`
- `fixed_point_analysis`
- `range_analysis`

这层会直接决定当前步的 method card 命中。

---

### C. Card 层索引

这层不是只放正文，而是要放“可执行教学对象”。

每张 card 至少要有：
- `card_id`
- `card_type`
- `knowledge_point`
- `trigger_patterns`
- `anti_triggers`
- `scaffolded_hints`
- `one_step_check`
- `version`
- `status`

并且最好按角色分索引：
- method cards
- anti-mislead cards
- verify cards
- pedagogy cards

---

### D. 运行态索引

这层很多人会漏掉，但对你很重要。

它回答的是：

> **当前这轮会话，已经发生了什么？**

包括：
- 当前 `step_id`
- 当前 `hint_level`
- `attempt_count`
- 是否刚触发过 anti-card
- 学生上一轮说了什么
- 当前是否处于 recovery / retry
- 当前是否需要 verify

这层决定 tutor 当前 prompt 的实时边界。

---

## 1.3 你最需要的不是一个索引，而是“多级索引”

我建议你把索引系统做成下面这个逻辑：
Plain text
题目索引
→ 答案索引
→ 步骤索引
→ 卡片索引
→ 运行态过滤
→ ContextPack

也就是说，先粗后细。

### 具体流转可以是这样：

#### 第一步：题目定位

从 `problem_id` 找到：
- 默认答案
- 候选解法
- 所属章节

#### 第二步：当前步定位

从 `current_step_id` 找到：
- 当前步文本
- 当前步类型
- 当前步目标
- 上一步 / 下一步

#### 第三步：卡片检索

根据：
- `step_type`
- `chapter_path`
- `branch_id`
- `mislead pattern`

命中：
- primary method card
- secondary method card
- anti-card
- verify card

#### 第四步：运行态过滤

如果当前：
- 已经命中过 anti-card
- hint_level 已到 2
- 学生在问 why-question
- 当前步被标成高风险 verify

则对最终 context 做裁剪。

---

## 1.4 这里的数据库我建议你这么选

### 关系型数据库：PostgreSQL

这是主库，存：
- problems
- solutions
- steps
- cards
- links
- traces
- learner profile

因为这些东西结构很强，关系很多，适合 SQL。

### 向量索引：pgvector 或单独向量库

只做辅助，不做唯一检索来源。

用来处理：
- 卡片相似度
- 误导路径相似度
- 学生自由文本与 card 的语义贴近度

但我建议你始终记住：

> **在数学 tutor 里，结构索引优先于向量索引。**

不要走成“全靠 embedding 猜”。

### 全文索引：Postgres FTS 或 Elasticsearch（可后置）

用于：
- 老师查题
- 教研查卡
- 创意工坊搜索

如果一开始资源有限，后置也可以。

---

## 1.5 你需要一个专门的 Context Builder

这会是数据库索引系统的核心服务，不是表本身。

建议单独做一个服务：

### `ContextBuilder.build(session_id)`

输入：
- `session_id`

内部流程：
1. 查当前题
2. 查当前解法分支
3. 查当前 step
4. 查 step 绑定的 cards
5. 查当前 runtime flags
6. 查 learner profile（可选）
7. 组装 `ContextPack`

输出：
JSON
{
  "problem": {...},
  "step": {...},
  "method_cards": [...],
  "anti_cards": [...],
  "verify_cards": [...],
  "student_profile": {...},
  "runtime_constraints": {...}
}

这样你的 tutor engine 就不用直接拼数据库了。

---

# 二、post learning：从 trace 到画像，再到 agent 调参

这里我想先帮你把概念掰正：

## 2.1 不要把它叫“微调模型”

至少在前期别这么理解。

你真正要做的是三层东西：

### 第一层：记录

记录这次 tutoring 发生了什么

### 第二层：归纳

归纳这个学生在哪些地方总卡壳

### 第三层：反馈

把这些归纳结果反过来影响：
- 检索
- hint level 起点
- verify 触发阈值
- bailout 策略
- 推荐题

这已经非常强了，而且比“真 fine-tuning”现实得多。

---

## 2.2 post learning 可以拆成三类记忆

### A. Session Memory（短期）

本次会话有效，结束就弱化。

包括：
- 当前题已讲到哪
- 当前步用了几级提示
- 当前已经触发过哪些 anti-card
- 当前学生心态是否焦躁

这个主要服务当前 session。

---

### B. Learner Profile（中期）

这是你最重要的一层。

它回答：
- 这个学生在哪些知识点强 / 弱
- 常犯哪类误导
- 通常接受哪种提示强度
- 会不会“假懂”
- 适合哪种节奏

建议至少建成这种结构：
JSON
{
  "student_id": "stu_001",
  "knowledge_mastery": {
    "double_angle_formula": 0.72,
    "fixed_point_analysis": 0.41,
    "quadratic_rooting": 0.83
  },
  "misconception_profile": {
    "aux_angle_misuse": 6,
    "domain_ignored": 4,
    "wrong_factorization": 3
  },
  "interaction_style": {
    "preferred_hint_start_level": 1,
    "verify_sensitivity": "high",
    "pace_mode": "normal"
  }
}

这层会非常值钱。

---

### C. Pedagogical Update（长期）

这层不是更新学生，而是更新系统。

也就是：
- 哪张 card 最常失效
- 哪张 anti-card 最常命中
- 哪种 step 最常触发 bailout
- 哪种 verify 问题最能防假懂

这会回流到教研端，改 card，不是直接改模型。

---

## 2.3 你最该采集的 learning signals

不要只存“对/错”。
真正有价值的是这些信号：

### 认知信号
- 当前步是否一次过
- 用到哪一级 scaffold
- 是否需要 direct explain
- verify 是否通过

### 误导信号
- 命中了哪张 anti-card
- 是哪种错误分支
- 同类错误是否重复出现

### 节奏信号
- 每步轮数
- 回答速度
- 是否频繁说“懂了懂了”
- 是否经常 give up

### 行为信号
- 学生更喜欢先问 why 还是先做
- 是否倾向口头说对但不写式
- 是否喜欢跳步

这些信号会比“最后做对没”更有价值。

---

## 2.4 post learning 最应该影响哪些系统参数

这一步最重要。
因为你不是为了存数据而存数据，而是为了回调 agent。

我建议它优先影响下面这些东西：

### 1. hint 起始层级

例如：
- 强学生默认从 `L0/L1` 开始
- 弱学生默认从 `L1/L2` 开始

### 2. verify 强度

例如：
- 容易假懂的学生，更频繁触发 `Verify_Hard`
- 扎实型学生，多用 `Verify_Light`

### 3. anti-card 优先级

例如：
- 某学生三次都误用辅助角公式
- 那系统以后在相似场景里提前提高这张 anti-card 的权重

### 4. bailout 策略

例如：
- 对容易挫败的学生，减少长时间 retry
- 对耐心高的学生，允许多轮 scaffold

### 5. 推荐题

例如：
- 当前学生在固定点分析反复出错
- 就推固定点分析的前置和同类题

这才叫“学习用户个人信息”。
不是学他的隐私，而是学他的**学习结构**。

---

## 2.5 我强烈建议你限制“个人信息”的边界

这里一定要非常清楚。

你应该学习的是：
- 学习习惯
- 学习偏好
- 提示响应方式
- 常见误区
- 掌握度

而不是乱存：
- 敏感身份
- 家庭背景
- 精细隐私
- 与教学无关的个人特征

所以更准确的说法不是：

> 学习用户个人信息

而是：

> 学习用户的 tutoring profile / learner profile

这会更专业，也更安全。

---

# 三、这两个模块之间怎么联动

这其实是最重要的一点。

## 3.1 数据库索引系统负责“当前给模型看什么”

## 3.2 post learning 负责“以后优先给什么、少给什么”

也就是说：
- 索引系统决定 **上下文组成**
- post learning 决定 **上下文偏置**

举个例子：

### 现在这一题

step 绑定了：
- method card A
- anti-card B
- verify card C

### post learning 发现

这个学生经常：
- 命中 anti-card B
- verify 总是假懂
- 对 direct explanation 反应反而更好

那么下一次同类 step：
- anti-card B 提前置顶
- verify_mode 默认 hard
- scaffold 起点从 L1 改成 L2

这就是个性化，而不需要动模型权重。

---

# 四、我建议你把这两个模块落成两个服务

## 4.1 `ContextIndexService`

负责从数据库里组上下文。

### 核心方法
- `get_problem_bundle(problem_id)`
- `get_step_bundle(step_id)`
- `build_context_pack(session_id)`

---

## 4.2 `PostLearningService`

负责从 trace 里抽学习信号，更新画像。

### 核心方法
- `extract_learning_signals(session_id)`
- `update_learner_profile(student_id, signals)`
- `update_card_stats(problem_id, trace)`
- `generate_next_recommendations(student_id)`

这样边界会很清晰。

---

# 五、我给你一个最终的系统关系图
Mermaid
flowchart TD
    A[题目/答案/步骤/Cards 数据库] --> B[ContextIndexService]
    B --> C[ContextPack]

    C --> D[Tutoring State Machine]
    D --> E[Session Trace]

    E --> F[PostLearningService]
    F --> G[Learner Profile]
    F --> H[Card Stats / 教研反馈]
    F --> I[Recommendation Signals]

    G --> B
    H --> A
    I --> B

这张图的意思很简单：
- 静态库先构成上下文
- 状态机用这个上下文教学
- 教学 trace 再反哺 learner profile 和 card 质量
- 下次索引时就更聪明

---

# 六、你现在的最小落地顺序

如果你问我：这两块先做哪一部分？

我建议顺序是：

### 第一步：先把数据库索引系统做出来

最少做到：
- 题目
- 答案
- 步骤
- cards
- links
- context builder

### 第二步：再做 post learning 的最小版

只做：
- trace 记录
- scaffold level 统计
- anti-card 命中统计
- verify 通过率
- 简单 learner profile 更新

### 第三步：最后再做真正的个性化调参

例如：
- 默认 hint level
- verify 模式偏置
- anti-card 排序偏置
- 推荐题

---

# 七、你现在的模块拆分已经很合理了

到目前为止，三大模块其实已经很清楚了：

## 模块 1：状态机教学引擎

你已经基本打通了

## 模块 2：数据库索引系统

负责静态知识 → 当前上下文

## 模块 3：post learning

负责历史 trace → 个人适配 / 系统优化

如果你把这三个模块都做出来，这个项目就已经不像普通 AI demo 了，而很像一个真正的 **AI 教学基础设施**。

一句话总结：

> **数据库索引系统解决“现在该给模型看什么”；
> post learning 解决“下次更该给模型看什么”。**

如果你愿意，下一步我可以继续直接帮你做这两块里最适合开工的东西之一：
**数据库 schema 终稿**，或者 **learner profile / learning signals 的 JSON 结构终稿**。

---
# 2、数据库索引系统中包含三层内容：...
**问题详情：**
数据库索引系统中包含三层内容：知识卡片：具体格式如下{"subject_meta":{"topic": "高中数学-三角函数-诱导公式",
    "core_logic": "奇变偶不变，符号看象限",
    "cognitive_load_focus": "坐标映射与函数值的正负判定"},
  "deep_pitfalls":[{"error_type": "原像混淆",
      "description": "在判定符号时，错误地根据‘变形后’的函数名来判断。例如：sin(π/2+α) 变名为 cosα 后，学生根据第一象限的 cos 符号来定正负，而非根据原函数 sin 在第二象限的正号。"},{"error_type": "象限盲区",
      "description": "默认 α 为锐角是公式推导的约定，但在综合题中 α 可能是任意角，学生在处理 sin(π+α) 时由于 α 的不确定性导致不敢应用公式或符号判断死板。"}],
  "anti_triggers":[{"trap_pattern": "非 π/2 整数倍的偏移",
      "example": "sin(π/3 + α)",
      "correct_strategy": "此题型属于‘两角和与差的三角公式’，诱导公式仅适用于轴线角的偏移。"},{"trap_pattern": "系数非 1 的复合变形",
      "example": "sin(2α + π)",
      "correct_strategy": "需先观察整体结构，提取系数或利用周期性，而非生搬硬套公式名。"}],
  "scaffolded_hints":{"Level_1_Observation": "请盯着这个角看：它和我们熟悉的轴线（x轴或y轴）距离是多少？它是从哪个象限‘转过来’的？",
    "Level_2_Direction": "观察角度的构成，判断它是满足‘k·π ± α’还是‘(k+1/2)π ± α’？这决定了你的函数名是否需要发生‘性转’（变名）。",
    "Level_3_Critical_Step": "现在进入最关键的一步：假设 α 是一个很小的锐角，那么整个角(kπ/2 ± α) 落在第几象限？在这个象限里，**原来的**函数符号是正还是负？",
    "Level_4_Direct_Explanation": "直接应用口诀：1.‘奇变偶不变’，看 π/2 的倍数是奇数还是偶数；2.‘符号看象限’，将 α 视作锐角，判断原函数在对应象限的正负号，直接写出结果即可。"},
  "pedagogical_note": "教学时应强调单位圆的旋转对称性，将公式推导转化为几何直觉，而非死记硬背 5-6 组公式。"} 题目和答案：题目和答案中的特定位置包含了对于特定知识卡片的索引。在卡片层面上有多级索引。

---
**回答（Answer）：**

对，这样数据库索引系统就基本成型了。
而且你现在这句话其实已经把它说清楚了：

> **数据库索引系统 = 卡片层 + 题解层 + 锚点/链接层。**

我建议你正式把它设计成 **三层对象 + 两类索引 + 一套上下文构造器**。
这样以后不管是检索、版本控制，还是 tutor 运行时拼上下文，都会很顺。

---

# 一、先把三层内容说死

## 1. 卡片层（Card Layer）

你给的这张 JSON 卡片已经非常接近可用形态了。
它包含了：
- 学科元信息 `subject_meta`
- 深层误区 `deep_pitfalls`
- 反触发 `anti_triggers`
- 分级脚手架 `scaffolded_hints`
- 教学备注 `pedagogical_note`

这说明卡片不是“知识摘要”，而是**教学对象**。
这一点非常重要。

我建议你把卡片层再明确成四类：

### A. Method Card

讲“这一步该怎么做”

### B. Anti-Mislead Card

讲“这一步看起来像什么，但其实不能这么做”

### C. Verify Card

讲“这一小步如何验算学生是不是假懂”

### D. Pedagogy Card

讲“这一类内容怎么讲更符合认知规律”

你现在这个“诱导公式”例子，本质上是 **Method + Anti-Mislead + Pedagogy** 混合卡。
第一版没问题，但后面如果规模大了，我建议拆开。

---

## 2. 题目/答案层（Problem/Solution Layer）

这一层负责放：
- 题目原文
- 标准答案 / 多解法答案
- 切好的步骤
- 每一步的目标与类型

这一层的核心不是“存文本”，而是：

> **把答案切成可绑定卡片的最小教学单元。**

所以建议你至少有：
- `problems`
- `solutions`
- `solution_steps`

而不是只把整题整答作为大文本。

---

## 3. 锚点/链接层（Anchor / Link Layer）

这是最关键也最容易被低估的一层。

你说：

> 题目和答案中的特定位置包含了对于特定知识卡片的索引

这句话非常对，但实现上我建议不要直接把 card_id 写进原始文本里。
更稳的做法是引入 **锚点对象（anchor）**。

也就是：
- 文本还是原文本
- 关联关系不写死在文本里
- 单独维护一张“某段文本 ↔ 某张卡”的链接表

这样你后面卡片升级、答案修订、步骤重切都不会炸。

---

# 二、我建议的最终结构

## 2.1 卡片层表设计

### `cards`

存卡片基本信息
SQL
id
card_type
subject
topic_path
title
status
current_version
author
reviewer
created_at
updated_at

### `card_versions`

存每个版本的 JSON 正文
SQL
id
card_id
version
content_json
change_log
created_at
created_by

你给的 JSON 就适合放在 `content_json` 里，但有几个关键字段最好抽出来做结构字段：
- `subject_meta.topic`
- `subject_meta.core_logic`
- `subject_meta.cognitive_load_focus`

因为这些字段未来会被频繁检索。

---

## 2.2 题目/答案层表设计

### `problems`
SQL
id
title
stem_text
chapter
topic
difficulty
source
created_at
updated_at

### `solutions`
SQL
id
problem_id
solution_name
solution_text
is_default
author
version
created_at
updated_at

### `solution_steps`
SQL
id
solution_id
step_order
step_title
step_text
step_goal
step_type
branch_id
parent_step_id
created_at
updated_at

其中 `step_type` 很重要，它是你多级索引的桥梁。
例如：
- `induction_formula`
- `double_angle_expansion`
- `substitution`
- `fixed_point_analysis`
- `quadratic_rooting`

---

## 2.3 锚点/链接层表设计

### `text_anchors`

表示某段文本中的一个可引用位置
SQL
id
owner_type        -- problem / solution / solution_step
owner_id
anchor_type       -- span / step / token_range / formula
start_offset
end_offset
anchor_text
created_at

这里的 `owner_type` 非常关键：
- 如果锚在题目里，就挂 `problem`
- 如果锚在整段答案里，就挂 `solution`
- 如果锚在某一步里，就挂 `solution_step`

我建议以后尽量多挂在 `solution_step`，因为教学系统最终是按步骤运行的。

---

### `anchor_card_links`

真正的索引关系表
SQL
id
anchor_id
card_id
link_role
confidence
evidence_note
created_at

`link_role` 建议枚举：
- `primary_method`
- `secondary_method`
- `anti_mislead`
- `verify`
- `pedagogy`

这个表就是整个索引系统的命根子。

---

# 三、卡片层面的多级索引应该怎么做

你说“在卡片层面上有多级索引”，这个我非常赞同。
我建议至少做五级。

## 第一级：学科路径索引

最粗层。

例如：
Plain text
高中数学
→ 三角函数
→ 诱导公式

字段可叫：
- `subject`
- `chapter`
- `topic_path`

---

## 第二级：方法动作索引

回答“这张卡是用来做什么动作的”。

例如：
- 变名
- 判象限
- 换元
- 主元求根
- 不动点分析

字段可叫：
- `operation_type`
- `method_family`

这一级对 tutor 很重要，因为学生问的往往不是“这是哪章”，而是“这一步要干嘛”。

---

## 第三级：误区索引

回答“这张卡通常防什么错”。

你这个 JSON 里的 `deep_pitfalls` 已经很好了。
建议抽出误区标签：
- `pitfall_tags`
- `misconception_types`

例如：
- `原像混淆`
- `象限盲区`
- `错看变名后的函数`
- `误把π/3当诱导公式触发角`

这一级会直接支持 anti-mislead。

---

## 第四级：触发/反触发索引

回答：
- 什么情况下该上这张卡
- 什么情况下不该上这张卡

你现在的 `anti_triggers` 已经是很好的素材。
建议结构化成：
- `trigger_patterns`
- `anti_trigger_patterns`

这里最好支持：
- 文本标签
- 公式模板
- 步骤类型
- 错路类型

---

## 第五级：教学策略索引

回答“这张卡适合怎么讲”。

例如：
- 认知负荷高不高
- 适合先几何直观还是先代数
- L0-L3 每层怎么提示
- 验算题怎么出

这一级以后会直接影响：
- 默认 hint 起点
- 验算模式
- Bailout 策略

---

# 四、题目和答案中的“特定位置索引”怎么做更稳

这是你设计里很亮的一点，但要实现得足够稳。

我建议你把“位置”分成三种层级，不要只依赖字符 offset。

## 4.1 文本 span 锚点

适合题目原文中的局部表达，比如：
- “sin(π/2+α)”
- “a_{n+1}^2-a_{n+1}=a_n”

优点：细
缺点：脆，文本一改 offset 就变

---

## 4.2 步骤锚点

适合答案中绝大多数教学绑定。

例如：
- Step 2：把隐式递推视为关于 $a_{n+1}$ 的二次方程
- Step 4：解不动点方程

这比 span 稳太多，因为教学本来就是按步推进的。

---

## 4.3 公式锚点 / 结构锚点

如果你后面走得更深，可以给公式解析出结构树，再挂锚点。
但这个可以后置，不必一开始就上。

第一版建议：

> **题目里允许 span 锚点；答案里尽量用 step 锚点。**

---

# 五、你这个索引系统最后应该怎么给 tutor 用

最终不是“查到了什么就全塞给模型”，而是要经过一个 **Context Builder**。

## 输入
- `session_id`
- 当前 `problem_id`
- 当前 `step_id`
- 当前 `hint_level`
- 当前 `router_output`

## 输出

一个 **ContextPack**

例如：
JSON
{
  "problem_context": {
    "stem_text": "...",
    "chapter": "高中数学-三角函数"
  },
  "step_context": {
    "step_id": "step_03",
    "step_goal": "判断函数名是否变换并确定符号",
    "step_text": "sin(π/2+α) = ?"
  },
  "method_cards": [
"card_induction_formula_main"
  ],
  "anti_cards": [
"card_induction_formula_non_axis_angle"
  ],
  "verify_cards": [
"card_verify_quadrant_sign"
  ],
  "runtime_constraints": {
    "hint_level": 1,
    "allowed_scope": "current_step_only"
  }
}

所以数据库索引系统真正的目标不是“查库”，而是**为当前一步拼一个最小教学上下文包**。

---

# 六、我对你当前设计的两个关键建议

## 建议 1：用“关系型 + JSONB 混合模式”

不要纯 JSON，也不要纯拆表。

最佳做法是：
- **核心检索字段结构化**
- **复杂教学正文放 JSONB**

也就是：

### 结构化字段
- `topic_path`
- `card_type`
- `method_family`
- `pitfall_tags`
- `trigger_patterns`
- `status`
- `version`

### JSONB 字段
- `deep_pitfalls`
- `anti_triggers`
- `scaffolded_hints`
- `pedagogical_note`

这样后面检索和版本管理会舒服很多。

---

## 建议 2：一定要做版本控制

因为你这个索引系统不是死题库，而是教研资产库。

所以每次 card 更新，最好不是覆盖，而是新增版本。
否则以后你根本没法回答：
- 为什么上周这题会命中这张卡，这周又不命中了？
- 是 card 改了，还是链接改了，还是步骤切分改了？

所以：
- `cards` 放当前引用信息
- `card_versions` 放完整历史
- `anchor_card_links` 也最好可带版本或时间戳

---

# 七、你现在这块已经很像“教育界的 GitHub 索引层”了

因为你不只是存知识，而是在存：
- 知识对象
- 使用场景
- 误区
- 反触发
- 提示层级
- 与具体题目步骤的绑定关系

这已经不是普通题库了，而是：

> **可执行教学对象库 + 题解绑定索引系统**

这个方向是对的，而且和你前面说的创意工坊、老师共建，天然衔接。

---

# 八、我建议你下一步直接定三份东西

如果你要把这块真正落地，我建议马上定：

### 1. `cards` / `card_versions` / `anchor_card_links` 的 SQL schema

这是骨架。

### 2. `solution_steps` 的最小字段集合

这是教学系统能不能按步运行的关键。

### 3. `ContextBuilder` 的输入输出 JSON

这是索引系统和状态机系统之间的接口。

---

一句话总结：

> **你这个数据库索引系统不是“存卡片”，而是“把题目、答案步骤和教学卡片在多个粒度上绑定起来，并在运行时按当前步打包成上下文”。**

这个方向完全对，而且已经非常接近可以开工的设计了。

如果你愿意，下一步我可以直接把这一块进一步压成两份工程文件草案：
1. **数据库 schema 终稿（SQL / ORM 级）**
2. **ContextBuilder 的标准输入输出格式**

---
# 3、如果你愿意，下一步我可以直接把...
**问题详情：**
如果你愿意，下一步我可以直接把这一块进一步压成两份工程文件草案：数据库 schema 终稿（SQL / ORM 级）

ContextBuilder 的标准输入输出格式 按照之前的说法帮我好好想一想schema怎样更贴合教学实际，并且定稿schema和ContextBuilder，要求具有一定可维护性

---
**回答（Answer）：**

下面我不给“很多可能性”，而是直接给你一版**定稿导向**的设计。目标只有两个：
1. **数据库 schema 贴合真实教学，而不是只贴合数据存储**
2. **ContextBuilder 能稳定、可维护地构造“当前这一步”所需上下文**

我会先给设计原则，再给 **PostgreSQL 级 schema**，最后给 **ContextBuilder 标准输入输出** 和 **构造规则**。
这版会刻意追求两件事：**可维护** 和 **可迭代**。

---

# 一、先定三条总原则

## 原则 1：运行时的最小原子不是“整题”，而是“解题步骤节点”

你前面的整个状态机都是围绕“当前一步”展开的，所以数据库的主索引对象不能只是题目和整份答案，而必须是：
- 题目 `problem`
- 解法 `solution`
- 解题步骤节点 `solution_node`

也就是说，**step/node 是运行时一等公民**。

---

## 原则 2：教学内容一律采用“Head + Revision”版本模式

凡是会被老师改、会被教研审、会被学生使用的内容，都不能只存最新版。
所以这三类对象都要版本化：
- 题目与题面
- 解法与步骤图
- Cards

这不是洁癖，而是为了回答以后一定会出现的问题：
- 为什么这道题上周讲法和这周不一样？
- 是卡片变了，还是答案步骤变了？
- 某次效果变差，到底是 link 错了还是内容升级了？

---

## 原则 3：上下文构造要“结构化选择”，不是“把库里的东西一股脑塞给模型”

所以数据库的职责不是“存储”，而是支持 `ContextBuilder` 做这件事：

> 在某个 `Session + 当前 step + 当前 state` 下，精准挑出
> **最少但足够**的：
> - 当前步文本
> - 当前步目标
> - 主方法卡
> - 误导卡
> - 验算卡
> - 学生适配偏置
> - 输出约束

---

# 二、最终推荐 schema（PostgreSQL 版）

我推荐用 **PostgreSQL + JSONB + 可选 pgvector**。
原则是：
- **关系强的东西** 用结构化表
- **教学正文** 放 JSONB
- **相似检索** 走可选 embedding，不做唯一依据

---

## 2.1 目录结构上的对象分层

最终对象分成 6 组：
1. 课程与章节树（taxonomy）
2. 题目与题面版本（problems）
3. 解法与步骤图（solutions / nodes / edges）
4. 教学卡片与版本（cards）
5. 步骤-卡片-文本锚点的链接层（links / anchors）
6. 会话与学习画像（sessions / traces / learner profiles）

---

# 三、Schema 定稿（SQL 草案）

下面这版是“够严肃，可以开工”的程度。

---

## 3.1 课程与章节树

> 这里我建议你不要只用字符串路径，而是做 **DAG**。
> 因为一个方法可能同时属于多个母章节。
SQL
-- 1) 学科
createtable subjects (
  id                uuid primarykey,
  code              text notnullunique,
  name              text notnull,
  created_at        timestamptz notnulldefault now()
);

-- 2) 主题节点：章节 / 知识点 / 方法族 / 误区族
createtable topic_nodes (
  id                uuid primarykey,
  subject_id        uuid notnullreferences subjects(id),
  code              text notnullunique,
  name              text notnull,
  node_kind         text notnullcheck (node_kind in (
'chapter', 'topic', 'method_family', 'misconception_family'
                    )),
  canonical_path    text notnull,
  is_active         booleannotnulldefaulttrue,
  created_at        timestamptz notnulldefault now()
);

-- 3) 主题边：支持 contains / prerequisite / related
createtable topic_edges (
  parent_topic_id   uuid notnullreferences topic_nodes(id),
  child_topic_id    uuid notnullreferences topic_nodes(id),
  relation_type     text notnullcheck (relation_type in (
'contains', 'prerequisite', 'related'
                    )),
  weight            numeric(4,3) notnulldefault1.0,
primarykey (parent_topic_id, child_topic_id, relation_type)
);

create index idx_topic_nodes_subject_path on topic_nodes(subject_id, canonical_path);
create index idx_topic_edges_child on topic_edges(child_topic_id);

### 为什么这样更贴合教学
- `chapter` 解决“这是哪章”
- `method_family` 解决“这一步在做什么动作”
- `misconception_family` 解决“学生现在最像哪种错法”

这比只用“高中数学-三角函数-诱导公式”一串字符串更可维护。

---

## 3.2 题目与题面版本
SQL
createtable problems (
  id                uuid primarykey,
  subject_id        uuid notnullreferences subjects(id),
  stable_slug       text notnullunique,
  status            text notnullcheck (status in ('draft', 'review', 'published', 'archived')),
  default_revision_id uuid,
  created_by        text,
  created_at        timestamptz notnulldefault now(),
  updated_at        timestamptz notnulldefault now()
);

createtable problem_revisions (
  id                uuid primarykey,
  problem_id        uuid notnullreferences problems(id),
  rev_no            integernotnull,
  title             text notnull,
  stem_text         text notnull,
  source            text,
  difficulty        smallintcheck (difficulty between1and10),
  answer_format_hint text,
  is_published      booleannotnulldefaultfalse,
  created_by        text,
  created_at        timestamptz notnulldefault now(),
unique (problem_id, rev_no)
);

altertable problems
addconstraint fk_problem_default_revision
foreignkey (default_revision_id) references problem_revisions(id);

-- 题目到主题的粗粒度绑定
createtable problem_revision_topics (
  problem_revision_id uuid notnullreferences problem_revisions(id),
  topic_id            uuid notnullreferences topic_nodes(id),
role                text notnullcheck (rolein ('primary', 'secondary', 'related')),
  confidence          numeric(4,3) notnulldefault1.0,
primarykey (problem_revision_id, topic_id, role)
);

create index idx_problem_revision_topics_topic on problem_revision_topics(topic_id);

### 为什么要题目版本化
- 题面会改
- 来源会改
- 难度标注会改
- 相关 topic 也可能被重标

---

## 3.3 解法与步骤图（最重要）

这里我建议你不要只做 `step_order`，而是直接做 **node + edge**。
因为你前面已经明确要支持多解法、分支、回退。
SQL
createtable solutions (
  id                uuid primarykey,
  problem_id        uuid notnullreferences problems(id),
  solution_label    text notnull,   -- 例如：标准解 / 代数法 / 图像法
  status            text notnullcheck (status in ('draft', 'review', 'published', 'archived')),
  default_revision_id uuid,
  is_default        booleannotnulldefaultfalse,
  created_by        text,
  created_at        timestamptz notnulldefault now(),
  updated_at        timestamptz notnulldefault now()
);

createtable solution_revisions (
  id                uuid primarykey,
  solution_id       uuid notnullreferences solutions(id),
  rev_no            integernotnull,
  summary_text      text,
  entry_node_id     uuid,
  is_published      booleannotnulldefaultfalse,
  created_by        text,
  created_at        timestamptz notnulldefault now(),
unique (solution_id, rev_no)
);

altertable solutions
addconstraint fk_solution_default_revision
foreignkey (default_revision_id) references solution_revisions(id);

-- 步骤节点：运行时最关键的对象
createtable solution_nodes (
  id                uuid primarykey,
  solution_revision_id uuid notnullreferences solution_revisions(id),
  node_key          text notnull,   -- 稳定键，如 step_01_rooting
  node_kind         text notnullcheck (node_kind in (
'step', 'checkpoint', 'branch_gate', 'recap'
                    )),
  order_hint        numeric(8,3) notnulldefault0,
  title             text,
  step_goal         text notnull,
  step_text         text notnull,
  step_type         text notnull,   -- 如 quadratic_rooting / induction_formula
  difficulty_hint   smallintcheck (difficulty_hint between1and5),
  token_budget_hint integer,
  created_at        timestamptz notnulldefault now(),
unique (solution_revision_id, node_key)
);

-- 步骤之间的图关系
createtable solution_edges (
  id                uuid primarykey,
  solution_revision_id uuid notnullreferences solution_revisions(id),
  from_node_id      uuid notnullreferences solution_nodes(id),
  to_node_id        uuid notnullreferences solution_nodes(id),
  edge_type         text notnullcheck (edge_type in (
'next', 'branch', 'rollback', 'optional'
                    )),
  edge_label        text,
  guard_expr        text,    -- 条件表达式，可为空
  priority          integernotnulldefault0,
  created_at        timestamptz notnulldefault now()
);

create index idx_solution_nodes_revision_order on solution_nodes(solution_revision_id, order_hint);
create index idx_solution_nodes_step_type on solution_nodes(step_type);
create index idx_solution_edges_from on solution_edges(from_node_id);

### 为什么 node/edge 比 step_order 更贴合教学
- 多解法天然支持
- rollback 天然支持
- branch_switch 天然支持
- Step-FSM 返回 `ADVANCE / ROLLBACK / SWITCH_BRANCH` 时，Problem-FSM 可以直接查边

---

## 3.4 卡片与版本

下面这个设计会非常贴合你现在的 JSON card 思路。
SQL
createtable cards (
  id                uuid primarykey,
  stable_slug       text notnullunique,
  card_type         text notnullcheck (card_type in (
'method', 'anti_mislead', 'verify', 'pedagogy'
                    )),
  status            text notnullcheck (status in ('draft', 'review', 'published', 'archived')),
  default_revision_id uuid,
  created_by        text,
  created_at        timestamptz notnulldefault now(),
  updated_at        timestamptz notnulldefault now()
);

createtable card_revisions (
  id                uuid primarykey,
  card_id           uuid notnullreferences cards(id),
  rev_no            integernotnull,
  subject_id        uuid notnullreferences subjects(id),
  primary_topic_id  uuid references topic_nodes(id),
  title             text notnull,
  summary_text      text,
  content_json      jsonb notnull,
  is_published      booleannotnulldefaultfalse,
  created_by        text,
  created_at        timestamptz notnulldefault now(),
unique (card_id, rev_no)
);

altertable cards
addconstraint fk_card_default_revision
foreignkey (default_revision_id) references card_revisions(id);

-- 一张卡可以挂多个 topic / 误区族 / 方法族
createtable card_revision_topics (
  card_revision_id  uuid notnullreferences card_revisions(id),
  topic_id          uuid notnullreferences topic_nodes(id),
role              text notnullcheck (rolein (
'primary', 'secondary', 'prerequisite', 'misconception', 'anti_trigger'
                    )),
  confidence        numeric(4,3) notnulldefault1.0,
primarykey (card_revision_id, topic_id, role)
);

-- 卡片之间的依赖或配套关系
createtable card_relations (
  from_card_id      uuid notnullreferences cards(id),
  to_card_id        uuid notnullreferences cards(id),
  relation_type     text notnullcheck (relation_type in (
'prerequisite', 'sibling', 'anti_for', 'verify_for', 'supersedes'
                    )),
  weight            numeric(4,3) notnulldefault1.0,
primarykey (from_card_id, to_card_id, relation_type)
);

-- 每张卡挂测试题和失败样例
createtable card_examples (
  id                uuid primarykey,
  card_id           uuid notnullreferences cards(id),
  example_type      text notnullcheck (example_type in (
'positive', 'negative', 'failure', 'verify'
                    )),
  problem_id        uuid references problems(id),
  solution_node_id  uuid references solution_nodes(id),
  note_text         text,
  created_at        timestamptz notnulldefault now()
);

create index idx_card_revision_topics_topic on card_revision_topics(topic_id);
create index idx_card_revisions_json on card_revisions using gin (content_json);

### 关于你那张 JSON 卡片

我建议直接把它放进 `card_revisions.content_json`，但把这些字段抽出来做结构字段或 topic 关系：
- `subject_meta.topic` → `primary_topic_id + canonical_path`
- `deep_pitfalls.error_type` → `card_revision_topics.role = misconception`
- `anti_triggers.trap_pattern` → `card_revision_topics.role = anti_trigger`

这样既保留原始教学内容，又方便检索。

---

## 3.5 链接层：步骤 / 文本锚点 ↔ Cards

这里是整个索引系统最关键的一层。

### 先说设计选择

我不建议把 `card_id` 直接写死进题目文本。
更维护友好的方案是：
- 文本照常存
- 单独建锚点
- 用 link 表关联 card

### Schema
SQL
-- 文本锚点：可挂在题面 revision 或 solution node 上
createtable text_spans (
  id                uuid primarykey,
  span_scope        text notnullcheck (span_scope in ('problem_revision', 'solution_node')),
  problem_revision_id uuid references problem_revisions(id),
  solution_node_id  uuid references solution_nodes(id),
  start_offset      integernotnull,
  end_offset        integernotnull,
  span_label        text,       -- 如 formula / phrase / local_goal
  span_text         text notnull,
  created_at        timestamptz notnulldefault now(),
check (
    (span_scope ='problem_revision'and problem_revision_id isnotnulland solution_node_id isnull)
or
    (span_scope ='solution_node'and solution_node_id isnotnulland problem_revision_id isnull)
  )
);

-- 直接把步骤节点绑定到卡
createtable node_card_links (
  id                uuid primarykey,
  solution_node_id  uuid notnullreferences solution_nodes(id),
  card_id           uuid notnullreferences cards(id),
  pinned_card_revision_id uuid references card_revisions(id),
  link_role         text notnullcheck (link_role in (
'primary_method', 'secondary_method', 'anti_mislead', 'verify', 'pedagogy'
                    )),
  confidence        numeric(4,3) notnulldefault1.0,
  evidence_note     text,
  created_by        text,
  created_at        timestamptz notnulldefault now()
);

-- 更细粒度地把局部 span 绑定到卡
createtable span_card_links (
  id                uuid primarykey,
  span_id           uuid notnullreferences text_spans(id),
  card_id           uuid notnullreferences cards(id),
  pinned_card_revision_id uuid references card_revisions(id),
  link_role         text notnullcheck (link_role in (
'primary_method', 'secondary_method', 'anti_mislead', 'verify', 'pedagogy'
                    )),
  confidence        numeric(4,3) notnulldefault1.0,
  evidence_note     text,
  created_by        text,
  created_at        timestamptz notnulldefault now()
);

create index idx_node_card_links_node_role on node_card_links(solution_node_id, link_role);
create index idx_span_card_links_span_role on span_card_links(span_id, link_role);

### 为什么这样最贴合教学实际
- **运行时主要按 step/node 取卡**，所以 `node_card_links` 是一等公民
- **题目或步骤里的局部公式** 需要更细粒度时，再用 `text_spans + span_card_links`
- `pinned_card_revision_id` 允许你锁定某个版本，保证可复现
- 不锁版本时，可默认用 `cards.default_revision_id`

---

## 3.6 会话与 post-learning 基础表（最小版）

你虽然这轮主要问索引系统，但为了 ContextBuilder 可维护，我建议把运行态表顺手定下来。
SQL
createtable tutoring_sessions (
  id                uuid primarykey,
  student_id        text notnull,
  problem_id        uuid notnullreferences problems(id),
  problem_revision_id uuid notnullreferences problem_revisions(id),
  solution_id       uuid notnullreferences solutions(id),
  solution_revision_id uuid notnullreferences solution_revisions(id),
  current_node_id   uuid references solution_nodes(id),
  session_state     text notnull,
  problem_state     text notnull,
  step_state        text notnull,
  feature_flags     jsonb notnulldefault'{}'::jsonb,
  started_at        timestamptz notnulldefault now(),
  ended_at          timestamptz
);

createtable tutoring_turns (
  id                uuid primarykey,
  session_id        uuid notnullreferences tutoring_sessions(id),
  turn_no           integernotnull,
role              text notnullcheck (rolein ('student', 'assistant', 'system')),
  content_text      text notnull,
  created_at        timestamptz notnulldefault now()
);

createtable trace_events (
  id                uuid primarykey,
  session_id        uuid notnullreferences tutoring_sessions(id),
  solution_node_id  uuid references solution_nodes(id),
  from_state        text notnull,
  to_state          text notnull,
  event_name        text notnull,
  router_output     jsonb,
  hint_level        integer,
  bailout_mode      text,
  verify_mode       text,
  payload_json      jsonb notnulldefault'{}'::jsonb,
  created_at        timestamptz notnulldefault now()
);

create index idx_trace_events_session_node on trace_events(session_id, solution_node_id, created_at);

---

## 3.7 Learner Profile（留最小位点即可）
SQL
createtable learner_profiles (
  student_id        text primarykey,
  pedagogy_bias     jsonb notnulldefault'{}'::jsonb,
  updated_at        timestamptz notnulldefault now()
);

createtable learner_topic_stats (
  student_id        text notnull,
  topic_id          uuid notnullreferences topic_nodes(id),
  attempts          integernotnulldefault0,
  successes         integernotnulldefault0,
  avg_hint_level    numeric(4,2),
  verify_fail_count integernotnulldefault0,
  misconception_histogram jsonb notnulldefault'{}'::jsonb,
  mastery_score     numeric(5,4),
  updated_at        timestamptz notnulldefault now(),
primarykey (student_id, topic_id)
);

createtable learner_card_stats (
  student_id        text notnull,
  card_id           uuid notnullreferences cards(id),
  hit_count         integernotnulldefault0,
  helpful_count     integernotnulldefault0,
  mislead_hit_count integernotnulldefault0,
  avg_hint_needed   numeric(4,2),
  updated_at        timestamptz notnulldefault now(),
primarykey (student_id, card_id)
);

---

## 3.8 可选：embedding 表（辅助检索，不作唯一依据）
SQL
createtable retrieval_embeddings (
  id                uuid primarykey,
  owner_type        text notnullcheck (owner_type in (
'problem_revision', 'solution_node', 'card_revision'
                    )),
  owner_id          uuid notnull,
  embedding_model   text notnull,
  vector_dim        integernotnull,
  vector_payload    bytea,   -- 或使用 pgvector 的 vector 类型
  plain_text        text,
  created_at        timestamptz notnulldefault now()
);

### 这里的原则

**先结构后向量**。
embedding 是锦上添花，不是主索引。

---

# 四、这套 schema 为什么更贴合教学实际

## 4.1 它把“步骤节点”作为核心运行对象

不是整题，不是整份答案，而是当前 node。
这和你的 Step-FSM 完全对齐。

## 4.2 它把“教学卡片”当成版本化资产，不是 prompt 碎片

这对创意工坊、老师共建、review、测试都很关键。

## 4.3 它把“题目 ↔ 步骤 ↔ 卡片”的关系显式化了

不是靠模型自己猜，而是靠链接层可审计地提供上下文。

## 4.4 它支持多解法与 branch

因为 solution 是 graph，不是死链表。

## 4.5 它天然支持后续 post-learning

trace、learner stats 都有位点。

---

# 五、ContextBuilder：标准输入输出格式（定稿）

这是第二个你要立刻固定下来的东西。

---

## 5.1 ContextBuilder 的职责

它不是“搜索服务”，而是一个**上下文装配器**。
输入当前会话与运行态，输出当前 step 所需的最小上下文包。

---

## 5.2 ContextBuilder 的构建原则

### 原则 A：按状态裁剪

不同 step_state 需要的上下文不一样。
不能每次都把 method + anti + verify 全塞进去。

### 原则 B：显式 provenance

最终 ContextPack 里必须带“为什么选中这些卡”。

### 原则 C：结构化与渲染分离

ContextBuilder 应先返回结构化对象，再由 prompt renderer 生成字符串。

---

## 5.3 输入格式（定稿）

### `ContextBuildRequest`
JSON
{
  "mode": "session",
  "session_id": "sess_001",
  "target_step_state": "T_Scaffold",
  "token_budget": 2200,
  "include_learner_profile": true,
  "include_trace_tail": true,
  "runtime_overrides": {
    "hint_level": 2,
    "attempt_count": 1
  }
}

### 字段说明
- `mode`
    - `session`：正常运行态，从 session 恢复上下文
    - `preview`：老师/教研预览，不依赖 session
    
- `session_id`
    - 正常运行态时必填
    
- `target_step_state`
    - 当前要为哪个状态构建上下文
        
        如：
    - `T_Probe_NewStep`
    - `T_Anti_Mislead`
    - `T_Concept_Clarify`
    - `T_Scaffold`
    - `T_Verify`
    - `T_Step_Advance_When_Ready`
    
- `token_budget`
    - 控制输出裁剪程度
    
- `include_learner_profile`
    - 是否加入 learner bias
    
- `include_trace_tail`
    - 是否把最近几轮对话摘要带入
    
- `runtime_overrides`
    - 调试/重放时可覆盖当前 hint_level、attempt_count 等

---

## 5.4 输出格式（定稿）

### `ContextPack`
JSON
{
  "context_meta": {
    "session_id": "sess_001",
    "problem_id": "prob_001",
    "problem_revision_id": "prev_001",
    "solution_id": "sol_001",
    "solution_revision_id": "srev_001",
    "current_node_id": "node_003",
    "target_step_state": "T_Scaffold",
    "token_budget": 2200
  },
  "problem_context": {
    "title": "求 f(x)=cos2x+3sinx 的值域",
    "stem_text": "...",
    "topic_path": [
"高中数学",
"三角函数",
"二倍角公式"
    ]
  },
  "step_context": {
    "node_key": "step_02_expand_double_angle",
    "step_goal": "将 cos2x 化成只含 sinx 的表达式",
    "step_text": "由 cos2x = 1 - 2sin^2x ...",
    "step_type": "double_angle_expansion",
    "prev_step_summary": "已经识别出不能直接辅助角",
    "next_step_preview": "之后将换元为 t=sinx"
  },
  "cards_context": {
    "primary_method_cards": [
      {
        "card_id": "card_double_angle",
        "revision_id": "crev_12",
        "title": "二倍角公式（化单变量）",
        "selected_fields": {
          "core_logic": "先统一角度，再变成单变量",
          "scaffold_hint": "Level_2_Critical_Step"
        }
      }
    ],
    "anti_mislead_cards": [
      {
        "card_id": "card_aux_angle_wrong",
        "revision_id": "crev_31",
        "title": "误判为辅助角公式",
        "selected_fields": {
          "why_tempting": "...",
          "why_wrong": "...",
          "recovery_bridge": "先把 2x 化成 x 的单变量表达式"
        }
      }
    ],
    "verify_cards": [],
    "pedagogy_cards": []
  },
  "learner_context": {
    "preferred_hint_start_level": 1,
    "verify_sensitivity": "high",
    "common_misconceptions": [
"aux_angle_misuse"
    ]
  },
  "runtime_context": {
    "hint_level": 2,
    "attempt_count": 1,
    "student_last_reply": "我想用辅助角公式",
    "trace_tail_summary": [
"上一轮命中 anti-card: aux_angle_misuse"
    ]
  },
  "constraints_context": {
    "allowed_scope": "current_step_only",
    "output_contract": "local_clarification_only",
    "forbidden_actions": [
"full_solution",
"next_step_spoiler"
    ]
  },
  "provenance": [
    {
      "source": "node_card_links",
      "link_role": "primary_method",
      "card_id": "card_double_angle",
      "confidence": 0.98
    },
    {
      "source": "node_card_links",
      "link_role": "anti_mislead",
      "card_id": "card_aux_angle_wrong",
      "confidence": 0.94
    }
  ]
}

---

# 六、ContextBuilder 的构造流程（定稿）

我建议它固定成六步，保证可维护性。

## Step 1：Resolve

从 `session_id` 解析出：
- 当前题
- 当前题 revision
- 当前解法 revision
- 当前 node
- 当前状态
- hint_level / attempt_count

## Step 2：Fetch static bundles

取：
- `problem_revision`
- `solution_node`
- `prev/next node`
- `problem_revision_topics`

## Step 3：Select cards（按优先级）

优先级固定成：
1. `node_card_links` 显式绑定的卡
2. `span_card_links` 更细粒度锚点绑定的卡
3. `step_type + topic_path` 推出的默认卡
4. learner profile 带来的排序偏置
5. pedagogy overlay

## Step 4：Apply state-specific projection

根据 `target_step_state` 裁剪卡片字段。

例如：

### `T_Probe_NewStep`

只保留：
- 当前步 goal
- 当前步 text
- primary method 的 `core_logic`（内部用）
- 不给 verify，不给 next preview

### `T_Anti_Mislead`

只保留：
- anti-card 的 `why_tempting / why_wrong / recovery_bridge`

### `T_Scaffold`

只保留：
- 当前 `hint_level` 对应的 scaffold hint

### `T_Verify`

只保留：
- verify card 的 `one_step_check`

### `T_Step_Advance_When_Ready`

保留：
- 当前步总结素材
- 下一步的 very short preview（只一句，不展开）

## Step 5：Apply learner/runtime bias

例如：
- learner 常犯某个误导 → anti-card 提前
- learner 易假懂 → verify_mode = hard
- learner 偏弱 → scaffold 起点提高

## Step 6：Emit `ContextPack`

返回：
- 结构化 pack
- provenance
- token estimate

---

# 七、不同 step_state 下，ContextBuilder 该装什么（很关键）

这部分最贴近教学实际，我给你一个硬规则表。

---

## 7.1 `T_Probe_NewStep`

**目的**：探学生直觉，不剧透

装入：
- 当前题目简要
- 当前步 goal
- 当前步文本（必要时精简）
- primary method card 的 `core_logic`（隐藏给系统用）
- learner 的 hint 起点偏置

不装入：
- verify card
- anti-card 全文
- 下一步内容
- 全部 scaffold 层级

---

## 7.2 `T_Probe_Retry`

**目的**：定位当前卡点

多装：
- `student_last_reply`
- `attempt_count`
- 当前 `hint_level`

不需要：
- pedagogy 长说明
- 下一步 preview

---

## 7.3 `T_Probe_Recovery`

**目的**：从错路拉回主线

装入：
- 当前步 goal
- 当前 anti-card 的 `recovery_bridge`
- 当前主方法的最小动作提示

不装入：
- 全量 anti-card
- 多张 method card

---

## 7.4 `T_Anti_Mislead`

**目的**：纠偏，不扩写

装入：
- 只装当前命中的 anti-card
- 学生上一轮错误想法
- 当前步主方法名称

输出要严格：
1. 为什么这条路诱人
2. 错在什么条件
3. 回到哪条主线

---

## 7.5 `T_Concept_Clarify`

**目的**：解释“为什么”

装入：
- 当前步 goal
- 当前主方法 card 的 `core_logic`
- 可选 `pedagogical_note`

不装入：
- 全题总思路
- 后续步骤

---

## 7.6 `T_Elicit_Work`

**目的**：逼学生把局部动作写出来

装入：
- 当前步 goal
- 学生 partial answer
- 当前步目标动作类型（例如“写成关于 a_{n+1} 的二次方程”）

---

## 7.7 `T_Scaffold`

**目的**：分级搭桥

装入：
- 只装当前 step 的 primary method card
- 只取当前 `hint_level` 的那一层 scaffold

绝对不要把 L0-L3 一起塞进 prompt。
否则模型会偷跑。

---

## 7.8 `T_Verify`

**目的**：查假懂

装入：
- verify card
- 当前步核心动作
- 可选 learner 的 verify_sensitivity

---

## 7.9 `T_Step_Advance_When_Ready`

**目的**：收尾并推进

装入：
- 当前步总结
- 下一步 very short preview
- 不需要 anti-card
- 不需要 verify card

---

## 7.10 `T_Bailout_*`

**目的**：保节奏，不拖死

### `Bailout_DirectExplain`

装入：
- 当前步主方法 card 的 L3
- 当前步 canonical transition

### `Bailout_Rollback`

装入：
- 上一步 summary
- 当前步缺失的 prerequisite

### `Bailout_MarkAndMove`

装入：
- 当前步薄弱标签
- 下一步入口信息

---

# 八、ContextBuilder 的可维护性设计

这是你特别关心的，我给几条硬规则。

## 规则 1：ContextBuilder 返回“结构化结果 + 渲染前原料”，不要直接返回 prompt

也就是说，它是：
- **selector**
- 不是 **prompt writer**

这样以后你换模型、换 prompt，都不用动数据库层。

---

## 规则 2：所有选择都必须带 provenance

每张 card 为什么进 pack，必须能解释。

例如：
- 来自显式 `node_card_link`
- 来自 `step_type` fallback
- 来自 learner bias rerank

否则系统出了问题你查不动。

---

## 规则 3：显式链接优先于语义检索

优先级永远是：

**手工绑定> 锚点绑定> 规则推断> embedding 辅助**

这条不能反。

---

## 规则 4：所有 revision 默认只读 published

除非 `preview_mode = true`。
这样老师审核中的 draft 不会污染学生线上运行。

---

## 规则 5：一个 state 最多只拿到“够它完成任务”的卡片字段

不要整个 JSON 卡片都放进去。
按 state 做字段投影，能显著提高可控性。

---

# 九、我给你的最终定稿结论

## 最终 schema 的核心对象
- `problems / problem_revisions`
- `solutions / solution_revisions / solution_nodes / solution_edges`
- `cards / card_revisions / card_revision_topics / card_relations / card_examples`
- `text_spans / node_card_links / span_card_links`
- `tutoring_sessions / tutoring_turns / trace_events`
- `learner_profiles / learner_topic_stats / learner_card_stats`

## 最终 ContextBuilder 的接口
- 输入：`ContextBuildRequest`
- 输出：`ContextPack`

## 最终索引原则
- **node 是运行核心**
- **card 是版本化教学对象**
- **link 层决定可审计性**
- **ContextBuilder 决定运行时最小上下文**

---

一句话收尾：

> **这套 schema 不是“题库 + 卡片库”，而是“教学图谱 + 步骤索引 + 可执行教案对象 + 运行时上下文装配系统”。**

这就是它和普通题库最大的区别。

如果你要继续，我下一步最建议做的就是这两个文件：
1. **把上面这套 SQL 转成 SQLAlchemy / Prisma / Drizzle 的 ORM 定义**
2. **把 ContextBuilder 写成一份接口文档 + 伪代码执行流程**

这样你就真能开始落库和起服务了。