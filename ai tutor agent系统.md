# 1、来帮我实现最终的设计吧，包括所...
**问题详情：**
来帮我实现最终的设计吧，包括所有的状态转移图和嵌套关系

---
**回答（Answer）：**

我先把它收束成一份可直接落地的 V6 终局设计：统一命名、画出分层状态机与嵌套关系，并补齐完整转移规则、feature flags 和中断机制。

---
**回答（Answer）：**

我会直接把它落实成可开工的终版：先给总嵌套图，再展开 Session/Problem/Step 三层转移图、共享上下文字段、关键 guard 条件和最小启用路径。

---
**回答（Answer）：**

可以，下面这版我直接按**最终可实施的 V6 设计**来写，不再停留在讨论层。
目标是三件事：
1. 把 **V1–V5 像套娃一样嵌进 V6**
2. 把 **所有状态、转移、父子关系** 定清楚
3. 让你后面可以直接拿去做 **FastAPI + LangGraph / 自研 orchestrator**

我先给一句总纲：

> **V6 不是一个平面大状态机，而是三层嵌套状态机 + 若干可使能子策略。**
> 其中：
> - **Session-FSM** 管“一整次学习”
> - **Problem-FSM** 管“这一道题”
> - **Step-FSM** 管“当前这一步怎么教”
> - **Verify / Bailout / Anti-Mislead / Branch / Reflection** 都不是平铺同级，而是作为子状态或侧向策略嵌进去

---

# 一、V6 的最终嵌套关系

## 1.1 总体嵌套图
Mermaid
flowchart TD
    A[Session-FSM] --> B[Problem-FSM]
    B --> C[Step-FSM]

    C --> C1[Probe Layer]
    C --> C2[Router Layer]
    C --> C3[Rescue Layer]
    C --> C4[Scaffold Layer]
    C --> C5[Verify Layer]
    C --> C6[Advance Layer]

    B --> B1[Strategy Build]
    B --> B2[Branch Select / Switch]
    B --> B3[Problem Recap]

    A --> A1[Goal Setting]
    A --> A2[Tutoring]
    A --> A3[Reflection]
    A --> A4[Recommendation]

    C3 --> M1[Anti_Mislead]
    C3 --> M2[Concept_Clarify]
    C3 --> M3[Affect_Repair]
    C3 --> M4[Bailout]

    M4 --> M41[Bailout_DirectExplain]
    M4 --> M42[Bailout_Rollback]
    M4 --> M43[Bailout_MarkAndMove]

    C4 --> S1[Scaffold_L0]
    C4 --> S2[Scaffold_L1]
    C4 --> S3[Scaffold_L2]
    C4 --> S4[Scaffold_L3]

    C5 --> V1[Verify_Light]
    C5 --> V2[Verify_Hard]

## 1.2 V1–V5 如何嵌进 V6
- **V1 核心**：`Probe -> Router -> Scaffold -> Step_Advance`
- **V2 增强**：`Probe_Retry / Probe_Recovery / Anti_Mislead / Concept_Clarify / Elicit_Work`
- **V3 增强**：`Verify_Light / Verify_Hard`
- **V4 增强**：`Affect_Repair / Bailout_*`
- **V5 增强**：`Branch_Select / Branch_Switch`
- **V6 终局**：`Session-FSM + Problem-FSM + Step-FSM` 统一调度上面所有模块

也就是说，**V6 不是替代前面版本，而是包住它们。**

---

# 二、Session-FSM：整次学习会话

Session 负责整次会话，不直接碰“当前这一步怎么问”。

## 2.1 Session 状态
- `Session_Start`
- `Goal_Setting`
- `Problem_Loading`
- `Tutoring`
- `Reflection`
- `Recommendation`
- `Session_Suspend`
- `Session_End`

## 2.2 Session 状态图
Mermaid
图表不受支持

## 2.3 Session 转移规则

### `Session_Start -> Goal_Setting`

触发：进入系统
动作：初始化 session 上下文

### `Goal_Setting -> Problem_Loading`

触发：用户选题 / 系统派题 / 老师布题
动作：绑定 problem_id、mode、goal

### `Problem_Loading -> Tutoring`

触发：题目、答案、steps、cards 载入完成
动作：初始化 Problem-FSM

### `Tutoring -> Reflection`

触发：一道题结束
动作：拉取 trace、learning signals

### `Reflection -> Recommendation`

触发：反思结束
动作：生成同类题 / 相关 card 推荐

### `Recommendation -> Problem_Loading`

触发：继续下一题

### `Recommendation -> Session_End`

触发：结束本次学习

### `任何状态 -> Session_Suspend`

触发：用户中断 / 网络断开
动作：保存快照

---

# 三、Problem-FSM：一道题的生命周期

Problem-FSM 管“这道题从加载到讲完”的全流程。

## 3.1 Problem 状态
- `Problem_Initialized`
- `Problem_Preprocess`
- `Strategy_Build`
- `Step_Enter`
- `Step_Tutor`
- `Branch_Select`
- `Problem_Recap`
- `Problem_Close`
- `Problem_Abort`

> 说明：
> `Step_Tutor` 不是一个简单状态，而是 **调用 Step-FSM 的外壳状态**。

## 3.2 Problem 状态图
Mermaid
stateDiagram-v2
    [*] --> Problem_Initialized
    Problem_Initialized --> Problem_Preprocess: EVT_PROBLEM_LOADED
    Problem_Preprocess --> Strategy_Build: EVT_STEPS_READY
    Strategy_Build --> Step_Enter: EVT_STRATEGY_READY
    Step_Enter --> Step_Tutor: EVT_STEP_CONTEXT_READY

    Step_Tutor --> Branch_Select: EVT_BRANCH_PROPOSED
    Branch_Select --> Step_Enter: EVT_BRANCH_CONFIRMED

    Step_Tutor --> Step_Enter: EVT_ADVANCE_STEP
    Step_Tutor --> Step_Enter: EVT_ROLLBACK_STEP
    Step_Tutor --> Problem_Recap: EVT_PROBLEM_COMPLETE
    Step_Tutor --> Problem_Abort: EVT_ABORT_PROBLEM

    Problem_Recap --> Problem_Close: EVT_RECAP_DONE
    Problem_Close --> [*]
    Problem_Abort --> [*]

