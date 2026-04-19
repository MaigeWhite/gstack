# gstack × Claude Code 探索要点总结

> 探索日期：2026-04-19  
> 项目路径：`/home/user/gstack`  
> Claude Code 二进制：`/opt/claude-code/bin/claude`

---

## 一、gstack 是什么

**gstack（Garry's Stack）** 不是一个普通的 prompt 集合，而是一个为 Claude Code 打造的**团队工程工作流平台**，核心价值是把团队最佳实践编码为 AI 可执行的结构化工作流。

### 三个设计原则（ETHOS.md）

| 原则 | 含义 |
|------|------|
| **Boil the Lake** | AI 时代完整实现的边际成本趋近于零，杜绝"先快后补" |
| **Search Before Building** | 优先复用已有方案，三层知识（tried-and-true → new-popular → first-principles） |
| **User Sovereignty** | AI 建议，人类决定，永远不自动驾驶 |

### 效率压缩参考

| 任务类型 | 人工团队 | CC+gstack | 压缩比 |
|----------|---------|-----------|--------|
| 脚手架/样板 | 2 天 | 15 分钟 | ~100x |
| 测试编写 | 1 天 | 15 分钟 | ~50x |
| 功能开发 | 1 周 | 30 分钟 | ~30x |
| Bug 修复 | 4 小时 | 15 分钟 | ~20x |
| 架构设计 | 2 天 | 4 小时 | ~5x |

---

## 二、Skill 系统

### SKILL.md 生成链

```
SKILL.md.tmpl（模板，人工维护）
    ↓ scripts/gen-skill-docs.ts（支持 10 种宿主平台）
SKILL.md（Claude 读取并执行的指令文件）
```

生成时注入：CLI 命令列表、快照标志、术语表、平台专属路径重写、工具名映射。

### Skill 声明的关键字段

```yaml
name: ship
preamble-tier: 4
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Agent
  - AskUserQuestion
hooks:
  PreToolUse:
    - matcher: "Edit"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/bin/check-freeze.sh"
triggers:
  - ship it
  - create a pr
```

### Preamble 机制

每个 skill 启动时自动注入共享基础设施（bash 代码，确定性执行）：

```
更新检查 → 会话追踪 → 用户偏好读取 → 仓库状态检测
→ 历史学习记录加载 → 遥测初始化
```

输出结构化状态：`BRANCH / PROACTIVE / REPO_MODE / LEARNINGS / EXPLAIN_LEVEL`，后续步骤基于这些确定性事实运行。

### 30+ 工作流技能覆盖完整研发生命周期

| 阶段 | 技能 | 作用 |
|------|------|------|
| 创意/规划 | `/office-hours` | YC 式初创诊断 + 创意头脑风暴 |
| 架构审查 | `/plan-ceo-review` `/plan-eng-review` | 战略 + 架构双维度审查 |
| 自动化审查 | `/autoplan` | CEO → 设计 → 工程 → DX 流水线 |
| 调试 | `/investigate` | 系统性根因分析，无根因不提修复 |
| QA | `/qa` | 浏览器自动化测试，迭代修复 |
| 安全 | `/cso` | OWASP Top 10 + STRIDE 审计 |
| 代码审查 | `/review` | 差异分析，SQL 安全，LLM 信任边界 |
| 发布 | `/ship` | 合并 → 测试 → CHANGELOG → PR 全流程 |
| 上线监控 | `/canary` | 部署后持续监控循环 |

---

## 三、团队功能触发机制

### 触发方式

| 方式 | 实现 |
|------|------|
| 手动输入 `/ship` `/qa` 等 | Skill tool 直接调用 |
| 自动路由 | CLAUDE.md 的 `## Skill routing` 规则 |
| 上下文感知 | `PROACTIVE=true` 时 Claude 识别意图自动 invoke |
| 语音别名 | frontmatter `triggers` 字段 |

### CLAUDE.md 路由规则示例

```markdown
## Skill routing
When the user's request matches an available skill, ALWAYS invoke it
using the Skill tool as your FIRST action.

- Bugs, errors → invoke investigate
- Ship, deploy, create PR → invoke ship
- QA, test the site → invoke qa
```

---

## 四、多角色协作模式

### 核心结论：默认共享上下文，隔离是例外

