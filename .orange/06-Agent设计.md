# 06 Agent 设计

## 架构理念：中间件驱动的 Agent

DeerFlow 的 Agent 设计核心是**中间件链（Middleware Chain）**，而非传统的节点图。每个 Middleware 在 `before_model`（LLM 调用前）和 `after_model`（LLM 调用后）各有一个执行钩子，职责单一，易于组合和扩展。

```
消息进入
   ↓
[Middleware 1].before_model()
[Middleware 2].before_model()
...
[Middleware N].before_model()
   ↓
LLM 调用（可能包含工具执行循环）
   ↓
[Middleware N].after_model()
...
[Middleware 2].after_model()
[Middleware 1].after_model()
   ↓
响应输出
```

## 完整 Middleware 链（18 个）

| # | Middleware | 阶段 | 功能 |
|---|-----------|------|------|
| 1 | `ThreadDataMiddleware` | before | 创建/确保线程目录（workspace/uploads/outputs）|
| 2 | `UploadsMiddleware` | before | 将上传文件列表注入 HumanMessage 上下文 |
| 3 | `SandboxMiddleware` | before | 获取沙箱环境，记录 sandbox_id 到 state |
| 4 | `DanglingToolCallMiddleware` | before | 注入缺失的 ToolMessage 占位符，防 tool_call_id 不匹配 |
| 5 | `LLMErrorHandlingMiddleware` | before | 归一化 LLM 调用失败，转为可恢复错误 |
| 6 | `GuardrailMiddleware` | before（工具前）| 可插拔工具调用授权（Allowlist / OAP 协议）|
| 7 | `SandboxAuditMiddleware` | before（工具前）| 安全审计日志 |
| 8 | `ToolErrorHandlingMiddleware` | after | 工具异常转 ToolMessage，不中断 |
| 9 | `SummarizationMiddleware` | after | token 超限时压缩旧消息（保留最近 N 条 + Skills）|
| 10 | `TodoListMiddleware` | after | Plan 模式下任务追踪（提供 write_todos 工具）|
| 11 | `TokenUsageMiddleware` | after | 收集 token 使用统计 |
| 12 | `TitleMiddleware` | after | 首次完整对话后 LLM 生成标题 |
| 13 | `MemoryMiddleware` | after | 过滤消息，入队异步记忆更新（30s debounce）|
| 14 | `ViewImageMiddleware` | before（每次 LLM 前）| 将 viewed_images 图片 base64 注入 |
| 15 | `DeferredToolFilterMiddleware` | before | 隐藏延迟加载工具 schema（tool_search 模式）|
| 16 | `SubagentLimitMiddleware` | after | 截断超出 MAX_CONCURRENT_SUBAGENTS(3) 的 task 调用 |
| 17 | `LoopDetectionMiddleware` | after | 检测重复工具调用循环，强制生成文本答案 |
| 18 | `ClarificationMiddleware` | after（必须最后）| 拦截 ask_clarification，发出 Command(goto=END) |

## ThreadState 数据结构

LangGraph 的状态对象，贯穿整个 Agent 执行：

```python
class ThreadState(AgentState):
    messages: list[BaseMessage]           # LangGraph 标准消息历史
    sandbox: SandboxState | None          # {sandbox_id}
    thread_data: ThreadDataState | None   # {workspace_path, uploads_path, outputs_path}
    title: str | None                     # 自动生成的会话标题
    artifacts: list[str]                  # 生成物路径列表（自动去重）
    todos: list | None                    # Plan 模式任务列表
    uploaded_files: list[dict] | None     # 上传文件信息
    viewed_images: dict[str, ViewedImageData]  # 图片 base64 缓存（可清空节省内存）
```

## 系统提示结构

`apply_prompt_template()` 生成的系统提示包含以下 XML 块（按顺序）：

```xml
<role>          Agent 名称和基本身份 </role>
<soul>          SOUL.md 内容（自定义 Agent 专属人格）</soul>
<memory>        记忆注入（top-15 Facts + 上下文摘要，≤2000 token）</memory>
<thinking_style>  思维指引（先思考再行动）</thinking_style>
<clarification_system>  澄清规则（有歧义先问清楚）</clarification_system>
<skill_system>  可用 Skills 目录（按需渐进加载）</skill_system>
<subagent_system>  子 Agent 编排规则（并发限制、分批策略）</subagent_system>
<working_directory>  虚拟文件路径说明 </working_directory>
<response_style>  输出风格规范（Markdown、引用格式）</response_style>
<citations>     引用规范（强制 web 搜索后标注来源）</citations>
<critical_reminders>  关键提醒（避免幻觉、完成任务才停）</critical_reminders>
<current_date>  当前日期（防止时间幻觉）</current_date>
```