## 3.3 Problem 的核心职责

### `Problem_Preprocess`

负责：
- 读取答案
- 切步
- 每步打标签
- 找 method card / anti-card / verify card

### `Strategy_Build`

负责：
- 建当前题的 `solution graph`
- 确定主分支与备选分支
- 标记风险步（高误导 / 高 verify 需求）

### `Step_Enter`

负责：
- 装载当前 step 的上下文
- 初始化 `hint_level = 0`
- 初始化 `attempt_count = 0`
- 决定进入哪种 probe（new / retry / recovery）

### `Step_Tutor`

负责：
- 调用 Step-FSM
- 接收 Step-FSM 的事件返回值：
    - `ADVANCE_STEP`
    - `ROLLBACK_STEP`
    - `SWITCH_BRANCH`
    - `PROBLEM_COMPLETE`
    - `ABORT_PROBLEM`

### `Branch_Select`

负责：
- 当学生提出另一种正确思路
- 或系统判定当前分支不适合当前学生时
- 切换 `branch_id`

### `Problem_Recap`

负责：
- 总结整题主线
- 总结误导分支
- 输出题后反思

---

# 四、Step-FSM：当前这一步怎么教

这是整个 tutor 的核心。

---

## 4.1 Step 状态总表

### 探路与推进模块
- `Probe_NewStep`
- `Probe_Retry`
- `Probe_Recovery`
- `Elicit_Work`
- `Step_Advance_When_Ready`

### 辅导与抢救模块
- `Anti_Mislead`
- `Concept_Clarify`
- `Scaffold`
- `Affect_Repair`
- `Bailout`

### 诊断与校验模块
- `Router`
- `Verify`

---

## 4.2 Step 顶层状态图
Mermaid
stateDiagram-v2
    [*] --> Probe_NewStep

    Probe_NewStep --> Router: EVT_STUDENT_REPLY
    Probe_Retry --> Router: EVT_STUDENT_REPLY
    Probe_Recovery --> Router: EVT_STUDENT_REPLY
    Elicit_Work --> Router: EVT_STUDENT_REPLY
    Scaffold --> Router: EVT_STUDENT_REPLY

    Router --> Anti_Mislead: mislead_hit == true
    Router --> Concept_Clarify: student_intent == concept_question
    Router --> Affect_Repair: affect_state in {impatient, confused, overloaded}
    Router --> Elicit_Work: needs_student_work == true
    Router --> Verify: step_correctness == correct and verify_needed == true
    Router --> Step_Advance_When_Ready: step_correctness == correct and verify_needed == false
    Router --> Bailout: attempt_count >= max_attempts_per_step
    Router --> Scaffold: default

    Anti_Mislead --> Probe_Recovery: EVT_RECOVERY_READY
    Concept_Clarify --> Probe_Retry: EVT_CLARIFIED
    Affect_Repair --> Scaffold: EVT_REFOCUSED
    Affect_Repair --> Bailout: affect_state == overloaded

    Verify --> Step_Advance_When_Ready: EVT_VERIFY_PASS
    Verify --> Scaffold: EVT_VERIFY_FAIL

    Bailout --> Step_Advance_When_Ready: direct_explain / mark_and_move
    Bailout --> [*]: rollback_step

    Step_Advance_When_Ready --> [*]

---

# 五、Step-FSM 的所有嵌套关系

## 5.1 Probe 层的嵌套关系
Mermaid
stateDiagram-v2
    state ProbeLayer {
        [*] --> Probe_NewStep
        Probe_NewStep --> [*]

        [*] --> Probe_Retry
        Probe_Retry --> [*]

        [*] --> Probe_Recovery
        Probe_Recovery --> [*]
    }

### 什么时候进入哪种 Probe
- `Probe_NewStep`
    
    新进入一步时使用
- `Probe_Retry`
    
    当前步已经尝试过一次以上，但没偏航
- `Probe_Recovery`
    
    刚从误导路径拉回主线时使用
    
    它的提问必须**限制问题空间**，不能再开放式瞎问

---

## 5.2 Scaffold 的嵌套关系
Mermaid
图表不受支持

### Scaffold 四层定义
- `Scaffold_L0`
    
    镜像 / 观察
    
    不给真正提示，只照亮当前局部结构
- `Scaffold_L1`
    
    指方向
    
    例如“先统一角度”“先把 $a_{n+1}$ 看成主元”
- `Scaffold_L2`
    
    给关键一步
    
    例如“写成关于 $a_{n+1}$ 的一元二次方程”
- `Scaffold_L3`
    
    直接讲当前步
    
    但**只允许讲当前步**，不允许整题泄洪

---

## 5.3 Verify 的嵌套关系
Mermaid
stateDiagram-v2
    state Verify {
        [*] --> Verify_Light
        Verify_Light --> Verify_Hard: EVT_SUSPECT_FAKE_MASTERY
        Verify_Light --> [*]: EVT_PASS
        Verify_Hard --> [*]: EVT_PASS
        Verify_Hard --> [*]: EVT_FAIL
    }

### Verify 两层定义
- `Verify_Light`
    
    一步小检验
    
    适合正常情况
- `Verify_Hard`
    
    变式检验 / 灵魂拷问
    
    适合学生答太快、只会复述、疑似背答案

---

## 5.4 Bailout 的嵌套关系
Mermaid
stateDiagram-v2
    state Bailout {
        [*] --> Bailout_DirectExplain
        Bailout_DirectExplain --> [*]: EVT_DIRECT_DONE

        [*] --> Bailout_Rollback
        Bailout_Rollback --> [*]: EVT_ROLLBACK_REQUEST

        [*] --> Bailout_MarkAndMove
        Bailout_MarkAndMove --> [*]: EVT_MARK_AND_MOVE
    }

### Bailout 三种策略

#### `Bailout_DirectExplain`

当前步连卡数轮，直接讲当前步
但**不讲完整题**

#### `Bailout_Rollback`

不是当前步太难，而是前置桥梁断了
回退上一步

#### `Bailout_MarkAndMove`

当前步暂时不强求完全掌握
先标记薄弱，再继续整题节奏

---

# 六、Router：整个系统最重要的隐藏状态

Router 不应该是“自由语言判断器”，而必须输出结构化对象。

