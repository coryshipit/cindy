# 火山方舟 Coding Plan 用量显示：需求与调研基线

> 状态：需求定义 + 调研基线，**尚未进入实现**。本文不含实现方案、数据模型终稿或 UI 设计。
> 核对日期：2026-11-08。Cindy 源码基线：`feadb203dee25ff5ca7ca4051642a413a832c234`。
> 文档所在分支：`feat/volcengine-coding-plan-usage`。
> 接手者须知：本文的「已确认」与「待验证」分区必须分开读，不要把官方字段**定义**当成已跑通的**输出**。

目标是在 Cindy 中展示火山方舟 Coding Plan 的订阅用量。本阶段只回答「能不能读、能读到什么」，
不预设读取通道、不预设展示位置、不锁定数据模型。

## 1. 需求

### 1.1 目标

用户使用火山方舟 Coding Plan 跑任务时，能在 Cindy 内看到该订阅的额度使用情况与窗口重置时刻，
不必切到火山控制台。价值主张是**提前看到窗口余量**，避免在工作途中撞上限流。

### 1.2 已拍板的边界

| 决定 | 结论 | 理由 |
| --- | --- | --- |
| 本期范围 | **只做调研，不定实现方式** | 读取通道尚未落实，不宜先写死方案 |
| 周期范围 | **只显示官方实际返回的窗口**，不硬凑固定三个 | 官方按套餐返回 `periods[]` 数组，数量不固定 |
| 数据模型 | **本期不定**，待实机验证字段后再定 | 避免按推测结构固化 schema |
| 展示位置 | **后议**（设置页用量卡 / 新任务输入区均未定） | 数据层未成立前讨论 UI 无意义 |
| 调研深度 | **纸面调研**：只读官方文档与 CLI 文档，不实际执行 | 不在本期引入实机依赖 |
| 身份归属表述 | **账号/订阅级共享用量，不归因当前任务或模型** | 官方不支持按模型明细，归因即编造 |

### 1.3 明确不做

- 不抓取火山控制台或官网后台网页。
- 不记录、不落盘、不向 renderer 传递任何令牌或密钥。
- 不做成独立插件，走 Cindy 原生 provider 用量管线。
- 不编造请求次数、金额或按模型明细；官方无此数据。

## 2. 现状基线（已核实）

### 2.1 provider 预置已存在，不需要新建

| 事实 | 出处 |
| --- | --- |
| `id: volcengine-coding-plan`，中文名「火山方舟 Coding Plan」，英文名「Volcengine Ark Coding Plan」 | `packages/model-providers/catalog/providers.json` |
| 模型 `ark-code-latest`；`regionHint: cn`；`docsUrl: https://www.volcengine.com/docs/82379/1925114` | 同上 |
| Claude Code runtime：`https://ark.cn-beijing.volces.com/api/coding`；Codex / Pi：`.../api/coding/v3`（openai-chat） | 同上 |
| `authKind: key`（API Key 连接，不是 OAuth 订阅） | `packages/model-compat/src/upstream-profiles.json` |
| 品牌映射到 `volcengine`；控制台入口 `https://console.volcengine.com/ark/` | `providerBranding.ts`、`providerSetupLinks.ts` |

结论：**不需要新建 provider**，工作量在用量读取与展示，不在目录接入。

### 2.2 用量读取已有 5 个来源，火山会成为第 6 个

`apps/desktop/src/shared/providerQuotaSource.ts` 的 `providerWeeklyQuotaSource()` 是唯一分发点，
当前返回 `'codex' | 'claude' | 'xai' | 'kimi' | 'opencode-go' | null`。

| 来源 | 读取方式 |
| --- | --- |
| `claude` | **内置 CLI** 的 `get_usage` 控制请求 |
| `codex` | 内部接口 / web usage |
| `xai` | HTTP `billing?format=credits` |
| `kimi` | HTTP 官方 `/coding/v1/usages` |
| `opencode-go` | HTTP |

结论：4/5 走 HTTP，只有 Claude 走 CLI。**火山若只能走 CLI，将是第一个「Coding Plan + 外部 CLI」的
来源**，与现有主流模式不同，需要在方案阶段单独评估。

