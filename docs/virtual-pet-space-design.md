# 配置驱动 + AI 决策驱动的 2D 虚拟宠物空间系统详细设计

## 1. 设计目标

本系统目标是构建一个**纯 2D 虚拟宠物空间**，其中：

1. 宠物是配置化定义的实体，包含基础形态、行为能力、渲染能力、状态字段。
2. 虚拟空间中的对象（如猫砂、猫爬架、猫窝、床）也是配置化定义的实体。
3. AI 不直接操纵底层渲染资源，而是基于宠物状态、空间状态、对象定义进行阶段性行为决策。
4. 决策结果输出为**规范化语义动作**与**对象语义状态意图**。
5. 本地 planner / executor 将 AI 输出转译为：
   - 空间中的移动路径
   - Live2D 动作播放
   - 表情和参数变化
   - 对象状态切换
   - 对象渲染状态变化
6. 渲染层根据实体当前状态与配置映射完成最终表现。

---

## 2. 总体架构

整体分为 7 层：

### 2.1 配置层
负责描述系统内所有可配置实体和规则：

- Pet Definition
- World Definition
- Object Definition
- Behavior Template Definition
- Render Mapping Definition
- AI Exposure Definition

### 2.2 运行时状态层
负责维护运行中状态：

- 宠物当前状态
- 对象当前状态
- 空间状态
- 历史事件
- 当前执行计划
- 当前渲染态

### 2.3 上下文编译层
负责把原始配置和运行态编译为 AI 可消费上下文：

- 宠物能力摘要
- 当前可行动作
- 可达对象与 affordance
- 决策提示
- 限制条件

### 2.4 AI 决策层
负责根据上下文输出：

- 宠物行为意图
- 目标对象
- 目标区域
- 表现风格标签
- 对象状态意图

### 2.5 Planner / Orchestrator 层
负责将 AI 输出转换为执行序列：

- 移动路径
- 动作模板序列
- 对象状态切换计划
- 失败回退逻辑

### 2.6 Render Execution 层
负责将执行计划作用于渲染系统：

- 控制 Live2D 宠物
- 控制对象显示状态
- 控制空间层 UI / 特效

### 2.7 渲染层
负责最终绘制：

- 2D 空间对象
- 宠物 Live2D 模型
- 特效 / 状态贴图 / overlay

---

## 3. 核心设计原则

### 3.1 AI 决定语义，不决定底层渲染细节
AI 只输出：

- 做什么行为
- 对哪个对象交互
- 对象应进入什么语义状态
- 采用什么风格/情绪

AI 不输出：

- sprite 文件路径
- Live2D 参数 ID
- motion 文件名
- 像素级轨迹

### 3.2 对象行为意义与渲染实现解耦
例如“猫窝 occupied”是语义状态。
具体如何渲染 occupied：

- 换图
- 加阴影
- 显示“被占用”样式
- 播放柔和呼吸动画

由对象自己的 render config 决定。

### 3.3 移动和演出分离
空间中的宠物移动要拆成：

- **空间层位移**：位置、路径、朝向、目标点
- **角色演出层**：walk/jump/sleep 等动作与表情

Live2D 负责演出，不负责路径求解。

### 3.4 所有实体都应有统一的数据模型
宠物与对象都属于 Entity，只是能力不同。
统一模型有利于：

- AI 统一理解
- planner 统一处理
- renderer 统一消费

---

## 4. 实体模型设计

系统中的实体统一抽象为 `Entity`，分为以下类型：

- `pet`
- `world_object`
- `environment`

### 4.1 Entity 通用结构

```ts
export type EntityType = 'pet' | 'world_object' | 'environment'

export interface BaseEntity {
  id: string
  entityType: EntityType
  type: string
  tags?: string[]
}
```

### 4.2 宠物实体模型

```ts
export interface PetEntity extends BaseEntity {
  entityType: 'pet'
  species: string
  persona: {
    traits: string[]
    styleTags: string[]
  }
  capabilities: {
    locomotion: string[]
    interactions: string[]
    expressions: string[]
  }
  physicalConstraints: {
    canFly: boolean
    jumpHeight: 'none' | 'low' | 'medium' | 'high'
    movementRange: 'ground_only' | 'ground_and_platform'
  }
  renderer: {
    type: 'live2d'
    modelSrc: string
    modelId: string
  }
}
```

### 4.3 空间对象实体模型

