# 虚拟宠物空间相关开源项目调研与可复用性分析

## 1. 背景

当前目标是构建一个**配置驱动 + AI 决策驱动的 2D 虚拟宠物空间系统**，系统需要支持：

- 配置化定义宠物、空间对象与环境
- 基于对象状态、宠物状态和空间上下文进行 AI 行为决策
- 输出结构化动作计划或计划草稿
- 将行为决策映射为 2D 场景移动与 Live2D 演出

在这个背景下，需要调研现有开源项目中，是否已经存在：

1. 对“世界对象模型交互”做过深度建模或训练的系统
2. 可直接复用的 object-centric planner / policy / simulation framework
3. 可借鉴的对象 schema、动作 schema、状态转移模型与训练数据构造方式

本文聚焦于相关开源项目的**能力边界、适配度和可复用性**。

---

## 2. 结论摘要

对当前系统最有参考价值的项目，不一定是“能直接拿来用”的项目，而是：

- 已经把**对象、空间、状态、动作**抽象得很清楚的项目
- 已经把**高层任务 -> 可执行动作程序**做过系统化设计的项目
- 已经形成比较成熟的**environment API / affordance schema / task representation** 的项目

### 最值得优先研究的项目方向

1. **ALFWorld**
2. **AI2-THOR / ProcTHOR**
3. **VirtualHome**
4. **Habitat / Habitat-Lab**

### 重要结论

- **没有一个项目可以开箱即用直接作为你的 2D Live2D 虚拟宠物空间 planner**
- **最可复用的是 schema、抽象方式、训练样本格式和 planner 设计思想**
- **最适合你的路线是：自定义 schema + 借鉴 benchmark + 训练小模型辅助 planner，而不是直接复用现成模型权重**

---

## 3. 项目分类

结合你的目标，可以将相关项目分为 4 类：

### 3.1 文本环境 / 对象交互规划类
特点：
- 对象抽象清晰
- world state 明确
- action space 通常结构化
- 适合借鉴 planner 输入输出协议

代表项目：
- ALFWorld

### 3.2 3D Embodied AI 仿真环境类
特点：
- 有真实空间、对象、导航
- 支持 agent 在场景中的移动与交互
- 通常偏重空间理解和导航

代表项目：
- Habitat / Habitat-Lab
- AI2-THOR / ProcTHOR

### 3.3 行为程序 / 任务脚本类
特点：
- 将高层任务表示为可执行程序或脚本
- 特别适合从语义行为映射到执行计划

代表项目：
- VirtualHome

### 3.4 机器人操作 / Vision-Language-Action 类
特点：
- 更偏真实机器人或机械臂控制
- 模型从观察直接输出动作
- 适合参考 policy 学习方式
- 不太适合直接复用于 2D 宠物空间

代表项目：
- VIMA
- OpenVLA
- Octo
- RT-1 / RT-2 / RT-X
- ManiSkill
- RLBench

---

## 4. 重点项目分析

---

## 4.1 ALFWorld

### 项目定位
ALFWorld 是一个将 embodied household tasks 抽象为文本环境任务的系统。它将 3D 家居环境中的对象状态、交互逻辑与任务流程映射到文本世界表示中。

### 核心能力
- 对象与房间状态抽象
- 文本化 action space
- 高层任务到动作链规划
- goal-conditioned planning

### 与当前系统的匹配点
最贴近的是：

- 你的系统也需要把世界压缩成 AI 可理解的结构化上下文
- 你也需要把对象 affordance 抽象成动作词表
- 你也需要定义统一的 action schema 和 planner 协议

### 可借鉴部分
1. **世界状态表示方式**
   - 适合借鉴其文本 / 符号化 world state 抽象
2. **动作词表设计**
   - 适合参考如何定义有限、可校验的动作空间
3. **任务规划结构**
   - 适合借鉴 goal -> action sequence 的表示方式
4. **LLM / policy 消费上下文方式**
   - 对你的 `DecisionContext` 设计很有帮助

### 不适合直接复用的部分
- 它不是 2D 宠物空间系统
- 它不负责你的 Live2D 执行问题
- 其 action space 需要结合你自己的对象类型和行为模板重构

### 适配建议
建议将 ALFWorld 作为：
- **AI 上下文设计参考**
- **planner 输入输出协议参考**
而不是直接复用其 runtime。