```
/autoplan 执行模型（同一个 Claude context window）：

Phase 0：读入所有 SKILL.md → 全部进 context
    ↓
Phase 1 (CEO 视角)：分析 → 结论留在 context
    ↓
Phase 2 (设计视角)：直接读 Phase 1 结论（无需传递）
    ↓
Phase 3 (工程视角)：同上
    ↓
Final Gate：汇总所有 taste decisions → AskUserQuestion（唯一需用户确认处）
```

强制规则（SKILL.md 原文）：
> "Phases MUST execute in strict order. NEVER run phases in parallel — each builds on the previous."

### 三种信息传递机制

| 场景 | 机制 | 说明 |
|------|------|------|
| Phase 间通信 | 同一 context window | 顺序执行，天然共享，无需额外传递 |
| 独立第二意见 | Agent tool（隔离 subagent）+ Codex CLI | 防主分析污染，代码层保证隔离 |
| 跨会话状态 | 文件系统 JSONL（`~/.gstack/projects/$SLUG/`）| context 清空后仍可恢复 |

### 真正的多 AI 场景

| 机制 | 实现方式 |
|------|---------|
| `/codex` skill 第二意见 | Bash 调用 OpenAI Codex CLI 子进程 |
| OpenClaw 编排 | `$OPENCLAW_SESSION` 环境变量感知，自动切换无交互模式 |
| skill 内部并行 | `Agent` tool（仅高权限 skill 如 ship 可用） |

---

## 五、gstack 基础设施层

Skill 是用户界面层，让工程真正 work 的是底层六层基础设施。

### 1. Browse 持久化浏览器守护进程

```
首次 $B 命令 → 读 .gstack/browse.json（port + token + pid）
    ↓ 进程不存在
启动 Bun HTTP server（随机端口 10000-60000，UUID bearer token）
    ↓
后续命令：POST localhost:{port}/command（~100ms/命令）
    ↓
闲置 30 分钟 → 自动关闭
```

- Cookie、登录态、Tab **跨命令持久**
- 版本不匹配时自动重启
- state file 权限 0o600，仅 localhost 绑定
- 50+ 命令：`goto / snapshot / click / fill / screenshot / responsive` 等

### 2. 33 个 bin/ 工具脚本（运行时工具链）

不依赖 AI，bash/TypeScript 实现，输出完全可预测：

| 类别 | 工具 | 作用 |
|------|------|------|
| 配置 | `gstack-config` | `~/.gstack/config.yaml` get/set |
| 项目标识 | `gstack-slug` | git remote 派生项目 slug，跨会话一致 |
| 学习记忆 | `gstack-learnings-log/search` | JSONL 持久化，置信度衰减 + 去重 |
| 问题偏好 | `gstack-question-preference` | 记录用户问题偏好，防 profile 污染 |
| 版本管理 | `gstack-update-check` | 拉远端 VERSION，管理 snooze 状态 |
| 遥测 | `gstack-telemetry-log/sync` | 本地 JSONL + 可选远端，含 crash recovery |
| 时间线 | `gstack-timeline-log/read` | 记录每个 skill 的 started/completed |
| 团队 | `gstack-team-init` | 生成 CLAUDE.md 路由规则 + 安装检查 hook |

### 3. 开发者心理画像系统

```
用户回答 AskUserQuestion
    ↓ gstack-question-log（记录选择 + 信号 key）
scripts/psychographic-signals.ts（5 维度，±0.03~0.06 delta）
    ↓
~/.gstack/developer-profile.json：
{
  scope_appetite:     0.73,  // 倾向做完整 vs 快速
  risk_tolerance:     0.41,
  detail_preference:  0.68,
  autonomy:           0.55,
  architecture_care:  0.82
}
    ↓ 影响 skill 行为（问题频率、输出详细程度等）
```

### 4. 多平台 Host 配置系统

支持 10 种 AI 工具，通过声明式配置生成各平台专属 SKILL.md：

```typescript
// openclaw 配置示例
{
  pathRewrites: [
    { from: '~/.claude/skills/gstack', to: '~/.openclaw/skills/gstack' },
    { from: 'CLAUDE.md', to: 'AGENTS.md' }
  ],
  toolRewrites: {
    'use the Bash tool':  'use the exec tool',
    'use the Agent tool': 'use sessions_spawn'
  },
  suppressedResolvers: ['CODEX_SECOND_OPINION', 'DESIGN_OUTSIDE_VOICES']
}
```