## 6.1 Router 输出 schema
JSON
{
  "student_intent": "solve_attempt",
  "step_correctness": "partial_correct",
  "mislead_hit": true,
  "anti_card_id": "anti_wrong_factorization",
  "concept_question": false,
  "needs_student_work": true,
  "verify_needed": false,
  "affect_state": "normal",
  "recommended_next_state": "Anti_Mislead",
  "confidence": 0.86
}

## 6.2 枚举值

### `student_intent`
- `solve_attempt`
- `concept_question`
- `give_up`
- `fast_guess`
- `reflection`

### `step_correctness`
- `wrong`
- `partial_correct`
- `correct_but_incomplete`
- `correct`

### `affect_state`
- `normal`
- `impatient`
- `confused`
- `overloaded`

---

# 七、所有状态的输入 / 输出契约（实现时必须写死）

这是你后面写代码最重要的部分。

---

## 7.1 `Probe_NewStep`

**输入**
- `current_step_goal`
- `method_cards`
- `allowed_scope = current_step_only`

**输出限制**
- 只能输出一句探测性问题
- 不得给公式结果
- 不得剧透下一步

**退出事件**
- `EVT_STUDENT_REPLY`

---

## 7.2 `Probe_Retry`

**输入**
- `attempt_count >= 1`
- `student_last_reply`
- `hint_level`

**输出限制**
- 只能定位当前卡点
- 不得重讲整题

**退出事件**
- `EVT_STUDENT_REPLY`

---

## 7.3 `Probe_Recovery`

**输入**
- `anti_card_id`
- `recovery_bridge`

**输出限制**
- 必须限制问题空间
- 不得再开放式发散

**退出事件**
- `EVT_STUDENT_REPLY`

---

## 7.4 `Anti_Mislead`

**输入**
- `anti_card`

**输出必须包含**
1. 为什么这条路诱人
2. 错在什么条件
3. 怎么回到主线

**退出事件**
- `EVT_RECOVERY_READY`

---

## 7.5 `Concept_Clarify`

**输入**
- `concept_question_type`
- `current_step_goal`
- `method_card`

**输出限制**
- 只解释当前步
- 不讲全题
- 不剧透下一步

**退出事件**
- `EVT_CLARIFIED`

---

## 7.6 `Elicit_Work`

**输入**
- `current_step_goal`
- `student_partial_answer`

**输出限制**
- 只能要求学生补一个局部动作 / 算式
- 不能自己直接写全

**退出事件**
- `EVT_STUDENT_REPLY`

---

## 7.7 `Scaffold`

**输入**
- `method_card`
- `hint_level`

**输出限制**
- 只能读取当前层级提示
- 不得发明卡片外的新路线
- 只能润色，不改数学动作

**退出事件**
- `EVT_STUDENT_REPLY`
- `EVT_HINT_EXHAUSTED`

---

## 7.8 `Verify`

**输入**
- `verify_card`
- `verify_mode = light | hard`

**输出限制**
- 只能检验当前步
- 不得变成长篇讲解

**退出事件**
- `EVT_VERIFY_PASS`
- `EVT_VERIFY_FAIL`

---

## 7.9 `Bailout_*`

**输入**
- `attempt_count`
- `hint_level`
- `affect_state`

**输出限制**
- DirectExplain：只讲当前步
- Rollback：只负责回退，不重讲全题
- MarkAndMove：简短收束，标记薄弱再推进

**退出事件**
- `EVT_DIRECT_DONE`
- `EVT_ROLLBACK_REQUEST`
- `EVT_MARK_AND_MOVE`

---

## 7.10 `Step_Advance_When_Ready`

**输入**
- 当前步已完成
- `has_next_step`

**输出限制**
- 总结当前步核心动作
- 然后推进到下一步
- 不得回头重讲整题

**退出事件**
- `EVT_ADVANCE_STEP`
- `EVT_PROBLEM_COMPLETE`

---

# 八、Step-FSM 的真实教学典型路径

## 8.1 学生走错路
Plain text
Probe_NewStep
→ Router(mislead_hit=true)
→ Anti_Mislead
→ Probe_Recovery
→ Router(partial_correct)
→ Scaffold_L1
→ Router(correct_but_incomplete)
→ Elicit_Work
→ Router(correct)
→ Verify_Light
→ Step_Advance_When_Ready

## 8.2 学生一直问“为什么”
Plain text
Probe_NewStep
→ Router(concept_question=true)
→ Concept_Clarify
→ Probe_Retry
→ Router(partial_correct)
→ Scaffold_L1
→ Router(correct)
→ Step_Advance_When_Ready

## 8.3 学生方向对，但只会嘴上说
Plain text
Probe_NewStep
→ Router(correct_but_incomplete)
→ Elicit_Work
→ Router(correct)
→ Verify_Light
→ Step_Advance_When_Ready

## 8.4 学生连续三轮卡死
Plain text
Probe_NewStep
→ Router(wrong)
→ Scaffold_L1
→ Router(wrong)
→ Scaffold_L2
→ Router(wrong)
→ Bailout_DirectExplain
→ Verify_Light
→ Step_Advance_When_Ready

## 8.5 学生说“懂了懂了”
Plain text
Router(correct, verify_needed=true)
→ Verify_Hard
→ fail
→ Scaffold_L2
→ Router(correct)
→ Step_Advance_When_Ready

---

# 九、Problem-FSM 和 Step-FSM 的父子关系

这是最关键的嵌套接口。

## 9.1 Problem 调用 Step-FSM

`Problem-FSM` 进入 `Step_Tutor` 后，只做三件事：
1. 装载当前步上下文
2. 运行 Step-FSM
3. 接收 Step-FSM 返回事件

## 9.2 Step-FSM 允许返回给 Problem-FSM 的事件
- `ADVANCE_STEP`
- `ROLLBACK_STEP`
- `SWITCH_BRANCH`
- `PROBLEM_COMPLETE`
- `ABORT_PROBLEM`

所以 Step-FSM 不是孤立的。
它像一个“子过程”，每轮运行结束后把控制权还给 Problem-FSM。

---

# 十、Session-FSM 和 Problem-FSM 的父子关系

Session 只关心宏观节奏：
- 当前题讲完没
- 需不需要反思
- 要不要推荐下一题
- 会话是否结束

