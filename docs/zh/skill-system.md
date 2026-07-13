# 技能系统设计

> 技能发现、YAML 格式、Skills Hub、技能创建与管理流程

---

## 目录

- [概述](#概述)
- [技能目录结构](#技能目录结构)
- [技能格式](#技能格式)
- [技能发现与索引](#技能发现与索引)
- [技能系统提示词注入](#技能系统提示词注入)
- [Skills Hub](#skills-hub)
- [技能管理工具](#技能管理工具)
- [技能安全机制](#技能安全机制)
- [技能创建流程](#技能创建流程)
- [平台过滤](#平台过滤)

---

## 概述

技能系统是 Hermes Agent 的**过程性知识库**，以 YAML + Markdown 格式存储具体操作步骤和领域知识。与工具（提供能力）不同，技能（提供知识）告诉 Agent **如何**使用工具完成特定任务。

```
技能 vs 工具:
+------------------+  +------------------+
|  工具 (能力)      |  |  技能 (知识)      |
|  terminal        |  |  git-workflow    |
|  browser         |  |  deploy-docker   |
|  file_operations |  |  research-paper  |
|  web_search      |  |  debug-python    |
+------------------+  +------------------+
         |                      |
         v                      v
    "我能做什么"           "我该怎么做"
```

---

## 技能目录结构

技能按类别组织在 `skills/` 目录下，当前有 30+ 个类别：

```
skills/
|-- software-development/    # 软件开发
|   |-- index.yaml               # 类别索引
|   |-- git-workflow.md          # Git 工作流
|   |-- debug-python.md         # Python 调试
|   +-- ...
|
|-- devops/                  # 运维
|   |-- index.yaml
|   |-- docker-deploy.md
|   +-- ...
|
|-- research/                # 研究
|   |-- index.yaml
|   |-- academic-paper.md
|   +-- ...
|
|-- creative/                # 创意
|-- data-science/            # 数据科学
|-- email/                   # 邮件
|-- gaming/                  # 游戏
|-- github/                  # GitHub
|-- media/                   # 媒体
|-- social-media/            # 社交媒体
|-- smart-home/              # 智能家居
|-- productivity/            # 生产力
|-- note-taking/             # 笔记
|-- feeds/                   # 信息流
|-- diagramming/             # 绘图
|-- mcp/                     # MCP
|-- mlops/                   # ML 运维
|-- domain/                  # 领域知识
|-- red-teaming/             # 红队测试
|-- leisure/                 # 休闲
|-- apple/                   # Apple 生态
|-- autonomous-ai-agents/    # 自主 AI Agent
|-- inference-sh/            # 推理服务
|-- gifs/                    # GIF 制作
|-- dogfood/                 # 自用技能
+-- index-cache/             # 索引缓存
```

此外还有 `optional-skills/` 目录，存放不默认加载的可选技能。

---

## 技能格式

### index.yaml -- 类别索引

每个技能类别有一个 `index.yaml` 文件：

```yaml
# skills/software-development/index.yaml
category: software-development
description: "Software development skills and best practices"
skills:
  - name: git-workflow
    file: git-workflow.md
    description: "Git branching, committing, and PR workflow"
    conditions:
      - "user mentions git"
      - "user asks about version control"
    platforms: ["cli", "telegram", "discord"]
    
  - name: debug-python
    file: debug-python.md
    description: "Python debugging techniques"
    conditions:
      - "user has a Python error"
      - "user asks to debug"
```

### 技能文件 -- Markdown

```markdown
---
name: git-workflow
description: Git branching and PR workflow
version: 1.0
author: nous-research
conditions:
  - "user mentions git"
  - "user asks about branching"
platforms: ["cli"]
---

# Git Workflow

## 分支策略

1. 从 main 创建 feature 分支
2. 使用有意义的分支名 (feature/xxx, fix/xxx)
3. 频繁提交，小步前进

## 提交规范

```bash
git commit -m "type(scope): description"
```

类型: feat, fix, docs, style, refactor, test, chore

## PR 流程

1. 推送分支到远端
2. 创建 PR，填写描述
3. 请求代码审查
4. 合并后删除分支
```

### 前置 Frontmatter 字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | 技能唯一标识 |
| `description` | 是 | 技能描述 |
| `version` | 否 | 版本号 |
| `author` | 否 | 作者 |
| `conditions` | 否 | 触发条件列表 |
| `platforms` | 否 | 适用平台列表 |
| `requires_tools` | 否 | 依赖的工具列表 |

---

## 技能发现与索引

### 扫描流程

```
build_skills_system_prompt()
    |
    |-- 1. 获取所有技能目录
    |       get_all_skills_dirs()
    |       -> [skills/, optional-skills/, ~/.hermes/skills/]
    |
    |-- 2. 遍历每个目录的 index.yaml
    |       iter_skill_index_files()
    |
    |-- 3. 解析 frontmatter
    |       parse_frontmatter(content)
    |       -> 提取 name, description, conditions
    |
    |-- 4. 平台过滤
    |       skill_matches_platform(skill, current_platform)
    |
    |-- 5. 排除已禁用的技能
    |       get_disabled_skill_names()
    |
    +-- 6. 生成技能索引摘要
            -> 注入系统提示词
```

### 索引缓存

为了避免每次启动都重新扫描，技能索引支持缓存：

```
skills/index-cache/
    index.json         # 缓存的技能索引
```

---

## 技能系统提示词注入

技能不是作为完整内容注入的，而是生成一个**索引**告诉模型有哪些技能可用：

```
系统提示词中的技能部分:

[Available Skills]
You have access to the following skills. When a user request matches
a skill's conditions, use the skill_view tool to load the full skill
content before proceeding.

Categories:
- software-development: git-workflow, debug-python, code-review, ...
- devops: docker-deploy, kubernetes, ci-cd, ...
- research: academic-paper, literature-review, ...
...
```

### 按需加载

技能内容只在需要时才通过 `skill_view` 工具加载到上下文中，避免不必要的 token 消耗：

```
用户: "帮我设置 Docker 部署"
    |
    |-- 模型识别: 匹配 devops/docker-deploy 技能
    |
    |-- 调用 skill_view(skill="docker-deploy")
    |       -> 读取 skills/devops/docker-deploy.md
    |       -> 返回完整技能内容
    |
    +-- 模型按技能指导执行操作
```

---

## Skills Hub

`tools/skills_hub.py` (97KB) 是技能的社区分享平台客户端：

### 功能

| 功能 | 说明 |
|------|------|
| 浏览 | 浏览社区分享的技能 |
| 搜索 | 按关键词搜索技能 |
| 安装 | 下载并安装社区技能 |
| 发布 | 将本地技能发布到 Hub |
| 更新 | 检查并更新已安装技能 |

### 使用方式

```bash
# CLI 命令
hermes skills hub browse          # 浏览社区技能
hermes skills hub search "docker" # 搜索技能
hermes skills hub install <name>  # 安装技能
hermes skills hub publish <path>  # 发布技能
```

或通过工具调用：

```
skill_manage(action="hub_search", query="docker deployment")
skill_manage(action="hub_install", skill_name="docker-deploy")
```

---

## 技能管理工具

### skills_list

列出所有可用技能：

```python
registry.register(
    name="skills_list",
    toolset="skills",
    handler=_handle_skills_list,
    # 返回技能索引摘要
)
```

### skill_view

查看特定技能的完整内容：

```python
registry.register(
    name="skill_view",
    toolset="skills",
    handler=_handle_skill_view,
    schema={
        "function": {
            "parameters": {
                "properties": {
                    "skill": {"type": "string", "description": "Skill name"},
                },
            },
        },
    },
)
```

### skill_manage

技能管理操作：

```python
registry.register(
    name="skill_manage",
    toolset="skills",
    handler=_handle_skill_manage,
    schema={
        "function": {
            "parameters": {
                "properties": {
                    "action": {
                        "type": "string",
                        "enum": [
                            "create", "edit", "delete",
                            "enable", "disable",
                            "hub_search", "hub_install", "hub_publish",
                        ],
                    },
                    "skill_name": {"type": "string"},
                    "content": {"type": "string"},
                },
            },
        },
    },
)
```

---

## 技能安全机制

`tools/skills_guard.py` (42KB) 实现了技能安全检查：

### 安全检查项

| 检查 | 说明 |
|------|------|
| 路径穿越 | 防止技能文件引用上级目录 |
| 注入检测 | 检查技能内容中的提示注入模式 |
| 大小限制 | 限制单个技能文件的大小 |
| 格式验证 | 验证 YAML frontmatter 格式 |

### 社区技能审核

从 Skills Hub 安装的社区技能在加载前会经过安全扫描：

```
Hub 安装
    |
    |-- 下载技能内容
    |
    |-- 安全扫描
    |       - 检查注入模式
    |       - 检查危险指令
    |       - 检查文件大小
    |
    |-- 扫描通过 -> 安装到 ~/.hermes/skills/
    |
    +-- 扫描失败 -> 拒绝安装，提示用户
```

---

## 技能创建流程

### 通过 CLI 创建

```bash
# 交互式创建
hermes skills create

# 指定参数
hermes skills create --name "my-skill" --category "devops"
```

### 通过工具调用创建

```
skill_manage(
    action="create",
    skill_name="my-deployment-skill",
    content="""---
name: my-deployment-skill
description: Custom deployment workflow
conditions:
  - "user asks about deployment"
---

# My Deployment Skill

## Steps
1. Build the project
2. Run tests
3. Deploy to staging
4. Verify
5. Deploy to production
"""
)
```

### Agent 自主创建

Agent 在执行任务后可以将成功的操作步骤提取为新技能：

```
用户: "帮我设置 Nginx 反向代理"
    |
    |-- Agent 完成任务
    |
    |-- Agent 识别: 这个过程可以提取为技能
    |
    +-- 调用 skill_manage(action="create", ...)
            -> 创建 nginx-reverse-proxy.md
            -> 下次类似请求可以直接使用
```

这是 "自进化" (self-improving) 的核心机制之一。

---

## 平台过滤

技能可以指定适用的平台，在不适用的平台上不会出现在索引中：

```python
def skill_matches_platform(skill: dict, platform: str) -> bool:
    """检查技能是否适用于当前平台。
    
    如果技能没有指定 platforms，则适用于所有平台。
    """
    platforms = skill.get("platforms", [])
    if not platforms:
        return True  # 没有限制 = 适用所有平台
    return platform in platforms
```

### 平台过滤示例

| 技能 | 适用平台 | 说明 |
|------|----------|------|
| git-workflow | cli | 仅 CLI (需要终端) |
| quick-reply | telegram, discord | 消息平台专用 |
| code-review | cli, acp | 开发环境 |
| smart-home | homeassistant | 智能家居专用 |
| research | (全部) | 无限制 |

---

## 技能同步

`tools/skills_sync.py` (11KB) 处理技能的同步和版本管理：

### 功能

| 功能 | 说明 |
|------|------|
| 本地到远端 | 将本地技能同步到 Skills Hub |
| 远端到本地 | 从 Skills Hub 拉取更新 |
| 版本比较 | 检查本地版本与远端版本差异 |
| 冲突处理 | 处理本地修改与远端更新的冲突 |

---

## 相关文档

- [工具系统](tool-system.md) -- 技能管理工具的注册
- [核心 Agent 循环](core-agent-loop.md) -- 技能在系统提示词中的注入
- [CLI 系统](cli-system.md) -- 技能管理 CLI 命令
- [架构概述](architecture-overview.md) -- 技能在系统中的定位