### 综合评价
- **对象建模参考价值：高**
- **planner 设计参考价值：高**
- **直接工程复用价值：中低**

---

## 4.2 AI2-THOR / ProcTHOR

### 项目定位
AI2-THOR 是一个面向 embodied AI 的 3D 家居交互仿真环境，提供大量 household objects、对象状态和 agent interaction API。ProcTHOR 进一步提供程序化生成的丰富室内环境。

### 核心能力
- household object catalog
- 对象状态转换
- 场景导航
- agent-object interaction
- object-centric environment API

### 与当前系统的匹配点
这个方向和你的系统最接近的地方在于：

- 都是“家庭/生活空间中的对象交互”
- 都强调对象类型、对象状态和可交互行为
- 都需要 agent 基于空间对象做决策

### 可借鉴部分
1. **对象 schema**
   - 对象类型、状态、属性、是否可交互等定义方式
2. **affordance 建模**
   - 哪些对象能做什么交互，如何表达
3. **对象状态机设计**
   - clean / dirty / open / closed / occupied 等状态体系
4. **空间中的对象 API**
   - object-centric world modeling

### 不适合直接复用的部分
- 运行时是 3D household simulator，太重
- 不适配你的 2D 渲染和 Live2D 演出需求
- 其动作更偏第一人称 embodied agent，而不是“宠物角色演出”

### 适配建议
非常适合作为：
- **世界对象建模参考标准**
- **affordance / state transition 设计参考**
- **对象配置库设计参考**

### 综合评价
- **对象交互建模参考价值：非常高**
- **环境 schema 参考价值：非常高**
- **直接工程复用价值：中**

---

## 4.3 VirtualHome

### 项目定位
VirtualHome 的核心思想是把家庭环境中的高层行为表示为程序脚本，再在虚拟环境中执行这些脚本。

### 核心能力
- 高层行为 program representation
- 对象交互脚本表示
- 行为序列执行
- object-centric action decomposition

### 与当前系统的匹配点
这个项目对你最重要的地方在于：

> 你也需要从 AI 的高层语义行为输出，过渡到可执行的 ExecutionPlan。

这和 VirtualHome 的行为脚本思想非常接近。

### 可借鉴部分
1. **行为脚本表示**
   - 很适合你设计 `PlanDraft`
2. **高层任务拆解**
   - 从一个 intent 映射到多步执行序列
3. **对象驱动动作链**
   - 不同对象会触发不同的执行脚本
4. **程序式 planner 表示法**
   - 适合做 AI 输出的中间层协议

### 不适合直接复用的部分
- 它不是 2D 宠物空间
- 它不直接提供 Live2D / 2D scene execution
- 对象和动作库仍需重建为你的场景版本

### 适配建议
非常适合作为：
- **`AIDecisionOutput -> PlanDraft -> ExecutionPlan` 的设计参考**
- **行为程序 DSL 的设计参考**

### 综合评价
- **程序式行为设计参考价值：非常高**
- **ExecutionPlan 设计参考价值：非常高**
- **直接工程复用价值：中**

---

## 4.4 Habitat / Habitat-Lab

### 项目定位
Habitat 是面向 embodied AI 的高性能 3D 仿真平台，强调导航、空间感知与语义环境表示。

### 核心能力
- 场景导航
- semantic map
- scene graph / spatial reasoning
- embodied agent simulation

### 与当前系统的匹配点
对于你的系统，它最值得借鉴的是：

- 空间建模思想
- 可达性判断
- agent 的空间上下文表示

### 可借鉴部分
1. **空间语义建模**
2. **reachable object / navigable area 思路**
3. **world state abstraction**

### 不适合直接复用的部分
- 过于偏 3D 和视觉导航
- 实现重、依赖大
- 对 2D 虚拟宠物空间来说成本过高

### 适配建议
更适合作为：
- **空间层与导航语义设计参考**
而不是作为你的主运行时。

### 综合评价
- **空间建模参考价值：高**
- **直接工程复用价值：低**

---

## 5. 机器人 / VLA 类项目的可复用性

这类项目包括：

- VIMA
- OpenVLA
- Octo
- RT-1 / RT-2 / RT-X
- ManiSkill
- RLBench

### 共同特点
- 输入通常是图像、文本、状态
- 输出通常是低层动作或控制策略
- 偏向机械臂、抓取、导航、机器人操作