Session 不介入 Step 级别的教学决策。

这点很重要，因为它保证了层次清晰：
- **Session 管节奏**
- **Problem 管路线**
- **Step 管教学**

---

# 十一、哪些东西不是状态，而是状态的副作用

这个边界一定要清楚。

以下内容我建议不要做成独立 FSM 状态，而是作为**状态转移的副作用**：
- `trace_log` 记录
- `postmortem` 生成
- `learning_signals` 更新
- `learner_state_update`
- `card_usage_stats`
- `teacher_review_queue`

也就是说：
- 每次状态转移时写 trace
- 每题结束时生成 recap
- 每 session 结束时更新 learner state

这样系统不会因为“治理功能”把状态空间搞爆炸。

---

# 十二、V6 的 feature flags 最终分层

你前面说“我可以从 V6 开始，只需要加几个使能”，这个方向是对的。
我建议这样分层：
JSON
{
  "step_kernel": {
    "enable_probe_retry": true,
    "enable_probe_recovery": true,
    "enable_elicit_work": true
  },
  "rescue_layer": {
    "enable_anti_mislead": true,
    "enable_concept_clarify": true,
    "enable_affect_repair": true,
    "enable_bailout": true
  },
  "verify_layer": {
    "enable_verify_light": true,
    "enable_verify_hard": true
  },
  "branch_layer": {
    "enable_branch_select": true,
    "enable_branch_switch": true
  },
  "session_layer": {
    "enable_reflection": true,
    "enable_recommendation": true,
    "enable_learner_state_update": false
  }
}

这意味着：
- 你**架构上就是 V6**
- 但运行时可以先只打开一部分能力
- 后面逐步点亮，不需要重构

---

# 十三、最终推荐的最小可达路径

虽然你是按 V6 设计，但我建议初始运行时只保证下面这条路径一定通：
Plain text
Session_Start
→ Goal_Setting
→ Problem_Loading
→ Problem_Preprocess
→ Strategy_Build
→ Step_Enter
→ Probe_NewStep
→ Router
→ {Anti_Mislead / Concept_Clarify / Elicit_Work / Scaffold / Verify / Step_Advance}
→ Problem_Recap
→ Reflection
→ Recommendation
→ Session_End

Branch / Affect / Rollback 可以后续逐步打开。

---

# 十四、你现在就可以拿去实现的最终设计结论

## 顶层结论
- **V6 采用三层嵌套 FSM**
- **V1–V5 作为子状态 / 子策略被包含**
- **日志、复盘、learner state 是副作用，不单独做状态**
- **使能开关决定哪些状态当前可达**

## 最终三层结构
- `Session-FSM`
- `Problem-FSM`
- `Step-FSM`

## Step-FSM 最终状态集合
- `Probe_NewStep`
- `Probe_Retry`
- `Probe_Recovery`
- `Router`
- `Anti_Mislead`
- `Concept_Clarify`
- `Affect_Repair`
- `Elicit_Work`
- `Scaffold_L0`
- `Scaffold_L1`
- `Scaffold_L2`
- `Scaffold_L3`
- `Verify_Light`
- `Verify_Hard`
- `Step_Advance_When_Ready`
- `Bailout_DirectExplain`
- `Bailout_Rollback`
- `Bailout_MarkAndMove`

---

如果你愿意，下一步我可以继续把这套最终设计**再压成一份工程用的单文件规范**，比如：
- 一份 **YAML 状态机定义**
- 一份 **Router schema**
- 一份 **父子状态机事件表**

这样你就可以直接扔给 AI 辅助编程去生代码了。

---
# 2、如果你愿意，下一步我可以继续把...
**问题详情：**
如果你愿意，下一步我可以继续把这套最终设计再压成一份工程用的单文件规范，比如：一份 YAML 状态机定义

一份 Router schema

一份 父子状态机事件表

这样你就可以直接扔给 AI 辅助编程去生代码了。好，你做吧

---
**回答（Answer）：**

下面直接给你三份可落地的工程规范草案。你可以分别保存成：
- `tutoring_v6.yaml`
- `router.schema.json`
- `fsm_events.yaml`

这三份已经尽量对齐前面我们讨论的 V6 终局设计，适合直接喂给 AI 辅助编程生成后端骨架。

---

## 1) `tutoring_v6.yaml`
YAML
version: "1.0.0"
machine_name: "tutoring_v6"
description: >
  Hierarchical FSM for AI tutoring.
  Session-FSM -> Problem-FSM -> Step-FSM.
  V1-V5 capabilities are nested as sub-policies inside V6.

naming_conventions:
  session_states_prefix: "S_"
  problem_states_prefix: "P_"
  step_states_prefix: "T_"
  events_prefix: "EVT_"

feature_flags:
  step_kernel:
    enable_probe_retry: true
    enable_probe_recovery: true
    enable_elicit_work: true
  rescue_layer:
    enable_anti_mislead: true
    enable_concept_clarify: true
    enable_affect_repair: true
    enable_bailout: true
  verify_layer:
    enable_verify_light: true
    enable_verify_hard: true
  branch_layer:
    enable_branch_select: true
    enable_branch_switch: true
  session_layer:
    enable_reflection: true
    enable_recommendation: true
    enable_learner_state_update: false

enums:
  student_intent:
    - solve_attempt
    - concept_question
    - give_up
    - fast_guess
    - reflection
  step_correctness:
    - wrong
    - partial_correct
    - correct_but_incomplete
    - correct
  affect_state:
    - normal
    - impatient
    - confused
    - overloaded
  card_role:
    - primary_method
    - secondary_method
    - anti_mislead
    - verify
    - pedagogy
  bailout_mode:
    - direct_explain
    - rollback
    - mark_and_move
  verify_mode:
    - light
    - hard