### 5. Chrome 扩展

MV3 扩展连接 browse 守护进程，在页面注入 `@e1 @e2 @e3` ref 标注，使 `$B snapshot -i` 返回的引用与真实 DOM 元素一一对应。

### 6. Design 二进制（AI 图像生成 CLI）

OpenAI Image API CLI，支持多轮设计迭代：`generate / variants / iterate / compare / design-to-code`。

---

## 六、Claude Code 的 WebSearch 实现

### 不是 MCP，是 Anthropic Server-side Tool

从 Claude Code 二进制（`/opt/claude-code/bin/claude`）提取的三种 tool 类型处理路径完全独立：

```javascript
case "tool_use":               // Claude Code 内置工具（本地执行）
case "mcp_tool_use":           // MCP 协议工具（本地进程）
case "server_tool_use":        // Anthropic 服务端工具 ← WebSearch 在这里
case "web_search_tool_result": // 结果直接从 API 流返回
```

用量统计也独立：
```javascript
server_tool_use: {
  web_search_requests: ...,   // 独立计费
  web_fetch_requests:  ...
}
```

### 搜索关键词生成逻辑

**主 Claude 模型直接生成，Claude Code 不做任何 query 改写。**

从二进制提取的 inputSchema：
```typescript
{
  query: string.min(2),          // "The search query to use" — Claude 自己填
  allowed_domains?: string[],
  blocked_domains?: string[]
}
```

完整调用链（从二进制 `call()` 函数还原）：

```
① 主 Claude 模型理解对话 → 自行决定搜索词
   tool_use { query: "javascript best practices 2025" }
    ↓
② Claude Code call() 函数接管
   构造：user message = "Perform a web search for the query: " + query
   发起第二次独立 API 调用：
     systemPrompt: "You are an assistant for performing a web search tool use"
     toolChoice:   { type: "tool", name: "web_search" }  ← 强制触发
     extraToolSchemas: [{ type: "web_search_20250305", max_uses: 8 }]
    ↓
③ Anthropic 服务端执行搜索
   流式返回：server_tool_use → content_block_delta（实时显示 "Searching: xxx"）
           → web_search_tool_result（结果）
    ↓
④ 结果格式化注入主对话
   "REMINDER: You MUST include the sources above using markdown hyperlinks."
```

**关键点**：第二次 API 调用里的 Claude 不做任何推理，`toolChoice` 强制它立刻触发服务端工具，只是个"执行器"。

### vLLM 无法使用 WebSearch

从二进制提取的 `isEnabled()` 白名单：

```javascript
function uq() {   // provider 判断（基于环境变量，与模型名无关）
  if (CLAUDE_CODE_USE_BEDROCK)       return "bedrock";
  if (CLAUDE_CODE_USE_FOUNDRY)       return "foundry";
  if (CLAUDE_CODE_USE_ANTHROPIC_AWS) return "anthropicAws";
  if (CLAUDE_CODE_USE_VERTEX)        return "vertex";
  return "firstParty";   // 默认
}

isEnabled() {
  if (provider === "firstParty")   return true;
  if (provider === "anthropicAws") return true;
  if (provider === "foundry")      return true;
  if (provider === "vertex")       return model.includes("claude-*-4");
  return false;   // vLLM → false
}
```

**改模型名无效**：provider 判断完全基于 `CLAUDE_CODE_USE_*` 环境变量，模型名只在 Vertex 分支里用。

**"绕过"路径**：不设任何 `CLAUDE_CODE_USE_*` + 设 `ANTHROPIC_BASE_URL=http://localhost:8000` → provider 默认 `firstParty` → `isEnabled()=true`，但：

```
web_search_20250305 是 Anthropic 服务端专属工具类型
本地 vLLM 无法执行实际网络搜索
→ isEnabled() 可骗过，搜索功能仍然不工作
```

**结论**：绕不过。本地模型要搜索，唯一可行方案是接 **MCP 搜索服务**（Brave/Tavily/SearXNG）。

---

## 七、gstack 遵循的 Claude Code 规范

### 代码层硬约束（Claude Code runtime 执行，不可绕过）

