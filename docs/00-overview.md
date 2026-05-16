# 用 OCP Catalog 包阿里妈妈：技术方案

> 这是**汇报版**。10 分钟读完。
> 想看深度分析见 [01-research.md](01-research.md) / [02-technical-design.md](02-technical-design.md) / [03-workflow.md](03-workflow.md)。
> 实际执行状态见 [06-coding-flow.md](06-coding-flow.md);业务申请清单见 [07-接入申请清单.md](07-接入申请清单.md)。

---

## 一句话

**做一个适配服务，把阿里妈妈淘宝联盟的商品搜索接口"包"成 OCP Catalog 协议**——AI Agent 用标准 OCP 协议就能发现淘宝商品；用户点击我们生成的链接成交，我们拿联盟佣金。

技术工作量 **3 周**：第一周本地 mock 跑通、第二周改 OCP-Catalog 加个回调机制、第三周接真实阿里。

## 实际进度（2026-05-15 更新）

```text
✅ Day 1  基础层(types/sign/mapper)         50 测试通过
✅ Day 2  适配层(client/config/registration) 累计 50
✅ Day 3  服务层 + Checkpoint A             query/sync 端到端
✅ Day 4  OCP-Catalog 协议扩展 + Checkpoint B   resolve hook 双向验证
✅ Day 5  真转链 + Checkpoint C             3 个 ActionBinding(view + 券 + 天猫券)
✅ Day 6  cron + ledger + Checkpoint D      自动同步 + 佣金统计
⏳ Day 7  真实 Alimama 联调                  等业务方拿 AppKey,见 07 文档
```

**当前能演示什么**:本地起 catalog + provider,Agent 用 OCP 协议查 fixture 商品 → 看到带 mock affiliate URL 的 ActionBinding → 后台看 ledger 自动累计 4 个订单 / 532.80 元 GMV / 89.21 元预估佣金。

---

## 它怎么工作

```
   ┌──────────────────────────────────────────────────────────────┐
   │                                                              │
   │     AI Agent                                                 │
   │       │  OCP 标准协议:                                       │
   │       │  POST /ocp/query   keyword="无线耳机"                │
   │       │  POST /ocp/resolve entry_id="xxx"                    │
   │       ▼                                                      │
   │   ┌──────────────────────┐                                   │
   │   │  OCP Catalog         │  (现成的,要小改一点)              │
   │   │  /ocp/query          │                                   │
   │   │  /ocp/resolve        │                                   │
   │   │  /ocp/objects/sync   │                                   │
   │   └────────┬─────────────┘                                   │
   │            │ 1. 同步时:接收商品入库                          │
   │            │ 2. resolve 时:回调 provider 拿购买链接           │
   │            ▼                                                  │
   │   ┌──────────────────────┐                                   │
   │   │  alimama-provider    │  ← 我们要写的核心服务            │
   │   │                      │                                   │
   │   │  - 定时拉商品同步     │                                   │
   │   │  - 接收 resolve 回调  │                                   │
   │   │  - 调阿里 API 生成链接 │                                  │
   │   │  - 拉订单存佣金账本   │                                   │
   │   └────────┬─────────────┘                                   │
   │            │ HTTPS                                            │
   │            ▼                                                  │
   │   ┌──────────────────────┐                                   │
   │   │  阿里妈妈 / 淘宝联盟 │  (外部)                            │
   │   │  gw.api.taobao.com   │                                   │
   │   └──────────────────────┘                                   │
   │                                                              │
   └──────────────────────────────────────────────────────────────┘
```

**关键就是中间那个 alimama-provider**：往上对 OCP 协议，往下对阿里妈妈 HTTP API。一句话就是个翻译适配层。

---

## 用到的阿里妈妈接口（聚焦在这 4 个就够）