shared_context:
  session:
    session_id: "string"
    student_id: "string"
    teacher_mode: false
    session_state: "string"
    current_problem_id: "string|null"
    current_goal: "string|null"
    current_recommendations: []
  problem:
    problem_state: "string"
    solution_id: "string|null"
    current_branch_id: "string|null"
    current_step_id: "string|null"
    current_step_order: 0
    has_next_step: false
    candidate_branches: []
    problem_done: false
  step:
    step_state: "string"
    step_goal: "string|null"
    step_status: "not_started"
    hint_level: 0
    attempt_count: 0
    max_attempts_per_step: 3
    student_last_reply: ""
    assistant_last_reply: ""
    student_intent: null
    step_correctness: null
    mislead_hit: false
    anti_card_id: null
    concept_question: false
    needs_student_work: false
    verify_needed: false
    verify_mode: "light"
    affect_state: "normal"
    recommended_next_state: null
    confidence: 0.0
  cards:
    method_card_ids: []
    anti_card_ids: []
    verify_card_ids: []
    cards_loaded: []
  runtime:
    trace_log_ids: []
    last_event: null
    snapshot_version: 0

side_effects:
  on_every_transition:
    - append_trace_log
    - persist_session_snapshot
  on_problem_close:
    - build_problem_postmortem
    - build_learning_signals
  on_session_end:
    - persist_recommendations
    - update_learner_state_if_enabled

contracts:
  outputs:
    question_only:
      must:
        - "end_with_question"
      must_not:
        - "full_solution"
        - "next_step_spoiler"
        - "multiple_questions"
    constrained_explanation:
      must:
        - "why_tempting"
        - "why_wrong"
        - "recovery_bridge"
      must_not:
        - "full_solution"
        - "next_step_spoiler"
    local_clarification_only:
      must:
        - "explain_current_step_only"
        - "return_to_current_step"
      must_not:
        - "full_problem_explanation"
        - "next_step_spoiler"
    ask_for_local_action:
      must:
        - "request_single_local_formula_or_action"
      must_not:
        - "solve_entire_problem"
    one_step_check:
      must:
        - "check_current_step_only"
      must_not:
        - "new_chapter_explanation"
    summary_plus_transition:
      must:
        - "summarize_current_step_core_action"
        - "transition_to_next_step_or_problem_recap"
      must_not:
        - "repeat_whole_solution"
    current_step_direct_only:
      must:
        - "explain_current_step_only"
      must_not:
        - "full_solution"

session_fsm:
  initial_state: "S_Session_Start"
  states:
    S_Session_Start:
      kind: visible
      description: "User enters tutoring session."
      on:
        EVT_SESSION_INIT:
          target: "S_Goal_Setting"

    S_Goal_Setting:
      kind: visible
      description: "Select goal, mode, or target problem."
      on:
        EVT_GOAL_CONFIRMED:
          target: "S_Problem_Loading"

    S_Problem_Loading:
      kind: background
      description: "Load problem, solution, steps, cards, caches."
      entry_actions:
        - load_problem
        - load_solution
        - init_problem_fsm
      on:
        EVT_PROBLEM_READY:
          target: "S_Tutoring"
        EVT_LOAD_FAILED:
          target: "S_Session_End"

    S_Tutoring:
      kind: visible
      description: "Main tutoring phase. Delegates to Problem-FSM."
      invokes: "problem_fsm"
      on:
        EVT_PROBLEM_DONE:
          target: "S_Reflection"
        EVT_SESSION_PAUSE:
          target: "S_Session_Suspend"

    S_Reflection:
      kind: visible
      description: "Reflect on completed problem."
      enabled_if: "feature_flags.session_layer.enable_reflection"
      entry_actions:
        - summarize_problem_trace
        - generate_reflection
      on:
        EVT_REFLECTION_DONE:
          target: "S_Recommendation"

    S_Recommendation:
      kind: visible
      description: "Recommend next problems/cards."
      enabled_if: "feature_flags.session_layer.enable_recommendation"
      entry_actions:
        - fetch_related_problems
        - fetch_related_cards
      on:
        EVT_NEXT_PROBLEM:
          target: "S_Problem_Loading"
        EVT_FINISH_SESSION:
          target: "S_Session_End"

    S_Session_Suspend:
      kind: background
      description: "Persist snapshot and wait for resume."
      entry_actions:
        - persist_snapshot
      on:
        EVT_RESUME:
          target: "S_Tutoring"
        EVT_ABORT:
          target: "S_Session_End"

    S_Session_End:
      kind: terminal
      description: "End session and persist final artifacts."

problem_fsm:
  initial_state: "P_Problem_Initialized"
  states:
    P_Problem_Initialized:
      kind: background
      on:
        EVT_PROBLEM_LOADED:
          target: "P_Problem_Preprocess"

    P_Problem_Preprocess:
      kind: background
      description: "Split steps, tag steps, attach cards."
      entry_actions:
        - split_solution_steps
        - tag_solution_steps
        - link_cards_to_steps
      on:
        EVT_STEPS_READY:
          target: "P_Strategy_Build"
        EVT_PREPROCESS_FAILED:
          target: "P_Problem_Abort"

    P_Strategy_Build:
      kind: background
      description: "Build solution graph and branch metadata."
      entry_actions:
        - build_solution_graph
        - detect_candidate_branches
        - mark_verify_risk_steps
      on:
        EVT_STRATEGY_READY:
          target: "P_Step_Enter"

    P_Step_Enter:
      kind: background
      description: "Prepare current step context and choose initial step state."
      entry_actions:
        - load_current_step_context
        - reset_hint_level_if_new_step
        - reset_attempt_count_if_new_step
      on:
        EVT_STEP_CONTEXT_READY:
          target: "P_Step_Tutor"

    P_Step_Tutor:
      kind: visible
      description: "Delegate current step tutoring to Step-FSM."
      invokes: "step_fsm"
      on:
        EVT_ADVANCE_STEP:
          target: "P_Step_Enter"
        EVT_ROLLBACK_STEP:
          target: "P_Step_Enter"
        EVT_BRANCH_PROPOSED:
          target: "P_Branch_Select"
        EVT_PROBLEM_COMPLETE:
          target: "P_Problem_Recap"
        EVT_ABORT_PROBLEM:
          target: "P_Problem_Abort"

    P_Branch_Select:
      kind: background
      description: "Select or switch to a valid solution branch."
      enabled_if: "feature_flags.branch_layer.enable_branch_switch"
      entry_actions:
        - evaluate_candidate_branches
        - select_branch
      on:
        EVT_BRANCH_CONFIRMED:
          target: "P_Step_Enter"
        EVT_BRANCH_REJECTED:
          target: "P_Step_Tutor"

    P_Problem_Recap:
      kind: visible
      description: "Summarize whole problem."
      entry_actions:
        - generate_problem_recap
        - generate_misconception_summary
      on:
        EVT_RECAP_DONE:
          target: "P_Problem_Close"

    P_Problem_Close:
      kind: terminal
      description: "Problem finished."

    P_Problem_Abort:
      kind: terminal
      description: "Problem aborted."

