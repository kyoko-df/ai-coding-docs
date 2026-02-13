# Claude Code CLI 完整指南

> **版本**: v2.1.0  
> **适用于**: Claude Code CLI 最新版本  

## 目录

### 基础篇
1. [工具安装指南](#工具安装指南)
2. [常用命令详解](#常用命令详解)
3. [高效使用技巧](#高效使用技巧)

### 高级功能篇
4. [MCP (Model Context Protocol) 集成](#mcp-model-context-protocol-集成)
5. [Headless Mode 和自动化](#headless-mode-和自动化)
6. [Hooks 和自定义斜杠命令](#hooks-和自定义斜杠命令)
7. [Permissions 权限管理](#permissions-权限管理)
8. [CLAUDE.md 文件配置](#claudemd-文件配置)
9. [@-mentions 和文件引用功能](#-mentions-和文件引用功能)
10. [Plan Mode 和高级功能](#plan-mode-和高级功能)

### 实践篇
11. [SPEC方法论驱动的开发流程](#spec方法论驱动的开发流程)
12. [常见问题解决方案](#常见问题解决方案)
13. [最佳实践案例](#最佳实践案例)
   - [1. 项目初始化](#1-项目初始化)
   - [2. 代码重构](#2-代码重构)
   - [3. 功能开发](#3-功能开发)
   - [4. 代码质量保证](#4-代码质量保证)
   - [5. 文档维护](#5-文档维护)
   - [6. Output Styles 最佳实践](#6-output-styles-最佳实践)
   - [7. 团队协作最佳实践](#7-团队协作最佳实践)
   - [8. 企业级部署指南](#8-企业级部署指南)

---

## 工具安装指南

### Claude Code CLI 安装

#### 系统要求
- Node.js 18.0 或更高版本
- npm 包管理器
- 操作系统：macOS 10.15+、Ubuntu 20.04+/Debian 10+、Windows（需要 WSL2）
- 至少 4GB 内存
- 稳定的网络连接（用于身份验证和 AI 处理）

#### 安装步骤

**通过 npm 全局安装**
```bash
npm install -g @anthropic-ai/claude-code
```

#### 验证安装
```bash
claude --version
```

#### 首次启动和身份验证
```bash
# 进入项目目录
cd your-project-directory

# 启动 Claude Code
claude
```

首次启动时会要求进行身份验证：
- 按照 OAuth 流程使用 Anthropic 控制台账户登录
- 确保在 https://console.anthropic.com/ 上的账单信息有效
- 或者使用 API Key 模式（需要绑定信用卡）

### 其他相关工具安装

#### Git CLI
```bash
# macOS
brew install git

# Ubuntu/Debian
sudo apt-get install git

# Windows
winget install Git.Git
```

#### GitHub CLI
```bash
# macOS
brew install gh

# Ubuntu/Debian
sudo apt install gh

# Windows
winget install GitHub.cli
```

---

## 常用命令详解

### Claude Code 核心命令

#### CLI 命令

**启动交互式会话**
```bash
claude
```

**使用初始提示启动**
```bash
claude "query"
```

**运行一次性查询**
```bash
claude -p "query"
```

**处理管道内容**
```bash
cat file | claude -p "query"
```

**查看和修改配置**
```bash
claude config
claude config list
claude config set <KEY> <VALUE>
```

**更新到最新版本**
```bash
claude update
```

**恢复历史会话**
```bash
claude --continue  # 或 claude -c
claude --resume    # 或 claude -r
```

**跳过权限确认（高级用法）**
```bash
claude --dangerously-skip-permissions
```

#### 交互式斜杠命令

在 Claude Code 会话中使用的命令：

**项目初始化**
```bash
/init
/init --force  # 强制覆盖更新
```

**模型切换**
```bash
/model
```

**恢复历史会话**
```bash
/resume
```

**查看所有命令**
```bash
/
```

**报告问题**
```bash
/bug
```

### Git 常用命令

#### 基础操作
```bash
# 克隆仓库
git clone <repository-url>

# 查看状态
git status

# 添加文件
git add <file>
git add .  # 添加所有文件

# 提交更改
git commit -m "commit message"

# 推送到远程
git push origin <branch-name>

# 拉取更新
git pull origin <branch-name>
```

#### 分支管理
```bash
# 创建分支
git branch <branch-name>
git checkout -b <branch-name>  # 创建并切换

# 切换分支
git checkout <branch-name>
git switch <branch-name>  # 新语法

# 合并分支
git merge <branch-name>

# 删除分支
git branch -d <branch-name>
```

---

## 高效使用技巧

### 1. 命令别名设置

#### Bash/Zsh 别名
```bash
# 添加到 ~/.bashrc 或 ~/.zshrc
alias cc="claude"
alias claude-skip="claude --dangerously-skip-permissions"

# Git 别名
alias gs="git status"
alias ga="git add"
alias gc="git commit"
alias gp="git push"
alias gl="git pull"
```

#### Git 内置别名
```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
```

### 2. 配置文件优化

#### Claude Code 配置

**查看当前配置**
```bash
claude config list
```

**常用配置项**
```bash
# 设置首选模型
claude config set preferredModel claude-3-sonnet
claude config set preferredModel claude-3-opus

# 禁用自动更新
claude config set autoUpdateStatus disabled

# 设置主题
claude config set theme dark
claude config set theme light
```

**环境变量配置**
```bash
# API 认证（如使用 API 模式）
export ANTHROPIC_AUTH_TOKEN=your-api-key
export ANTHROPIC_BASE_URL=https://api.anthropic.com
```

### 3. 工作流自动化

#### 使用 npm scripts
```json
{
  "scripts": {
    "claude": "claude",
    "claude-init": "claude '/init'",
    "claude-skip": "claude --dangerously-skip-permissions"
  }
}
```

#### 使用 Makefile
```makefile
.PHONY: claude-init claude-dev

claude-init:
	claude '/init'

claude-dev:
	claude --dangerously-skip-permissions
```

### 4. Claude Code 高级使用技巧

#### 深度思考模式
Claude Code 支持不同级别的深度思考，消耗更多计算资源但提供更好的结果：

```bash
# 基础思考
claude "think 如何优化这个函数"

# 深度思考（按消耗递增）
claude "think hard 设计一个可扩展的架构"
claude "think harder 解决复杂的性能问题"
claude "ultrathink 全面重构整个系统"  # 最高级别，费用最高
```

#### 权限管理和自动化
```bash
# 跳过所有权限确认（高级用法）
claude --dangerously-skip-permissions

# 设置别名简化使用
alias claude-auto='claude --dangerously-skip-permissions'

# 在会话中临时设置不再询问权限
# 当 Claude 询问权限时，选择 "Don't ask again for this session"
```

#### 历史会话管理
```bash
# 恢复最近的会话
claude --continue
claude -c

# 选择历史会话恢复
claude --resume
claude -r

# 在交互模式中恢复会话
/resume
```

#### 图片和多媒体处理
```bash
# 在 macOS 中粘贴图片到 Claude Code
# 使用 Ctrl + V（不是 Cmd + V）

# 常用图片处理指令
claude "分析这个错误截图并提供解决方案"
claude "根据这个设计图实现网页布局"
claude "这个图片显示了什么问题？"
```

#### 终端快捷键
- `Ctrl + R`：搜索历史命令
- `Ctrl + L`：清屏
- `Ctrl + C`：终止当前命令
- `ESC`：中断 Claude Code 当前任务
- `ESC ESC`：编辑上一条消息
- `Option + Enter`：换行（macOS）
- `Tab`：命令自动补全
- `↑/↓`：浏览命令历史

#### 高效提示词技巧
```bash
# 明确具体的需求
claude "修复用户登录时不输入密码出现的空指针错误"

# 限制操作范围
claude "only modify this function, do not do anything else"

# 分步执行复杂任务
claude "分析思路并列出重构步骤（先不写代码）"
# 确认后再执行
claude "现在执行第一步的代码修改"

# 使用上下文文件
cat error.log | claude -p "分析这个错误日志并提供解决方案"
```

#### Output Styles（输出样式）功能

Output Styles 是 Claude Code 中的一种强大机制，用来控制模型生成内容的表达方式和结构模板。它不会改变 Claude 的核心能力或工具权限，而是通过预设的写作框架影响输出格式和风格。

**核心概念**：
- Output Styles 本质是一个系统提示词文件，存放在 `.claude/output-styles/` 目录下
- 可以包含元数据（名称、描述）和正文模板
- 允许将 Claude Code 用作不同类型的代理，同时保持核心功能不变

**内置样式类型**：
```bash
# 默认样式（Default）
# 标准的 Claude Code 输出格式

# 解释型样式（Explanatory）
# 提供详细分析和解释的输出格式

# 学习型样式（Learning）
# 循序渐进的教学式输出格式
```

**使用方法**：
```bash
# 在交互模式中切换输出样式
/output-style explanatory
/output-style learning
/output-style default

# 查看可用的输出样式
/output-style
```

**自定义样式创建**：
```bash
# 创建自定义样式目录
mkdir -p .claude/output-styles

# 创建自定义样式文件
cat > .claude/output-styles/code-review.md << 'EOF'
---
name: "Code Review"
description: "专业代码审查报告格式"
---

# 代码审查报告

## 概述
[简要描述审查的代码模块和主要功能]

## 代码质量评估
### 优点
- [列出代码的优秀之处]

### 问题和建议
- [列出发现的问题和改进建议]

## 安全性检查
[安全相关的评估和建议]

## 性能分析
[性能相关的评估和优化建议]

## 总结
[整体评估和下一步行动建议]
EOF

# 使用自定义样式
/output-style code-review
```

**常用自定义样式示例**：

1. **PRD模板样式**：
```bash
cat > .claude/output-styles/prd-template.md << 'EOF'
---
name: "PRD Template"
description: "产品需求文档模板格式"
---

# 产品需求文档 (PRD)

## 1. 产品概述
[产品背景、目标和价值主张]

## 2. 用户需求
[目标用户、用户场景和需求分析]

## 3. 功能需求
[详细功能列表和优先级]

## 4. 非功能性需求
[性能、安全、可用性等要求]

## 5. 技术方案
[技术架构和实现方案]

## 6. 项目计划
[时间安排和里程碑]
EOF
```

2. **API文档样式**：
```bash
cat > .claude/output-styles/api-docs.md << 'EOF'
---
name: "API Documentation"
description: "API文档标准格式"
---

# API 文档

## 端点信息
- **URL**: [API端点URL]
- **方法**: [HTTP方法]
- **描述**: [功能描述]

## 请求参数
| 参数名 | 类型 | 必需 | 描述 |
|--------|------|------|------|
| [参数] | [类型] | [是/否] | [说明] |

## 响应格式
```json
{
  "status": "success",
  "data": {},
  "message": "操作成功"
}
```

## 错误码
| 错误码 | 描述 | 解决方案 |
|--------|------|----------|
| [代码] | [说明] | [处理方法] |

## 示例
[请求和响应示例]
EOF
```

**高级配置技巧**：

#### MCP (Model Context Protocol) 集成

MCP 是 Anthropic 开发的开放标准，允许 Claude Code 连接到数百个外部工具和数据源。通过 MCP 服务器，Claude Code 可以访问工具、数据库和 API。<mcreference link="https://docs.anthropic.com/en/docs/claude-code/mcp" index="1">1</mcreference>

**MCP 核心概念**：
- **Host**: 发起请求的应用程序（如 Claude Code）
- **Client**: 管理主机和服务器之间通信的中介
- **Server**: 提供数据或功能的工具或服务（如 GitHub、数据库）

**MCP 命令详解**：

```bash
# 添加 MCP 服务器
claude mcp add <server-name> [options]

# 通过 HTTP 传输添加服务器
claude mcp add --transport http <server-name> <url>

# 通过 SSE 传输添加服务器
claude mcp add --transport sse <server-name> <url>

# 添加带环境变量的服务器
claude mcp add <server-name> --env KEY=VALUE --env KEY2=VALUE2 -- <command>

# 列出已安装的 MCP 服务器
claude mcp list

# 移除 MCP 服务器
claude mcp remove <server-name>

# 更新 MCP 服务器
claude mcp update <server-name>

# 检查 MCP 服务器状态
claude mcp status
```

**常用 MCP 服务器安装示例**：

1. **开发和测试工具**：
```bash
# Sentry - 错误监控和调试
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp

# Socket - 安全依赖分析
claude mcp add --transport http socket https://mcp.socket.dev/

# Hugging Face - AI 模型和应用访问
claude mcp add --transport http hugging-face https://huggingface.co/mcp

# Jam - 调试录制访问
claude mcp add --transport http jam https://mcp.jam.dev/mcp
```

2. **项目管理和文档**：
```bash
# Asana - 项目管理
claude mcp add --transport sse asana https://mcp.asana.com/sse

# Jira 和 Confluence
claude mcp add --transport sse atlassian https://mcp.atlassian.com/v1/sse

# ClickUp - 任务管理
claude mcp add clickup --env CLICKUP_API_KEY=YOUR_KEY --env CLICKUP_TEAM_ID=YOUR_ID -- npx -y @hauptsache.net/clickup-mcp

# Linear - 问题跟踪
claude mcp add --transport sse linear https://mcp.linear.app/sse

# Notion - 文档管理
claude mcp add --transport http notion https://mcp.notion.com/mcp
```

3. **数据库和数据管理**：
```bash
# Airtable - 数据库管理
claude mcp add airtable --env AIRTABLE_API_KEY=YOUR_KEY -- npx -y airtable-mcp-server

# HubSpot - CRM 数据
claude mcp add --transport http hubspot https://mcp.hubspot.com/anthropic

# Daloopa - 财务数据
claude mcp add --transport http daloopa https://mcp.daloopa.com/server/mcp
```

4. **支付和商务**：
```bash
# Stripe - 支付处理
claude mcp add --transport http stripe https://mcp.stripe.com

# PayPal - 支付集成
claude mcp add --transport http paypal https://mcp.paypal.com/mcp

# Square - 商务平台
claude mcp add --transport sse square https://mcp.squareup.com/sse
```

5. **设计和媒体**：
```bash
# Figma - 设计工具（需要 Figma Desktop）
claude mcp add --transport http figma-dev-mode-mcp-server http://127.0.0.1:3845/mcp

# Canva - 设计平台
claude mcp add --transport http canva https://mcp.canva.com/mcp

# InVideo - 视频创建
claude mcp add --transport sse invideo https://mcp.invideo.io/sse
```

**MCP 使用示例**：

```bash
# 使用 @-mentions 引用 MCP 资源
claude "基于 @jira-issue-ENG-4521 实现功能并在 @github 创建 PR"

# 分析监控数据
claude "检查 @sentry 和 @statsig 中 ENG-4521 功能的使用情况"

# 查询数据库
claude "从 @postgres 数据库中找到使用 ENG-4521 功能的 10 个随机用户邮箱"

# 集成设计
claude "根据 @slack 中发布的新 @figma 设计更新我们的标准邮件模板"
```

**MCP 配置文件管理**：

MCP 服务器配置存储在 `.claude.json` 或 `~/.claude.json` 文件中：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@composio/mcp@latest", "github"],
      "env": {
        "GITHUB_TOKEN": "your-token"
      }
    },
    "sentry": {
      "transport": "http",
      "url": "https://mcp.sentry.dev/mcp"
    }
  }
}
```

**MCP 安全注意事项**：
- 仅安装来自可信来源的 MCP 服务器 <mcreference link="https://docs.anthropic.com/en/docs/claude-code/mcp" index="1">1</mcreference>
- 定期审查已安装的 MCP 服务器
- 妥善管理 API 密钥和环境变量
- 使用最小权限原则配置服务器访问

#### Headless Mode 和自动化

Claude Code 包含 headless mode，专为非交互式环境设计，如 CI、pre-commit hooks、构建脚本和自动化流程。<mcreference link="https://www.anthropic.com/engineering/claude-code-best-practices" index="2">2</mcreference>

**Headless Mode 命令**：

```bash
# 基本 headless 模式
claude --headless "prompt"

# 结合跳过权限确认
claude --headless --dangerously-skip-permissions "prompt"

# 从文件读取提示
claude --headless --file prompt.txt

# 处理管道输入
cat input.txt | claude --headless "分析这个文件"

# 设置输出格式
claude --headless --output-format json "生成 API 响应"
claude --headless --output-format markdown "生成文档"

# 指定最大 token 数
claude --headless --max-tokens 1000 "简短总结"

# 设置超时时间
claude --headless --timeout 30 "快速分析"
```

**CI/CD 集成示例**：

1. **GitHub Actions 工作流**：
```yaml
name: Code Review with Claude
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  claude-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code
      - name: Review Changes
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          git diff HEAD~1 | claude --headless --dangerously-skip-permissions \
            "审查这些代码变更，提供改进建议" > review.md
      - name: Comment PR
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('review.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: review
            });
```

2. **Pre-commit Hook**：
```bash
#!/bin/sh
# .git/hooks/pre-commit

# 检查暂存的文件
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(js|ts|py|java)$')

if [ -n "$STAGED_FILES" ]; then
    echo "Running Claude Code review..."
    
    for FILE in $STAGED_FILES; do
        # 使用 Claude Code 检查每个文件
        REVIEW=$(claude --headless --dangerously-skip-permissions \
            "检查这个文件的代码质量和潜在问题: $(cat $FILE)")
        
        # 如果发现严重问题，阻止提交
        if echo "$REVIEW" | grep -i "严重\|critical\|error"; then
            echo "发现严重问题，提交被阻止:"
            echo "$REVIEW"
            exit 1
        fi
    done
fi

exit 0
```

3. **构建脚本集成**：
```bash
#!/bin/bash
# build-with-claude.sh

set -e

echo "开始构建流程..."

# 代码质量检查
echo "执行代码质量检查..."
claude --headless --dangerously-skip-permissions \
    "分析项目代码质量，生成改进建议" > quality-report.md

# 生成文档
echo "生成 API 文档..."
claude --headless --dangerously-skip-permissions \
    "基于代码生成 API 文档" > api-docs.md

# 运行测试
echo "运行测试套件..."
npm test

# 构建项目
echo "构建项目..."
npm run build

echo "构建完成！"
```

**自动化脚本示例**：

1. **批量文件处理**：
```bash
#!/bin/bash
# batch-process.sh

for file in src/**/*.js; do
    echo "处理文件: $file"
    claude --headless --dangerously-skip-permissions \
        "优化这个 JavaScript 文件的性能: $(cat $file)" > "${file}.optimized"
done
```

2. **日志分析自动化**：
```bash
#!/bin/bash
# analyze-logs.sh

LOG_FILE="/var/log/application.log"
REPORT_FILE="log-analysis-$(date +%Y%m%d).md"

# 分析最近的日志
tail -n 1000 "$LOG_FILE" | claude --headless --dangerously-skip-permissions \
    "分析这些应用日志，识别错误模式和性能问题" > "$REPORT_FILE"

echo "日志分析报告已生成: $REPORT_FILE"
```

3. **代码迁移助手**：
```bash
#!/bin/bash
# migrate-code.sh

SOURCE_DIR="$1"
TARGET_FRAMEWORK="$2"

find "$SOURCE_DIR" -name "*.js" | while read file; do
    echo "迁移文件: $file"
    claude --headless --dangerously-skip-permissions \
        "将这个 JavaScript 代码迁移到 $TARGET_FRAMEWORK: $(cat $file)" \
        > "${file%.js}.${TARGET_FRAMEWORK}.js"
done
```

**环境变量配置**：

```bash
# 设置 API 认证
export ANTHROPIC_API_KEY="your-api-key"
export ANTHROPIC_BASE_URL="https://api.anthropic.com"

# 设置默认行为
export CLAUDE_HEADLESS_MODE="true"
export CLAUDE_SKIP_PERMISSIONS="true"
export CLAUDE_OUTPUT_FORMAT="markdown"
export CLAUDE_MAX_TOKENS="2000"
export CLAUDE_TIMEOUT="60"
```

**Docker 容器中使用**：

```dockerfile
# Dockerfile
FROM node:18-alpine

# 安装 Claude Code
RUN npm install -g @anthropic-ai/claude-code

# 设置工作目录
WORKDIR /app

# 复制项目文件
COPY . .

# 设置环境变量
ENV CLAUDE_HEADLESS_MODE=true
ENV CLAUDE_SKIP_PERMISSIONS=true

# 运行分析脚本
CMD ["claude", "--headless", "--dangerously-skip-permissions", "分析项目代码并生成报告"]
```

**性能优化建议**：
- 使用 `--max-tokens` 限制输出长度以提高速度
- 设置合理的 `--timeout` 值避免长时间等待
- 在 CI 环境中缓存 Claude Code 安装
- 使用批处理减少 API 调用次数
- 合理使用 `--dangerously-skip-permissions` 在安全环境中

#### Hooks 和自定义斜杠命令

Claude Code 支持自定义 hooks 和斜杠命令，允许开发者创建可重复的工作流和自动化流程。<mcreference link="https://www.anthropic.com/engineering/claude-code-best-practices" index="2">2</mcreference>

**Hooks 管理命令**：

```bash
# 查看可用的 hooks
/hooks

# 列出已安装的 hooks
claude hooks list

# 安装 hook
claude hooks install <hook-name>

# 移除 hook
claude hooks remove <hook-name>

# 启用/禁用 hook
claude hooks enable <hook-name>
claude hooks disable <hook-name>

# 查看 hook 详情
claude hooks info <hook-name>
```

**自定义斜杠命令**：

自定义斜杠命令存储在 `.claude/commands/` 目录中，每个命令是一个包含提示模板的文件。

**创建自定义斜杠命令**：

1. **代码审查命令**：
```bash
# 创建命令目录
mkdir -p .claude/commands

# 创建代码审查命令
cat > .claude/commands/review.md << 'EOF'
---
name: "review"
description: "执行全面的代码审查"
usage: "/review [文件路径或模式]"
---

请对指定的代码进行全面审查，包括：

1. **代码质量**：
   - 代码风格和一致性
   - 命名规范
   - 代码复杂度

2. **功能性**：
   - 逻辑正确性
   - 边界条件处理
   - 错误处理

3. **性能**：
   - 算法效率
   - 内存使用
   - 潜在瓶颈

4. **安全性**：
   - 输入验证
   - 权限检查
   - 敏感数据处理

5. **可维护性**：
   - 代码结构
   - 文档完整性
   - 测试覆盖率

请提供具体的改进建议和示例代码。
EOF

# 使用自定义命令
/review src/auth.js
```

2. **测试生成命令**：
```bash
cat > .claude/commands/test.md << 'EOF'
---
name: "test"
description: "为指定代码生成单元测试"
usage: "/test <文件路径>"
---

为指定的代码文件生成完整的单元测试，要求：

1. **测试覆盖率**：
   - 覆盖所有公共方法
   - 包含正常和异常情况
   - 边界条件测试

2. **测试结构**：
   - 使用适当的测试框架
   - 清晰的测试描述
   - 合理的测试分组

3. **测试数据**：
   - 提供测试数据和 mock
   - 设置和清理代码
   - 独立的测试用例

4. **断言**：
   - 明确的期望结果
   - 适当的断言方法
   - 错误消息验证

请生成可直接运行的测试代码。
EOF
```

3. **文档生成命令**：
```bash
cat > .claude/commands/docs.md << 'EOF'
---
name: "docs"
description: "生成 API 文档"
usage: "/docs [模块或文件]"
---

为指定的代码生成详细的 API 文档，包括：

1. **概述**：
   - 模块/类的用途
   - 主要功能描述
   - 使用场景

2. **API 参考**：
   - 方法签名
   - 参数说明
   - 返回值描述
   - 异常情况

3. **使用示例**：
   - 基本用法
   - 高级用法
   - 最佳实践

4. **注意事项**：
   - 性能考虑
   - 安全提醒
   - 兼容性说明

请使用 Markdown 格式生成文档。
EOF
```

4. **重构命令**：
```bash
cat > .claude/commands/refactor.md << 'EOF'
---
name: "refactor"
description: "重构代码以提高质量"
usage: "/refactor <文件路径> [重构类型]"
---

对指定代码进行重构，重点关注：

1. **代码结构**：
   - 提取重复代码
   - 分离关注点
   - 优化类和方法设计

2. **性能优化**：
   - 算法改进
   - 减少不必要的计算
   - 优化数据结构

3. **可读性**：
   - 改善命名
   - 简化复杂逻辑
   - 添加必要注释

4. **设计模式**：
   - 应用适当的设计模式
   - 提高代码的可扩展性
   - 增强代码的可测试性

请保持功能不变，只改进代码质量。
EOF
```

**高级自定义命令示例**：

1. **项目初始化命令**：
```bash
cat > .claude/commands/init-project.md << 'EOF'
---
name: "init-project"
description: "初始化新项目结构"
usage: "/init-project <项目类型> <项目名称>"
---

根据指定的项目类型创建完整的项目结构：

支持的项目类型：
- `react`: React 应用
- `node`: Node.js 后端
- `python`: Python 项目
- `java`: Java 应用
- `go`: Go 项目

创建内容包括：
1. 目录结构
2. 配置文件
3. 依赖管理文件
4. 基础代码模板
5. README 文档
6. CI/CD 配置
7. 测试框架设置

请创建生产就绪的项目模板。
EOF
```

2. **安全审计命令**：
```bash
cat > .claude/commands/security-audit.md << 'EOF'
---
name: "security-audit"
description: "执行安全审计"
usage: "/security-audit [范围]"
---

对代码进行全面的安全审计，检查：

1. **输入验证**：
   - SQL 注入风险
   - XSS 漏洞
   - 命令注入

2. **身份认证**：
   - 密码策略
   - 会话管理
   - 权限控制

3. **数据保护**：
   - 敏感数据加密
   - 数据传输安全
   - 日志安全

4. **依赖安全**：
   - 已知漏洞检查
   - 依赖版本审计
   - 许可证合规性

提供详细的安全报告和修复建议。
EOF
```

**命令管理**：

```bash
# 列出所有可用的自定义命令
ls .claude/commands/

# 查看命令内容
cat .claude/commands/review.md

# 编辑命令
$EDITOR .claude/commands/review.md

# 删除命令
rm .claude/commands/old-command.md

# 备份命令
cp -r .claude/commands/ .claude/commands.backup/
```

**命令参数处理**：

自定义命令可以接受参数，在命令模板中使用 `{{arg1}}`、`{{arg2}}` 等占位符：

```bash
cat > .claude/commands/analyze.md << 'EOF'
---
name: "analyze"
description: "分析指定类型的问题"
usage: "/analyze <类型> <目标>"
---

请分析 {{arg1}} 类型的问题，重点关注 {{arg2}}。

分析维度：
- 问题识别
- 根因分析
- 解决方案
- 预防措施

请提供详细的分析报告。
EOF

# 使用：/analyze performance database
```

**团队共享命令**：

```bash
# 将命令添加到版本控制
git add .claude/commands/
git commit -m "添加团队共享的自定义命令"

# 团队成员同步命令
git pull origin main

# 设置命令权限
chmod +x .claude/commands/*.md
```

**高级配置技巧**：
```bash
# 为特定项目类型创建样式集合
mkdir -p .claude/output-styles/frontend
mkdir -p .claude/output-styles/backend
mkdir -p .claude/output-styles/devops

# 前端开发样式
cat > .claude/output-styles/frontend/component-docs.md << 'EOF'
---
name: "React Component Docs"
description: "React组件文档格式"
---

# [组件名称]

## 概述
[组件功能和用途描述]

## Props
| 属性名 | 类型 | 默认值 | 必需 | 描述 |
|--------|------|--------|------|------|
| [prop] | [type] | [default] | [required] | [description] |

## 使用示例
```jsx
// 基础用法
<ComponentName prop="value" />

// 高级用法
<ComponentName 
  prop1="value1"
  prop2={value2}
  onEvent={handleEvent}
/>
```

## 样式定制
[CSS类名和样式说明]

## 注意事项
[使用时的注意点和最佳实践]
EOF

# 后端开发样式
cat > .claude/output-styles/backend/service-docs.md << 'EOF'
---
name: "Service Documentation"
description: "后端服务文档格式"
---

# [服务名称]

## 服务概述
[服务功能和职责描述]

## 接口列表
[API端点列表和简要说明]

## 数据模型
[相关数据结构和关系]

## 业务逻辑
[核心业务流程说明]

## 依赖服务
[外部依赖和集成点]

## 监控和日志
[监控指标和日志格式]
EOF
```

#### Permissions 权限管理

Claude Code 提供了细粒度的权限管理系统，确保代码操作的安全性和可控性。<mcreference link="https://www.anthropic.com/engineering/claude-code-best-practices" index="2">2</mcreference>

**权限管理命令**：

```bash
# 查看当前权限设置
/permissions
claude permissions list

# 查看特定工具的权限
claude permissions show <tool-name>

# 授予权限
claude permissions grant <tool-name>
claude permissions grant <tool-name> --scope <scope>

# 撤销权限
claude permissions revoke <tool-name>

# 重置所有权限
claude permissions reset

# 导出权限配置
claude permissions export > permissions.json

# 导入权限配置
claude permissions import permissions.json
```

**跳过权限确认**：

```bash
# 跳过所有权限确认（谨慎使用）
claude --dangerously-skip-permissions "你的提示"

# 跳过特定工具的权限确认
claude --skip-permissions=file_operations "你的提示"

# 临时授予权限
claude --temp-permissions=network,file_write "你的提示"
```

**权限类型和范围**：

1. **文件系统权限**：
```bash
# 文件读取权限
claude permissions grant file_read
claude permissions grant file_read --scope "/path/to/project"

# 文件写入权限
claude permissions grant file_write
claude permissions grant file_write --scope "/path/to/safe/directory"

# 文件删除权限
claude permissions grant file_delete
claude permissions grant file_delete --scope "/tmp/*"

# 目录创建权限
claude permissions grant dir_create
```

2. **网络权限**：
```bash
# 网络访问权限
claude permissions grant network
claude permissions grant network --scope "https://api.example.com"

# HTTP 请求权限
claude permissions grant http_request
claude permissions grant http_request --scope "GET,POST"

# 下载权限
claude permissions grant download
claude permissions grant download --scope "*.zip,*.tar.gz"
```

3. **系统权限**：
```bash
# 命令执行权限
claude permissions grant command_exec
claude permissions grant command_exec --scope "npm,git,python"

# 环境变量访问权限
claude permissions grant env_access
claude permissions grant env_access --scope "PATH,NODE_ENV"

# 进程管理权限
claude permissions grant process_mgmt
```

**权限配置文件**：

项目级权限配置 (`.claude/permissions.json`)：
```json
{
  "version": "1.0",
  "permissions": {
    "file_operations": {
      "read": {
        "allowed": true,
        "scope": ["/project/src", "/project/docs"]
      },
      "write": {
        "allowed": true,
        "scope": ["/project/src", "/project/tests"],
        "exclude": ["/project/src/config/secrets.json"]
      },
      "delete": {
        "allowed": false
      }
    },
    "network": {
      "http_requests": {
        "allowed": true,
        "scope": ["https://api.github.com", "https://registry.npmjs.org"]
      }
    },
    "commands": {
      "execution": {
        "allowed": true,
        "whitelist": ["npm", "git", "python", "node"],
        "blacklist": ["rm", "sudo", "chmod"]
      }
    }
  },
  "security": {
    "require_confirmation": true,
    "log_operations": true,
    "max_file_size": "10MB",
    "timeout": 30
  }
}
```

**安全最佳实践**：

```bash
# 权限最小化原则
claude permissions grant file_read --scope "/project/src"

# 定期审查权限
claude permissions audit

# 设置权限过期时间
claude permissions grant network --expires "1h"
claude permissions grant file_write --expires "2024-12-31"

# 保护敏感文件
echo "/project/config/secrets.json" >> .claude/protected_files

# 启用操作日志
claude config set security.log_operations true
claude config set security.log_file "/var/log/claude.log"

# 启用操作确认
claude config set security.require_confirmation true
```

**权限故障排除**：

```bash
# 检查权限状态
claude permissions status

# 诊断权限问题
claude permissions diagnose

# 查看权限日志
claude permissions logs
claude permissions logs --filter "denied"

# 临时提升权限（调试用）
claude --debug-permissions "你的提示"

# 恢复默认权限
claude permissions restore-defaults

# 紧急权限重置
claude permissions emergency-reset
```

#### CLAUDE.md 文件配置

CLAUDE.md 文件是项目的核心配置文件，用于定义项目上下文、开发规范和 Claude Code 的行为模式。<mcreference link="https://www.anthropic.com/engineering/claude-code-best-practices" index="2">2</mcreference>

**CLAUDE.md 文件结构**：

```markdown
# 项目名称

## 项目概述
[项目的简要描述和目标]

## 技术栈
- **前端**: React 18, TypeScript, Tailwind CSS
- **后端**: Node.js, Express, PostgreSQL
- **工具**: Vite, ESLint, Prettier, Jest
- **部署**: Docker, AWS ECS

## 项目结构
```
src/
├── components/     # React 组件
├── pages/         # 页面组件
├── hooks/         # 自定义 hooks
├── utils/         # 工具函数
├── types/         # TypeScript 类型定义
└── api/           # API 接口
```

## 开发规范

### 代码风格
- 使用 TypeScript 进行类型安全开发
- 遵循 ESLint 和 Prettier 配置
- 组件使用函数式组件和 hooks
- 使用 CSS-in-JS 或 Tailwind CSS

### 命名规范
- 组件使用 PascalCase: `UserProfile.tsx`
- 文件和目录使用 kebab-case: `user-profile.utils.ts`
- 常量使用 UPPER_SNAKE_CASE: `API_BASE_URL`
- 变量和函数使用 camelCase: `getUserData`

### Git 工作流
- 主分支: `main`
- 功能分支: `feature/feature-name`
- 修复分支: `fix/bug-description`
- 提交信息格式: `type(scope): description`

## Claude Code 配置

### 默认行为
- 优先使用 TypeScript
- 遵循项目的 ESLint 规则
- 生成完整的类型定义
- 包含单元测试
- 添加适当的错误处理

### 输出格式
- 代码审查使用 `/output-style code-review`
- API 文档使用 `/output-style api-docs`
- 组件文档使用 `/output-style component-docs`

### 安全要求
- 不在代码中硬编码敏感信息
- 使用环境变量管理配置
- 实施输入验证和输出编码
- 遵循 OWASP 安全最佳实践

## 部署和运维

### 环境配置
- **开发环境**: `npm run dev`
- **测试环境**: `npm run test`
- **生产构建**: `npm run build`

### 监控和日志
- 使用 Winston 进行日志记录
- 集成 Sentry 进行错误监控
- 使用 Prometheus 进行性能监控

## 团队协作

### 代码审查
- 所有 PR 需要至少一人审查
- 使用 `/review` 命令进行 AI 辅助审查
- 关注代码质量、性能和安全性

### 文档维护
- API 变更需要更新文档
- 组件变更需要更新 Storybook
- 重要决策记录在 ADR 中
```

**CLAUDE.md 最佳实践**：

1. **项目上下文设置**：
```markdown
## 项目上下文

### 业务背景
[项目的业务价值和目标用户]

### 技术决策
[关键技术选择的原因和权衡]

### 架构原则
- 单一职责原则
- 开闭原则
- 依赖倒置原则
- 接口隔离原则

### 性能要求
- 页面加载时间 < 2秒
- API 响应时间 < 500ms
- 支持并发用户数 > 1000
```

2. **开发工作流定义**：
```markdown
## 开发工作流

### 功能开发流程
1. 创建功能分支
2. 编写测试用例
3. 实现功能代码
4. 运行测试和代码检查
5. 提交 PR 并请求审查
6. 合并到主分支

### 代码质量标准
- 测试覆盖率 > 80%
- 代码复杂度 < 10
- 无 ESLint 错误
- 通过安全扫描

### 发布流程
1. 创建发布分支
2. 更新版本号和变更日志
3. 执行完整测试套件
4. 部署到预发布环境
5. 执行用户验收测试
6. 部署到生产环境
```

3. **Claude Code 特定配置**：
```markdown
## Claude Code 配置

### 代码生成偏好
- 使用函数式编程风格
- 优先使用 TypeScript 严格模式
- 包含 JSDoc 注释
- 生成对应的测试文件

### 审查重点
- 类型安全性
- 错误处理完整性
- 性能优化机会
- 安全漏洞检查

### 输出样式
- 代码: 包含类型定义和注释
- 文档: 使用 Markdown 格式
- 测试: 使用 Jest 和 Testing Library
- 配置: 使用 JSON 或 YAML 格式

### 自定义命令
- `/component`: 生成 React 组件和测试
- `/api`: 生成 API 端点和文档
- `/migration`: 生成数据库迁移脚本
- `/deploy`: 生成部署配置
```

4. **团队协作规范**：
```markdown
## 团队协作

### 沟通渠道
- 技术讨论: Slack #tech-discussion
- 代码审查: GitHub PR 评论
- 架构决策: 周会和 ADR 文档

### 知识分享
- 每周技术分享会
- 代码审查最佳实践
- 新技术调研报告

### 问题解决
- Bug 报告使用 GitHub Issues
- 紧急问题使用 Slack @channel
- 技术债务在 Sprint 计划中讨论

### 文档维护
- API 文档自动生成
- 架构图定期更新
- 运维手册持续完善
```

**CLAUDE.md 模板示例**：

```bash
# 创建标准 CLAUDE.md 模板
cat > CLAUDE.md << 'EOF'
# [项目名称]

## 项目概述
[项目描述、目标和价值主张]

## 技术栈
- **语言**: [主要编程语言]
- **框架**: [使用的框架]
- **数据库**: [数据存储方案]
- **工具**: [开发和构建工具]

## 项目结构
```
[目录结构说明]
```

## 开发规范

### 代码风格
- [代码风格指南]
- [命名规范]
- [注释规范]

### Git 工作流
- [分支策略]
- [提交信息格式]
- [PR 流程]

## Claude Code 配置

### 默认行为
- [代码生成偏好]
- [输出格式要求]
- [质量标准]

### 安全要求
- [安全编码规范]
- [敏感信息处理]
- [权限控制]

## 部署和运维

### 环境配置
- [开发环境设置]
- [测试环境配置]
- [生产环境要求]

### 监控和日志
- [监控指标]
- [日志格式]
- [告警规则]

EOF
```

**CLAUDE.md 维护建议**：

```bash
# 定期更新 CLAUDE.md
# 1. 技术栈变更时更新
# 2. 开发规范调整时更新
# 3. 新团队成员加入时审查
# 4. 项目里程碑时回顾

# 版本控制
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md with new development standards"

# 团队同步
# 在团队会议中讨论 CLAUDE.md 的变更
# 确保所有成员理解新的规范和要求
```

#### @-mentions 和文件引用功能

Claude Code 支持使用 @-mentions 语法来引用项目中的文件、函数、类和外部资源，提供精确的上下文感知能力。<mcreference link="https://www.anthropic.com/engineering/claude-code-best-practices" index="2">2</mcreference>

**基础 @-mentions 语法**：

```bash
# 引用整个文件
claude "请审查 @src/components/UserProfile.tsx 的代码质量"

# 引用特定函数
claude "优化 @src/utils/api.ts:fetchUserData 函数的性能"

# 引用特定类
claude "为 @src/models/User.ts:User 类添加新的方法"

# 引用多个文件
claude "比较 @src/components/Header.tsx 和 @src/components/Footer.tsx 的实现"

# 引用目录
claude "分析 @src/components/ 目录下所有组件的一致性"
```

**高级引用语法**：

1. **行号范围引用**：
```bash
# 引用特定行号范围
claude "重构 @src/api/auth.ts:15-30 这段代码"

# 引用函数的特定部分
claude "优化 @src/utils/helpers.ts:calculateTotal:5-10 的逻辑"
```

2. **模式匹配引用**：
```bash
# 引用所有测试文件
claude "更新 @**/*.test.ts 中的测试用例"

# 引用特定类型的文件
claude "检查 @src/**/*.tsx 组件的 TypeScript 类型"

# 引用配置文件
claude "分析 @*.config.js 配置文件的设置"
```

3. **Git 相关引用**：
```bash
# 引用最近的提交
claude "分析 @git:HEAD~1 提交中的变更"

# 引用特定分支的文件
claude "比较 @git:main:src/api.ts 和当前版本的差异"

# 引用暂存区的文件
claude "审查 @git:staged 中的所有变更"
```

**MCP 资源引用**：

```bash
# 引用 GitHub 资源
claude "基于 @github:issue-123 创建相应的功能实现"
claude "将 @github:pr-456 的变更合并到当前分支"

# 引用 Jira 任务
claude "实现 @jira:PROJ-789 任务的需求"

# 引用 Notion 文档
claude "根据 @notion:design-spec 更新组件设计"

# 引用 Figma 设计
claude "基于 @figma:component-library 实现新组件"

# 引用数据库
claude "查询 @postgres:users 表中的活跃用户数据"

# 引用 Slack 消息
claude "根据 @slack:tech-discussion 的讨论实现功能"
```

**文件引用最佳实践**：

1. **精确引用**：
```bash
# 好的做法：精确指定要引用的内容
claude "优化 @src/hooks/useAuth.ts:useAuth 钩子的错误处理"

# 避免：过于宽泛的引用
claude "优化 @src/ 目录下的所有代码"
```

2. **上下文相关引用**：
```bash
# 提供相关上下文
claude "基于 @src/types/User.ts 的类型定义，更新 @src/api/users.ts 的接口"

# 引用相关测试
claude "为 @src/utils/validation.ts 创建对应的 @src/utils/validation.test.ts"
```

3. **批量操作引用**：
```bash
# 批量文件处理
claude "为 @src/components/**/*.tsx 所有组件添加 PropTypes 验证"

# 批量重构
claude "将 @src/pages/**/*.js 文件迁移到 TypeScript"
```

**引用语法扩展**：

1. **函数和类成员引用**：
```bash
# 引用类的特定方法
claude "重构 @src/services/ApiService.ts:ApiService.get 方法"

# 引用接口的特定属性
claude "扩展 @src/types/Config.ts:AppConfig.database 配置"

# 引用枚举值
claude "使用 @src/constants/Status.ts:UserStatus.ACTIVE 状态"
```

2. **依赖关系引用**：
```bash
# 引用包依赖
claude "升级 @package.json:react 到最新版本"

# 引用环境变量
claude "配置 @.env:DATABASE_URL 环境变量"

# 引用配置文件
claude "更新 @tsconfig.json:compilerOptions 设置"
```

3. **文档和注释引用**：
```bash
# 引用 README 文档
claude "根据 @README.md 的说明更新安装步骤"

# 引用 API 文档
claude "基于 @docs/api.md 实现客户端接口"

# 引用代码注释
claude "实现 @src/api.ts:// TODO: 添加缓存机制 的功能"
```

**引用解析和智能提示**：

```bash
# 查看可用的引用
claude "/references"

# 搜索可引用的内容
claude "/find UserProfile"

# 查看文件的可引用元素
claude "/symbols @src/components/UserProfile.tsx"

# 查看引用的依赖关系
claude "/deps @src/api/users.ts"
```

**引用配置和自定义**：

在 `.claude/config.json` 中配置引用行为：

```json
{
  "references": {
    "auto_complete": true,
    "case_sensitive": false,
    "include_patterns": [
      "src/**/*.{ts,tsx,js,jsx}",
      "docs/**/*.md",
      "*.config.{js,ts,json}"
    ],
    "exclude_patterns": [
      "node_modules/**",
      "dist/**",
      "*.log"
    ],
    "max_file_size": "1MB",
    "symbol_extraction": {
      "functions": true,
      "classes": true,
      "interfaces": true,
      "types": true,
      "constants": true
    }
  },
  "mcp_references": {
    "github": {
      "enabled": true,
      "auto_link_issues": true,
      "auto_link_prs": true
    },
    "jira": {
      "enabled": true,
      "project_key": "PROJ"
    },
    "notion": {
      "enabled": true,
      "workspace_id": "your-workspace-id"
    }
  }
}
```

**引用性能优化**：

```bash
# 缓存引用索引
claude config set references.cache_enabled true
claude config set references.cache_ttl 3600

# 限制引用范围
claude config set references.max_depth 3
claude config set references.max_files 1000

# 异步加载引用
claude config set references.async_loading true
```

**引用安全和权限**：

```bash
# 设置引用权限
claude permissions grant file_reference --scope "/project/src"
claude permissions grant mcp_reference --scope "github,jira"

# 保护敏感文件
echo "config/secrets.json" >> .claude/no_reference_files
echo ".env*" >> .claude/no_reference_patterns

# 审计引用使用
claude references audit
claude references logs --filter "external"
```

**实际使用示例**：

```bash
# 代码审查场景
claude "请审查 @src/components/UserProfile.tsx 组件，特别关注 @src/components/UserProfile.tsx:handleSubmit 方法的错误处理"

# 功能实现场景
claude "基于 @github:issue-123 的需求，在 @src/api/users.ts 中添加新的用户管理接口"

# 重构场景
claude "将 @src/utils/helpers.ts:formatDate 函数重构为更通用的日期处理工具，并更新所有引用该函数的文件"

# 测试场景
claude "为 @src/services/AuthService.ts:login 方法创建全面的单元测试，覆盖 @src/types/Auth.ts:LoginRequest 的所有场景"

# 文档场景
claude "根据 @src/api/users.ts 的实现，更新 @docs/api/users.md 的 API 文档"
```

#### Plan Mode 和高级功能

Claude Code 提供了多种高级功能来提升开发效率和代码质量，包括计划模式、协作功能、AI 助手集成等。

**Plan Mode（计划模式）**：

计划模式允许 Claude Code 在执行代码变更前先制定详细的执行计划，提供更好的可预测性和控制性。

```bash
# 启用计划模式
claude --plan "重构用户认证系统"
claude plan "添加新的支付功能模块"

# 查看当前计划
claude plan show
claude plan status

# 执行计划的特定步骤
claude plan execute --step 1
claude plan execute --steps 1-3

# 修改计划
claude plan edit --step 2
claude plan add "添加单元测试"
claude plan remove --step 4

# 保存和加载计划
claude plan save --name "auth-refactor"
claude plan load --name "auth-refactor"
claude plan list
```

**计划模式配置**：

```bash
# 设置计划模式偏好
claude config set plan.auto_generate true
claude config set plan.detail_level "detailed"
claude config set plan.require_approval true
claude config set plan.backup_before_execute true

# 计划模板
claude plan template create --name "feature-development"
claude plan template use "feature-development"
```

**协作功能**：

1. **团队会话共享**：
```bash
# 创建共享会话
claude session create --shared --name "team-review"
claude session invite --email "teammate@company.com"

# 加入共享会话
claude session join --id "session-123"

# 会话权限管理
claude session permissions --user "teammate@company.com" --role "editor"
claude session permissions --user "manager@company.com" --role "viewer"

# 会话同步
claude session sync
claude session export --format "markdown"
```

2. **代码审查协作**：
```bash
# 创建代码审查请求
claude review create --files "src/components/*.tsx" --reviewers "team@company.com"

# 响应审查请求
claude review respond --id "review-123" --status "approved"
claude review comment --id "review-123" --line 45 --message "建议优化性能"

# 审查状态管理
claude review list --status "pending"
claude review merge --id "review-123"
```

3. **知识库集成**：
```bash
# 连接团队知识库
claude kb connect --type "confluence" --url "https://company.atlassian.net"
claude kb connect --type "notion" --workspace "team-workspace"

# 搜索知识库
claude kb search "API 设计规范"
claude kb get --id "page-123"

# 更新知识库
claude kb update --id "page-123" --content "@docs/api-guidelines.md"
```

**AI 助手集成**：

1. **多模型支持**：
```bash
# 切换 AI 模型
claude model use "claude-3-opus"
claude model use "claude-3-sonnet"
claude model use "claude-3-haiku"

# 模型比较
claude compare --models "opus,sonnet" --prompt "优化这个算法"

# 模型配置
claude model config --temperature 0.7 --max_tokens 4000
```

2. **专业化助手**：
```bash
# 启用专业助手
claude assistant use "security-expert"
claude assistant use "performance-optimizer"
claude assistant use "code-reviewer"

# 自定义助手
claude assistant create --name "react-specialist" --prompt "@.claude/assistants/react.md"

# 助手管理
claude assistant list
claude assistant info "security-expert"
```

3. **上下文增强**：
```bash
# 项目上下文分析
claude context analyze
claude context summary

# 智能建议
claude suggest --type "refactoring"
claude suggest --type "testing"
claude suggest --type "documentation"

# 代码质量检查
claude quality check
claude quality report --format "html"
```

**高级工作流**：

1. **自动化流水线**：
```bash
# 创建工作流
claude workflow create --name "ci-cd-pipeline"
claude workflow add-step --name "test" --command "npm test"
claude workflow add-step --name "build" --command "npm run build"
claude workflow add-step --name "deploy" --command "npm run deploy"

# 执行工作流
claude workflow run "ci-cd-pipeline"
claude workflow status

# 工作流模板
claude workflow template list
claude workflow template use "react-app-deployment"
```

2. **智能重构**：
```bash
# 大规模重构
claude refactor --type "extract-component" --target "@src/pages/UserProfile.tsx:UserForm"
claude refactor --type "move-to-hook" --target "@src/components/UserList.tsx:fetchUsers"

# 架构迁移
claude migrate --from "class-components" --to "functional-components" --path "src/components/"
claude migrate --from "javascript" --to "typescript" --path "src/"

# 依赖升级
claude upgrade --package "react" --version "18.0.0" --update-usage
```

3. **性能分析**：
```bash
# 性能分析
claude perf analyze --target "@src/components/Dashboard.tsx"
claude perf benchmark --iterations 100

# 性能优化建议
claude perf suggest
claude perf optimize --auto-apply

# 性能监控
claude perf monitor --duration 60s
claude perf report --format "json"
```

**实验性功能**：

1. **AI 代码生成**：
```bash
# 启用实验性功能
claude experimental enable "ai-codegen"
claude experimental enable "smart-completion"

# AI 代码生成
claude generate component --name "UserCard" --props "user,onEdit,onDelete"
claude generate api --resource "users" --operations "CRUD"
claude generate test --target "@src/utils/validation.ts"

# 智能补全
claude complete --context "React functional component"
```

2. **代码理解**：
```bash
# 代码解释
claude explain --target "@src/algorithms/sorting.ts:quickSort"
claude explain --complexity

# 依赖分析
claude deps analyze --target "@src/components/App.tsx"
claude deps graph --output "deps.svg"

# 影响分析
claude impact --change "@src/types/User.ts:User.email"
```

3. **智能调试**：
```bash
# 智能调试
claude debug --error "TypeError: Cannot read property 'name' of undefined"
claude debug --performance --slow-queries

# 日志分析
claude logs analyze --file "app.log" --pattern "error"
claude logs suggest-fixes
```

**高级配置示例**：

在 `.claude/config.json` 中配置高级功能：

```json
{
  "plan_mode": {
    "enabled": true,
    "auto_generate": true,
    "detail_level": "detailed",
    "require_approval": true,
    "backup_before_execute": true,
    "templates_dir": ".claude/plan-templates"
  },
  "collaboration": {
    "team_workspace": "company-dev-team",
    "default_reviewers": ["senior-dev@company.com"],
    "auto_invite_on_pr": true,
    "knowledge_base": {
      "confluence": {
        "url": "https://company.atlassian.net",
        "space": "DEV"
      }
    }
  },
  "ai_assistant": {
    "default_model": "claude-3-sonnet",
    "specialized_assistants": {
      "security": "security-expert",
      "performance": "performance-optimizer",
      "review": "code-reviewer"
    },
    "context_enhancement": true,
    "smart_suggestions": true
  },
  "experimental": {
    "ai_codegen": true,
    "smart_completion": true,
    "intelligent_debugging": true,
    "performance_analysis": true
  }
}
```

**使用最佳实践**：

1. **计划驱动开发**：
```bash
# 始终从计划开始
claude plan "实现用户认证功能" --template "feature-development"
claude plan review
claude plan execute --step-by-step
```

2. **协作最佳实践**：
```bash
# 创建共享上下文
claude session create --shared --name "feature-auth"
claude context share --include "requirements,design,progress"
```

3. **质量保证**：
```bash
# 集成质量检查
claude workflow add-step --name "quality-gate" --command "claude quality check"
claude review create --auto-assign --quality-checks
```

---

## SPEC方法论驱动的开发流程

### 概述

SPEC（Specification-driven Programming）方法论是一种以规范为驱动的系统化开发方法，结合Claude Code的AI能力，可以构建高效、可靠的软件开发流程。该方法论强调在开发过程中明确定义需求规范、架构设计、实现标准和验证准则。

### 核心理念

基于搜索结果的软件工程最佳实践，SPEC方法论包含四个核心阶段：
- **S**pecification（规范定义）：明确需求和功能规范
- **P**lanning（计划设计）：架构设计和技术选型
- **E**xecution（执行实现）：代码实现和持续集成
- **C**ertification（认证验证）：测试验证和质量保证

### 1. Specification（规范定义阶段）

#### 1.1 需求分析与捕获

**使用Claude Code进行需求分析**：
```bash
# 项目初始化和需求理解
cd your-project
claude '/init'
claude "分析项目需求文档，提取核心功能点和非功能性需求"

# 需求分类和优先级排序
claude "将需求分为功能需求、质量需求和约束需求三类，并设定优先级"

# 创建用户故事
claude "基于需求分析结果，创建详细的用户故事和验收标准"
```

**需求文档模板**：
```bash
# 创建需求规范文档
claude "创建requirements.md文件，包含以下结构：
1. 项目概述和目标
2. 功能需求列表
3. 非功能性需求（性能、安全、可用性）
4. 技术约束和限制
5. 验收标准
6. 风险评估"
```

#### 1.2 领域建模

```bash
# 业务领域分析
claude "分析业务领域，创建领域模型图和核心实体关系"

# 数据模型设计
claude "基于领域模型设计数据库schema，包含实体、属性和关系"

# API接口规范
claude "设计RESTful API规范，定义端点、请求/响应格式和错误处理"
```

### 2. Planning（计划设计阶段）

#### 2.1 架构设计

**系统架构设计**：
```bash
# 整体架构分析
claude "基于需求设计系统整体架构，包括：
- 分层架构（表现层、业务层、数据层）
- 组件划分和职责定义
- 技术栈选择和依赖关系
- 部署架构和环境规划"

# 创建架构文档
claude "创建architecture.md文档，包含架构图、组件说明和技术决策"
```

**技术选型和评估**：
```bash
# 技术栈评估
claude "评估以下技术选项的优缺点：
- 前端框架：React vs Vue vs Angular
- 后端框架：Express vs Fastify vs Koa
- 数据库：PostgreSQL vs MongoDB vs MySQL
- 部署方案：Docker vs Serverless vs Traditional"

# 创建技术决策记录
claude "创建ADR（Architecture Decision Records）文档记录技术选型理由"
```

#### 2.2 开发计划制定

**敏捷开发规划**：
```bash
# 迭代规划
claude "基于用户故事创建Sprint计划：
1. 将用户故事分解为具体任务
2. 估算开发工作量
3. 制定2-4周的Sprint计划
4. 定义每个Sprint的交付目标"

# 创建开发路线图
claude "创建development-roadmap.md，包含：
- 里程碑和关键节点
- 依赖关系和风险点
- 资源分配和时间安排"
```

### 3. Execution（执行实现阶段）

#### 3.1 测试驱动开发（TDD）

**TDD工作流**：
```bash
# 编写测试用例
claude "基于用户故事编写单元测试，遵循TDD红-绿-重构循环：
1. 编写失败的测试（红）
2. 编写最小可行代码（绿）
3. 重构优化代码（重构）"

# 集成测试
claude "创建集成测试套件，验证组件间交互和API端点"

# 端到端测试
claude "设计E2E测试场景，模拟用户完整操作流程"
```

#### 3.2 持续集成/持续部署（CI/CD）

**CI/CD管道设置**：
```bash
# 创建CI配置
claude "创建.github/workflows/ci.yml文件，包含：
1. 代码检查和格式化
2. 单元测试和覆盖率报告
3. 集成测试执行
4. 安全扫描和依赖检查
5. 构建和打包"

# 部署自动化
claude "设计自动化部署流程：
- 开发环境自动部署
- 测试环境集成部署
- 生产环境蓝绿部署"
```

#### 3.3 代码质量保证

**代码审查和质量控制**：
```bash
# 代码审查
claude "review当前代码，检查：
1. 代码规范和风格一致性
2. 设计模式和架构原则遵循
3. 性能和安全最佳实践
4. 测试覆盖率和质量"

# 重构优化
claude "识别代码异味并进行重构：
- 消除重复代码
- 简化复杂方法
- 优化性能瓶颈
- 改善可读性和维护性"
```

### 4. Certification（认证验证阶段）

#### 4.1 质量验证

**多层次测试验证**：
```bash
# 功能测试
claude "执行功能测试，验证所有用户故事的验收标准"

# 性能测试
claude "进行性能测试和压力测试：
1. 负载测试验证系统容量
2. 压力测试找出性能极限
3. 稳定性测试验证长期运行"

# 安全测试
claude "执行安全测试：
- SQL注入和XSS漏洞扫描
- 身份认证和授权验证
- 数据加密和传输安全
- 依赖包安全漏洞检查"
```

#### 4.2 用户验收测试（UAT）

```bash
# UAT计划
claude "制定用户验收测试计划：
1. 定义测试场景和用例
2. 准备测试数据和环境
3. 组织用户测试会话
4. 收集反馈和问题报告"

# 缺陷管理
claude "建立缺陷跟踪和修复流程：
- 缺陷分类和优先级定义
- 修复验证和回归测试
- 发布准备和上线检查"
```

### 5. SPEC方法论实施指南

#### 5.1 团队协作最佳实践

**协作工具和流程**：
```bash
# 文档协作
claude "建立团队文档协作规范：
1. 使用Markdown格式统一文档
2. Git版本控制管理文档变更
3. 定期文档审查和更新
4. 知识分享和培训计划"

# 代码协作
claude "制定代码协作规范：
- Git分支策略（GitFlow或GitHub Flow）
- Pull Request审查流程
- 代码合并和发布流程
- 冲突解决和回滚策略"
```

#### 5.2 质量度量和改进

**关键指标监控**：
```bash
# 开发效率指标
claude "建立开发效率监控：
- 代码提交频率和质量
- 功能交付速度和准确性
- 缺陷发现和修复时间
- 测试覆盖率和通过率"

# 持续改进
claude "实施持续改进机制：
1. 定期回顾会议（Sprint Retrospective）
2. 流程优化和工具改进
3. 团队技能提升计划
4. 最佳实践分享和推广"
```

#### 5.3 风险管理和应急预案

```bash
# 风险识别和评估
claude "识别项目风险并制定应对策略：
- 技术风险：技术选型、性能、安全
- 进度风险：资源、依赖、变更
- 质量风险：测试、集成、部署
- 业务风险：需求变更、市场变化"

# 应急预案
claude "制定应急响应预案：
1. 生产环境故障处理流程
2. 数据备份和恢复策略
3. 安全事件响应计划
4. 业务连续性保障措施"
```

### 6. SPEC方法论工具链

#### 6.1 推荐工具集成

**开发工具链**：
- **需求管理**：Jira、Azure DevOps、Linear
- **设计工具**：Figma、Miro、Draw.io
- **代码管理**：Git、GitHub/GitLab、Bitbucket
- **CI/CD**：GitHub Actions、GitLab CI、Jenkins
- **测试工具**：Jest、Cypress、Playwright、Postman
- **监控工具**：Sentry、DataDog、New Relic

**Claude Code集成脚本**：
```bash
# 自动化工作流脚本
#!/bin/bash
# spec-workflow.sh

echo "SPEC方法论自动化工作流"

# 阶段1：规范定义
echo "1. 执行需求分析..."
claude "分析项目需求并创建规范文档"

# 阶段2：计划设计
echo "2. 执行架构设计..."
claude "基于需求设计系统架构"

# 阶段3：执行实现
echo "3. 执行代码实现..."
claude "基于TDD方法实现功能"
npm test

# 阶段4：认证验证
echo "4. 执行质量验证..."
claude "执行全面测试和质量检查"
npm run test:e2e

echo "SPEC工作流完成！"
```

---

## 常见问题解决方案

### 1. 安装问题

#### 问题：npm 安装失败
**症状**：
```
npm ERR! code EACCES
npm ERR! syscall access
```

**解决方案**：
```bash
# 方法一：使用 sudo（不推荐）
sudo npm install -g @anthropic-ai/claude-code

# 方法二：配置 npm 全局目录
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# 方法三：使用 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install node
nvm use node
```

#### 问题：版本冲突
**症状**：
```
Error: Unsupported Node.js version
```

**解决方案**：
```bash
# 检查 Node.js 版本
node --version

# 升级 Node.js
nvm install --lts
nvm use --lts

# 或使用包管理器
brew upgrade node  # macOS
```

### 2. 配置问题

#### 问题：命令未找到
**症状**：
```
claude: command not found
```

**解决方案**：
```bash
# 检查安装路径
which claude
npm list -g @anthropic-ai/claude-code

# 添加到 PATH
echo 'export PATH="$PATH:$(npm config get prefix)/bin"' >> ~/.bashrc
source ~/.bashrc

# 重新安装
npm uninstall -g @anthropic-ai/claude-code
npm install -g @anthropic-ai/claude-code
```

#### 问题：权限错误和授权中断
**症状**：
```
Permission denied
或频繁的权限确认弹窗
```

**解决方案**：
```bash
# 检查文件权限
ls -la ~/.claude

# 修复权限
chmod 755 ~/.claude
chown -R $USER ~/.claude

# 使用跳过权限模式（高级用法）
claude --dangerously-skip-permissions

# 设置别名避免重复输入
alias claude-skip='claude --dangerously-skip-permissions'
```

### 3. 性能问题

#### 问题：命令执行缓慢或网络连接问题
**解决方案**：
```bash
# 清理缓存
npm cache clean --force

# 检查网络连接
ping api.anthropic.com

# 重启 Claude Code
# 按 Ctrl+C 退出当前会话，然后重新启动
claude

# 检查系统资源
top
df -h

# 如果使用代理，确保代理设置正确
# 建议使用新加坡或美国节点
```

### 4. Git 相关问题

#### 问题：合并冲突
**解决方案**：
```bash
# 查看冲突文件
git status

# 手动解决冲突后
git add <resolved-files>
git commit -m "Resolve merge conflicts"

# 或使用工具
git mergetool
```

#### 问题：误提交敏感信息
**解决方案**：
```bash
# 从历史中移除文件
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch <sensitive-file>' \
  --prune-empty --tag-name-filter cat -- --all

# 强制推送
git push origin --force --all
```

---

## 最佳实践案例

### 1. 项目初始化最佳实践

#### 标准项目结构
```
project-name/
├── .claude/              # Claude Code 配置目录
├── CLAUDE.md            # Claude 项目说明文档
├── .gitignore           # Git 忽略文件
├── package.json         # 项目依赖
├── README.md           # 项目说明
├── src/                # 源代码
│   ├── components/     # 组件
│   ├── services/       # 服务
│   ├── utils/         # 工具函数
│   └── tests/         # 测试文件
└── docs/              # 文档
```

#### 初始化脚本
```bash
#!/bin/bash
# init-project.sh

PROJECT_NAME=$1

if [ -z "$PROJECT_NAME" ]; then
  echo "Usage: $0 <project-name>"
  exit 1
fi

# 创建项目目录
mkdir $PROJECT_NAME
cd $PROJECT_NAME

# 初始化 Git
git init

# 启动 Claude Code 并初始化项目
claude '/init'

# 添加初始文件到 Git
git add .
git commit -m "Initial commit with Claude Code setup"

# 设置远程仓库（可选）
# git remote add origin <repository-url>
# git push -u origin main

echo "Project $PROJECT_NAME initialized successfully with Claude Code!"
```

### 2. 代码质量保证

#### 预提交钩子设置
```bash
# 安装 husky
npm install --save-dev husky
npx husky install

# 添加预提交钩子
npx husky add .husky/pre-commit "npm run lint && npm run test"
```

#### 持续集成配置 (.github/workflows/ci.yml)
```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Install Claude Code
      run: npm install -g @anthropic-ai/claude-code
    
    - name: Run tests with Claude Code
      run: |
        export ANTHROPIC_AUTH_TOKEN=${{ secrets.ANTHROPIC_API_KEY }}
        claude -p "运行项目测试并生成覆盖率报告"
      env:
        ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    
    - name: Code analysis with Claude
      run: |
        export ANTHROPIC_AUTH_TOKEN=${{ secrets.ANTHROPIC_API_KEY }}
        claude -p "分析代码质量并生成报告"
      env:
        ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### 3. 团队协作最佳实践

#### 分支策略
```bash
# 功能开发流程
git checkout -b feature/new-feature

# 使用 Claude Code 开发功能
claude "实现新功能：用户注册验证"

# 运行测试和分析
claude "运行测试并修复任何失败的测试"
claude "分析代码质量并优化"

# 提交代码
git add .
git commit -m "feat: add user registration validation"
git push origin feature/new-feature

# 使用 Claude Code 创建 Pull Request
claude "创建 Pull Request 到 main 分支，标题为 'Feature: Add User Registration Validation'"
```

#### 提交信息规范
```
type(scope): description

[optional body]

[optional footer]
```

**类型说明**：
- `feat`：新功能
- `fix`：修复 bug
- `docs`：文档更新
- `style`：代码格式化
- `refactor`：重构
- `test`：测试相关
- `chore`：构建过程或辅助工具的变动

### 4. 性能优化实践

#### 代码分析和优化
```bash
# 使用 Claude Code 进行性能分析
claude "分析项目性能瓶颈并生成优化建议报告"

# 自动化代码重构
claude "重构 src/ 目录下的代码，优化性能和可读性"

# 深度思考模式进行复杂优化
claude "think harder 关于如何优化这个应用的整体架构"
```

#### 构建优化
```bash
# 使用 Claude Code 优化构建流程
claude "优化项目构建配置，减少包大小并提高性能"

# 分析构建产物
claude "分析构建输出，识别可以优化的地方"
```

### 5. 文档维护

#### 自动生成文档
```bash
# 使用 Claude Code 生成 API 文档
claude "为项目生成完整的 API 文档"

# 生成组件文档
claude "为所有 React 组件生成详细的使用文档"

# 更新 README
claude "更新 README.md，包含最新的项目信息和使用说明"
```

#### 文档同步脚本
```bash
#!/bin/bash
# sync-docs.sh

echo "Syncing documentation with Claude Code..."

# 生成项目文档
claude -p "生成项目的完整文档，包括 API、组件和使用指南"

# 提交文档更新
git add docs/ README.md CLAUDE.md
git commit -m "docs: update documentation with Claude Code"
git push origin main

echo "Documentation updated and synced with Claude Code!"
```

### 6. Output Styles 最佳实践

#### 使用场景和建议

**1. 项目文档标准化**
```bash
# 为团队创建统一的文档样式
mkdir -p .claude/output-styles/team-standards

# API文档标准
/output-style api-docs
claude "为用户认证模块生成API文档"

# 代码审查标准
/output-style code-review
claude "审查用户注册功能的代码质量"
```

**2. 不同开发阶段的样式切换**
```bash
# 需求分析阶段
/output-style prd-template
claude "分析用户管理系统的需求"

# 开发阶段
/output-style component-docs
claude "为登录组件生成文档"

# 测试阶段
/output-style test-report
claude "生成单元测试报告"
```

**3. 团队协作优化**
```bash
# 为不同角色创建专用样式
# 产品经理
/output-style prd-template

# 前端开发
/output-style component-docs

# 后端开发
/output-style api-docs

# QA测试
/output-style test-report
```

#### 配置管理最佳实践

**1. 版本控制集成**
```bash
# 将样式文件纳入版本控制
git add .claude/output-styles/
git commit -m "feat: add custom output styles for team"

# 为不同项目分支创建不同样式
git checkout feature/api-redesign
# 修改或添加特定的API文档样式
```

**2. 样式文件组织结构**
```
.claude/output-styles/
├── team-standards/          # 团队标准样式
│   ├── code-review.md
│   ├── api-docs.md
│   └── meeting-notes.md
├── project-specific/        # 项目特定样式
│   ├── frontend/
│   │   ├── component-docs.md
│   │   └── ui-specs.md
│   └── backend/
│       ├── service-docs.md
│       └── database-schema.md
└── personal/               # 个人偏好样式
    ├── learning-notes.md
    └── quick-reference.md
```

**3. 样式模板管理脚本**
```bash
#!/bin/bash
# manage-styles.sh

function create_style() {
    local style_name=$1
    local style_path=".claude/output-styles/${style_name}.md"
    
    if [ -f "$style_path" ]; then
        echo "样式 $style_name 已存在"
        return 1
    fi
    
    cat > "$style_path" << EOF
---
name: "$style_name"
description: "自定义样式描述"
---

# $style_name 模板

## 概述
[内容概述]

## 详细内容
[具体内容模板]
EOF
    
    echo "样式 $style_name 创建成功"
}

function list_styles() {
    echo "可用的输出样式："
    find .claude/output-styles -name "*.md" -exec basename {} .md \;
}

function backup_styles() {
    tar -czf "output-styles-backup-$(date +%Y%m%d).tar.gz" .claude/output-styles/
    echo "样式文件已备份"
}

# 使用示例
# ./manage-styles.sh create "bug-report"
# ./manage-styles.sh list
# ./manage-styles.sh backup
```

#### 注意事项和限制

**1. 性能考虑**
```bash
# 避免过于复杂的样式模板
# 样式文件大小建议控制在 2KB 以内

# 检查样式文件大小
find .claude/output-styles -name "*.md" -exec ls -lh {} \;
```

**2. 兼容性注意事项**
- Output Styles 功能需要 Claude Code 最新版本支持
- 样式文件使用 YAML front matter 格式，需要严格遵循语法
- 模板中的占位符应该清晰明确，避免歧义

**3. 安全和隐私**
```bash
# 避免在样式模板中包含敏感信息
# 检查样式文件中的敏感内容
grep -r "password\|token\|secret" .claude/output-styles/

# 为敏感项目创建独立的样式配置
mkdir -p .claude/output-styles/private
echo ".claude/output-styles/private/" >> .gitignore
```

**4. 团队协作建议**
- 建立团队样式规范和命名约定
- 定期审查和更新样式模板
- 为新团队成员提供样式使用培训
- 建立样式模板的反馈和改进机制

**5. 故障排除**
```bash
# 检查样式文件语法
# 确保 YAML front matter 格式正确
head -10 .claude/output-styles/your-style.md

# 重置到默认样式
/output-style default

# 清理损坏的样式文件
find .claude/output-styles -name "*.md" -size 0 -delete
```

---

### 7. Claude Code 实际应用场景

#### 日常开发工作流
```bash
# 1. 项目启动
cd my-project
claude '/init'  # 让 Claude 理解项目结构

# 2. 功能开发
claude "实现用户注册功能，包括表单验证和数据库存储"

# 3. 代码审查和优化
claude "review 这个函数的代码质量并提出改进建议"

# 4. 测试和调试
claude "运行测试并修复任何失败的用例"

# 5. 文档更新
claude "更新 README 文件，添加新功能的使用说明"
```

#### 代码理解和学习
```bash
# 理解不熟悉的代码库
claude "解释这个支付处理系统的工作原理"
claude "查找用户权限检查的代码位置"
claude "分析缓存层的实现逻辑"

# 学习新技术
claude "解释这个 React Hook 的使用方法"
claude "这个 SQL 查询是如何优化的？"
```

#### Git 操作自动化
```bash
# 智能提交
claude "分析当前的代码更改并生成合适的提交信息"

# 解决合并冲突
claude "帮我解决这个合并冲突"

# 创建 Pull Request
claude "创建 PR 到 main 分支，总结这次的功能改进"

# 查看历史
claude "查找 12 月份添加 markdown 测试的提交"
```

#### 问题诊断和修复
```bash
# 错误诊断
claude "分析这个错误日志并找出根本原因"

# 性能问题
claude "这个页面加载很慢，帮我找出性能瓶颈"

# 安全审查
claude "检查代码中的安全漏洞"
```

#### 长期项目管理
```bash
# 创建项目计划文件
echo "# 项目重构计划" > refactor_plan.md
claude "分析当前项目架构，制定重构计划并更新 refactor_plan.md"

# 跨会话任务跟踪
claude "我们完成了重构的第一阶段，请列出下一阶段的具体任务"

# 定期代码审查
claude "对过去一周的代码提交进行质量分析"
```

---

## 总结

本手册基于 Claude Code 的官方文档、实际使用经验和现代软件工程最佳实践，提供了完整的使用指南和系统化开发流程。特别是通过引入SPEC方法论，结合Claude Code的AI能力，构建了从需求分析到质量验证的完整开发周期。

### 核心价值

通过遵循本手册的指南和SPEC方法论，开发者和团队可以：

1. **快速上手**：通过准确的安装指南和命令说明快速掌握Claude Code
2. **系统化开发**：运用SPEC方法论实现规范驱动的开发流程 <mcreference link="https://blog.csdn.net/weixin_44799217/article/details/120550553" index="5">5</mcreference>
3. **提高效率**：利用深度思考模式、权限管理和自动化脚本提升开发效率
4. **保证质量**：通过TDD、CI/CD和多层次测试验证确保代码质量 <mcreference link="https://blog.csdn.net/weixin_44799217/article/details/120550553" index="6">6</mcreference>
5. **敏捷协作**：遵循敏捷开发原则，通过短期迭代缩短DevOps生命周期 <mcreference link="https://blog.csdn.net/weixin_44799217/article/details/120550553" index="7">7</mcreference>
6. **持续改进**：建立持续集成/持续部署流程，实现快速、频繁和可靠的软件交付 <mcreference link="https://blog.csdn.net/weixin_44799217/article/details/120550553" index="8">8</mcreference>

### SPEC方法论的核心优势

基于搜索结果的软件工程最佳实践，SPEC方法论提供了：

- **需求驱动**：通过明确的需求分析和规范定义，确保开发目标清晰 <mcreference link="https://blog.csdn.net/weixin_44799217/article/details/120550553" index="5">5</mcreference>
- **架构先行**：系统化的架构设计和技术选型，降低技术风险
- **测试驱动**：TDD和BDD方法确保代码质量和功能正确性 <mcreference link="https://blog.csdn.net/weixin_44799217/article/details/120550553" index="8">8</mcreference>
- **持续验证**：多层次的质量验证和用户验收测试，保障交付质量

### 重要提醒

- **费用控制**：注意使用 `ultrathink` 等高级功能的成本 <mcreference link="https://www.cnblogs.com/javastack/p/18978280" index="2">2</mcreference>
- **安全考虑**：避免在敏感项目中使用第三方 API 服务 <mcreference link="https://www.cnblogs.com/xiaoqi/p/18980593/claude-code-free" index="4">4</mcreference>
- **版本管理**：建议使用稳定版本，避免频繁更新带来的兼容性问题
- **网络要求**：确保稳定的网络连接，特别是在使用代理时选择合适的节点 <mcreference link="https://blog.axiaoxin.com/post/claude-code-full-guide/" index="1">1</mcreference>

### 实施建议

建议开发者和团队根据以下原则实施SPEC方法论和Claude Code最佳实践：

1. **渐进式采用**：从小型项目开始实践SPEC方法论，逐步扩展到大型项目
2. **工具集成**：将Claude Code集成到现有的开发工具链中，形成统一的工作流
3. **团队培训**：定期组织SPEC方法论和Claude Code使用培训，提升团队整体能力
4. **度量改进**：建立关键指标监控，持续优化开发流程和质量标准
5. **文档维护**：保持文档的及时更新，确保实践指南与实际操作保持一致

Claude Code 作为一个强大的 AI 编程助手，结合SPEC方法论的系统化开发流程，能够显著提升开发效率和软件质量。通过合理使用和持续优化，可以为团队和项目带来最大价值。

---

*版本：2.1（新增Output Styles功能详解）*
*基于网络搜索结果优化，融合现代软件工程最佳实践，新增Output Styles功能的详细介绍和最佳实践指导*