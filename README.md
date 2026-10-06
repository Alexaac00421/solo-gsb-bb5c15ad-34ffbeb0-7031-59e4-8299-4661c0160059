# 🌍 跨时区软件发布窗口编排

在多时区环境下编排软件发布：给定**服务依赖、候选发布时间窗、团队禁发时段、维护冻结、回滚耗时与共享资源**，
系统用 Google OR-Tools CP-SAT 求出满足全部约束的可执行排程；无解时返回**阻塞它的约束组合**而不是部分结果。

## 约束语义

| 约束 | 规则 |
|---|---|
| 候选时间窗 | 每个服务的**部署 + 回滚整个占用区间**必须完整落在其某个候选窗内；支持多窗 |
| 依赖先后 | `下游开始 ≥ 上游部署结束 + gap_minutes` |
| 团队禁发 | 仅**部署区间**不得与所属团队禁发时段重叠（回滚允许进入禁发期） |
| 维护冻结 | **占用区间（部署+回滚）**不得与任何冻结重叠 |
| 共享资源 | 同名资源在任意时刻的**占用数（含回滚窗口）**不得超过其容量。容量由请求顶层的 `resource_capacities` 声明（正整数）；**未声明的资源容量为 1**，即同一时间只能被一个服务占用。容量为 2 时两个服务可并行，第三个必须错峰。还可用 `resource_capacity_periods` 为资源声明**临时容量时段**（开始、结束、时区、非负整数容量，左闭右开）：时段内任意时刻占用数不得超过当时容量（**0 表示该时段不可占用**），时段外沿用 `resource_capacities` 基准容量；降容时刻落在既有占用期间同样生效 |
| 夏令时 | 不带偏移量的本地墙上时间若在 DST 切换时**重复**（回拨）或**不存在**（前拨），请求被拒绝（422），并给出 gap 两侧 / 重复两次的绝对时间供修正；带偏移量的绝对时间（如 `...+01:00`、`...Z`）无歧义，直接通过 |
| 无解 | 求解器返回一个足以导致矛盾的约束子集（sufficient assumption core），即“阻塞组合”，放松其中任意一项后可重新求解；**不返回部分排程** |

目标函数：所有服务部署开始时刻之和最小（排程尽量早、结果确定可复现）。

## 排程调整：锁定时刻与参考时刻（本轮新增）

评审中需要“固定住几项已确认发布、再重排其余服务”时，可在 `/api/schedule` 请求体顶层追加
两个**可选**数组（省略或传 `[]` 时行为与旧版本完全一致；`/api/schedule/diff` 两侧请求也各自支持）：

| 字段 | 语义 |
|---|---|
| `locked_starts` | **已确认服务**的部署开始时刻，必须**原时刻部署**。锁定不豁免任何约束：其部署与回滚占用仍须满足候选窗、依赖、团队禁发、维护冻结与资源分时容量。锁定与任一其他约束冲突时，阻塞组合（`blockers`）中会包含 `{"code": "locked_start:<服务ID>", ...}` 以及与之冲突的约束，**不返回部分排程**。 |
| `reference_starts` | **未确认服务**的参考开始时刻，仅作为优化偏好，**不豁免任何约束**（窗外/冻结/资源等约束照常生效，时刻会被排到最近的合法位置）。未给参考时刻的服务只参与最后一层目标。 |

两类锚点的每项写法相同：

```json
{"service": "payments", "start": "2026-11-13T22:00:00Z", "timezone": "UTC"}
```

- `start` 沿用现有 ISO 8601 解析规则：带偏移量（`Z` / `+01:00`）时为绝对时刻；
  不带偏移量时为墙上时间，`timezone` 字段给 IANA 时区（DST 回拨重复/前拨不存在按 422 拒绝，
  错误格式与候选窗完全一致，且给出可直接修正的绝对时刻建议）；
- **`timezone` 可省略**：省略时取该服务自身的 `timezone`（展示时区）。

含参考时刻时，对“给了参考时刻的服务”按以下**分层（字典序）目标**求解，三层依次最优：

1. **偏离参考时刻的服务数最少**（开始时刻与参考完全一致才算不偏离，提前/推迟一秒都算偏离）；
2. 在偏离数最少的前提下，**偏移总时长最少**（各服务绝对偏差秒数之和）；
3. 在前两层都最优的前提下，**沿用原尽早发布目标**（所有服务部署开始时刻之和最小）。