```ts
export interface WorldObjectEntity extends BaseEntity {
  entityType: 'world_object'
  spatial: SpatialConfig
  collision: CollisionConfig
  anchors: Record<string, AnchorPoint>
  affordances: ObjectAffordance[]
  stateMachine: ObjectStateMachine
  render: ObjectRenderConfig
  aiExposure?: ObjectAIExposure
}
```

### 4.4 环境实体模型

```ts
export interface EnvironmentEntity extends BaseEntity {
  entityType: 'environment'
  variables: Record<string, string | number | boolean>
}
```

---

## 5. 空间设计

因为系统是纯 2D 空间，所以必须有明确的空间抽象。

### 5.1 空间基础模型

```ts
export interface WorldDefinition {
  id: string
  size: {
    width: number
    height: number
  }
  zones: ZoneDefinition[]
  navPoints: NavPoint[]
  edges: NavEdge[]
  objects: WorldObjectEntity[]
}
```

### 5.2 区域定义

```ts
export interface ZoneDefinition {
  id: string
  type: string
  label?: string
  polygon?: { x: number, y: number }[]
  tags?: string[]
}
```

用途：

- 标记床区、猫砂区、活动区
- 给 AI 和 planner 提供空间语义

### 5.3 导航点与边

```ts
export interface NavPoint {
  id: string
  x: number
  y: number
  zoneId?: string
  tags?: string[]
}

export interface NavEdge {
  from: string
  to: string
  moveType: 'walk' | 'jump_up' | 'jump_down'
  cost?: number
}
```

用途：

- planner 进行路径计算
- 支持“走到床边，再 jump_up 到床上”
- 支持爬架/高台的二维抽象

---

## 6. 对象配置设计

对象是系统中最关键的配置化部分之一。

### 6.1 对象空间属性

```ts
export interface SpatialConfig {
  position: { x: number, y: number }
  size?: { width: number, height: number }
  layer?: 'background' | 'midground' | 'foreground'
  zSortByY?: boolean
}

export interface CollisionConfig {
  blocking: boolean
  walkable: boolean
}
```

### 6.2 锚点设计

锚点用于 planner 和执行器，不给 AI 暴露底层坐标细节。

```ts
export interface AnchorPoint {
  x: number
  y: number
  description?: string
}
```

例如猫窝可定义：

- approach
- enter
- sleep

床可定义：

- approach_left
- jump_up
- rest

### 6.3 Affordance 设计

对象告诉系统“这个对象可被怎样使用”。

```ts
export interface ObjectAffordance {
  action: string
  roles: string[]
  requirements?: AffordanceRequirement[]
}

export interface AffordanceRequirement {
  type: string
  value?: string | number | boolean
}
```

例子：

- 猫窝支持 `sleep_on_object`
- 猫砂支持 `use_litter`
- 猫爬架支持 `climb_object`, `perch_on_object`

### 6.4 对象状态机

```ts
export interface ObjectStateMachine {
  initial: string
  states: Record<string, ObjectStateDefinition>
}

export interface ObjectStateDefinition {
  renderVariant: string
  transitions?: string[]
}
```

典型对象状态：

- normal
- occupied
- dirty
- active
- highlighted

### 6.5 对象渲染配置

```ts
export interface ObjectRenderConfig {
  renderer: 'sprite' | 'animated_sprite'
  variants: Record<string, ObjectRenderVariant>
}

export interface ObjectRenderVariant {
  asset: string
  effects?: string[]
}
```

### 6.6 AI 暴露摘要

```ts
export interface ObjectAIExposure {
  visible: boolean
  summary: Record<string, number | string | boolean>
}
```

例如：

- `comfortValue`
- `sleepValue`
- `toiletValue`
- `playValue`

这会帮助 AI 进行偏好判断。

---

## 7. 宠物配置设计

宠物配置是 AI 与 renderer 的核心桥梁。

### 7.1 宠物能力配置

```ts
export interface PetCapabilityConfig {
  locomotion: string[]
  interactions: string[]
  expressions: string[]
}
```

例如：

- locomotion: `idle`, `walk`, `jump_short`, `sleep`
- interactions: `approach_object`, `look_at_object`, `sleep_on_object`
- expressions: `happy`, `curious`, `sleepy`

### 7.2 宠物状态模型

