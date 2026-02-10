# Plan Analyzer（任务分析器）

[English](README.md)

一个 Claude Code 插件，在执行前对任务进行复杂度分析，并路由到最省 token 的执行路径。

> *"站在月球看地球"* — [老王来了@dlw2023](https://www.youtube.com/@dlw2023)

这句话正是 Plan Analyzer 的设计理念：先退一步，看清全貌，再行动。不急于一头扎进执行，而是站在高处俯瞰任务 — 它的复杂度、可用的工具、每条路径的成本 — 然后选择最高效的那条。正如站在月球上才能看清地球的真实尺度，站在全局视角分析任务，才能找到最简洁的执行路径。

**先分级，再路由** — 不同于盲目降级模型，Plan Analyzer 根据任务特征精确匹配资源。

## 安装

### 从 Marketplace 安装

```bash
# 添加 marketplace 源
/plugin marketplace add https://github.com/huacheng/moonview

# 安装插件
/plugin install plan-analyzer
```

### 手动安装

将仓库克隆到 Claude Code 插件目录：

```bash
git clone https://github.com/huacheng/moonview.git ~/.claude/plugins/local/plan-analyzer
```

然后在 `~/.claude/plugins/installed_plugins.json` 中注册：

```json
{
  "plan-analyzer@local": [
    {
      "scope": "user",
      "installPath": "~/.claude/plugins/local/plan-analyzer",
      "version": "1.0.0"
    }
  ]
}
```

## 使用方式

通过以下任意触发词调用：

```
/plan-analyzer refactor auth to use JWT
pa: add validation to the login form
plan-analyze: refactor the database module
```

**别名：** `pa`, `plan-analyze`, `analyze-plan`, `plan-budget`, `budget-plan`

## 工作原理

Plan Analyzer 是一个通用的预执行复杂度分析器，覆盖**整个 Claude Code 工具链**：

- 原生工具（Read / Edit / Write / Grep / Glob / Bash）
- 内置 agent（Explore / Plan / general-purpose / Bash agent）
- oh-my-claudecode (OMC) 插件 agent
- MCP 工具及其他已加载的插件能力

### 第一步：分级复杂度（L0–L4）

| 等级 | 名称 | 标准 | 示例 |
|------|------|------|------|
| **L0** | Trivial | 单文件, <10 行, 明显变更 | 修错别字, 改配置值, 加 import |
| **L1** | Simple | 单文件, 逻辑清晰, <50 行 | 加函数, 修明确 bug, 加验证 |
| **L2** | Moderate | 2-3 文件, 模式明确 | 加 API endpoint + handler, 重构单模块 |
| **L3** | Complex | 4+ 文件, 横切关注点 | 新功能含 DB + API + tests, 大型重构 |
| **L4** | Massive | 系统级, 架构性, 10+ 文件 | 新子系统, 全栈功能, 迁移 |

### 第二步：路由到最低成本策略

| 等级 | 策略 | Agent 数量 | 验证方式 |
|------|------|-----------|---------|
| **L0** | 原生工具直接操作 | 0 | 无 |
| **L1** | 原生工具或 1 个轻量 agent | 0-1 | `lsp_diagnostics` |
| **L2** | 1-2 个 agent 顺序执行 | 1-2 | 构建 / 测试运行 |
| **L3** | 2-4 个 agent，可能并行 | 2-4 | 完整测试套件 + 审查 |
| **L4** | 编排模式 | 3-6 | 完整验证协议 |

### 第三步：输出并执行

```
[TASK ANALYSIS] L2: Add health endpoint with DB check
Strategy: single-agent
Toolchain: native+omc
Steps:
  1. Read existing route patterns -> Read tool
  2. Implement endpoint + handler -> executor-low:haiku
  3. Verify -> lsp_diagnostics + test run
Verification: build check
Est. overhead: low
```

## 工具选择层级

从最便宜到最贵：

```
Layer 0: 原生工具 (Read/Edit/Write/Grep/Glob/Bash)
Layer 1: 内置轻量 agent (Explore:haiku)
Layer 2: OMC 低tier agent (executor-low:haiku)
Layer 3: 内置通用 agent (general-purpose:sonnet)
Layer 4: OMC 中tier agent (executor:sonnet)
Layer 5: OMC 高tier agent (executor-high:opus)
Layer 6: 编排模式 (ultrawork / autopilot / ralph)
```

## 快速规则

| 规则 | 说明 |
|------|------|
| **文件数** | 要修改的文件 ≤ 1 → 直接编辑，不用 agent |
| **已知路径** | 文件路径已知 → 直接 `Read`，绝不用 Explore agent |
| **关键字搜索** | 直接用 `Grep`/`Glob`，绝不用 Bash grep/find |
| **批处理** | 相关变更合入一个 agent prompt（最多 5 文件） |
| **验证** | 匹配复杂度（L0=无, L1=lsp_diagnostics, L2+=tests） |
| **模型 tier** | 先试最便宜（haiku → sonnet → opus），仅失败时升级 |
| **MCP 工具** | 直接调用 `lsp_diagnostics`, `ast_grep_search`，无需 agent |

## 反浪费模式

| 浪费模式 | 修复方法 | 节省 |
|---------|---------|------|
| Agent 做单文件编辑 | 直接 Edit 工具 | ~2.5K tokens |
| Opus 做已知模式任务 | Haiku/Sonnet | ~5K tokens |
| Explore agent 查已知路径 | 直接 Read 工具 | ~1.8K tokens |
| L0-L1 做 Architect 验证 | 跳过或 lsp_diagnostics | ~4.5K tokens |
| 逐文件 spawn agent | 批处理到一个 agent | ~2.5K × (N-1) tokens |

## 与 OMC 模式集成

Plan Analyzer 作为 OMC 编排模式的**预处理器**：

| 模式 | Plan Analyzer 角色 |
|------|-------------------|
| **ecomode** | 先分级，ecomode 再进一步降级 tier |
| **ultrawork** | 决定是否值得并行化 |
| **ralph** | 优化每次迭代的 agent 选择 |
| **autopilot** | 在 autopilot 规划前做预筛选 |

未安装 OMC 时，Plan Analyzer 通过原生工具和内置 agent 同样有效。

## 与 Ecomode 对比

| 方面 | Plan Analyzer | Ecomode |
|------|--------------|---------|
| 范围 | 全工具链 | 仅 OMC agent |
| 方式 | 先分级再路由 | 盲目降级 tier |
| 直接编辑 | 是，L0-L1 跳过 agent | 仍委派给 agent |
| 批处理 | 合并相关文件 | 无批处理意识 |
| 无 OMC | 完全可用 | 不适用 |

**最佳组合：** Plan Analyzer + Ecomode 一起使用提供最高 token 效率。

## 许可证

MIT