只有三层都在 `max_solve_seconds` 内**证明最优**才返回 `200、"status": "optimal"` 与排程；
任一层限时内未能证明最优则返回 `200、feasible=false、"status": "unknown"`，
`blockers` 为 `solver_unknown`，**不输出排程**（可增大 `max_solve_seconds` 重试）。
不含参考时刻时求解路径与旧版完全一致（单目标尽早，可行但未证最优时返回 `feasible`）。

输入错误（沿用现有 422 `errors` 数组格式，可与其他错误一并返回）：

| code | 触发条件 |
|---|---|
| `unknown_service` | 锚点引用了不存在的服务（`field` 指明 `locked_starts[i]` / `reference_starts[i]`） |
| `duplicate_locked_start` | 同一服务在 `locked_starts` 中重复出现 |
| `duplicate_reference_start` | 同一服务在 `reference_starts` 中重复出现 |
| `anchor_in_both_lists` | 同一服务同时列入两类（已确认与参考互斥） |
| `bad_time` | 时间串无法按 ISO 8601 解析 |
| `dst_ambiguous` / `dst_nonexistent` / `bad_timezone` | 墙上时间 DST 歧义/不存在、时区未知（与候选窗同一套格式与建议） |

## 快速开始（Docker Compose）

```bash
docker compose up --build
```

- 操作页面：http://localhost:8000/
- 健康检查：http://localhost:8000/api/health
- 交互式 API 文档（Swagger）：http://localhost:8000/docs

停止：`docker compose down`。

## 仅用 Docker

```bash
docker build -t release-orchestrator:1.0 .
docker run --rm -p 8000:8000 release-orchestrator:1.0
```

## 本地直接运行（Python 3.11+）

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

> 时区数据：镜像内同时安装了系统 `tzdata` 与 Python `tzdata` 包，保证 `zoneinfo` 可解析全部 IANA 时区
> （`Asia/Shanghai`、`Europe/Berlin`、`America/New_York` 等）。

## HTTP API

### `POST /api/schedule`

请求体：

```json
{
  "services": [
    {
      "id": "payments",
      "name": "支付服务",
      "timezone": "Europe/Berlin",
      "duration_minutes": 45,
      "rollback_minutes": 60,
      "resources": ["payment-db", "k8s-prod-east"],
      "windows": [
        {"start": "2026-11-13T22:00", "end": "2026-11-14T05:00", "timezone": "Europe/Berlin"}
      ]
    }
  ],
  "dependencies": [
    {"upstream": "payments", "downstream": "api-gateway", "gap_minutes": 30}
  ],
  "service_teams": [
    {"service": "payments", "team": "payment-team"}
  ],
  "team_blackouts": [
    {"team": "payment-team", "start": "2026-11-13T23:30", "end": "2026-11-14T00:30", "timezone": "Europe/Berlin"}
  ],
  "freezes": [
    {"label": "周末封网", "start": "2026-11-14T02:00", "end": "2026-11-14T03:00", "timezone": "UTC"}
  ],
  "resource_capacities": {
    "payment-db": 1,
    "k8s-prod-east": 2
  },
  "resource_capacity_periods": [
    {"resource": "k8s-prod-east", "start": "2026-11-14T01:00", "end": "2026-11-14T02:00",
     "timezone": "UTC", "capacity": 0}
  ],
  "locked_starts": [
    {"service": "payments", "start": "2026-11-13T22:00:00+01:00"}
  ],
  "reference_starts": [
    {"service": "api-gateway", "start": "2026-11-13T23:30:00Z"}
  ],
  "max_solve_seconds": 10
}
```

`resource_capacities` 为可选字段：

- 键是资源名，值是该资源的**并行容量（正整数）**。服务 `resources` 中每列出一个资源，
  就在自己的部署 + 回滚占用区间内占用该资源 **1 个名额**；任意时刻同资源占用数不得超过容量。
  容量为 2 时两项发布可并行，第三项必须错峰。
- **未在 `resource_capacities` 中出现的资源按容量 1 处理**；省略整个字段或传 `{}` 时，
  行为与旧版本完全一致（所有共享资源串行）。