step_fsm:
  initial_state: "T_Probe_NewStep"
  states:
    T_Probe_NewStep:
      kind: visible
      output_contract: "question_only"
      description: "First probe for a new step."
      on:
        EVT_STUDENT_REPLY:
          target: "T_Router"

    T_Probe_Retry:
      kind: visible
      enabled_if: "feature_flags.step_kernel.enable_probe_retry"
      output_contract: "question_only"
      description: "Probe after failed attempt on same step."
      on:
        EVT_STUDENT_REPLY:
          target: "T_Router"

    T_Probe_Recovery:
      kind: visible
      enabled_if: "feature_flags.step_kernel.enable_probe_recovery"
      output_contract: "question_only"
      description: "Probe after anti-mislead recovery."
      on:
        EVT_STUDENT_REPLY:
          target: "T_Router"

    T_Router:
      kind: background
      description: "Structured diagnosis and next-state selection."
      entry_actions:
        - run_router_classifier
      on:
        EVT_ROUTED_MISLEAD:
          target: "T_Anti_Mislead"
        EVT_ROUTED_CONCEPT:
          target: "T_Concept_Clarify"
        EVT_ROUTED_AFFECT:
          target: "T_Affect_Repair"
        EVT_ROUTED_ELICIT:
          target: "T_Elicit_Work"
        EVT_ROUTED_VERIFY:
          target: "T_Verify"
        EVT_ROUTED_ADVANCE:
          target: "T_Step_Advance_When_Ready"
        EVT_ROUTED_BAILOUT:
          target: "T_Bailout"
        EVT_ROUTED_DEFAULT:
          target: "T_Scaffold"

    T_Anti_Mislead:
      kind: visible
      enabled_if: "feature_flags.rescue_layer.enable_anti_mislead"
      output_contract: "constrained_explanation"
      description: "Explain tempting wrong path and return to mainline."
      on:
        EVT_RECOVERY_READY:
          target: "T_Probe_Recovery"

    T_Concept_Clarify:
      kind: visible
      enabled_if: "feature_flags.rescue_layer.enable_concept_clarify"
      output_contract: "local_clarification_only"
      description: "Clarify why current step uses this move."
      on:
        EVT_CLARIFIED:
          target: "T_Probe_Retry"

    T_Affect_Repair:
      kind: visible
      enabled_if: "feature_flags.rescue_layer.enable_affect_repair"
      output_contract: "local_clarification_only"
      description: "Reduce cognitive load and repair pacing."
      on:
        EVT_REFOCUSED:
          target: "T_Scaffold"
        EVT_OVERLOADED:
          target: "T_Bailout"

    T_Elicit_Work:
      kind: visible
      enabled_if: "feature_flags.step_kernel.enable_elicit_work"
      output_contract: "ask_for_local_action"
      description: "Ask student to produce the local formula/action."
      on:
        EVT_STUDENT_REPLY:
          target: "T_Router"

    T_Scaffold:
      kind: visible
      output_contract: "local_clarification_only"
      description: "Run hint ladder from L0 to L3."
      sub_states:
        T_Scaffold_L0:
          description: "Mirror / observation."
        T_Scaffold_L1:
          description: "Direction hint."
        T_Scaffold_L2:
          description: "Critical local step."
        T_Scaffold_L3:
          description: "Direct current-step explanation."
      on:
        EVT_STUDENT_REPLY:
          target: "T_Router"
        EVT_HINT_EXHAUSTED:
          target: "T_Bailout"

    T_Verify:
      kind: visible
      description: "Verify current-step understanding."
      enabled_if: "feature_flags.verify_layer.enable_verify_light"
      output_contract: "one_step_check"
      sub_states:
        T_Verify_Light:
          description: "Simple one-step verification."
        T_Verify_Hard:
          description: "Hard verification for suspected fake mastery."
      on:
        EVT_VERIFY_PASS:
          target: "T_Step_Advance_When_Ready"
        EVT_VERIFY_FAIL:
          target: "T_Scaffold"

    T_Step_Advance_When_Ready:
      kind: visible
      output_contract: "summary_plus_transition"
      description: "Summarize current step and advance."
      on:
        EVT_HAS_NEXT_STEP:
          target: "__RETURN_EVT_ADVANCE_STEP__"
        EVT_NO_NEXT_STEP:
          target: "__RETURN_EVT_PROBLEM_COMPLETE__"

    T_Bailout:
      kind: visible
      enabled_if: "feature_flags.rescue_layer.enable_bailout"
      description: "Protect pacing and prevent step deadlock."
      sub_states:
        T_Bailout_DirectExplain:
          output_contract: "current_step_direct_only"
          description: "Directly explain current step only."
        T_Bailout_Rollback:
          description: "Rollback to previous step for missing prerequisite."
        T_Bailout_MarkAndMove:
          description: "Mark weak mastery and continue."
      on:
        EVT_DIRECT_DONE:
          target: "T_Verify"
        EVT_ROLLBACK_REQUEST:
          target: "__RETURN_EVT_ROLLBACK_STEP__"
        EVT_MARK_AND_MOVE:
          target: "T_Step_Advance_When_Ready"

guards:
  router_to_mislead: "mislead_hit == true"
  router_to_concept: "student_intent == 'concept_question' or concept_question == true"
  router_to_affect: "affect_state in ['impatient', 'confused', 'overloaded']"
  router_to_elicit: "step_correctness == 'correct_but_incomplete' or needs_student_work == true"
  router_to_verify: "step_correctness == 'correct' and verify_needed == true"
  router_to_advance: "step_correctness == 'correct' and verify_needed == false"
  router_to_bailout: "attempt_count >= max_attempts_per_step or affect_state == 'overloaded'"

