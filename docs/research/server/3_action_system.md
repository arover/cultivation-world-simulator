# 动作系统 (Action System)

本文档详细解读了修仙世界模拟器服务端的动作系统，包括动作的定义、注册、运行时状态管理以及单人动作与双人互动动作的实现机制。

## 1. 概述

动作系统是驱动游戏世界运转的核心引擎。在每个游戏月（回合）中，AI 决策系统会为每个 NPC（Avatar）生成一系列动作计划（Action Plan），随后这些计划会被提交并执行。动作系统负责处理动作的生命周期（开始、执行、完成/中断），并生成相应的游戏事件（Event）和状态变更。

## 2. 核心架构

动作系统的代码主要分布在 `src/classes/action/` 和 `src/classes/mutual_action/` 目录下，以及 `src/classes/action_runtime.py` 和 `src/classes/core/avatar/action_mixin.py` 中。

### 2.1 运行时状态 (`action_runtime.py`)

为了规范动作的执行过程，系统定义了以下几个核心数据结构：

- **`ActionStatus` (枚举)**：定义了动作的生命周期状态，包括 `RUNNING`（进行中）、`COMPLETED`（已完成）、`FAILED`（失败）、`CANCELLED`（被取消）和 `INTERRUPTED`（被打断）。
- **`ActionResult`**：所有动作的 `step()` 方法必须返回此对象。它包含当前状态、执行过程中产生的事件列表（`events`），以及可选的附加数据（`payload`）和建议的下一个动作（`next_action`）。
- **`ActionPlan`**：表示角色计划执行但尚未开始的动作。包含动作名称、参数、优先级等。
- **`ActionInstance`**：表示角色当前正在执行的动作实例，包含实例化的 `Action` 对象及其参数。

### 2.2 动作基类 (`action.py`)

`Action` 是所有动作的抽象基类。为了支持不同类型的动作，系统提供了一系列 Mixin 和子类：

- **`ActualActionMixin`**：定义了实际可执行动作的标准接口：
  - `can_start(**params) -> tuple[bool, str]`：检查动作是否满足执行条件。
  - `start(**params) -> Event | None`：动作开始时调用，通常返回一个表示动作开始的事件。
  - `step(**params) -> ActionResult`：动作的核心执行逻辑，每个游戏月调用一次。
  - `finish(**params) -> list[Event]`：动作完成时调用，进行清理并返回结束事件。
- **`InstantAction`**：一次性动作，在一次 `step` 内即可完成（如：购买物品、装备法宝）。
- **`TimedAction`**：长态动作，需要持续多个游戏月（如：闭关修炼、长途移动）。通过 `duration_months` 控制持续时间。

### 2.3 动作注册表 (`registry.py`)

`ActionRegistry` 负责管理所有可用的动作。
- 通过 `@register_action()` 装饰器，将动作类注册到系统中。
- AI 决策时，可以通过注册表获取所有“实际可执行”的动作列表，并将其作为 Prompt 的一部分提供给 LLM。
- 角色执行动作时，通过动作名称从注册表中实例化具体的动作对象。

## 3. 角色的动作管理 (`action_mixin.py`)

`Avatar` 类通过继承 `ActionMixin` 来获得动作管理能力。核心流程如下：

1. **加载计划 (`load_decide_result_chain`)**：将 AI 决策返回的动作列表转换为 `ActionPlan` 并加入角色的 `planned_actions` 队列。
2. **提交计划 (`commit_next_plan`)**：如果角色当前没有正在执行的动作，从队列中取出下一个计划，调用 `can_start` 检查条件。如果通过，则调用 `start` 触发开始事件，并将动作设为 `current_action`（即 `ActionInstance`）。
3. **执行动作 (`tick_action`)**：在每个游戏月的执行阶段，调用当前动作的 `step` 方法。
   - 如果返回 `COMPLETED`，则调用 `finish`，并清空 `current_action`（除非在执行过程中发生了动作抢占/切换）。
   - 收集执行过程中产生的所有事件，存入角色的 `_pending_events` 中，等待 Simulator 统一归档。

## 4. 互动动作 (`mutual_action.py`)

`MutualAction` 是一种特殊的动作类型，表示角色 A 对角色 B 发起的社交互动（如：交谈、赠礼、攻击、双修）。

### 4.1 异步反馈机制

互动动作的最大特点是**目标角色 B 会根据自身性格和与 A 的关系给出反馈**。这个反馈过程是由 LLM 异步决定的：

1. **发起 (`start`)**：A 发起互动，生成“A 对 B 发起某动作”的事件。
2. **首帧 (`step` 第一次调用)**：
   - 收集 A 和 B 的状态信息、关系数据。
   - 创建一个异步任务，调用 LLM（`interaction_feedback` 任务）来决定 B 的反馈。
   - 返回 `RUNNING` 状态。
3. **后续帧 (`step` 后续调用)**：
   - 检查 LLM 异步任务是否完成。
   - 如果未完成，继续返回 `RUNNING`。
   - 如果已完成，解析 LLM 返回的反馈（如：接受、拒绝、反击、逃跑）。
   - 调用 `_settle_feedback` 将反馈落地为具体的游戏逻辑（例如，如果 B 决定反击，则强制中断 B 当前的动作，并让 B 立即执行 `Attack` 动作）。
   - 生成反馈事件，返回 `COMPLETED`。

这种设计使得游戏中的社交互动极具动态性和不可预测性，NPC 之间的恩怨情仇完全由 LLM 根据上下文自然演化。

## 5. 总结

修仙世界模拟器的动作系统采用了高度抽象和模块化的设计。通过 `ActionStatus` 和 `ActionResult` 规范了动作的生命周期；通过 `ActionRegistry` 实现了动作的解耦和动态发现；通过 `ActionMixin` 将动作调度逻辑无缝集成到角色实体中。特别是 `MutualAction` 引入的异步 LLM 反馈机制，为游戏带来了极其丰富的涌现式社交玩法。