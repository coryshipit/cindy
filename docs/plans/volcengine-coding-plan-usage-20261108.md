# 火山方舟 Coding Plan 用量显示：需求与纸面调研结论

> 状态：需求定义 + 纸面调研结论，**尚未进入实现**。本文不含实现方案、数据模型终稿或 UI 设计。
> 核对日期：2026-11-08。Cindy 源码基线：`feadb203dee25ff5ca7ca4051642a413a832c234`。
> 本版补入鉴权模型与可行性结论，结论见 §6。
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
读取日期 2026-11-08。用到的官方文件：

- `skills/arkcli-usage/SKILL.md`、`skills/arkcli-usage/references/arkcli-usage-plan.md`（命令与输出字段）
- `skills/arkcli-auth/SKILL.md`、`skills/arkcli-auth/references/auth-modes.md`（认证模型）
- `skills/arkcli-docs/SKILL.md`（公开 OpenAPI 契约目录）
- `skills/arkcli-api-explorer/SKILL.md`（本机 Action 与公开目录的分工）
- `README.md`（安装方式）

该仓库是**分发仓库**（含 README、skills、dsh-plugins），不含 CLI 源码，因此字段来自官方 skills 文档，
而非源码类型定义。

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

### 3.4 鉴权前提与调用面

用量查询属于**控制面**，与模型调用的**数据面**使用完全不同的凭证，这是本需求最关键的前提。

| 平面 | 凭证 | 覆盖范围 |
| --- | --- | --- |
| 数据面 | **ARK API Key**（`Authorization: Bearer`） | `+chat` / `+gen` / `+understand` / embeddings 等模型调用 |
| **控制面（含 usage）** | **SSO 派生的 STS**（V4 自签 + `X-Security-Token`），或长效 AK/SK | 列模型、endpoint、API Key、`usage` 等查询 |

依据 `skills/arkcli-auth/references/auth-modes.md`，其中明确把 `usage` 列为控制面接口。由此：

- **API Key 不能用于查用量**——这是两套凭证，不是同一把 Key 权限不足的问题。
- `auth login` 的 AK/SK 通道在 0.1.16 已**关闭**，SSO 是唯一登录入口；STS 过期需 refresh，
  refresh 失败要重登。
- `auth_method="sts"` 是合法登录态，可来自 `source=ve`（借用本机 `volcengine-cli` 登录态）、
  `init-volc`、云开发机或 CI 注入 `VOLC_INIT_*`。这是潜在的免交互路径，但有额外前置条件。
- `arkcli auth apikey` 只是「选一把 API Key 写入凭证存储」，属数据面准备动作，**不能把它
  当作已获得 usage 权限**。

调用面（见 `arkcli-usage-plan.md`「调用面对齐」）：

| 产品 | 接口 |
| --- | --- |
| Coding Plan 个人版 | `/open/GetCodingPlanUsage`（OpenAPI） |
| Coding Plan 团队版 | `/open/GetSeatInfoUsage`（OpenAPI） |
| Agent Plan 个人版 / 团队版 | `/open/GetAFPUsage` / `/open/GetSeatAFPUsage` |

arkcli 走公网 OpenTOP 入口。火山另有公开 OpenAPI 契约目录，可用 `arkcli docs apis list` /
`docs apis spec --id` 读取（见 `skills/arkcli-docs`）。**该目录是否收录 `GetCodingPlanUsage`、
其 spec 与鉴权要求为何，只能由实机命令确认，纸面未知。**

## 4. 关键风险（方案阶段必须先回答）

### 4.1 SSO 登录态是独立凭证（最高优先级）

用量查询需要一份 **SSO 派生的 STS 登录态**，它与 Cindy 里配置的 ARK API Key 是两套凭证（见 §3.4）。
因此：

- 用户即便已经在 Cindy 里配置了可用的火山 API Key，**仍然无法读到用量**；
- 要读用量，用户必须额外完成一次 SSO 授权（浏览器登录），并接受 STS 需 refresh 维持；
- 登录的 SSO 身份与 Cindy 连接所用的 Key 可能属于**不同账号或不同订阅**，读到的配额未必是当前
  任务实际消耗的那份。

必须在方案阶段明确：如何让用户理解并完成这次额外授权；多个身份并存时如何确认一致性；
不一致时如何降级或拒绝展示。这条是产品决策点，不是技术难点。

### 4.2 只有百分比，没有绝对量

Coding Plan 后端只给 `percent`。因此：

- 只能表达「已用 X%」与「重置时刻」，**不能表达「还剩多少次」**；
- 与 `provider-settings.md` §4 现有规则相反方向的换算是必须的：该规则要求**统一显示剩余百分比**
  （100 − 已用，限制在 0–100），而 arkcli 给的是已用百分比，展示层必须做这个转换。

### 4.3 两条通道及其成本

`arkcli` 在当前仓库**零引用**，Cindy 未内置。STS 门槛（§4.1）在两条通道上都存在，区别在谁承担
实现成本：