| API | 干什么 | 权限 |
|---|---|---|
| **`taobao.tbk.dg.material.optional`** | **商品搜索**（关键词、类目、价格、佣金率筛选，分页） | 基础（注册就有） |
| `taobao.tbk.item.info.get` | 商品详情批量查询 | 基础 |
| **`taobao.tbk.privilege.get`** | **生成带我们 PID 的购买链接**（含优惠券） | 邀请制 ⚠️ |
| `taobao.tbk.order.get` | 拉成交订单（统计佣金） | 基础 |

**领导问的"搜索的接口"就是 `taobao.tbk.dg.material.optional`**。返回字段包含：标题、主图、小图、价格（原价 + 折扣价）、优惠券信息、佣金率、销量、店铺名、所在地——这些字段足够直接映射到 OCP 的 `CommercialObject` 标准结构。

> ⚠️ **唯一的卡点**：`privilege.get` 转链接口是邀请制，需要"月日均点击 5 万"或"月日均 alipay 10 万"才能拿到。这个问题有兜底方案，见下文。

---

## OCP Catalog 怎么"包"

### Step 1：把搜索结果同步进 OCP Catalog（同步链路）

定时跑 `material.optional` → 字段映射 → 调 OCP Catalog 的 `/ocp/objects/sync` 入库。

**字段映射**（阿里字段 → OCP 字段）：

```
阿里返回                             OCP CommercialObject
─────────                            ─────────────────────
title                                product.core.v1.title
pict_url + small_images.string[]     product.core.v1.image_urls (绝对化 URL)
shop_title                           product.core.v1.brand
item_url                             product.core.v1.product_url
num_iid                              product.core.v1.sku
zk_final_price (折扣价,字符串)        price.v1.amount (人民币)
reserve_price (原价)                  price.v1.list_amount
commission_rate (基点)                attributes.commission_rate_bp
coupon_info / coupon_*                attributes.coupon (对象)
volume (30天销量)                     attributes.sales_volume_30d
user_type (0淘宝/1天猫)              attributes.platform
```

OCP 标准的三个 descriptor pack 全用得上：`product.core.v1`、`price.v1`、`inventory.v1`（取 `unknown` 因为阿里不给库存数）。

### Step 2：Agent 查询（不用我们写代码）

Agent 调 `POST /ocp/query` 关键词搜索 → OCP Catalog 用现成的检索逻辑（关键词 / filter）从已同步的商品池返结果。**这部分 OCP-Catalog 仓库已经实现好了**，复用即可。

### Step 3：Resolve 时实时转链（核心动作）

Agent 选中某商品 → 调 `POST /ocp/resolve` → OCP Catalog 回调我们 provider → provider 调 `privilege.get` 拿带 PID 的链接 → 通过 OCP `ActionBinding` 返给 Agent。

Agent 拿到的链接形如：`https://s.click.taobao.com/xxxx`，里面嵌了我们的 PID。用户点击 → 跳手淘 App → 下单 → 联盟系统按 PID 把佣金记给我们。

### Step 4：佣金统计（异步，不阻塞主链路）

每日 cron 跑 `taobao.tbk.order.get` 拉订单数据，存进 `commission_ledger` 表。后台能看到每天产生了多少 GMV、估算佣金、确认结算佣金。

---

## 商业逻辑（领导说的"卖出去赚钱"）

简单的链条：

```
用户买 100 元商品
  └─ 商家给阿里联盟 15 元佣金（佣金率商家设的）
       └─ 阿里扣 10% 技术服务费 = 1.5 元
            └─ 剩 13.5 元打到我们的联盟账户（次月 20 号月结）
```

**净佣金 ≈ 订单金额 × 佣金率 × 0.9**。佣金率因品类不同从 3% 到 50% 不等。

**前提**：要有 `privilege.get` 转链权限（否则链接里没我们的 PID，归不了佣金）。这就是为什么链接 API 权限是真正的卡点。