- 容量必须为正整数（`0`、负数按 422 拒绝，错误码 `bad_resource_capacity`）；
  同一服务在 `resources` 中重复列出同一资源也按 422 拒绝（`duplicate_resource`）。
- 容量不满足时（如三个服务只能挤在容量 2 的同一时段）仍返回 **200、`feasible=false`**，
  `blockers` 中给出含资源容量约束的阻塞组合，不返回部分排程。

`resource_capacity_periods` 为可选字段（临时限流/扩容），每项是一个容量时段：

- `resource` 为资源名，`start` / `end` / `timezone` 与候选窗写法相同（墙上时间 + 时区，
  或带偏移量的绝对时间），`capacity` 为该时段内的并行容量（**非负整数**）。
- 时段按**左闭右开**处理：占用区间恰在时段起点结束、或恰在时段终点开始的，不受该时段限制。
- 时段内任意时刻，同资源所有服务的部署 + 回滚占用并发数不得超过**当时容量**；
  `capacity: 0` 表示该时段内资源完全不可占用（常用于发布前临时限流）。
  降容时刻落在既有占用期间也须遵守——例如容量在 01:00 降为 1，则 01:00 之后
  仍在占用中的服务也计入这 1 个名额。
- 时段外沿用 `resource_capacities` 的基准容量；未声明基准容量的资源仍为 1。
  时段容量高于基准容量时表示临时扩容，同样生效。
- 同一资源的容量时段**不得重叠**（首尾相接允许），违反按 422 拒绝
  （`capacity_period_overlap`）；容量为负数按 422 拒绝（`bad_capacity_period`）；
  结束不晚于开始、时区未知、本地时间 DST 歧义等与候选窗相同的输入错误格式拒绝。
- 省略该字段或传 `[]` 时，行为与旧版本完全一致。
- 临时容量导致无解时同样返回 **200、`feasible=false`**，`blockers` 中给出对应的
  资源阻塞组合（基准容量阻塞码为 `resource:<资源名>`，临时时段阻塞码为
  `resource:<资源名>:<时段开始时间戳>`），不返回部分排程。

时间串支持两种写法：

- **墙上时间 + 时区字段**：`"2026-11-13T22:00"` + `"timezone": "Europe/Berlin"`，按该时区解释，参与 DST 检查；
- **绝对时间**：`"2026-11-13T21:00:00Z"` 或 `"2026-11-13T22:00:00+01:00"`，无 DST 歧义。

**可行（200）**：

```json
{
  "feasible": true,
  "status": "optimal",
  "schedule": [
    {
      "service_id": "payments",
      "name": "支付服务",
      "deploy_start_utc": "2026-11-13T21:00:00+00:00",
      "deploy_end_utc": "2026-11-13T21:45:00+00:00",
      "occupied_until_utc": "2026-11-13T22:45:00+00:00",
      "local_time": "2026-11-13T22:00:00+01:00",
      "timezone": "Europe/Berlin",
      "window_used": "2026-11-13T21:00:00+00:00 ~ 2026-11-14T04:00:00+00:00 UTC（候选窗1）",
      "resources": ["payment-db"]
    }
  ],
  "blockers": null
}
```

**不可行（200，`feasible=false`）**：`schedule` 为 `null`，`blockers` 为联合导致无解的约束组合：

```json
{
  "feasible": false,
  "status": "infeasible",
  "schedule": null,
  "blockers": [
    {"code": "freeze:周末封网:a", "message": "周末封网（…）期间服务 A 不得部署或处于回滚窗口内"},
    {"code": "dependency:a->b", "message": "依赖顺序：B 必须在 A 部署完成之后才能开始"},
    {"code": "resource:shared-db", "message": "共享资源 shared-db 同时只能被一个服务占用…"}
  ]
}
```

**输入错误（422）**：DST 重复/不存在、未知时区、依赖环、区间非法、引用不存在的服务、候选窗短于部署+回滚占用、资源容量非正（`bad_resource_capacity`）、同一服务重复列出资源（`duplicate_resource`）、容量时段容量为负（`bad_capacity_period`）、同资源容量时段重叠（`capacity_period_overlap`）等：