runtime_policies:
  hint_ladder:
    initial_level: 0
    max_level: 3
    increment_on:
      - EVT_ROUTED_DEFAULT
      - EVT_VERIFY_FAIL
  verify_policy:
    light_if:
      - "verify_needed == true"
      - "student_intent != 'fast_guess'"
    hard_if:
      - "student_intent == 'fast_guess'"
      - "confidence < 0.6"
      - "student_reply in ['懂了', '会了', '就是这样']"
  bailout_policy:
    choose_direct_explain_if:
      - "attempt_count >= max_attempts_per_step"
      - "step_goal is still locally teachable"
    choose_rollback_if:
      - "missing_prerequisite == true"
    choose_mark_and_move_if:
      - "session_pacing_priority == true"

logging:
  trace_fields:
    - session_id
    - problem_id
    - step_id
    - turn_index
    - from_state
    - to_state
    - student_reply
    - assistant_reply
    - router_output
    - hint_level
    - verify_mode
    - bailout_mode
    - mislead_hit
    - confidence

return_events_from_step_fsm:
  - EVT_ADVANCE_STEP
  - EVT_ROLLBACK_STEP
  - EVT_BRANCH_PROPOSED
  - EVT_PROBLEM_COMPLETE
  - EVT_ABORT_PROBLEM

---

## 2) `router.schema.json`
JSON
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/router.schema.json",
  "title": "Tutor Router Output Schema",
  "type": "object",
  "additionalProperties": false,
  "required": [
"student_intent",
"step_correctness",
"mislead_hit",
"anti_card_id",
"concept_question",
"needs_student_work",
"verify_needed",
"verify_mode",
"affect_state",
"recommended_next_state",
"confidence"
  ],
  "properties": {
    "student_intent": {
      "type": "string",
      "enum": [
"solve_attempt",
"concept_question",
"give_up",
"fast_guess",
"reflection"
      ]
    },
    "step_correctness": {
      "type": "string",
      "enum": [
"wrong",
"partial_correct",
"correct_but_incomplete",
"correct"
      ]
    },
    "mislead_hit": {
      "type": "boolean"
    },
    "anti_card_id": {
      "type": ["string", "null"],
      "description": "Matched anti-mislead card id if mislead_hit is true."
    },
    "concept_question": {
      "type": "boolean"
    },
    "needs_student_work": {
      "type": "boolean",
      "description": "True if student has right direction but must write local formula/action."
    },
    "verify_needed": {
      "type": "boolean"
    },
    "verify_mode": {
      "type": "string",
      "enum": ["light", "hard"]
    },
    "affect_state": {
      "type": "string",
      "enum": [
"normal",
"impatient",
"confused",
"overloaded"
      ]
    },
    "recommended_next_state": {
      "type": "string",
      "enum": [
"T_Anti_Mislead",
"T_Concept_Clarify",
"T_Affect_Repair",
"T_Elicit_Work",
"T_Scaffold",
"T_Verify",
"T_Step_Advance_When_Ready",
"T_Bailout"
      ]
    },
    "confidence": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0
    },
    "missing_prerequisite": {
      "type": "boolean",
      "default": false
    },
    "notes": {
      "type": "string"
    }
  },
  "allOf": [
    {
      "if": {
        "properties": { "mislead_hit": { "const": true } },
        "required": ["mislead_hit"]
      },
      "then": {
        "properties": {
          "recommended_next_state": { "const": "T_Anti_Mislead" }
        }
      }
    },
    {
      "if": {
        "properties": { "concept_question": { "const": true } },
        "required": ["concept_question"]
      },
      "then": {
        "properties": {
          "recommended_next_state": { "const": "T_Concept_Clarify" }
        }
      }
    },
    {
      "if": {
        "properties": { "needs_student_work": { "const": true } },
        "required": ["needs_student_work"]
      },
      "then": {
        "properties": {
          "recommended_next_state": { "const": "T_Elicit_Work" }
        }
      }
    },
    {
      "if": {
        "properties": {
          "step_correctness": { "const": "correct" },
          "verify_needed": { "const": true }
        },
        "required": ["step_correctness", "verify_needed"]
      },
      "then": {
        "properties": {
          "recommended_next_state": { "const": "T_Verify" }
        }
      }
    },
    {
      "if": {
        "properties": {
          "step_correctness": { "const": "correct" },
          "verify_needed": { "const": false }
        },
        "required": ["step_correctness", "verify_needed"]
      },
      "then": {
        "properties": {
          "recommended_next_state": { "const": "T_Step_Advance_When_Ready" }
        }
      }
    }
  ]
}

---

## 3) `fsm_events.yaml`
YAML
version: "1.0.0"
title: "Parent-Child FSM Event Table"

event_types:
  upward_events:
    description: "Events emitted from child FSM to parent FSM."
  downward_events:
    description: "Commands or setup signals emitted from parent FSM to child FSM."
  local_events:
    description: "Events consumed within the same FSM."

session_to_problem:
  downward_events:
    - event: EVT_PROBLEM_READY
      from: S_Problem_Loading
      to: P_Problem_Initialized
      meaning: "Problem assets loaded; initialize Problem-FSM."
    - event: EVT_SESSION_PAUSE
      from: S_Tutoring
      to: P_ANY
      meaning: "Pause tutoring and persist snapshots."
  upward_events:
    - event: EVT_PROBLEM_DONE
      from: P_Problem_Close
      to: S_Tutoring
      meaning: "Current problem finished; Session-FSM may enter Reflection."
    - event: EVT_ABORT_PROBLEM
      from: P_Problem_Abort
      to: S_Tutoring
      meaning: "Problem aborted; Session-FSM decides next action."

problem_to_step:
  downward_events:
    - event: EVT_STEP_CONTEXT_READY
      from: P_Step_Enter
      to: T_Probe_NewStep
      meaning: "Current step context loaded; start Step-FSM."
    - event: EVT_STEP_RETRY
      from: P_Step_Enter
      to: T_Probe_Retry
      meaning: "Re-enter same step after retry."
    - event: EVT_STEP_RECOVERY
      from: P_Step_Enter
      to: T_Probe_Recovery
      meaning: "Re-enter current step after anti-mislead recovery."
  upward_events:
    - event: EVT_ADVANCE_STEP
      from: T_Step_Advance_When_Ready
      to: P_Step_Tutor
      meaning: "Current step passed; advance to next step."
    - event: EVT_ROLLBACK_STEP
      from: T_Bailout_Rollback
      to: P_Step_Tutor
      meaning: "Rollback to previous step."
    - event: EVT_BRANCH_PROPOSED
      from: T_Router
      to: P_Step_Tutor
      meaning: "Alternative valid branch detected; Problem-FSM may switch branches."
    - event: EVT_PROBLEM_COMPLETE
      from: T_Step_Advance_When_Ready
      to: P_Step_Tutor
      meaning: "No next step; whole problem completed."
    - event: EVT_ABORT_PROBLEM
      from: T_Bailout
      to: P_Step_Tutor
      meaning: "Abort current problem."

