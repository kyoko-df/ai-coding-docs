# Superpowers Harness 编程实现分析

## 结论

Superpowers 本身不是一个从零实现的 agent runtime，也不是单独的执行引擎。它更像是一个叠加在 Claude Code、Cursor、Codex、OpenCode 这些宿主环境之上的工作流层。

它通过以下四类机制，把“写代码”变成一套接近 harness 编程的系统：

- 会话启动时注入 bootstrap 上下文
- 用 skills 作为控制平面
- 用子代理工作流实现编排
- 用 transcript 驱动的测试脚本验证行为

换句话说，Superpowers 的 harness 编程不是集中写在某个 runtime 里，而是分散在：

- hooks
- skills
- prompt templates
- headless 测试脚本

这些部件共同组成一个可执行、可验证的 agent workflow harness。

## 一、整体架构

可以把 Superpowers 的实现拆成四层：

### 1. 宿主 Harness 层

底层运行能力由外部平台提供，例如：

- Claude Code
- Cursor
- Codex
- OpenCode

这些平台负责提供：

- 会话生命周期
- 工具调用能力
- 子代理能力
- hook 机制
- transcript 持久化

Superpowers 自己不重复实现这些能力，而是复用宿主平台的基础设施。

在 README 中，它把自己定义为“建立在 skills 和初始指令之上的完整软件开发工作流”：  
[README.md:L3-L15](superpowers/README.md#L3-L15)

### 2. Bootstrap 注入层

Superpowers 在会话启动时，把 `using-superpowers` 这个元技能注入到系统上下文里，从一开始改变 agent 的默认行为。

### 3. Workflow 编排层

核心开发流程由一组 skills 串联起来：

- brainstorming
- writing-plans
- subagent-driven-development
- test-driven-development
- requesting-code-review
- finishing-a-development-branch

对应的主流程定义可以看这里：  
[README.md:L101-L117](superpowers/README.md#L101-L117)

### 4. Test / Eval Harness 层

Superpowers 通过 headless Claude 会话和 transcript 解析脚本，验证 skills 是否真的按预期执行，而不是只停留在文档层面。

## 二、Bootstrap 是怎么注入的

### 1. Claude Code / Cursor 使用 SessionStart hook

Claude Code 的 hook 配置在：  
[hooks.json](superpowers/hooks/hooks.json)

关键逻辑：

- 在 `startup|clear|compact` 时触发
- 调用 `hooks/session-start`

具体定义见：  
[hooks.json:L1-L15](superpowers/hooks/hooks.json#L1-L15)

### 2. `session-start` 会把 `using-superpowers` 注入上下文

核心文件：  
[session-start](superpowers/hooks/session-start)

关键逻辑在这里：  
[session-start:L17-L55](superpowers/hooks/session-start#L17-L55)

这个脚本做了几件关键的事：

1. 读取 `skills/using-superpowers/SKILL.md`
2. 做 JSON 转义
3. 包装成 `<EXTREMELY_IMPORTANT> ... </EXTREMELY_IMPORTANT>`
4. 通过不同平台需要的字段注入：
   - Cursor 用 `additional_context`
   - Claude Code 用 `hookSpecificOutput.additionalContext`

这意味着 Superpowers 的第一步不是等用户提需求再建议怎么做，而是**在会话初始化时先改写 agent 的决策框架**。

## 三、真正的“控制内核”：`using-superpowers`

Superpowers 最关键的 skill 不是某个业务 skill，而是 `using-superpowers`。

文件：  
[using-superpowers/SKILL.md](superpowers/skills/using-superpowers/SKILL.md)

关键内容：  
[using-superpowers/SKILL.md:L10-L45](superpowers/skills/using-superpowers/SKILL.md#L10-L45)

它建立了几个硬规则：

- 只要有 1% 概率某个 skill 适用，就必须先调用 skill
- 在任何响应前都要先判断 skill
- 即使是澄清问题，也不能跳过这一步
- 用户指令优先于 skill，skill 优先于默认系统提示

它的流程图在这里：  
[using-superpowers/SKILL.md:L42-L74](superpowers/skills/using-superpowers/SKILL.md#L42-L74)

这个流程本质上把 agent 的默认行为从：

- 收到请求
- 直接回答或直接操作

改写成：

- 收到请求
- 判断 skill 是否适用
- 先加载相关 skill
- 如果 skill 有 checklist，就转成 Todo
- 再按 skill 指导执行

所以从系统设计角度看，`using-superpowers` 才是整个 harness 的“元调度器”。

## 四、最像状态机的部分：`subagent-driven-development`

如果说 `using-superpowers` 定义了全局规则，那么 `subagent-driven-development` 就是最典型的执行型 harness。

文件：  
[subagent-driven-development/SKILL.md](superpowers/skills/subagent-driven-development/SKILL.md)

技能定义直接说明了它的核心模式：  
[subagent-driven-development/SKILL.md:L8-L12](superpowers/skills/subagent-driven-development/SKILL.md#L8-L12)

- 每个 task 派发 fresh subagent
- 每个 task 后做两阶段 review
- 顺序固定为 spec compliance review 在前，code quality review 在后

### 1. 它的流程图本质上就是 workflow state machine

完整流程图在：  
[subagent-driven-development/SKILL.md:L40-L84](superpowers/skills/subagent-driven-development/SKILL.md#L40-L84)

如果翻译成状态机，大致就是：

1. 读 plan
2. 抽取全部任务
3. 创建 Todo
4. 派 implementer
5. implementer 可提问
6. implementer 实现、测试、提交、自检
7. 派 spec reviewer
8. 如果 spec 不通过，回 implementer 修复并重审
9. spec 通过后再派 code quality reviewer
10. 如果质量不通过，再回 implementer 修复并复审
11. 任务完成后标记 Todo
12. 所有任务完成后做最终 review
13. 进入 finishing skill

这个结构已经不是“给模型写一段提示词”那么简单，而是一个具备：

- 阶段
- 关卡
- 回路
- 验收条件

的工作流 harness。

### 2. 它强调 controller 精确构造子代理上下文

关键说明在：  
[subagent-driven-development/SKILL.md:L8-L10](superpowers/skills/subagent-driven-development/SKILL.md#L8-L10)

这里的核心思想是：

- 子代理不应继承主会话全部上下文
- controller 必须把任务信息和上下文精确裁剪后再交给子代理

这使得主代理像一个 orchestrator，而子代理像一次次受控执行单元。

## 五、Prompt Templates 就是子代理 harness 的“接口协议”

Superpowers 没有把子代理调度写成一个 JS 调度器，而是通过 prompt template 形成稳定协议。

涉及三个关键模板：

- [implementer-prompt.md](superpowers/skills/subagent-driven-development/implementer-prompt.md)
- [spec-reviewer-prompt.md](superpowers/skills/subagent-driven-development/spec-reviewer-prompt.md)
- [code-quality-reviewer-prompt.md](superpowers/skills/subagent-driven-development/code-quality-reviewer-prompt.md)

### 1. Implementer 模板

关键片段：  
[implementer-prompt.md:L9-L18](superpowers/skills/subagent-driven-development/implementer-prompt.md#L9-L18)

要求 controller 在派发任务时直接提供：

- 任务完整文本
- 场景上下文
- 工作目录

并明确要求：

- 不要让子代理自己读 plan 文件
- 子代理如有疑问必须先提问
- 完成后要自检再汇报

自检要求在：  
[implementer-prompt.md:L74-L112](superpowers/skills/subagent-driven-development/implementer-prompt.md#L74-L112)

这相当于把 implementer 的行为 contract 写死了。

### 2. Spec Reviewer 模板

关键片段：  
[spec-reviewer-prompt.md:L21-L35](superpowers/skills/subagent-driven-development/spec-reviewer-prompt.md#L21-L35)

它的审查哲学非常鲜明：

- 不信 implementer 报告
- 必须独立读代码
- 逐条对照需求
- 检查缺失项、额外项、误解项

这使得 spec review 不是礼貌性 review，而是一个“独立验收器”。

### 3. Code Quality Reviewer 模板

关键片段：  
[code-quality-reviewer-prompt.md:L7-L18](superpowers/skills/subagent-driven-development/code-quality-reviewer-prompt.md#L7-L18)

这里明确规定：

- 只有 spec review 通过后，才能进入 code quality review
- code quality review 复用 `requesting-code-review` 提供的评审模板

这体现出 Superpowers 对 review 的分层设计：

- 第一层看“有没有按 spec 做”
- 第二层看“做得好不好”

## 六、跨平台适配：Harness 抽象的不是 API，而是能力

Superpowers 的一个重要特征，是它没有把 harness 逻辑绑定死在某个平台 API 上。

### 1. Cursor / Claude Code

Cursor 插件清单显式暴露了 skills、agents、commands、hooks：  
[plugin.json:L21-L24](superpowers/.cursor-plugin/plugin.json#L21-L24)

这意味着在这些平台上，Superpowers 借助原生插件机制接入。

### 2. Codex

Codex 走的是原生 skill discovery 方案：  
[README.codex.md:L50-L59](superpowers/docs/README.codex.md#L50-L59)

也就是：

- Codex 启动时扫描 `~/.agents/skills/`
- Superpowers 通过 symlink 把自己的 `skills/` 暴露给 Codex
- `using-superpowers` 自动参与调度

多代理能力则依赖 Codex 的 `multi_agent` 功能：  
[README.codex.md:L35-L39](superpowers/docs/README.codex.md#L35-L39)

工具映射定义在：  
[codex-tools.md:L5-L25](superpowers/skills/using-superpowers/references/codex-tools.md#L5-L25)

比如：

- `Task` 对应 `spawn_agent`
- `TodoWrite` 对应 `update_plan`
- `Skill` 在 Codex 中走原生技能加载

这说明 Superpowers 抽象的不是单个平台的工具名，而是“任务派发”“计划跟踪”“技能加载”这些能力接口。

### 3. OpenCode

OpenCode 这边有一个显式插件实现：  
[superpowers.js](superpowers/.opencode/plugins/superpowers.js)

关键逻辑：  
[superpowers.js:L49-L106](superpowers/.opencode/plugins/superpowers.js#L49-L106)

它做两件事：

1. 通过 `config` hook 注册 skills 路径
2. 通过 `experimental.chat.system.transform` 注入 bootstrap 内容

README 也直接总结了这点：  
[README.opencode.md:L91-L106](superpowers/docs/README.opencode.md#L91-L106)

所以从工程结构上看，Superpowers 是：

- 用宿主平台提供 runtime
- 用自己的 bootstrap 和 skill 体系覆盖行为
- 用适配层把能力映射到不同平台

## 七、测试 Harness：Superpowers 怎么验证自己真的工作

Superpowers 最有工程味的一部分，是它不仅有 workflow skill，还有一套测试 harness 来验证这些 workflow。

总览文档：  
[tests/claude-code/README.md](superpowers/tests/claude-code/README.md)

说明中提到：  
[tests/claude-code/README.md:L5-L8](superpowers/tests/claude-code/README.md#L5-L8)

这套测试是通过 `claude -p` 以 headless 模式运行真实会话，再检查行为。

### 1. 测试工具层

基础工具在：  
[test-helpers.sh](superpowers/tests/claude-code/test-helpers.sh)

核心运行逻辑：  
[test-helpers.sh:L4-L29](superpowers/tests/claude-code/test-helpers.sh#L4-L29)

它提供：

- `run_claude`
- `assert_contains`
- `assert_not_contains`
- `assert_count`
- `assert_order`
- 临时项目构建和清理

本质上，这已经是一个简洁的 shell-based eval harness。

### 2. Skill 语义测试

文件：  
[test-subagent-driven-development.sh](superpowers/tests/claude-code/test-subagent-driven-development.sh)

它通过问模型一系列问题来验证 skill 约束有没有被正确表达出来，例如：

- 是否知道先做 spec compliance review
- 是否强调 self-review
- 是否要求 plan 只读一次
- 是否要求 controller 提供完整 task text
- 是否强调不要直接在 main 上开发

这类测试验证的是“skill 语义 contract 是否稳定”。

### 3. 真正的集成测试 Harness

最关键的文件：  
[test-subagent-driven-development-integration.sh](superpowers/tests/claude-code/test-subagent-driven-development-integration.sh)

这个脚本做了完整闭环：

1. 创建临时 Node.js 项目  
   [L24-L45](superpowers/tests/claude-code/test-subagent-driven-development-integration.sh#L24-L45)

2. 写入 implementation plan  
   [L47-L105](superpowers/tests/claude-code/test-subagent-driven-development-integration.sh#L47-L105)

3. 初始化 git 仓库  
   [L107-L112](superpowers/tests/claude-code/test-subagent-driven-development-integration.sh#L107-L112)

4. 用 headless Claude 执行 plan  
   [L136-L158](superpowers/tests/claude-code/test-subagent-driven-development-integration.sh#L136-L158)

5. 从 `~/.claude/projects/...` 找 session transcript  
   [L164-L179](superpowers/tests/claude-code/test-subagent-driven-development-integration.sh#L164-L179)

6. 解析 transcript，验证工具调用是否发生  
   [L187-L217](superpowers/tests/claude-code/test-subagent-driven-development-integration.sh#L187-L217)

7. 检查实现产物是否真实存在、测试是否通过、commit 是否形成  
   [L219-L277](superpowers/tests/claude-code/test-subagent-driven-development-integration.sh#L219-L277)

这个测试方式特别重要，因为它不只是看最终回复写得像不像，而是检查：

- Skill 是否真的被调用
- Task 子代理是否真的被派发
- Todo 是否真的被创建
- 代码文件是否真的产生
- 测试是否真的通过
- Git 提交是否真的发生

这就是一个标准的 transcript-driven harness。

### 4. Skill Triggering Tests

除了执行测试，Superpowers 还测试“技能是否被正确触发”。

文件：  
[skill-triggering/run-test.sh](superpowers/tests/skill-triggering/run-test.sh)

关键逻辑在：  
[run-test.sh:L42-L63](superpowers/tests/skill-triggering/run-test.sh#L42-L63)

它会：

- 给 Claude 一个自然语言 prompt
- 让它跑多轮
- 输出为 stream-json
- 直接 grep `"name":"Skill"` 和对应 skill 名

也就是说，Superpowers 连“技能触发”本身都纳入了 harness 验证。

### 5. 显式技能请求测试

文件：  
[explicit-skill-requests/run-test.sh](superpowers/tests/explicit-skill-requests/run-test.sh)

其中一个很有价值的检查在：  
[run-test.sh:L97-L121](superpowers/tests/explicit-skill-requests/run-test.sh#L97-L121)

它会检查一个特定失败模式：

- 模型有没有在调用 Skill 之前就先用别的工具开始工作

这实际上是在验证一个非常基础的 harness 不变量：

- **先加载 skill**
- **再执行动作**

## 八、为什么这可以被称为 Harness 编程

如果从系统设计角度总结，Superpowers 的 harness 编程主要体现在五点：

### 1. 有明确入口

通过 hook 或系统注入，在会话开始时建立统一行为规则。

### 2. 有声明式控制面

skill frontmatter 负责“何时触发”，skill 内容负责“如何执行”。

### 3. 有工作流编排

不是单次 prompt，而是多阶段、多角色、多回路的流程控制。

### 4. 有跨平台适配

底层 harness 可以变化，但 workflow contract 尽量保持一致。

### 5. 有可验证机制

通过 transcript、产物检查、测试命令、git 历史来验证 agent 是否真正遵守流程。

## 九、一句话总结

Superpowers 的 harness 编程本质上是：

- 用 bootstrap 注入改写 agent 初始行为
- 用 `using-superpowers` 把 skill system 变成调度内核
- 用 `subagent-driven-development` 把 plan 执行流程写成多代理状态机
- 用 transcript-driven integration tests 把这些规则变成可验证 contract

所以它不是一个单体 runtime，而是一个：

- workflow harness
- skill orchestration system
- transcript-driven evaluation harness

的组合体。