```ts
export interface PetRuntimeState {
  mood: string
  energy: number
  hunger: number
  cleanliness: number
  comfort: number
  currentAction?: string
  location: {
    x: number
    y: number
    zoneId?: string
  }
  targetObjectId?: string
}
```

### 7.3 宠物渲染映射配置

将语义动作映射到 Live2D 可执行动作。

```ts
export interface PetRenderMapping {
  actionTemplates: Record<string, PetActionTemplate>
  expressionMappings: Record<string, PetExpressionMapping>
}

export interface PetActionTemplate {
  sequence: string[]
}

export interface PetExpressionMapping {
  type: 'group' | 'parameters'
  group?: string
  parameters?: {
    id: string
    blend: 'Add' | 'Multiply' | 'Overwrite'
    value: number
  }[]
}
```

---

## 8. AI 决策设计

AI 层不能直接吃原始配置，而要吃“编译后的世界模型摘要”。

### 8.1 AI 输入上下文

```ts
export interface DecisionContext {
  pet: {
    id: string
    mood: string
    energy: number
    hunger: number
    currentAction?: string
    styleTags: string[]
    allowedActions: string[]
    allowedExpressions: string[]
  }
  world: {
    timeOfDay?: string
    zones: {
      id: string
      type: string
    }[]
    reachableObjects: DecisionObjectSummary[]
  }
  hints?: string[]
}
```

### 8.2 对象摘要

```ts
export interface DecisionObjectSummary {
  id: string
  type: string
  state: string
  reachable: boolean
  affordances: string[]
  scores?: Record<string, number>
}
```

### 8.3 AI 输出协议

AI 输出必须结构化。

```ts
export interface AIDecisionOutput {
  action: string
  targetObjectId?: string
  targetZoneId?: string
  petExpression?: string
  durationSec?: number
  priority?: 'low' | 'normal' | 'high'
  objectStateIntents?: ObjectStateIntent[]
}

export interface ObjectStateIntent {
  objectId: string
  state: string
}
```

例子：

```json
{
  "action": "sleep_on_object",
  "targetObjectId": "cat_bed_01",
  "petExpression": "sleepy",
  "durationSec": 20,
  "priority": "normal",
  "objectStateIntents": [
    {
      "objectId": "cat_bed_01",
      "state": "occupied"
    }
  ]
}
```

---

## 9. 上下文编译层设计

AI 不直接读完整配置，而由 `ContextCompiler` 编译。

### 9.1 编译职责

负责从：

- pet config
- pet runtime state
- world object config
- object runtime state
- world nav state
- environment state

编译出：

- 当前可达对象
- 对象可用 affordance
- 当前行为候选
- 风格提示
- 限制条件

### 9.2 编译流程

1. 读取宠物当前状态
2. 计算可达对象
3. 过滤对象 affordance（满足 requirements）
4. 根据宠物状态生成 allowedActions
5. 汇总对象摘要
6. 生成 hints

### 9.3 示例 hints

- `low energy favors rest-related actions`
- `cat_bed_01 has highest sleepValue`
- `litter_box_01 is usable and reachable`

---

## 10. Planner / Orchestrator 设计

AI 输出的是语义动作，需要本地 planner 转为执行计划。

### 10.1 Planner 主要职责

1. 校验 AI 输出是否合法
2. 计算目标对象路径
3. 选择动作模板
4. 生成对象状态切换时机
5. 处理失败回退

### 10.2 执行计划结构

```ts
export interface ExecutionPlan {
  id: string
  steps: ExecutionStep[]
}

export type ExecutionStep =
  | MoveStep
  | PetMotionStep
  | PetExpressionStep
  | ObjectStateStep
  | WaitStep

export interface MoveStep {
  type: 'move'
  path: { x: number, y: number }[]
  moveStyle?: string
}

export interface PetMotionStep {
  type: 'pet_motion'
  templateId: string
}

export interface PetExpressionStep {
  type: 'pet_expression'
  expression: string
}

export interface ObjectStateStep {
  type: 'object_state'
  objectId: string
  state: string
}

export interface WaitStep {
  type: 'wait'
  durationSec: number
}
```

### 10.3 行为模板示例

`sleep_on_object` 可能被规划成：

1. move 到 `approach` anchor
2. pet_motion: `turn_to_target`
3. pet_expression: `sleepy`
4. move 到 `sleep` anchor（可选）
5. pet_motion: `lie_down`
6. object_state: `occupied`
7. pet_motion: `sleep_loop`

---

## 11. 移动系统设计