```json
{
  "errors": [
    {
      "code": "dst_nonexistent",
      "field": "services[x].windows[0].start",
      "message": "…本地时间在夏令时前拨时不存在（该时刻被跳过）…",
      "suggestions": [
        "2027-03-14T06:30:00+00:00  （gap 前，偏移 UTC-05:00）",
        "2027-03-14T07:30:00+00:00  （gap 后，偏移 UTC-04:00）"
      ]
    }
  ]
}
```

`dst_ambiguous`（重复时间）的建议分别给出回拨前后的两个绝对时刻（第一次/第二次）。
这些值可直接粘贴回输入框重新提交。

#### 共享资源并行容量示例（HTTP）

三个服务都要占用 `canary-pool`，声明容量 2：前两项并行，第三项自动错峰一小时。

```bash
curl -X POST http://localhost:8000/api/schedule \
  -H 'Content-Type: application/json' \
  -d '{
    "services": [
      {"id":"canary-a","name":"灰度 A","timezone":"UTC",
       "duration_minutes":30,"rollback_minutes":30,"resources":["canary-pool"],
       "windows":[{"start":"2026-12-01T00:00:00Z","end":"2026-12-01T03:00:00Z","timezone":"UTC"}]},
      {"id":"canary-b","name":"灰度 B","timezone":"UTC",
       "duration_minutes":30,"rollback_minutes":30,"resources":["canary-pool"],
       "windows":[{"start":"2026-12-01T00:00:00Z","end":"2026-12-01T03:00:00Z","timezone":"UTC"}]},
      {"id":"canary-c","name":"灰度 C","timezone":"UTC",
       "duration_minutes":30,"rollback_minutes":30,"resources":["canary-pool"],
       "windows":[{"start":"2026-12-01T00:00:00Z","end":"2026-12-01T03:00:00Z","timezone":"UTC"}]}
    ],
    "resource_capacities": {"canary-pool": 2}
  }'
```

返回（200，节选）：`canary-a` 与 `canary-b` 同在 `00:00` 开始、占用至 `01:00`，
`canary-c` 错峰到 `01:00` 开始、占用至 `02:00` —— 任意时刻池内占用数均为 2，未超容量。
把容量改为 1（或直接省略该字段）时三者串行到 `00:00 / 01:00 / 02:00` 开始；
窗口只够容纳两个占用时则返回 `feasible=false`，`blockers` 含
`{"code": "resource:canary-pool", ...}`。容量写成 `0` 或负数返回 422：

```json
{
  "errors": [
    {
      "code": "bad_resource_capacity",
      "field": "resource_capacities[canary-pool]",
      "message": "资源 canary-pool 的容量必须为正整数（实际: 0）；未声明的资源按容量 1 处理",
      "suggestions": null
    }
  ]
}
```

#### 共享资源临时容量时段示例（HTTP）

发布前对 `canary-pool` 临时限流：`2026-12-01 00:30 ~ 01:30 UTC` 容量降为 0（不可占用）。
两个服务基准容量 2 本可都在 `00:00` 并行，限流后占用区间（各 1 小时）不得与该时段重叠，
双双顺延到 `01:30` 开始：

```bash
curl -X POST http://localhost:8000/api/schedule \
  -H 'Content-Type: application/json' \
  -d '{
    "services": [
      {"id":"canary-a","name":"灰度 A","timezone":"UTC",
       "duration_minutes":30,"rollback_minutes":30,"resources":["canary-pool"],
       "windows":[{"start":"2026-12-01T00:00:00Z","end":"2026-12-01T03:00:00Z","timezone":"UTC"}]},
      {"id":"canary-b","name":"灰度 B","timezone":"UTC",
       "duration_minutes":30,"rollback_minutes":30,"resources":["canary-pool"],
       "windows":[{"start":"2026-12-01T00:00:00Z","end":"2026-12-01T03:00:00Z","timezone":"UTC"}]}
    ],
    "resource_capacities": {"canary-pool": 2},
    "resource_capacity_periods": [
      {"resource":"canary-pool","start":"2026-12-01T00:30:00Z",
       "end":"2026-12-01T01:30:00Z","timezone":"UTC","capacity":0}
    ]
  }'
```