### 2.3 已有可复用的标准快照形态

`apps/desktop/src/shared/kimiCodingUsage.ts` 给出了 Cindy 对 Coding Plan 用量的标准形态：

```ts
interface KimiCodingUsageSnapshot {
  kind: 'kimi-coding-quota';
  windows: Array<{ id: KimiUsageWindowId; usedPercent: number; resetsAt: number | null }>;
  fetchedAt: number;
}
```

其窗口集合 `['limit_5h', 'limit_7d', 'limit_month_total', 'limit_month_code']` 同时覆盖
5 小时、周、月，证明**三周期在 Cindy 里已有先例**，且官方缺失窗口时的处理原则是注释明写的：

> Missing windows are not unlimited or zero use.

即**缺窗口不等于无限或零使用**，不得用缺省值填充。这条原则应直接沿用。

### 2.4 Claude 的 CLI 读取范式可直接借鉴

`apps/desktop/src/main/maker-host/claude-native-cli.ts` 与
`apps/desktop/src/main/usage/claudeSubscriptionUsageRefresh.ts` 已实现：

- CLI 用自己的登录查询，**Cindy 不接触凭证**；
- IPC 立即返回缓存快照，后台按节流刷新（默认 180s）；
- 失败指数退避（5min 起 ×2 封顶 30min），**保留已有缓存**；
- 身份维度是 CLI 登录账号指纹，换号即清旧快照；
- 无法识别的输出形状按「解析失败」处理并退避，**不当作「没有套餐」**。

这套骨架是 `arkcli` 方案的现成模板，若最终选 CLI 通道可直接复用。

## 3. 官方 CLI 的字段定义（一手材料）

来源：`volcengine/ark-cli` 公开仓库（Apache-2.0，主分支 `main`，仓库最近推送 2026-10-29），
文件 `skills/arkcli-usage/SKILL.md` 与 `skills/arkcli-usage/references/arkcli-usage-plan.md`，
读取日期 2026-11-08。该仓库是**分发仓库**（含 README、skills、dsh-plugins），不含 CLI 源码，
因此字段来自官方 skills 文档，而非源码类型定义。

### 3.1 命令

```
arkcli usage plan --product coding-plan         # 个人版
arkcli usage plan --product coding-plan-team    # 团队版
arkcli usage plan                                # 默认探 4 个 SKU
arkcli usage plan --all                          # 4 桶全跑，不探订阅
```

默认探 `personal × 2 + team × 2`，**只有 `subscribed` 桶才下发 usage 请求**；`--product` 跳过探测单查。
与 `usage balance --type plan` 是同一份数据，`usage plan` 输出更完整（带 `subscribed` / `updated_at`）。

### 3.2 输出结构

顶层 `viewer` + `items[]`；`items[]` 以 `product` / `edition` / `tier` / `seat_id` 标识身份，
每项含 `periods[]`：

```json
{ "label": "weekly", "used": 12500, "total": 50000, "percent": 25, "reset_at": "2024-06-06T00:26:40+08:00" }
```

### 3.3 Coding Plan 与 Agent Plan 的字段差异

| 字段 | Agent Plan | Coding Plan |
| --- | --- | --- |
| `periods[].label` | `5h` / `weekly` / `monthly`（后端返的 `daily` 不展示） | **`session` / `weekly` / `monthly`**，按 canonical order 稳定排序 |
| `periods[].used` / `total` | 绝对值 | **缺失**：后端只返 `Percent`，`used`/`total` 通过 `omitempty` 不出现 |
| `periods[].percent` | 由 `used/total*100` 计算 | **直接透传后端 `Percent`**（0–100） |
| `periods[].reset_at` | RFC3339 北京时间（UTC+08:00） | 同左；是**额度窗口刷新时间**，不是订阅到期时间 |

官方同时明写：

- **Coding Plan / Coding Plan Team 当前不提供按模型套餐明细**；这类问题只能用
  `usage plan --product coding-plan` 给 quota 快照并说明能力边界。