你关心的“正常准确移动”，这里必须明确。

### 11.1 移动目标
实现：

- 宠物在 2D 空间中准确到达对象 anchor
- 移动过程中播放合理演出
- 不穿越障碍
- 可处理上下平台抽象

### 11.2 路径规划
使用 `navPoints + edges` 实现。
planner 输出路径点序列。

### 11.3 移动执行
移动执行器负责：

- 更新宠物场景节点位置
- 根据方向切换朝向
- 根据 moveStyle 切换 walking / jumping 等模板
- 到达终点后对齐 anchor

### 11.4 移动与 Live2D 的边界
Live2D 不负责求路径。
Live2D 只负责演出：

- walk_loop
- jump_short
- idle_stop
- turn

空间位置由外层容器移动。

---

## 12. Live2D 执行层设计

可借鉴 airi 的实现思路，但需要更高层封装。

### 12.1 Live2D 执行层职责
负责：

- 加载模型
- 播放 motion
- 应用 expression
- 应用参数修饰
- 接收外层位置/朝向控制

### 12.2 语义动作到 Live2D 的映射
例如：

- `walk_loop` -> motion group `Walk`
- `sleep_loop` -> motion group `Sleep`
- `sleepy` -> expression group `Sleepy`
- `curious` -> expression group 或参数组合

### 12.3 推荐保持的边界
Live2D executor 接口应接收：

- `playTemplate(templateId)`
- `setExpression(expressionId)`
- `setFacing(left|right)`
- `setWorldPosition(x, y)`

不要让 AI 或 planner 直接调用底层 Live2D parameter API。

---

## 13. 对象渲染系统设计

对象渲染与宠物渲染应分离。

### 13.1 对象渲染器职责
根据对象当前运行态：

- 选择 renderVariant
- 选择 asset
- 应用 effects
- 应用 layer/z-index

### 13.2 对象渲染状态来源
对象渲染状态由三部分决定：

1. 初始配置
2. 本地规则变更
3. AI 输出的 objectStateIntents

### 13.3 典型例子
AI 判定猫在猫窝睡觉。
planner 安排：

- 宠物 sleep_on_object
- 猫窝 state -> occupied

对象渲染器看到 `occupied`，自动切到 occupied 贴图。

---

## 14. 配置示例

下面给一个较完整的例子。

```json
{
  "world": {
    "id": "room_01",
    "size": { "width": 1280, "height": 720 },
    "zones": [
      { "id": "litter_zone", "type": "toilet_area" },
      { "id": "rest_zone", "type": "rest_area" },
      { "id": "bed_zone", "type": "bed_area" }
    ],
    "navPoints": [
      { "id": "p1", "x": 100, "y": 650, "zoneId": "litter_zone" },
      { "id": "p2", "x": 380, "y": 640, "zoneId": "rest_zone" },
      { "id": "p3", "x": 700, "y": 610, "zoneId": "bed_zone" },
      { "id": "p4", "x": 790, "y": 500, "zoneId": "bed_zone" }
    ],
    "edges": [
      { "from": "p1", "to": "p2", "moveType": "walk" },
      { "from": "p2", "to": "p3", "moveType": "walk" },
      { "from": "p3", "to": "p4", "moveType": "jump_up" }
    ]
  },
  "pet": {
    "id": "cat_01",
    "entityType": "pet",
    "type": "cat",
    "species": "cat",
    "persona": {
      "traits": ["curious", "clingy"],
      "styleTags": ["cute_soft", "small_movement"]
    },
    "capabilities": {
      "locomotion": ["idle", "walk", "jump_short", "sleep"],
      "interactions": ["approach_object", "sleep_on_object", "use_litter", "perch_on_object"],
      "expressions": ["happy", "curious", "sleepy"]
    },
    "physicalConstraints": {
      "canFly": false,
      "jumpHeight": "low",
      "movementRange": "ground_and_platform"
    },
    "renderer": {
      "type": "live2d",
      "modelSrc": "/pets/cat/model3.json",
      "modelId": "cat-main"
    }
  },
  "objects": [
    {
      "id": "litter_box_01",
      "entityType": "world_object",
      "type": "litter_box",
      "spatial": {
        "position": { "x": 120, "y": 620 },
        "layer": "midground",
        "zSortByY": true
      },
      "collision": { "blocking": false, "walkable": false },
      "anchors": {
        "approach": { "x": 140, "y": 660 }
      },
      "affordances": [
        { "action": "use_litter", "roles": ["pet"] }
      ],
      "stateMachine": {
        "initial": "clean",
        "states": {
          "clean": { "renderVariant": "clean" },
          "dirty": { "renderVariant": "dirty" }
        }
      },
      "render": {
        "renderer": "sprite",
        "variants": {
          "clean": { "asset": "/objects/litter/clean.png" },
          "dirty": { "asset": "/objects/litter/dirty.png" }
        }
      },
      "aiExposure": {
        "visible": true,
        "summary": {
          "toiletValue": 0.95
        }
      }
    },
    {
      "id": "cat_bed_01",
      "entityType": "world_object",
      "type": "cat_bed",
      "spatial": {
        "position": { "x": 420, "y": 610 },
        "layer": "midground",
        "zSortByY": true
      },
      "collision": { "blocking": false, "walkable": false },
      "anchors": {
        "approach": { "x": 390, "y": 640 },
        "sleep": { "x": 425, "y": 620 }
      },
      "affordances": [
        { "action": "sleep_on_object", "roles": ["pet"] }
      ],
      "stateMachine": {
        "initial": "normal",
        "states": {
          "normal": { "renderVariant": "default" },
          "occupied": { "renderVariant": "occupied" }
        }
      },
      "render": {
        "renderer": "sprite",
        "variants": {
          "default": { "asset": "/objects/cat-bed/default.png" },
          "occupied": { "asset": "/objects/cat-bed/occupied.png" }
        }
      },
      "aiExposure": {
        "visible": true,
        "summary": {
          "sleepValue": 0.9,
          "comfortValue": 0.85
        }
      }
    }
  ]
}
```