step_local_events:
  - event: EVT_STUDENT_REPLY
    from: T_Probe_NewStep|T_Probe_Retry|T_Probe_Recovery|T_Elicit_Work|T_Scaffold
    to: T_Router
    meaning: "Student responds; route next action."

  - event: EVT_ROUTED_MISLEAD
    from: T_Router
    to: T_Anti_Mislead
    meaning: "Wrong but tempting method detected."

  - event: EVT_ROUTED_CONCEPT
    from: T_Router
    to: T_Concept_Clarify
    meaning: "Student asks conceptual why-question."

  - event: EVT_ROUTED_AFFECT
    from: T_Router
    to: T_Affect_Repair
    meaning: "Student shows overload/impatience/confusion."

  - event: EVT_ROUTED_ELICIT
    from: T_Router
    to: T_Elicit_Work
    meaning: "Student direction is acceptable but must produce local formula/action."

  - event: EVT_ROUTED_VERIFY
    from: T_Router
    to: T_Verify
    meaning: "Student appears correct; verify before advance."

  - event: EVT_ROUTED_ADVANCE
    from: T_Router
    to: T_Step_Advance_When_Ready
    meaning: "Student step is sufficiently correct; may advance."

  - event: EVT_ROUTED_BAILOUT
    from: T_Router
    to: T_Bailout
    meaning: "Too many failures or overloaded affect state."

  - event: EVT_ROUTED_DEFAULT
    from: T_Router
    to: T_Scaffold
    meaning: "Default tutoring fallback."

  - event: EVT_RECOVERY_READY
    from: T_Anti_Mislead
    to: T_Probe_Recovery
    meaning: "Recovery bridge delivered; ask constrained follow-up."

  - event: EVT_CLARIFIED
    from: T_Concept_Clarify
    to: T_Probe_Retry
    meaning: "Concept clarified; retry same step."

  - event: EVT_REFOCUSED
    from: T_Affect_Repair
    to: T_Scaffold
    meaning: "Student stabilized; continue with scaffold."

  - event: EVT_OVERLOADED
    from: T_Affect_Repair
    to: T_Bailout
    meaning: "Affect repair failed; bail out."

  - event: EVT_HINT_EXHAUSTED
    from: T_Scaffold
    to: T_Bailout
    meaning: "Hint ladder exhausted."

  - event: EVT_VERIFY_PASS
    from: T_Verify
    to: T_Step_Advance_When_Ready
    meaning: "Verification passed."

  - event: EVT_VERIFY_FAIL
    from: T_Verify
    to: T_Scaffold
    meaning: "Verification failed; return to scaffold."

  - event: EVT_DIRECT_DONE
    from: T_Bailout_DirectExplain
    to: T_Verify
    meaning: "Current step directly explained; lightly verify if enabled."

  - event: EVT_ROLLBACK_REQUEST
    from: T_Bailout_Rollback
    to: P_Step_Tutor
    meaning: "Ask parent Problem-FSM to move current_step_order backward."

  - event: EVT_MARK_AND_MOVE
    from: T_Bailout_MarkAndMove
    to: T_Step_Advance_When_Ready
    meaning: "Mark weak mastery and continue."

interrupt_rules:
  global_interrupts:
    - source: ANY
      event: EVT_SESSION_PAUSE
      target: S_Session_Suspend
      rule: "Highest priority; persist all child snapshots."
    - source: ANY
      event: EVT_ABORT
      target: S_Session_End
      rule: "Terminate session gracefully."

  problem_interrupts:
    - source: P_ANY
      event: EVT_ABORT_PROBLEM
      target: P_Problem_Abort
      rule: "Abort only current problem, preserve session."

  step_interrupts:
    - source: T_ANY
      event: EVT_ROUTED_BAILOUT
      target: T_Bailout
      rule: "Overrides Scaffold/Probe loops."
    - source: T_Anti_Mislead
      event: EVT_STUDENT_REPLY
      target: T_Probe_Recovery
      rule: "Do not re-open broad probing after recovery."

priority_rules:
  router_priority:
    - mislead_hit
    - concept_question
    - affect_state_overloaded
    - needs_student_work
    - verify_needed
    - step_correctness_correct
    - bailout_threshold
    - default_scaffold

  verify_priority:
    - hard_verify_if_fast_guess
    - hard_verify_if_low_confidence
    - light_verify_if_normal_correct

  bailout_priority:
    - rollback_if_missing_prerequisite
    - direct_explain_if_current_step_teachable
    - mark_and_move_if_pacing_priority

return_contracts:
  from_step_to_problem:
    required_fields:
      - session_id
      - problem_id
      - current_step_id
      - emitted_event
      - trace_log_id
      - hint_level
      - attempt_count
  from_problem_to_session:
    required_fields:
      - session_id
      - problem_id
      - emitted_event
      - postmortem_id
      - learning_signal_ids

---

## 实现建议

这三份文件在工程里各自扮演的角色最好固定下来：
- `tutoring_v6.yaml`
    
    作为**单一真相源**，驱动状态机节点与转移规则
- `router.schema.json`
    
    作为 Router 输出校验器，保证后台判定结构稳定
- `fsm_events.yaml`
    
    作为父子状态机调度说明书，避免 Step / Problem / Session 混乱耦合

你后面可以直接让 AI 辅助编程按这三份规范生：
1. Pydantic 模型
2. LangGraph 节点与条件边
3. FastAPI 路由和状态持久化逻辑

如果你下一步要继续，我建议直接让 AI 先生成这四个文件：
- `schemas/router.py`
- `schemas/context.py`
- `state_machine/loader.py`
- `agents/langgraph/graph.py`

这样最容易把这套规范跑起来。