- **禁止改跑 `usage stats` 后把 platform / Endpoint Token 统计冒充 Coding Plan 套餐明细。**
- 订阅属于账号/子用户，**不由 active profile 类型决定**；platform profile 不等于没有套餐。

## 4. 关键风险（方案阶段必须先回答）

### 4.1 身份错配（最高优先级）

火山在 Cindy 里的上游连接是 **API Key**（`authKind: key`），而 `arkcli` 的用量读取依赖**账号登录态**。
两者是不同的凭证体系，可能指向**不同身份或不同订阅**。

后果：读到的配额可能不是用户当前任务实际消耗的那份订阅。若不解决，展示的用量会误导用户。

必须在方案阶段明确：如何确认 CLI 登录身份与连接身份一致；不一致时如何降级或拒绝展示。

### 4.2 只有百分比，没有绝对量

Coding Plan 后端只给 `percent`。因此：

- 只能表达「已用 X%」与「重置时刻」，**不能表达「还剩多少次」**；
- 与 `provider-settings.md` §4 现有规则相反方向的换算是必须的：该规则要求**统一显示剩余百分比**
  （100 − 已用，限制在 0–100），而 arkcli 给的是已用百分比，展示层必须做这个转换。

### 4.3 依赖外部 CLI

`arkcli` 在当前仓库**零引用**，Cindy 未内置。若最终只能走 CLI，需要回答：谁安装、谁登录、如何发现
命令、如何隔离进程与凭证。这条未回答前，不应进入实现。

## 5. 待实机验证清单

纸面调研无法覆盖、必须在实机确认的项：

1. 当前账号实际返回哪几个 `periods`（`session` / `weekly` / `monthly` 是否总在）。
2. 档位（Lite / Pro）在 `tier` 里如何表达；个人版与团队版的实际差异。
3. 顶层 `viewer` 的内容；未订阅时 `subscribed: false` 那一行的完整形状。
4. `arkcli` 的安装方式、登录方式、登录态存放位置，以及能否被 Cindy 安全复用。
5. 是否存在**不依赖 CLI** 的官方配额 HTTP 接口（若存在可完全绕开 4.3）。
6. 调用频率限制与错误形态（未登录、无订阅、网络失败分别返回什么）。

## 6. 验收条件

本阶段（调研）完成的标准：

1. 依据官方文档与 `ark-cli` skills 文档，确认 `periods[]` 字段结构、周期 label、缺省字段语义与
   `reset_at` 语义 —— **已达成**（见 §3）。
2. 明确区分「官方字段定义已确认」与「真实输出已观测」；未实测的条目全部进入 §5 清单 —— **已达成**。
3. 给出读取通道的可行性结论：**可做 / 不可做 / 需官方支持**，并列出各通道的阻塞点 —— 待方案阶段。

**不构成验收**:「拿到了真实 JSON」。纸面调研拿不到真实输出，不得在文档中声称已跑通。

## 7. 待定事项

| 事项 | 状态 |
| --- | --- |
| 读取通道（内置 CLI / 用户自装 CLI / 官方 HTTP） | 未定 |
| 数据模型（快照 schema 与窗口 id 命名） | 未定，倾向复用 §2.3 形态 |
| 展示位置（设置页 / 新任务输入区 / 两处） | 未定 |
| 端范围（Desktop / Mobile 跟随） | 未定 |
| 身份错配解法（§4.1） | 未定，最高优先级 |
| UI 文案与术语（「火山方舟 Coding Plan」需入 `i18n/GLOSSARY.md`） | 未定 |

## 8. 相关正本

- 供应商用量产品语义：[`provider-settings.md`](../product-rules/provider-settings.md) §4
- 用量缓存回显与隔离：[`model-selector-unified.md`](../product-rules/model-selector-unified.md)
- 用量历史图表：[`usage-history-charts.md`](../design-rules/usage-history-charts.md)
- provider 接口协议审计：[`provider-interface-audit.md`](../dev-rules/provider-interface-audit.md)
- 项目知识归属：[`development-workflow.md`](../dev-rules/development-workflow.md) §4