返回（200，节选）：`canary-a` 与 `canary-b` 都在 `2026-12-01T01:30:00+00:00` 开始、
占用至 `02:30` —— 限流时段 `[00:30, 01:30)` 内无任何占用，时段外仍按基准容量 2 并行。

时段按左闭右开处理：若限流改为 `01:00 ~ 02:00` 容量 0，则 `00:00` 开始、`01:00` 结束占用
的服务不受影响（占用恰在时段起点结束）。降容落在既有占用期间也须遵守：基准容量 2、
`01:00 ~ 02:00` 降为 1 时，一个 90 分钟占用从 `00:00` 跨到 `01:30`，第二个服务必须等到
`01:30` 才能开始（`01:00` 之后池内同时只能有 1 个占用）。

限流时段覆盖全部候选窗时返回 `feasible=false`，`blockers` 含临时时段阻塞：

```json
{
  "feasible": false,
  "status": "infeasible",
  "schedule": null,
  "blockers": [
    {"code": "resource:canary-pool:1796083200",
     "message": "共享资源 canary-pool 在 2026-12-01T00:00:00+00:00 ~ 2026-12-01T02:00:00+00:00 UTC 期间临时限流为 0（不可占用）；涉及服务的部署+回滚窗口不得与该时段重叠: 灰度 A"}
  ]
}
```

同一资源的时段重叠、容量为负数均按 422 拒绝：

```json
{
  "errors": [
    {
      "code": "capacity_period_overlap",
      "field": "resource_capacity_periods[canary-pool]",
      "message": "资源 canary-pool 的容量时段重叠：…；同一资源的容量时段不得重叠（首尾相接允许）",
      "suggestions": null
    }
  ]
}
```

#### 排程调整示例：锁定时刻 + 参考时刻（HTTP）

`payments` 已确认在 `02:00 UTC` 发布（锁定）；`api-gateway`、`reports` 尚未确认，
参考时刻都给 `02:00`。三者串行依赖同一资源 `shared-db`（容量 1，每次部署 + 回滚占用 1 小时）。

```bash
curl -X POST http://localhost:8000/api/schedule \
  -H 'Content-Type: application/json' \
  -d '{
    "services": [
      {"id":"payments","name":"支付","timezone":"UTC","duration_minutes":30,"rollback_minutes":30,
       "resources":["shared-db"],
       "windows":[{"start":"2026-12-01T00:00:00Z","end":"2026-12-01T06:00:00Z","timezone":"UTC"}]},
      {"id":"api-gateway","name":"网关","timezone":"UTC","duration_minutes":30,"rollback_minutes":30,
       "resources":["shared-db"],
       "windows":[{"start":"2026-12-01T00:00:00Z","end":"2026-12-01T06:00:00Z","timezone":"UTC"}]},
      {"id":"reports","name":"报表","timezone":"UTC","duration_minutes":30,"rollback_minutes":30,
       "resources":["shared-db"],
       "windows":[{"start":"2026-12-01T00:00:00Z","end":"2026-12-01T06:00:00Z","timezone":"UTC"}]}
    ],
    "locked_starts": [
      {"service":"payments","start":"2026-12-01T02:00:00Z"}
    ],
    "reference_starts": [
      {"service":"api-gateway","start":"2026-12-01T02:00:00Z"},
      {"service":"reports","start":"2026-12-01T02:00:00Z"}
    ]
  }'
```

分层目标下：`payments` 必须在 `02:00`（占用到 `03:00`），把容量 1 的 `shared-db` 在
`02:00~03:00` 占满，因此两个未确认服务**都无法**准点于 `02:00`——最少偏离服务数为 **2**；
第二层把两者各安排到紧贴锁定占用两侧的 `01:00`（提前 1 小时）与 `03:00`（推迟 1 小时），
偏移总时长最小（2 小时；换成 `03:00`/`04:00` 同为 2 个偏离但总偏移更大）；
第三层尽早发布同样在等价格局中选总开始时刻更早的组合：

```json
{
  "feasible": true,
  "status": "optimal",
  "schedule": [
    {"service_id": "api-gateway", "deploy_start_utc": "2026-12-01T01:00:00+00:00", "...": "…"},
    {"service_id": "payments",     "deploy_start_utc": "2026-12-01T02:00:00+00:00", "...": "…"},
    {"service_id": "reports",      "deploy_start_utc": "2026-12-01T03:00:00+00:00", "...": "…"}
  ],
  "blockers": null
}
```