退款扣回 / 维权窗口 / 月结周期等细节见 [01-research.md §4](01-research.md#4-佣金结算与退款的完整生命周期)。

---

## 工程任务（按工作量）

| 任务 | 行数估计 | 位置 |
|---|---|---|
| 写 alimama-provider 服务 | ~1500 行 | OCP-Catalog 仓库 `apps/examples/alimama-provider-api/` |
| OCP-Catalog 加 resolve-hook 回调机制 | ~90 行 | `packages/catalog-core/` + `commerce-scenario.ts` |
| 字段映射 + 单测 | ~300 行 | provider 内部 |
| Mock 模式（脱离真实阿里也能跑） | ~100 行 | provider 内部 |
| **合计** | **~2000 行** | **1.5-2 人周** + 真实联调 1 周 |

详见 [02-technical-design.md §3-4](02-technical-design.md#3-仓库与模块组织)。

---

## 三周 PoC 计划

### Week 1：纯本地 mock 跑通（不依赖任何阿里凭据）

- 造 10-20 个商品的 JSON fixture（手编或网上截一份脱敏的）
- 写 provider 骨架 + 字段映射 + Mock client
- 跑通 Agent → OCP `/ocp/query` → 看到 fixture 商品

**交付**：能演示"AI 调 OCP 标准协议查到淘宝风格商品"，全本地。

### Week 2：OCP Catalog 协议扩展（resolve hook 机制）

- 改 `packages/catalog-core/` 的 Scenario interface：从同步改异步，加 `ResolveContext`
- 改 `commerce-scenario.ts`：检测 attributes 标记，回调 provider 的 `/provider/resolve_hook`
- 改 `ocp-schema`：`ResolveRequest` 加可选 `agent` 字段
- Provider 暴露 `/provider/resolve_hook` POST 端点

**交付**：协议级 demo，Agent resolve 时 catalog 动态从 provider 拿链接。

### Week 3：接真实阿里

- pub.alimama.com 注册 + 媒体备案 + 申请 AppKey（实际只要 1-3 天）
- 后台手动建 1 个 adzone_id
- 把 mock 切真实 `material.optional`
- 链接 API 权限到位：直接用 `privilege.get`
- 链接 API 没拿到：用 `item_url` 原始链接演示链路（不归因）或接 [订单侠](https://www.dingdanxia.com/price) 二房东（按调用量付费）
- 真机测试：点链接 → 跳手淘 → 下个测试单（取消即可）

**交付**：完整的"AI 推荐 → 用户买 → 我们能看到归因"链路。

详细 checklist 见 [03-workflow.md](03-workflow.md)。

---

## 还需要拍板的（精简版）

| 项 | 现在要回答的 |
|---|---|
| 联盟账号主体 | 个人测试 OR deeplumen 公司？PoC 用个人即可，正式要切公司 |
| 链接 API 等不到怎么办 | 接订单侠（¥200-500/月）OR 用裸链不归因（演示用） |
| 谁先接 OCP？ | 哪个 Agent 团队先做 demo？ |

完整 12 项决策见 [04-decisions.md](04-decisions.md)。

---

## 当前状态

- [x] 调研完成（API 表面、字段、合规、商业模型）
- [x] 技术设计完成（架构、改造点、字段映射）
- [x] 执行 workflow 写好
- [ ] **等业务/技术拍板上面三项决策**
- [ ] Week 1 PoC（拍板后立即开工）

---

## 一句话汇报模板（给领导）

> 技术上完全可行。本质是写一个 ~2000 行的适配服务（alimama-provider），把淘宝联盟的搜索 API 包成 OCP Catalog 标准协议，AI Agent 用 OCP 协议就能查淘宝商品。3 周可以做出能演示的版本——Week 1 本地 mock 跑通字段映射、Week 2 改 OCP Catalog 加个 resolve 回调机制（~90 行）、Week 3 接真实阿里接口。商业上每笔成交我们拿佣金（订单金额 × 商家设定的佣金率 × 0.9），但需要拿到阿里的"链接 API"邀请制权限，没到前可以用二房东 API 兜底。**最大的卡点不是技术，是申请权限**。
