# RL 训练环境

> Gym 环境基类、Agent 循环、工具上下文、基准测试、轨迹生成与压缩

---

## 目录

- [概述](#概述)
- [架构分层](#架构分层)
- [HermesAgentBaseEnv 基类](#hermesagentbaseenv-基类)
- [HermesAgentLoop 多轮循环](#hermesagentloop-多轮循环)
- [ToolContext 工具上下文](#toolcontext-工具上下文)
- [具体环境](#具体环境)
- [BatchRunner 并行轨迹生成](#batchrunner-并行轨迹生成)
- [TrajectoryCompressor 轨迹压缩](#trajectorycompressor-轨迹压缩)

---

## 概述

Hermes Agent 集成了 **Atropos** RL 训练框架，将 Agent 的工具调用能力暴露为标准的 Gym 环境。这套系统支持：

- 使用真实工具（终端、浏览器、文件系统）的强化学习训练
- SWE-Bench 风格的代码修复任务
- 多步 Web 研究任务
- On-Policy Distillation (OPD) 密集训练信号
- 并行轨迹生成和压缩

```
Atropos Trainer
  │
  ├─ HermesAgentBaseEnv (抽象基类)
  │    ├─ HermesAgentLoop (多轮 Agent 循环)
  │    │    └─ handle_function_call() (工具调度)
  │    ├─ ToolContext (奖励函数的工具访问)
  │    └─ ScoredDataGroup (训练数据)
  │
  ├─ 具体环境:
  │    ├─ TerminalTestEnv     (测试: 文件创建任务)
  │    ├─ HermesSWEEnv        (SWE-Bench: Modal 沙箱)
  │    ├─ WebResearchEnv      (Web 研究: FRAMES 数据集)
  │    └─ AgenticOPDEnv       (OPD: 密集信号训练)
  │
  └─ 离线工具:
       ├─ batch_runner.py            (并行轨迹生成)
       └─ trajectory_compressor.py   (轨迹压缩)
```

---

## 架构分层

### 四层架构

| 层 | 模块 | 职责 |
|----|------|------|
| L0 | `agent_loop.py` | 可复用的多轮 Agent 循环，标准 OpenAI 工具调用 |
| L1 | `tool_context.py` | 每个 rollout 的工具访问句柄，供奖励函数使用 |
| L2 | `hermes_base_env.py` | 抽象基类，Atropos 集成桩代码 |
| L3 | 具体环境 | 数据集加载、prompt 格式化、奖励计算 |

### 两阶段运行模式

| 阶段 | 服务器类型 | 说明 |
|------|-----------|------|
| Phase 1 | OpenAI 兼容服务器 | vLLM/SGLang/OpenRouter，标准 ChatCompletion |
| Phase 2 | VLLM ManagedServer | 客户端工具调用解析器，token 级跟踪 |

---

## HermesAgentBaseEnv 基类

### 类定义

```python
class HermesAgentBaseEnv(BaseEnv):
    """所有 Hermes Agent 环境共享的 Atropos 集成基类。"""

    # 子类只需实现:
    # setup()          — 加载数据集
    # get_next_item()  — 返回下一个任务项
    # format_prompt()  — 将任务项转换为用户消息
    # compute_reward() — 对 rollout 打分（可访问 ToolContext）
    # evaluate()       — 定期评估
```

### HermesAgentEnvConfig 配置

```python
class HermesAgentEnvConfig(BaseEnvConfig):
    """Agent 环境配置，扩展 BaseEnvConfig。"""

    # 工具配置
    toolsets: List[str]              # 启用的工具集
    disabled_toolsets: List[str]     # 禁用的工具集

    # 终端后端
    terminal_backend: str            # local/modal/docker/daytona

    # Agent 循环参数
    max_turns: int                   # 最大 LLM 调用次数
    tool_pool_size: int             # 工具线程池大小

    # 工具调用解析（Phase 2）
    tool_call_parser: str           # hermes/hermes_json/...
```

### 核心流程

```
collect_trajectories(item)
  │
  ├─ sample_toolsets_from_distribution()  ← 采样工具集组合
  │
  ├─ format_prompt(item) → user_message
  │
  ├─ HermesAgentLoop.run() → AgentResult
  │    └─ (多轮: LLM → 工具调用 → 结果 → LLM → ...)
  │
  ├─ ToolContext(task_id) → ctx
  │    └─ 提供工具访问给奖励函数
  │
  ├─ compute_reward(item, result, ctx) → score
  │
  └─ ScoredDataGroup(items=[ScoredDataItem(messages, score)])
```

---

## HermesAgentLoop 多轮循环

### 核心设计

`agent_loop.py` 实现了可复用的多轮 Agent 引擎：

```python
@dataclass
class AgentResult:
    messages: List[Dict[str, Any]]        # 完整对话历史
    managed_state: Optional[Dict] = None  # Phase 2 状态
    turns_used: int = 0                   # LLM 调用次数
    finished_naturally: bool = False      # 是否自然结束
    reasoning_per_turn: List[str] = []    # 每轮推理内容
    tool_errors: List[ToolError] = []     # 工具执行错误
```

### 执行循环

```python
async def run(
    self,
    user_message: str,
    system_prompt: str,
    tools: List[Dict],
    max_turns: int,
    task_id: str,
) -> AgentResult:
    """运行多轮 Agent 循环。

    循环逻辑:
    1. 调用 LLM (带 tools 参数)
    2. 如果有 tool_calls → 执行工具 → 追加结果 → 继续循环
    3. 如果无 tool_calls → 结束（自然完成）
    4. 达到 max_turns → 结束（预算耗尽）
    """
```

### 工具执行线程化

工具执行在独立线程池中运行，避免与 Atropos 事件循环死锁：

```python
_tool_executor = ThreadPoolExecutor(max_workers=128)

# 工具调用通过 loop.run_in_executor() 在工作线程中执行
# 工作线程获得干净的事件循环（不与 Atropos 的循环冲突）

def resize_tool_pool(max_workers):
    """运行时调整线程池大小。由 HermesAgentBaseEnv.__init__ 调用。"""
```

### 错误记录

每个工具调用错误都被记录到 `AgentResult.tool_errors`：

```python
@dataclass
class ToolError:
    turn: int              # 出错的轮次
    tool_name: str         # 工具名
    arguments: str         # 参数（截断）
    error: str             # 错误消息
    tool_result: str       # 返回给模型的结果
```

---

## ToolContext 工具上下文

### 设计理念

`ToolContext` 为奖励/验证函数提供对所有 Hermes 工具的无限制访问：

```python
class ToolContext:
    """每个 rollout 的工具访问句柄。

    使用与模型 rollout 相同的 task_id，因此终端/浏览器会话
    中的所有状态（文件、进程、浏览器标签）都被保留。
    """
```

### 使用示例

```python
async def compute_reward(self, item, result, ctx):
    # 在模型的终端沙箱中运行测试
    test = ctx.terminal("pytest -v")
    if test["exit_code"] == 0:
        return 1.0

    # 检查文件是否被创建
    content = ctx.read_file("/workspace/solution.py")
    if content.get("content"):
        return 0.5

    return 0.0
```

### 便捷方法

| 方法 | 对应工具 | 说明 |
|------|---------|------|
| `ctx.terminal(cmd)` | `terminal` | 执行终端命令 |
| `ctx.read_file(path)` | `read_file` | 读取文件 |
| `ctx.write_file(path, content)` | `write_file` | 写入文件 |
| `ctx.search_files(pattern)` | `search_files` | 搜索文件 |
| `ctx.call_tool(name, args)` | 任意工具 | 通用工具调用 |

### 清理

```python
async def cleanup(self):
    """清理 rollout 使用的资源。"""
    cleanup_vm(self.task_id)       # 终端沙箱
    cleanup_browser(self.task_id)  # 浏览器实例
```

---

## 具体环境

### SWE-Bench 环境

`hermes_swe_env/` — 代码修复任务，使用 Modal 沙箱：

```
任务: 给定一个 GitHub issue 和代码仓库，修复 bug
工具: 终端 (Modal 容器)、文件读写、搜索
奖励: pytest 通过率
数据集: SWE-Bench (HuggingFace)
```

### WebResearchEnv

`web_research_env.py` — 多步 Web 研究：

```python
# 奖励信号:
# - 答案正确性 (LLM 评审, 0.0-1.0)
# - 信息源多样性 (使用 >= 2 个不同域名)
# - 效率 (惩罚过多工具调用)
# - 工具使用 (奖励实际使用 web 工具)

# 数据集: FRAMES 基准 (Google, 2024)
# 多跳事实问题
```

### AgenticOPDEnv

`agentic_opd_env.py` — On-Policy Distillation 密集训练信号：

```
核心思想 (OpenClaw-RL, Princeton 2026):
  每次 Agent 收到下一状态信号（工具结果、错误跟踪、测试结果），
  该信号包含了关于 Agent 上一次响应如何改进的事后信息。

流程:
  1. 运行标准 Agent rollout（工具调用循环）
  2. 遍历对话，找到 (assistant_turn, next_state) 对
  3. 使用 LLM 评审从下一状态信号中提取 "hints"
  4. 构建增强 prompt（原始上下文 + hint）
  5. 通过 VLLM 的 prompt_logprobs 评分学生响应 token
  6. 打包教师的 top-K 预测为 distill_token_ids / distill_logprobs

训练器计算每 token 优势:
  A_t = teacher_logprob(token_t) - student_logprob(token_t)
  正值 → 教师认可此 token（上调权重）
  负值 → 教师反对（下调权重）
```

### Terminal Test 环境

`terminal_test_env/` — 简单的文件创建任务，用于测试整个栈的工作状态。

---

## BatchRunner 并行轨迹生成

### 功能

`batch_runner.py` 提供从数据集批量生成轨迹的能力：

```bash
# 基本用法
python batch_runner.py --dataset_file=data.jsonl --batch_size=10 --run_name=my_run

# 断点续传
python batch_runner.py --dataset_file=data.jsonl --run_name=my_run --resume

# 指定工具集分布
python batch_runner.py --dataset_file=data.jsonl --distribution=image_gen
```

### 核心特性

| 特性 | 说明 |
|------|------|
| 并行处理 | 基于 multiprocessing.Pool 的多进程并行 |
| 断点续传 | 检查点机制，中断后可恢复 |
| 轨迹格式 | from/value 对格式，兼容训练管道 |
| 工具统计 | 跨批次聚合工具使用统计 |
| 进度显示 | Rich 进度条（spinner + bar + ETA） |

### 工具集分布

通过 `toolset_distributions.py` 采样不同的工具集组合，增加训练多样性：

```python
from toolset_distributions import sample_toolsets_from_distribution

# 每个 rollout 使用不同的工具子集
toolsets = sample_toolsets_from_distribution("default")
```

### 输出格式

```json
{
  "conversations": [
    {"from": "system", "value": "..."},
    {"from": "human", "value": "..."},
    {"from": "gpt", "value": "...", "tool_calls": [...]},
    {"from": "tool", "value": "..."},
    {"from": "gpt", "value": "..."}
  ],
  "tool_stats": {
    "terminal": {"count": 5, "success": 4, "failure": 1},
    "read_file": {"count": 3, "success": 3, "failure": 0}
  }
}
```

---

## TrajectoryCompressor 轨迹压缩

### 设计目标

`trajectory_compressor.py` 对已完成的 Agent 轨迹进行后处理压缩，在目标 token 预算内保留训练信号质量。

### 压缩策略

```
保护区域:
  ├─ 头部: system + human + 第一个 gpt + 第一个 tool  (永不压缩)
  ├─ 尾部: 最后 N 轮 (最终行动和结论)
  └─ 中间: 从第 2 个 tool response 开始压缩

压缩方式:
  1. 仅压缩中间区域
  2. 按需压缩（只压缩够用的量）
  3. 用单条 human summary 消息替换被压缩的区域
  4. 保留未压缩的工具调用（模型在摘要后继续工作）
```

### 使用方式

```bash
# 压缩目录下的所有 JSONL 文件
python trajectory_compressor.py --input=data/my_run

# 压缩单个文件
python trajectory_compressor.py --input=data/trajectories.jsonl

# 15% 采样压缩
python trajectory_compressor.py --input=data/trajectories.jsonl --sample_percent=15

# 自定义 token 目标
python trajectory_compressor.py --input=data/trajectories.jsonl --target_max_tokens=16000
```

### 配置

```python
@dataclass
class CompressionConfig:
    tokenizer_name: str = "moonshotai/Kimi-K2-Thinking"
    trust_remote_code: bool = True
    # target_max_tokens: int  — 目标 token 预算
    # protect_last_n: int    — 保护尾部轮数
```

### 配置文件

支持 YAML 配置文件（见 `datagen-config-examples/trajectory_compression.yaml`）。

---

## 基准测试

### 目录结构

```
environments/benchmarks/
  ├─ terminalbench_2/    # Terminal-Bench 2.0 评估
  ├─ tblite/             # TB-Lite 轻量评估
  └─ yc_bench/           # YC-Bench 评估
```

### 运行模式

所有环境支持三种运行模式：

```bash
# serve — 连接 Atropos trainer 实时训练
python environments/xxx_env.py serve

# process — 离线数据生成
python environments/xxx_env.py process

# evaluate — 独立评估
python environments/xxx_env.py evaluate
```