## 子 Agent 执行架构

```
Lead Agent（主线程）
    │
    ├── task("分析 AWS", subagent_type="general-purpose")
    ├── task("分析 Azure", subagent_type="general-purpose")
    └── task("分析 GCP", subagent_type="general-purpose")
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
_scheduler_pool（3 workers 接收任务）
    │
    ▼
_execution_pool（3 workers 实际执行）
    │
    每个子 Agent 在独立线程中运行独立 asyncio 事件循环
    （防止父子 Agent 的 asyncio 事件循环冲突）
    │
    继承: sandbox_state + thread_data（共享沙箱文件系统）
    隔离: 消息历史 + Skills 加载（各自独立）
    │
    ▼
SubagentResult {
    task_id: str,
    status: "completed" | "timeout" | "error",
    result: str,           # 子 Agent 的最终响应
    ai_messages: list,     # 所有中间 AI 消息（含工具调用细节）
    started_at: datetime,
    completed_at: datetime
}
```

## 自定义 Agent 机制（SOUL）

每个自定义 Agent 可以定义：

```
agents/{name}/
├── SOUL.md       Agent 个性/角色定义（注入系统提示 <soul> 块）
├── config.yaml   工具白名单、Skills 白名单、模型配置
└── USER.md       用户级配置（可选，用户可覆盖）
```

典型 `SOUL.md` 示例：
```markdown
You are Aria, a senior financial analyst with 15 years of experience...
Your communication style is professional but approachable...
You always cite data sources and provide confidence levels...
```

IM 渠道通过 `assistant_id` 字段路由到指定 Agent：
- Telegram 渠道 → 路由到 `agent_a`（友好对话风格）
- Slack 渠道 → 路由到 `agent_b`（专业分析风格）

## 上下文压缩机制（SummarizationMiddleware）

当上下文超过配置的 token 限制时自动触发：

```
触发条件: current_tokens >= summarization.threshold × context_window
                                ↓
压缩策略:
1. 保留头部消息（系统提示 + 前 N 条用户消息，含 Skills）
2. 保留尾部消息（最近 M 条消息）
3. 对中间部分调用辅助 LLM 生成结构化摘要
4. 摘要替换中间部分
                                ↓
注意:
- Skills 注入的消息不压缩（保留完整 Skill 内容）
- 摘要明确标记为"[SUMMARY OF PREVIOUS CONTEXT]"
```

## 记忆系统设计

### 三层记忆

```
Layer 1: 工作记忆（当前对话消息列表）
  → 存于 LangGraph checkpoint（SQLite/PostgreSQL）
  → 受 SummarizationMiddleware 管理，超限自动压缩

Layer 2: 持久 Facts（跨会话长期记忆）
  → 存于 .deer-flow/users/{uid}/memory.json
  → 结构化：Facts + userContext + workContext + summary
  → 每次对话后 LLM 异步提取更新

Layer 3: 产物（文件形式的输出）
  → 存于 .../user-data/outputs/
  → 引用路径记录在 ThreadState.artifacts
```

### Facts 数据结构

```json
{
  "content": "用户偏好使用 Python 而非 JavaScript",
  "category": "preference",    // preference / knowledge / context / behavior / goal
  "confidence": 0.92,          // 0.0 ~ 1.0，LLM 估算
  "source": "conversation",
  "updated_at": "2026-05-01T10:00:00Z"
}
```

### 去重机制

- whitespace 归一化后比较（`" memory"` == `"memory"`）
- 相同内容更新 confidence 而非插入重复
- 低 confidence Facts（< 0.3）定期清理

## Skills 加载机制

```
Agent 初始化时：
  → 扫描 skills/ 目录
  → 构建技能索引（name + description + requires_tools）
  → 注入系统提示 <skill_system> 块（仅目录，不含全文）

Agent 执行时，若需要某个 Skill：
  → Agent 调用内置工具 read_skill(name)
  → SkillLoader 读取对应 SKILL.md 全文
  → 将全文注入为系统消息（标记为 skill_message）
  → 后续 LLM 调用包含此 Skill 的完整指导

注意：
  → SummarizationMiddleware 压缩时保留 skill_message（不压缩）
  → 每个 Skill 注入仅一次，相同 Skill 不重复注入
```
