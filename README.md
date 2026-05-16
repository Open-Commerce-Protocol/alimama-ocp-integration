# Alimama × OCP Catalog Integration

> 把阿里巴巴**阿里妈妈淘宝联盟**（CPS 联盟营销）包装成 **OCP（Open Commerce Protocol）** 标准协议——让 AI Agent 用统一的 OCP 接口就能发现淘宝商品、拿到带 PID 归因的购买链接、跟踪佣金回流。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Status: PoC Complete](https://img.shields.io/badge/Status-PoC%20Complete-success.svg)](docs/08-完成版汇报与使用文档.md)
[![Day 1-7](https://img.shields.io/badge/Day%201--7-✓%20all%20checkpoints-blue.svg)](docs/06-coding-flow.md)
[![Tests](https://img.shields.io/badge/tests-79%20passing-brightgreen.svg)](#项目状态)

---

## 这是什么

**一个翻译适配服务**（`alimama-provider`），夹在两个互不感知的系统之间：

```text
   AI Agent                  alimama-provider                Alimama API
   ━━━━━━━━━                ━━━━━━━━━━━━━━━━━              ━━━━━━━━━━━━
   说 OCP 协议    ⇄ 翻译 ⇄    同时懂两边        ⇄ 翻译 ⇄   说阿里 top API

   不知道阿里                                                不知道 Agent
```

业务故事一句话：**AI Agent 推荐淘宝商品 → 用户点链接成交 → 阿里联盟把佣金按 PID 记到我们账户**。

技术故事一句话：**OCP-Catalog 仓库改 90 行 + 写一个 ~3000 行的 alimama-provider 服务**。

---

## 项目状态

| 阶段 | 状态 | 凭证 |
| --- | --- | --- |
| 调研、协议设计、合规分析 | ✅ | [01-research](docs/01-research.md) / [02-technical-design](docs/02-technical-design.md) |
| Day 1 — 基础层（types、sign、mapper） | ✅ Checkpoint -- 29 测试 | [06 Day 1](docs/06-coding-flow.md) |
| Day 2 — 适配层（client、catalog-client、registration） | ✅ -- 50 测试 | |
| Day 3 — 服务层 + ⭐ **Checkpoint A** | ✅ query/sync 端到端 | |
| Day 4 — OCP-Catalog 协议扩展（90 行 patch）+ ⭐ **Checkpoint B** | ✅ resolve hook 双向 | [patches/](patches/) |
| Day 5 — 动态转链 + ⭐ **Checkpoint C** | ✅ 3 个 ActionBinding | |
| Day 6 — Cron + 佣金账本 + ⭐ **Checkpoint D** | ✅ 79 测试 | |
| **Day 7 — 真实 Alimama 联调 + ⭐ Checkpoint E** | ✅ **`s.click.taobao.com` 真短链** | [09-执行手册](docs/09-执行手册.md) |

**当前能做的事**：用 OCP 协议查到真实淘宝商品 → 拿到带 PID 的真 affiliate URL → 用户点击直跳手淘 → 佣金按 adzone 归因到我们账户。

---

## 5 分钟跑通

完整复制粘贴见 [docs/09-执行手册.md](docs/09-执行手册.md)。最短版本：

```bash
# 1. 起 Postgres
docker start ocp-catalog-postgres-wsl

# 2. 起 OCP Catalog(:4000)
cd OCP-Catalog && bun run commerce:catalog:api &

# 3. 起 alimama-provider(:4300,mock 模式不需要阿里凭据)
ALIMAMA_MOCK=true \
OCP_CATALOG_BASE_URL=http://localhost:4000 \
OCP_CATALOG_ID=cat_local_dev \
OCP_PROVIDER_ID=alimama_demo \
OCP_API_KEY=dev-api-key \
OCP_PROVIDER_BASE_URL=http://localhost:4300 \
PROVIDER_PORT=4300 \
bun run --cwd apps/examples/alimama-provider-api start &

# 4. 注册 + 同步 + 查询
curl -X POST http://localhost:4300/admin/register -d '{}'
curl -X POST http://localhost:4300/admin/sync \
  -H 'content-type: application/json' \
  -d '{"q":"耳机","pageSize":6}'

# 5. Agent 视角:用 OCP 协议查商品
curl -X POST http://localhost:4000/ocp/query \
  -H 'content-type: application/json' \
  -d '{"query_pack":"ocp.query.keyword.v1","filters":{"provider_id":"alimama_demo"},"limit":10}'

# 6. Resolve 拿带 PID 的 affiliate URL
curl -X POST http://localhost:4000/ocp/resolve \
  -H 'content-type: application/json' \
  -d '{"entry_id":"<上一步的 entry_id>","agent":{"agent_id":"agt_demo"}}'
```

> Windows 用户：必须用 **Git Bash** 不能 PowerShell（curl 别名问题）。Python 解析中文要先 `export PYTHONIOENCODING=utf-8`。

---

## 架构总览

```text
┌───────────────────────────────────────────────────────────────────────┐
│                                                                       │
│   ┌────────────┐                                                      │
│   │  AI Agent  │  query/resolve(OCP 协议)                              │
│   └────────────┘                                                      │
│         │                                                             │
│         ▼                                                             │
│   ┌──────────────────────┐  resolve 时回调                            │
│   │ commerce-catalog-api │ ────────────────┐                          │
│   │  (OCP-Catalog 改 90 行)│                 │                          │
│   └──────────────────────┘                 ▼                          │
│         │ sync(物料 push)            ┌────────────────────────┐       │
│         │                            │ alimama-provider-api ★ │       │
│         ▼                            │ (这是我们写的核心)      │       │
│   ┌────────────┐                     └────────────────────────┘       │
│   │ Postgres   │                              │                       │
│   │ (索引)     │                              │ taobao.tbk.*          │
│   └────────────┘                              ▼                       │
│                                       ┌───────────────────────┐       │
│                                       │ Alimama 阿里妈妈      │       │
│                                       │ gw.api.taobao.com     │       │
│                                       └───────────────────────┘       │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

**核心创新**：Day 4 给 OCP-Catalog 加了 **resolve 时回调 Provider 的机制**，让 catalog 能根据 Agent 身份动态生成 ActionBinding（而不是 sync 时静态预算）。这 90 行让"千 Agent 千归因 PID"成为可能。

详见 [docs/10-工作流程详解.md](docs/10-工作流程详解.md)。

---

## 仓库结构

```text
alimama-ocp-integration/
├── README.md                  ← 你正在看
├── LICENSE                    ← MIT
├── docs/                      ← 11 份文档(下面有导航)
├── patches/                   ← Day 4 对 OCP-Catalog 的 90 行 patch(unified diff)
│   ├── day-4-changes.md       ← 5 个 patch 含设计取舍
│   └── day-4-all-changes.patch ← git apply 可直接打
├── provider/                  ← alimama-provider 代码占位(实现在 OCP-Catalog 仓库)
└── fixtures/                  ← mock 数据占位(实现在 OCP-Catalog 仓库)
```

> **实际运行代码住在 OCP-Catalog 仓库**：`OCP-Catalog/apps/examples/alimama-provider-api/`
> 这个 repo 主要承载设计、决策、补丁、文档。

---

## 文档导航

按你想做的事找：

| 你想做的事 | 读哪份 |
| --- | --- |
| 5 分钟看懂这事是什么 | [00-overview.md](docs/00-overview.md) |
| 对外汇报 / 演示 | [08-完成版汇报与使用文档.md](docs/08-完成版汇报与使用文档.md) |
| 自己复制粘贴跑一遍 | [09-执行手册.md](docs/09-执行手册.md) |
| 在白板讲给别人 | [10-工作流程详解.md](docs/10-工作流程详解.md) |
| 申请阿里联盟账号 | [07-接入申请清单.md](docs/07-接入申请清单.md) |
| 接着写代码 | [06-coding-flow.md](docs/06-coding-flow.md) |
| 看战略 / 合规深度分析 | [01-research.md](docs/01-research.md) |
| 看技术设计 / 协议扩展 | [02-technical-design.md](docs/02-technical-design.md) |
| 看待决策清单 | [04-decisions.md](docs/04-decisions.md) |

---

## 关键技术发现

| 发现 | 影响 |
| --- | --- |
| **`buildSearchProjection` 默认丢弃 `attributes`** | 必须 patch，否则 resolve 时拿不到 hook URL → 这是 Day 4 的隐藏第 5 patch |
| **阿里 top 签名算法** = `md5(secret + sorted_kv + secret)` 大写 | `sign.ts` 实现需要严格按字典序拼接 |
| **图片 URL 常以 `//gw.alicdn.com/...` 形式返回** | mapper 必须绝对化加 `https:` 前缀，否则 OCP schema 校验拒绝 |
| **OCP `ActionBinding.method` 锁死为 GET** | 不能在 binding 里塞 POST 端点，Agent 想 POST 必须走 `attributes` 自定义协议 |
| **PowerShell `curl` 是 `Invoke-WebRequest` 别名** | 整套手册必须 Git Bash 跑，PS 命令需重写 |

完整记录见各 Day 的复盘和 [docs/04-decisions.md](docs/04-decisions.md)。

---

## 它对 OCP 生态的价值

OCP-Catalog 之前只是个**纯检索引擎**，无法跟外部 Provider 做动态交互。这个项目带来三个生态级贡献：

1. **协议扩展提案**：`ResolveRequest.agent`、`Scenario.buildResolveActions` async + `ResolveContext`、`buildSearchProjection` 透传 attributes。这套扩展让 OCP 支持"Agent 身份感知" + "Provider 动态 binding"，是 Agentic Commerce 的核心能力
2. **聚合范式**：alimama-provider 的架构能 1:1 复刻成 jd-union-provider / pdd-ddk-provider，OCP 自然演化成"多联盟桥接层"
3. **完整参考实现**：包括 mock 模式、cron 自动化、佣金账本、降级路径——其他人接其他联盟时可直接 fork 改

---

## 商业语境

参考 [01-research.md §7](docs/01-research.md) — 2026 年全球 Agentic Commerce 协议格局：

- **OpenAI ACP**、**Google UCP**（Shopping Graph 50 亿 SKU）、**Klarna APP**、**PayPal AP2**、**Visa TAP**、**Mastercard Agent Pay** 在打协议战
- 但**中国市场对这些都不可达**——OCP 在中文电商生态有结构性空间
- 阿里妈妈"AI 万相"是**商家侧 AI 工具**，不是 **Agent 侧协议**——这个 Agent ↔ 商家的协议层正是 OCP 要补的位置

---

## 已知限制 / Roadmap

- [ ] `commission_ledger` 当前是内存版，进程重启数据丢失 → 待持久化到 Postgres
- [ ] PoC 阶段所有 Agent 共用同一个 adzone_id → 升级为 per-agent adzone（需阿里联盟流量达标后申请高级权限）
- [ ] `taobao.tbk.privilege.get` 单品转链 → 需阿里联盟"链接 API"邀请制权限
- [ ] OCP 主仓 PR：把 `ResolveContext` + `commission.v1` descriptor pack 提案合并到上游
- [ ] 接第二个联盟（JD Union 或 拼多多多多客）验证多联盟聚合架构

---

## 贡献

issue / PR 欢迎。重大改动前请先在 issue 里讨论。

参与开发的代码规范见 [docs/06-coding-flow.md](docs/06-coding-flow.md)。

---

## License

[MIT](LICENSE)