| 规范 | Claude Code 如何执行 | gstack 如何使用 |
|------|---------------------|----------------|
| `allowed-tools` | API 层工具白名单，物理限制 | 每个 skill 精确声明最小权限 |
| `hooks/PreToolUse` | 工具调用前执行 bash，返回 `{"permissionDecision":"deny"}` 可拦截 | `/freeze` `/guard` 实现文件系统边界 |
| `hooks/SessionStart` | 会话启动时执行脚本 | 团队模式自动升级 gstack |
| Skill 发现路径 | `~/.claude/skills/{name}/SKILL.md` 目录约定 | setup 脚本按此路径安装 |
| `Skill` tool | `/skill-name` 路由到 SKILL.md | CLAUDE.md routing 规则触发 |
| `AskUserQuestion` | 结构化阻塞，等待用户输入 | 所有决策点、onboarding 引导 |
| `Agent` tool | 创建隔离 context 的 subagent | dual voice 独立审查，强隔离 |
| `ExitPlanMode` | plan mode 的退出门控 | autoplan 完成后退出 |

`check-freeze.sh` 示例（代码层拦截，非 prompt 约束）：

```bash
# 返回 deny → Claude Code 直接拦截 Edit 调用
printf '{"permissionDecision":"deny","message":"[freeze] Blocked: %s is outside boundary (%s)."}\n' \
  "$FILE_PATH" "$FREEZE_DIR"
```

### Prompt 层软约束（依赖模型遵循，存在漂移风险）

| 规范 | 漂移风险 | 兜底手段 |
|------|---------|---------|
| 顺序执行 Phase（CEO→设计→工程）| 中 | SKILL.md 明确警告 + E2E 测试 |
| 强制输出 artifact 到磁盘 | 中 | 验证文件是否存在 |
| 全深度执行不压缩 | 高（SKILL.md 原文警告）| LLM judge 评分 |
| STOP 关键词暂停 | 低 | 结构清晰 |
| Writing Style / 术语注释 | 高 | 可接受偶尔漂移 |

SKILL.md 原文承认漂移存在：
> "If you catch yourself writing fewer than 3 sentences for any review section, **you are likely compressing**."

### 分工原则

```
能出错就死  →  代码层（allowed-tools + hooks）
允许偶尔漂移 →  Prompt 层
漂移后能检测 →  E2E 测试（claude -p）+ LLM judge 评分
```

---

## 八、完整架构总结

```
用户 / Claude
    │
    ▼  Skill tool（Claude Code 原生，/skill-name 路由）
┌─────────────────────────────────────────────────┐
│              Skill 层（SKILL.md）                │
│  allowed-tools（API 层白名单）                    │  ← 代码层边界
│  hooks/PreToolUse（bash 拦截）                   │
│  AskUserQuestion（结构化交互）                    │
│  Agent tool（隔离 subagent）                     │
└────────────────┬────────────────────────────────┘
                 │ 调用 bin/ 工具脚本
    ┌────────────┼──────────────────┐
    ▼            ▼                  ▼
bin/ 工具链   Browse 守护进程      Design CLI
（33个脚本）  （Playwright HTTP）   （OpenAI API）
    │
    ├── gstack-config        确定性配置读写
    ├── gstack-learnings-*   跨会话知识记忆
    ├── gstack-developer-profile  心理画像（5维度）
    ├── gstack-timeline-*    执行历史
    └── gstack-telemetry-*   本地 JSONL + 可选远端
         │
         ▼
    ~/.gstack/（本地状态，不依赖 AI，可审计）
    ├── config.yaml
    ├── projects/{slug}/
    │   ├── learnings.jsonl
    │   ├── timeline.jsonl
    │   └── question-preferences.json
    └── analytics/skill-usage.jsonl

多平台支持（hosts/）
    gen-skill-docs.ts → 10 种 AI 工具各自的 SKILL.md
    （路径重写 + 工具名映射 + 禁用特定 resolver）
```

### 一句话总结

> gstack 把"不能出错的约束"全部压到 Claude Code 代码层（工具权限、文件系统边界、会话钩子），把"可以容忍漂移的部分"留给 prompt + 测试兜底。Skill 是用户界面，33 个 bin 工具是确定性骨架，两层分工明确——这是整个工程能可靠运行的根本原因。
