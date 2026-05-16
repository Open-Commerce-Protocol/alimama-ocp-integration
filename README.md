# Alimama × OCP Catalog Integration

阿里妈妈推广链接（淘宝联盟 CPS）接入 OCP Catalog 协议的**独立工作区**：调研、技术设计、执行流程、补丁归档、待办清单。

> **当前状态**：PoC 已完成 Day 1-Day 6（mock 模式端到端跑通，79 测试全绿）。
> 实际代码住在 `e:/homework/work/OCP-Catalog/apps/examples/alimama-provider-api/`（OCP-Catalog monorepo workspace）。
> 本目录只放**文档 + 补丁归档 + 设计稿**，不放业务代码。
>
> 下一步等业务方按 [07-接入申请清单.md](docs/07-接入申请清单.md) 拿到真实 AppKey 后进 Day 7。

---

## 第一次来看的话

👉 **要汇报 / 演示 Day 1-7 成果** → 直接读 [docs/08-完成版汇报与使用文档.md](docs/08-完成版汇报与使用文档.md)
👉 **想 10 分钟懂战略意义** → 读 [docs/00-overview.md](docs/00-overview.md)

其他文档都是它们的"展开版"，按需查阅。

## 目录速览

```text
alimama-ocp-integration/
├── README.md                       ← 你正在看(项目入口/状态板)
├── docs/
│   ├── 00-overview.md              ← ★ 汇报版:愿景 + 商业 + 3 周计划(10 分钟读)
│   ├── 01-research.md              ← 调研报告:战略/合规/市场背景(深)
│   ├── 02-technical-design.md      ← 技术设计:能不能/怎么做/改哪里(纯技术,深)
│   ├── 03-workflow.md              ← 真要做的话:一步步怎么走(Day0 到 Week5+)
│   ├── 04-decisions.md             ← 待决策清单,12 项,每项含影响和拍板人
│   ├── 05-implementation.md        ← 实现报告:三方角色 + 代码片段 + 验证命令
│   ├── 06-coding-flow.md           ← ★ 代码实现流程:从 Day 1 Step 1.1 开始写
│   ├── 07-接入申请清单.md           ← ★ 给业务/领导:申请 AppKey 需要做什么
│   └── 08-完成版汇报与使用文档.md   ← ★★ Day 1-7 完成后的汇报+使用文档
├── provider/                       ← 未来 alimama-provider 服务代码会在这里
│   └── README.md
├── patches/                        ← 对 OCP-Catalog 仓库的修改(diff 形式)
│   ├── README.md
│   ├── day-4-changes.md            ← Day 4 协议扩展 patch 详细说明
│   └── day-4-all-changes.patch     ← unified diff,git apply 即可
└── fixtures/                       ← 离线开发用的 mock 响应样本(JSON)
    └── README.md
```

---

## 按角色快速查阅

| 你是 | 读哪 |
| --- | --- |
| **要汇报 Day 1-7 成果** | **[08-完成版汇报与使用文档.md](docs/08-完成版汇报与使用文档.md) ★★** |
| 决策者 / 领导 (战略层) | [00-overview.md](docs/00-overview.md) 一份够了 |
| **业务/运营 (要申请阿里联盟账号)** | **[07-接入申请清单.md](docs/07-接入申请清单.md) ★** |
| 技术评审 | [08-完成版汇报](docs/08-完成版汇报与使用文档.md) + [05-implementation.md](docs/05-implementation.md) |
| **就要开始写代码** | **[06-coding-flow.md](docs/06-coding-flow.md) 从 Day 1 Step 1.1 走** |
| 看完整 4-6 周路线图 | [03-workflow.md](docs/03-workflow.md) |
| 想看完整工程细节 | [02-technical-design.md](docs/02-technical-design.md) |
| 法务 / 合规 | [01-research.md §5](docs/01-research.md) + [07 FAQ](docs/07-接入申请清单.md) |
| 想了解全球格局 | [01-research.md §7](docs/01-research.md) 2026 Agentic Commerce 协议战 |

---

## 一句话结论

**技术 100% 可行**。最小可演示版本（mock 模式）1.5 周可跑通；接通真实 Alimama 再 1 周。最大约束不在技术、在**链接 API 邀请制权限**（需流量证明或走二房东兜底）。

详细见 [docs/02-technical-design.md](docs/02-technical-design.md)。

---

## 与外部项目的关系

| 项目 | 路径 | 关系 |
| --- | --- | --- |
| OCP-Catalog | `e:/homework/work/OCP-Catalog/` | **会被本项目修改**：需在 `packages/catalog-core/`、`packages/ocp-schema/`、`apps/examples/commerce-catalog-api/commerce-scenario.ts` 加 90 行扩展；新增 `apps/examples/alimama-provider-api/` |
| betterfastener | `e:/homework/work/betterfastener/` | 不相关，搁置 |

如果决定执行，最终代码会以 PR 形式合入 OCP-Catalog。本项目（`alimama-ocp-integration/`）持续作为**设计文档 + 待办归档库**存在。

---

## 项目状态

- [x] 初步调研（中英文资源、API 表面、PID 归因模型、合规边界）
- [x] 技术设计（架构、字段映射、协议扩展、改造点）
- [x] 执行 workflow 草案
- [x] 待决策清单
- [x] **PoC 实现 Day 1-Day 6 全部完成**（mock 模式端到端跑通，79 测试通过）
- [x] **OCP-Catalog 协议扩展应用**（Day 4：resolve hook + ResolveContext，已在工作树）
- [x] 自动化层完成（material poller + order poller + commission ledger）
- [x] **业务侧拿到淘宝联盟 AppKey**（35325642 + adzone 116271650208）
- [x] **⭐ Day 7：真实 Alimama 联调完成** — Agent 用 OCP 协议查到真淘宝商品 + 拿到带 PID 的真短链
- [ ] 真机下测试单 + 等月结看后台佣金
- [ ] 把账号切到 deeplumen 公司主体
- [ ] commission_ledger 持久化到 Postgres
- [ ] 接入第一个真实内部 AI Agent

---

## 备注

`e:/homework/work/alimama-ocp-catalog-research.md` 和 `e:/homework/work/alimama-ocp-technical-design.md` 是本项目 docs/ 里两份文档的**原始副本**。现在已经整理进本项目，根目录的两份可以删除（如果你不想留双份）。