| 通道 | 判定 | 成本与阻塞 |
| --- | --- | --- |
| **纯 API Key 直连** | ❌ 不可行 | usage 属控制面，API Key 只覆盖数据面（§3.4） |
| **arkcli CLI**（npm 全局包） | ⚠️ 可行 | 用户需装 `@volcengine/ark-cli` 并做一次 SSO 登录；STS refresh 由 arkcli 负责。Cindy 侧要处理命令发现、进程隔离、凭证不进 renderer |
| **Cindy 直连 OpenAPI** | ⚠️ 可行但成本最高 | Cindy 需自实现 V4 签名 + STS 获取与刷新；且 `GetCodingPlanUsage` 的公开 spec 尚未证实 |

两条可行通道都需要用户做一次 SSO 授权，**没有能仅凭现有 API Key 完成的路径**。

## 5. 证据分级

### 5.1 纸面已确认（依据官方 skills 文档）

1. Coding Plan 的 `periods[].label` 为 `session` / `weekly` / `monthly`（§3.3）。
2. Coding Plan 只有 `percent`，无 `used` / `total`，另有 `updated_at`（§3.3）。
3. `reset_at` 为 RFC3339 北京时间，语义是窗口刷新而非订阅到期（§3.3）。
4. Coding Plan **通常没有 `tier` 字段**（§3.3）。
5. 未订阅形态为 `subscribed: false` 且 `periods` 为空（`arkcli-usage-plan.md`「错误处理」）。
6. 错误按 `items[].error` 分桶隔离，一桶失败不影响其它桶（同上）。
7. 用量属控制面，需 SSO/STS；API Key 只用于数据面（§3.4）。
8. Coding Plan 个人版 / 团队版的调用面分别为 `/open/GetCodingPlanUsage` / `/open/GetSeatInfoUsage`（§3.4）。

### 5.2 仍需实机确认

1. 当前账号实际返回哪几个 `periods`，三档是否总在。
2. `GetCodingPlanUsage` 是否收录在公开 OpenAPI 目录，其 spec 与鉴权要求。
3. `arkcli` 安装后的实际行为：postinstall 钩子、Skills 注入可否关闭（`ARKCLI_SKIP_SKILLS=1` /
   `ARKCLI_SKIP_POSTINSTALL=1`）、包体积。
4. 登录态存放位置与生命周期，以及 Cindy 能否在不接触用户凭证的前提下复用。
5. 调用频率限制，以及未登录 / 无订阅 / 网络失败的实际返回。
6. SSO 登录在 Cindy 内的可行交互形态（系统浏览器 / 设备码 `--no-browser`）。

## 6. 验收结论

**可做，前置条件是引入 SSO 登录态。**

| 通道 | 判定 |
| --- | --- |
| 纯 API Key | 不可行 |
| arkcli CLI | 可行 |
| Cindy 直连 OpenAPI | 可行但成本最高，spec 未证实 |

阻塞点按优先级：

1. **SSO 登录态必须由用户额外授权**，无法复用现有 API Key（§4.1）。这是产品决策点。
2. `GetCodingPlanUsage` 公开 spec 未证实，决定直连方案能否成立（§5.2）。
3. 落地形态（内置 CLI / 用户自装）需在方案阶段定，涉及打包、版本与登录交互。

本阶段达成情况：

1. 确认 `periods[]` 字段结构、周期 label、缺省字段语义与 `reset_at` 语义 —— **已达成**（§3）。
2. 区分「官方定义已确认」与「真实输出已观测」 —— **已达成**（§5 分两级）。
3. 给出可行性结论与阻塞点 —— **已达成**（本节）。

**不构成验收**：拿到了真实 JSON。纸面调研无法取得真实输出，不得声称已跑通。

## 7. 待定事项

| 事项 | 状态 |
| --- | --- |
| 读取通道 | 不可行项已排除，剩两条可行通道待选（§4.3） |
| 数据模型（快照 schema 与窗口 id 命名） | 未定，倾向复用 §2.3 形态 |
| 展示位置（设置页 / 新任务输入区 / 两处） | 未定 |
| 端范围（Desktop / Mobile 跟随） | 未定 |
| SSO 授权形态与身份一致性校验（§4.1） | 未定，最高优先级 |
| UI 文案与术语（「火山方舟 Coding Plan」需入 `i18n/GLOSSARY.md`） | 未定 |

## 8. 相关正本

- 供应商用量产品语义：[`provider-settings.md`](../product-rules/provider-settings.md) §4
- 用量缓存回显与隔离：[`model-selector-unified.md`](../product-rules/model-selector-unified.md)
- 用量历史图表：[`usage-history-charts.md`](../design-rules/usage-history-charts.md)
- provider 接口协议审计：[`provider-interface-audit.md`](../dev-rules/provider-interface-audit.md)
- 项目知识归属：[`development-workflow.md`](../dev-rules/development-workflow.md) §4