要点：

- **锁定不豁免约束**：把 `payments` 锁到候选窗外、冻结/禁发时段内，或与资源容量冲突时，
  返回 `200、feasible=false`，`blockers` 同时包含 `{"code": "locked_start:payments", ...}`
  与被冲突的约束（如 `resource:shared-db`、`freeze:…`、`dependency:…`），`schedule` 为 `null`。
- **参考不豁免约束**：参考时刻落在窗外/冻结段时不会报错也不会照排，服务被排到偏差最小的合法时刻。
- **墙上时间**：`{"service":"payments","start":"2026-12-01T03:00","timezone":"Asia/Shanghai"}`
  省略 `timezone` 时取服务自身时区；DST 歧义/不存在按 422 拒绝并给出两个绝对时刻修正建议。
- **输入错误（422）**：未知服务（`unknown_service`）、同服务在同一类重复
  （`duplicate_locked_start` / `duplicate_reference_start`）、同时列入两类
  （`anchor_in_both_lists`）、非法时刻（`bad_time`）均沿用 `errors` 数组格式，例如：

```json
{
  "errors": [
    {
      "code": "anchor_in_both_lists",
      "field": "locked_starts[0]",
      "message": "服务 payments 同时列入 locked_starts 与 reference_starts；已确认时刻的服务不能再给参考时刻（两类互斥）",
      "suggestions": null
    }
  ]
}
```

- **超时语义**：仅当存在 `reference_starts` 时需要证明三层字典序目标；任一层在
  `max_solve_seconds` 内未证明最优，返回 `200、feasible=false、status:"unknown"`，
  不输出排程。`/api/schedule/diff` 两侧请求可以分别携带锚点，锚点导致的 422 / 无解 / unknown
  仍只在 `failures` 的对应一侧返回，不输出差异。

### `POST /api/schedule/diff` — 变更影响预演

变更评审时，提交一份**当前排程请求**和一份**修改后的请求**，系统分别独立求解，
再按**服务 ID** 对齐两侧结果，预演变更会让哪些服务的发布时刻发生移动。

请求体：

```json
{
  "current": { ...一份完整的 /api/schedule 请求体... },
  "modified": { ...一份完整的 /api/schedule 请求体... }
}
```

比较规则：

- `changed`：两侧都存在、且**部署开始**或**占用结束**（部署结束 + 回滚窗口）的 UTC 时刻发生变化的服务，
  给出变化前后的 UTC 时刻与变化分钟数（`delta_minutes` 正为推迟、负为提前；未变化的字段为 `null`）；
- `added` / `removed`：仅在 `modified` 中出现（新增）/ 仅在 `current` 中出现（被移除）的服务；
- 两份请求结果完全一致时，三个列表均为空（空差异）。
- 两侧请求**各自携带自己的 `resource_capacities` 与 `resource_capacity_periods` 并独立求解**：
  例如 current 未声明容量（资源串行）、modified 把同一资源声明为容量 2（两项并行），
  或 modified 新增一段临时限流时段（容量 0），差异会体现在被提前/推迟的服务上；
  某一侧容量或容量时段非法（422）、或容量约束导致无解时，同样只在 `failures` 的
  该侧给出完整错误 / 阻塞组合（含资源约束），`diff` 为 `null`，不给部分排程。
- **任一请求输入错误或无解时，不输出差异结果**：在 `failures` 中按 `current` / `modified`
  分别给出该侧的完整输入错误（HTTP 422）或阻塞约束组合（HTTP 200、`feasible=false`），
  与直接调用 `/api/schedule` 得到的内容一致。
- 两侧请求均可携带本轮新增的 `locked_starts` / `reference_starts` 并独立求解；
  某侧锚点引用未知服务、重复或跨两类（422），或锁定时刻与该侧约束冲突（`blockers` 含
  `locked_start:<服务ID>`），或该侧分层目标限时未证明最优（`kind: "unknown"`）时，
  同样只在 `failures` 的对应一侧给出结果，`diff` 为 `null`，不给部分排程。

调用示例：

