# 服务端启动与配置加载流程

本文档详细解读了修仙世界模拟器服务端的启动流程、配置加载机制，特别是与大语言模型（LLM）API Key 相关的配置管理。

## 1. 启动流程概述

服务端的入口文件是 `src/server/main.py`，基于 FastAPI 框架构建。其启动流程主要分为以下几个阶段：

### 1.1 FastAPI Lifespan (生命周期)
在 FastAPI 应用启动时，会触发 `lifespan` 异步上下文管理器：
1. **日志过滤**：添加 `EndpointFilter` 过滤掉前端高频轮询 `/api/init-status` 产生的日志噪音。
2. **语言与路径初始化**：
   - 读取 `CONFIG.system.language`（默认为 `zh-CN`）。
   - 调用 `update_paths_for_language()` 更新静态数据（CSV）和模板（Templates）的加载路径。
   - 调用 `reload_game_configs()` 重新加载多语言配置。
3. **静态数据重载**：调用 `reload_all_static_data()` 强制刷新内存中的业务静态数据（如宗门、功法、法宝等），确保与当前语言设置一致。
4. **启动游戏循环**：通过 `asyncio.create_task(game_loop())` 启动后台游戏循环任务。此时游戏处于 `idle`（空闲）和 `paused`（暂停）状态，等待前端发送开始指令。
5. **开发模式前端启动**：如果带有 `--dev` 参数，服务端会自动通过 `subprocess` 在 `web` 目录下执行 `npm run dev` 启动前端服务。

### 1.2 游戏初始化 (`/api/game/start`)
当用户在前端点击“开始新游戏”时，会调用 `/api/game/start` 接口：
1. **保存本地配置**：将前端传递的参数（如初始 NPC 数量、宗门数量、世界历史等）写入 `static/local_config.yml`。
2. **合并配置**：将新写入的本地配置合并到全局 `CONFIG` 对象中。
3. **触发异步初始化**：调用 `init_game_async()` 开始真正的游戏世界生成。

### 1.3 异步生成世界 (`init_game_async`)
该函数负责从零构建游戏世界，包含多个阶段（Phase），并通过全局变量 `game_instance` 向前端同步进度：
- **Phase 0 (扫描资源)**：重置静态数据，扫描头像资源。
- **Phase 1 (加载地图)**：加载地图数据，初始化 SQLite 事件数据库。
- **Phase 2 (处理历史)**：处理用户输入的世界历史背景。
- **Phase 3 (初始化宗门)**：生成宗门并分配领地。
- **Phase 4 (生成角色)**：生成初始 NPC（Avatar）。
- **Phase 5 (检查 LLM)**：调用 `test_connectivity()` 检查 LLM 连通性。
- **Phase 6 (生成初始事件)**：生成角色初始关系和事件。

## 2. 配置加载机制

配置管理的核心逻辑位于 `src/utils/config.py`，使用了 `OmegaConf` 库来实现多层配置的合并。

### 2.1 配置文件结构
- **`static/config.yml`**：基础配置文件，包含默认的游戏参数、路径定义、LLM 默认模式等。受版本控制。
- **`static/local_config.yml`**：本地配置文件，用于存储用户自定义的设置（如 API Key、自定义游戏参数）。不受版本控制（通常在 `.gitignore` 中）。

### 2.2 加载逻辑 (`load_config`)
1. 创建空的 `base_config` 和 `local_config`。
2. 如果 `static/config.yml` 存在，则加载为 `base_config`。
3. 如果 `static/local_config.yml` 存在，则加载为 `local_config`。
4. 使用 `OmegaConf.merge(base_config, local_config)` 合并两者。**`local_config` 的优先级更高**，会覆盖 `base_config` 中的同名配置。
5. 将 `paths` 节点下的所有字符串路径转换为 `pathlib.Path` 对象。
6. 导出一个全局单例 `CONFIG` 供全项目使用。

## 3. LLM API Key 及相关配置

LLM 的配置是游戏运行的核心，相关逻辑主要在 `src/utils/llm/config.py` 和 `src/utils/llm/client.py` 中。

### 3.1 LLM 配置结构
在 `config.yml` / `local_config.yml` 中，LLM 配置通常位于 `llm` 节点下：
```yaml
llm:
  api_key: "sk-..."          # 用户的 API Key (通常保存在 local_config.yml)
  base_url: "https://..."    # OpenAI 兼容接口的 Base URL
  model_name: "gpt-4o"       # 默认/普通任务使用的模型
  fast_model_name: "gpt-4o-mini" # 快速任务使用的模型
  mode: "default"            # 全局模式覆盖 (normal/fast/default)
  default_modes:             # 细粒度任务模式配置
    action_decision: "normal"
    relation_resolver: "fast"
    # ...
```

### 3.2 LLM 模式 (LLMMode)
游戏定义了两种主要的 LLM 调用模式：
- **NORMAL**：用于需要深度推理的复杂任务（如动作决策 `action_decision`）。读取 `model_name`。
- **FAST**：用于需要快速响应、逻辑相对简单的任务（如关系结算 `relation_resolver`）。读取 `fast_model_name`。

`get_task_mode(task_name)` 函数会根据 `config.yml` 中的 `llm.default_modes` 映射表，自动为不同任务选择合适的模式。

### 3.3 配置读取 (`LLMConfig.from_mode`)
在每次发起 LLM 请求前，系统会调用 `LLMConfig.from_mode(mode)` 动态构建配置对象：
1. 从全局 `CONFIG.llm` 中读取 `api_key` 和 `base_url`。
2. 根据传入的 `mode`（NORMAL 或 FAST），选择读取 `model_name` 还是 `fast_model_name`。
3. 返回一个冻结的 `LLMConfig` 数据类实例。

## 4. LLM 连通性测试

为了防止游戏在运行中途因为 API Key 错误或网络问题崩溃，服务端在启动和配置阶段提供了连通性测试机制。

### 4.1 测试逻辑 (`test_connectivity`)
位于 `src/utils/llm/client.py` 中的 `test_connectivity` 函数负责执行测试：
1. 构造一个极简的 Prompt（`"test"`）。
2. 使用 `urllib` 向配置的 `base_url` 发送一个同步的 POST 请求。
3. 捕获异常并解析 HTTP 状态码：
   - **401 / invalid_api_key**：提示 API Key 无效。
   - **403 / Forbidden**：提示权限或配额不足。
   - **404**：提示 Base URL 错误（服务地址不存在）。
   - **Timeout / Connection**：提示网络连接问题。
4. 返回 `(bool, str)` 元组，表示是否成功及具体的中文错误提示。

### 4.2 触发时机
连通性测试在以下两个时机被触发：
1. **前端配置页**：前端调用 `/api/config/llm/status` 接口时，服务端会分别对 `NORMAL` 和 `FAST` 两个模型进行连通性测试，并将结果返回给前端展示。
2. **游戏初始化 (Phase 5)**：在 `init_game_async` 的第 5 阶段，系统会再次调用 `test_connectivity` 确保 LLM 服务可用，如果失败，初始化流程会中断并抛出错误。