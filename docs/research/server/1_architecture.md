# 修仙世界模拟器 - 服务端代码解读

本文档对位于 `src` 目录下的服务端代码进行全面解读，涵盖架构设计、核心模块、游戏循环、AI 决策机制以及数据流转。

## 1. 概述 (Overview)

该项目的服务端是一个基于 **FastAPI** 构建的异步 Python 应用。它不仅提供 RESTful API 供前端调用，还通过 WebSocket 实时推送游戏状态和事件。服务端的核心是一个**回合制模拟器**（1 回合 = 1 个游戏月），负责驱动整个修仙世界的运转，包括 NPC（Avatar）的修炼、移动、战斗、社交以及宗门发展等。

## 2. 核心架构 (Core Architecture)

服务端的代码结构清晰，主要分为以下几个核心目录：

- `src/server/`: 包含 FastAPI 应用的入口点和路由定义。
- `src/sim/`: 包含游戏模拟器（Simulator）、存档/读档逻辑以及实体管理器。
- `src/classes/`: 定义了游戏中的所有核心实体（如 Avatar、World、Sect、Action 等）。
- `src/systems/`: 实现了独立的游戏系统逻辑（如战斗、境界、时间、奇遇等）。
- `src/run/`: 包含游戏启动、地图加载、静态数据加载等脚本。
- `src/utils/`: 提供各种工具函数（如 LLM 调用、配置读取、CSV 解析等）。

### 2.1 入口与 API (`src/server/main.py`)

`main.py` 是服务端的入口文件，定义了 FastAPI 应用实例 `app`。
- **全局状态**: 维护了一个全局字典 `game_instance`，其中包含 `world`（世界状态）和 `sim`（模拟器实例）。
- **WebSocket 管理**: `ConnectionManager` 负责管理客户端的 WebSocket 连接，用于广播游戏事件和状态更新。
- **API 路由**: 提供了丰富的 API，如 `/api/state`（获取当前状态）、`/api/events`（获取事件）、`/api/map`（获取地图）、`/api/control/*`（控制游戏暂停/恢复/步进）等。

### 2.2 游戏模拟器 (`src/sim/simulator.py`)

`Simulator` 类是游戏的心脏。它的 `step()` 方法定义了每个游戏月（回合）的执行流程。一个完整的 `step` 包含以下阶段：

1. **感知与认知更新**: 更新存活角色的视野，自动占据无主洞府。
2. **长期目标思考**: 角色评估并设定长期目标。
3. **聚集结算 (Gathering)**: 处理多人聚集事件（如秘境探索、拍卖会）。
4. **决策阶段**: 调用 AI（LLM）为每个角色决定本月要执行的动作。
5. **提交阶段**: 角色提交计划好的动作。
6. **执行阶段**: 执行动作（Action Tick）。
7. **关系演化**: 根据角色的交互更新彼此的好感度和关系。
8. **结算死亡**: 处理寿元耗尽或被击杀的角色。
9. **年龄与新生**: 增加角色年龄，处理凡人觉醒或新生儿诞生。
10. **被动结算**: 处理丹药消化、时间buff/debuff、触发奇遇等。
11. **绰号生成**: 根据角色的行为和成就生成绰号。
12. **天地灵机更新**: 更新世界级的全局 Buff/Debuff。
13. **城市繁荣度更新**: 更新地图上城市的繁荣度。
14. **归档与时间推进**: 记录事件日志，推进游戏时间（月份 +1）。

## 3. 核心实体 (Core Entities)

### 3.1 世界 (`src/classes/core/world.py`)
`World` 类是整个游戏状态的容器。它包含了：
- `Map`: 游戏地图，由多个 `Tile` 和 `Region` 组成。
- `MonthStamp`: 当前游戏时间。
- `AvatarManager` & `MortalManager`: 管理所有修仙者和凡人。
- `EventManager`: 集中管理游戏中发生的所有事件。
- `CirculationManager`: 管理世界中流通的珍贵物品。