---

## 15. 运行时流程

### 15.1 启动阶段
1. 加载世界配置
2. 加载宠物配置
3. 加载对象配置
4. 初始化运行时状态
5. 初始化宠物渲染器
6. 初始化对象渲染器

### 15.2 决策循环
每个周期：

1. 读取 pet runtime state
2. 读取 object runtime state
3. 编译 DecisionContext
4. 调用 AI
5. 校验 AI 输出
6. planner 生成 ExecutionPlan
7. executor 执行

### 15.3 执行阶段
1. 移动宠物到目标 anchor
2. 播放 motion
3. 设置 pet expression
4. 设置 object state
5. renderer 响应状态变化

---

## 16. 失败与降级策略

必须有，因为 AI 输出不一定稳定。

### 16.1 非法 action
若 AI 输出不在 allowedActions 中：
- fallback 到 `idle`

### 16.2 不可达对象
若目标对象不可达：
- 选择同类型可达对象
- 或 fallback 到 idle / look_at

### 16.3 不支持的 expression
若表达不存在：
- fallback 到 neutral / sleepy / curious 等默认可用表达

### 16.4 模板缺失
若某动作模板不存在：
- 退化为 `move + idle`
- 或 `move + expression`

---

## 17. 与 Live2D 的匹配结论

这套系统和 Live2D 是匹配的，但边界要明确：

### 适合：
- 预置动作模板
- 表情与参数修饰
- 2D 场景位置移动
- 对象状态驱动渲染
- AI 决策语义行为

### 不适合：
- AI 直接生成底层动画参数
- 任意自由轨迹驱动全身动作
- 高自由度复杂运动学

因此正确路线是：

**AI -> 语义行为 -> planner -> 2D 场景移动 + Live2D 演出模板**

---

## 18. 后续扩展方向

可以逐步扩展：

1. 多宠物系统
2. 对象主动状态变化（如猫砂自动变脏）
3. 环境变量（昼夜/温度）影响 AI
4. 记忆系统（宠物偏爱某个猫窝）
5. 插件化对象类型
6. 行为评分系统替代纯 LLM 决策
7. 混合“规则 + AI”决策

---

## 19. 最终总结

这套系统的本质是：

> **用配置定义虚拟空间中的实体、能力、状态和渲染映射；用 AI 决定“谁在什么条件下做什么”；用 planner 把语义决策转译为可执行的 2D 空间移动与 Live2D 演出。**

最关键的三个设计点是：

1. **对象配置化不只是配图片，而是配 affordance + 状态机 + 渲染映射**
2. **AI 只输出语义意图和对象状态意图，不输出底层渲染命令**
3. **移动由空间系统控制，演出由 Live2D 执行**