### 对当前系统的价值
#### 有价值的地方
- 参考“模型做 policy”的架构设计
- 参考 action token / action head 设计思路
- 参考训练数据组织方式
- 参考小模型替代部分 planner 的路线

#### 不适合直接复用的地方
- 动作空间完全不同
- 训练目标完全不同
- 不适合直接映射到 2D 宠物行为模板
- 不适合直接输出你的 `ExecutionPlan`

### 结论
这类项目**更适合参考训练方法和 policy 架构**，不适合直接拿来作为虚拟宠物空间 planner 或行为模型。

---

## 6. 是否存在可直接复用的“世界对象交互小模型”

### 结论
严格来说：

**目前没有一个现成开源小模型，可以开箱即用直接适配你的 2D 配置驱动宠物空间，并输出稳定可执行的 `ExecutionPlan`。**

原因包括：

1. 你的动作空间是自定义的
2. 你的对象 schema 是自定义的
3. 你的运行时是 2D 场景 + Live2D
4. 你的目标是语义行为规划，不是机器人控制或第一人称导航

所以真正可行的方向不是“找一个现成完全匹配的模型”，而是：

- 借对象 schema
- 借 action schema
- 借行为脚本表示
- 借训练样本格式
- 在你自己的 schema 上训练小模型

---

## 7. 对当前系统最推荐的借鉴策略

### 7.1 借 ALFWorld 的什么
- AI 上下文组织方式
- 对象状态表达方式
- 动作词表设计
- world state -> action selection 思路

### 7.2 借 AI2-THOR / ProcTHOR 的什么
- household object schema
- affordance 建模方式
- 对象状态机
- object-centric interaction API

### 7.3 借 VirtualHome 的什么
- 行为脚本 / 程序表示
- 任务拆解
- 高层行为到动作序列映射
- 作为 `PlanDraft` 设计参考

### 7.4 借 Habitat 的什么
- reachable area
- semantic zones
- spatial reasoning 抽象

---

## 8. 推荐的系统演进路线

### 阶段 1：建立自有 schema
先定义你自己的：

- Pet schema
- Object schema
- Action schema
- PlanDraft schema
- ExecutionPlan schema

这是必须的基础。

### 阶段 2：规则 planner 跑通全链路
使用 deterministic planner：

- world state -> decision context
- high-level action -> plan draft
- plan draft -> execution plan

目的：
- 跑通系统
- 沉淀训练数据
- 保证可执行性

### 阶段 3：引入小模型做 planner 辅助
训练小模型做：

- target selection
- template selection
- style filling
- plan draft generation

### 阶段 4：保留 validator + compiler
即便引入模型，也不要直接执行模型输出。
保留：

- validator
- compiler
- fallback planner

确保结果稳定可执行。

---

## 9. 推荐的小模型训练方向

如果后续你要微调一个小模型，最推荐的训练目标不是“直接输出 final `ExecutionPlan`”，而是：

### 9.1 Target Selection Model
输入：
- pet state
- reachable object summaries
- intent

输出：
- targetObjectId

### 9.2 Template Selection Model
输入：
- intent
- pet state
- object summary

输出：
- templateId
- style tags

### 9.3 Plan Draft Model
输入：
- pet state
- object summary
- world state
- intended action

输出：
- `PlanDraft`

然后由规则层将 `PlanDraft` 编译为 final `ExecutionPlan`。

---

## 10. 最终建议

### 最值得研究的开源项目优先级
1. **VirtualHome**
2. **AI2-THOR / ProcTHOR**
3. **ALFWorld**
4. **Habitat / Habitat-Lab**

### 采纳建议
- **直接复用 runtime：不推荐**
- **借鉴对象建模、动作 schema 和行为程序设计：强烈推荐**
- **训练你自己的小模型：推荐，但应基于你自己的 schema 和训练样本**

### 最终结论

对当前系统最现实的路线是：

> **以自定义配置协议为核心，参考 ALFWorld / AI2-THOR / VirtualHome 的对象建模与行为表示方式，先用规则 planner 跑通系统，再用小模型逐步接管 target selection、template selection 和 plan draft generation。**

这种方式可以同时兼顾：

- 配置化扩展能力
- AI 泛化能力
- 可执行性
- 可验证性
- 与 Live2D 执行层的稳定衔接