```bash
curl -X POST http://localhost:8000/api/schedule/diff \
  -H 'Content-Type: application/json' \
  -d '{
    "current": {
      "services": [
        {"id": "payments", "name": "支付服务", "timezone": "UTC",
         "duration_minutes": 60, "rollback_minutes": 30,
         "windows": [{"start": "2026-11-13T00:00:00Z", "end": "2026-11-13T06:00:00Z", "timezone": "UTC"}]}
      ],
      "max_solve_seconds": 10
    },
    "modified": {
      "services": [
        {"id": "payments", "name": "支付服务", "timezone": "UTC",
         "duration_minutes": 60, "rollback_minutes": 30,
         "windows": [{"start": "2026-11-13T00:00:00Z", "end": "2026-11-13T06:00:00Z", "timezone": "UTC"}]}
      ],
      "freezes": [
        {"label": "临时冻结", "start": "2026-11-13T00:00:00Z", "end": "2026-11-13T01:00:00Z", "timezone": "UTC"}
      ],
      "max_solve_seconds": 10
    }
  }'
```

**有变化（200）**：

```json
{
  "feasible": true,
  "diff": {
    "changed": [
      {
        "service_id": "payments",
        "name": "支付服务",
        "deploy_start": {
          "before_utc": "2026-11-13T00:00:00+00:00",
          "after_utc": "2026-11-13T01:00:00+00:00",
          "delta_minutes": 60.0
        },
        "occupied_until": {
          "before_utc": "2026-11-13T01:30:00+00:00",
          "after_utc": "2026-11-13T02:30:00+00:00",
          "delta_minutes": 60.0
        }
      }
    ],
    "added": [],
    "removed": []
  },
  "failures": null
}
```

只调整回滚时长、开始时刻不变时，`deploy_start` 为 `null`，仅 `occupied_until` 给出变化。
两份请求无差异时返回 `"changed": [], "added": [], "removed": []`。

**某一侧输入错误（422）**：不输出 `diff`，在 `failures` 中指明出错的一侧及完整错误列表：

```json
{
  "feasible": false,
  "diff": null,
  "failures": {
    "modified": {
      "kind": "validation",
      "status": null,
      "errors": [
        {"code": "bad_timezone", "field": "services[payments].timezone",
         "message": "服务 payments: 未知时区 'Mars/Olympus'", "suggestions": null}
      ],
      "blockers": null
    }
  }
}
```

**某一侧无解（200，`feasible=false`）**：`kind` 为 `infeasible`（求解器超时未判定时为 `unknown`），
`blockers` 为该侧的阻塞约束组合：

```json
{
  "feasible": false,
  "diff": null,
  "failures": {
    "current": {
      "kind": "infeasible",
      "status": "infeasible",
      "errors": null,
      "blockers": [
        {"code": "freeze:维护冻结：全天封网:payments", "message": "…期间服务 支付服务（payments）不得部署或处于回滚窗口内"}
      ]
    }
  }
}
```

两侧同时失败时 `failures` 中会同时包含 `current` 与 `modified` 两个键。

## 项目结构

```
app/
  main.py        FastAPI 入口（/api/schedule、/api/schedule/diff、/api/health、静态页面）
  models.py      请求/响应 Pydantic 模型
  times.py       时区解析与 DST 重复/不存在分类（zoneinfo fold 规则）
  validator.py   输入校验：DST、引用完整性、依赖环检测、区间合法性
  scheduler.py   CP-SAT 建模、求解、不可行假设核（blocking core）提取
  diff.py        变更影响预演：双请求执行与按服务 ID 的时刻对齐求差
  web/index.html 操作页面（原生 JS，内置 3 个示例场景）
Dockerfile
docker-compose.yml
requirements.txt
```

## 不可行核的工作原理

每条业务约束都挂在一个布尔“强制字面量”上（通过可选区间实现：字面量为假时区间在
`NoOverlap` 中缺席），求解时把全部字面量作为 assumptions 断言为真。
CP-SAT 在判定 INFEASIBLE 时返回 `SufficientAssumptionsForInfeasibility`：
其中任意一个字面量放松（对应约束被修改/移除），其余约束才有可能成立。
因此返回的不是“所有相关约束”，而是一个最小化的、可直接据此谈判的阻塞组合。