### 3.2 角色 (`src/classes/core/avatar/core.py`)
`Avatar` 类代表游戏中的 NPC 或玩家角色。为了避免类过于庞大，它使用了 **Mixin 模式** 进行功能组合：
- `AvatarSaveMixin` / `AvatarLoadMixin`: 处理序列化和反序列化。
- `EffectsMixin`: 处理角色身上的各种状态效果（Buff/Debuff）。
- `InventoryMixin`: 管理角色的背包、灵石和材料。
- `ActionMixin`: 管理角色当前正在执行的动作和计划。

角色的核心属性包括：境界（`cultivation_progress`）、灵根（`root`）、性格（`personas`）、功法（`technique`）、法宝（`weapon`）、人际关系（`relations`）等。

### 3.3 动作系统 (`src/classes/action/` & `src/classes/mutual_action/`)
游戏中的行为被抽象为 `Action`（单人动作，如闭关、移动、炼器）和 `MutualAction`（双人交互，如论道、双修、战斗、赠送）。
每个动作都实现了 `step()` 方法，定义了该动作在一个月内产生的具体效果和事件。

## 4. 游戏系统 (Game Systems)

### 4.1 修仙境界 (`src/systems/cultivation.py`)
定义了修仙的境界（`Realm`：练气、筑基、金丹、元婴等）和阶段（`Stage`：前期、中期、后期）。境界的提升需要积累修为，并在突破时可能面临天劫。

### 4.2 战斗系统 (`src/systems/battle.py`)
战斗系统参考了《文明6》的战斗力计算机制。
- **基础战斗力**: 由角色的境界和阶段决定（如练气10，筑基20）。
- **属性克制**: 功法属性（金木水火土）之间存在克制关系，克制方获得额外战斗力加成。
- **胜率计算**: 使用 Sigmoid 函数基于双方的战斗力差值计算胜率，并引入一定的随机性来决定战斗结果和伤害。

## 5. AI 与大语言模型集成 (AI & LLM Integration)

该项目的一个显著特点是使用大语言模型（LLM）来驱动 NPC 的决策。

### 5.1 LLM 决策 (`src/classes/ai.py`)
`LLMAI` 类继承自抽象的 `AI` 基类。在每个回合的决策阶段：
1. 收集角色的当前状态、周围环境信息（视野内的其他角色、区域）。
2. 将这些信息与游戏中可用的动作列表（Action Infos）组合成 Prompt。
3. 异步调用 LLM（通过 `src/utils/llm/` 下的工具）。
4. LLM 返回一个 JSON 格式的决策结果，包含角色选择的动作（`action_name`）及其参数（`action_params`），以及角色的内心独白（`thinking`）。

这种机制使得 NPC 能够根据性格、人际关系和当前处境做出极其复杂和符合逻辑的行为，而不需要编写庞大的行为树。

## 6. 数据与配置 (Data & Configuration)

- **静态数据**: 游戏的基础数据（如宗门配置、功法列表、法宝属性、地名等）存储在 `static/game_configs/` 目录下的 CSV 文件中。
- **数据加载**: `src/utils/df.py` 负责解析这些 CSV 文件，并支持 i18n 多语言翻译注入。`src/run/data_loader.py` 负责在游戏初始化时将这些数据加载到内存中的注册表（Registry）里。
- **配置管理**: `src/utils/config.py` 使用 `OmegaConf` 读取 `static/config.yml`，管理游戏的全局参数（如 LLM API Key、游戏节奏参数等）。

## 7. 总结

修仙世界模拟器的服务端是一个高度模块化、数据驱动且深度集成 LLM 的系统。它通过 FastAPI 提供高效的并发处理能力，利用面向对象和 Mixin 模式构建了复杂的修仙世界实体，并通过 LLM 赋予了 NPC 极高的自由度和智能，从而实现了一个动态演化的沙盒修仙世界。