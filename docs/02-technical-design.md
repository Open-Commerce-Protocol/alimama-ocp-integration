# 阿里妈妈 × OCP Catalog 技术结合方案

> 纯技术视角的可行性 + 实施设计。
> 不讨论商业模式、合规边界、资质申请。
> 关联背景文档：[alimama-ocp-catalog-research.md](./alimama-ocp-catalog-research.md)
> 状态：技术设计稿

---

## 0. 技术可行性一句话

**完全可行**。淘宝联盟暴露的是标准 HTTP API（淘宝开放平台 `taobao.tbk.*`），OCP Catalog 暴露的是标准 HTTP API（`/ocp/objects/sync`、`/ocp/query`、`/ocp/resolve`），两者间需要的就是一个**翻译适配服务**。

但有 **一个非平凡的技术 gap** 需要在 OCP-Catalog 侧打开：当前 `commerce-scenario.ts` 的 `buildResolveActions` 只能从静态字段拼装 `view_product`，**无法在 resolve 时回调 Provider 动态生成 ActionBinding**。要支持"每个 Agent 一个 affiliate URL"必须扩展 catalog-core 的 resolve hook 机制。

---

## 1. 技术前提条件

| 项 | 最低需要 | 仅做 PoC 时的替代 |
|---|---|---|
| 淘宝开放平台 AppKey + AppSecret | 申请测试账号即可 | 无替代，必须有 |
| 基础 API 权限（material.optional） | 媒体备案后自动 | 无替代 |
| 链接 API 权限（privilege.get） | 邀请制 / 流量达标 | 用 mock 返回数据 / 二房东 HTTP 接口 |
| 订单 API 权限（order.get） | 基础包通常包含 | 用 mock |
| OCP-Catalog 本地可跑 | `bun install` + Docker Postgres | 已配置好（见 [.env](./OCP-Catalog/.env)） |
| 一个 adzone_id | 联盟后台手动创建 1 个即可 PoC | 无替代 |

**最小 PoC 集合**：基础 API + 1 个 adzone_id，**根本不需要链接 API 权限**——ActionBinding 暂时用未转链的原始 `item_url`，先把整条链路跑通。

---

## 2. 架构总览

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   Agent (用户/上游消费方)                                                    │
│       │                                                                      │
│       │ OCP 协议 (HTTP)                                                      │
│       ▼                                                                      │
│   ┌────────────────────────────────┐                                         │
│   │  commerce-catalog-api  :4000   │  (复用,不动核心)                        │
│   │  /ocp/query                    │                                         │
│   │  /ocp/resolve                  │ ◀── 需扩展:支持 Provider resolve hook  │
│   │  /ocp/objects/sync             │                                         │
│   └────────┬───────────────────────┘                                         │
│            │ sync (push)         │ resolve_hook (pull)                       │
│            │                     │                                           │
│            ▼                     ▼                                           │
│   ┌──────────────────────────────────────────────────────────────────────┐   │
│   │  alimama-provider-api  :4300  (新增,本次设计核心)                   │   │
│   │                                                                      │   │
│   │  cron: materialPoller         POST /provider/resolve_hook            │   │
│   │   └─▶ tbk.dg.material.optional └─▶ tbk.privilege.get (动态转链)     │   │
│   │   └─▶ map → CommercialObject     │                                  │   │
│   │   └─▶ POST /ocp/objects/sync     │                                  │   │
│   │                                  │                                  │   │
│   │  cron: orderPoller               │                                  │   │
│   │   └─▶ tbk.order.get              │                                  │   │
│   │   └─▶ upsert commission_ledger   │                                  │   │
│   │                                  │                                  │   │
│   │  agent_adzone_mapping  ◀─────────┘                                  │   │
│   │  (PostgreSQL table)                                                  │   │
│   └────────┬─────────────────────────────────────────────────────────────┘   │
│            │                                                                 │
│            │ HTTPS, app_key + sign                                          │
│            ▼                                                                 │
│   ┌────────────────────────────────┐                                         │
│   │  Alimama Open Gateway          │   (外部)                                │
│   │  gw.api.taobao.com/router/rest │                                         │
│   └────────────────────────────────┘                                         │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**三大流向**：
1. **物料同步**（异步，定时）：alimama → provider → catalog
2. **查询发现**（同步，Agent 主动）：Agent → catalog（不出 catalog 边界）
3. **解析转链**（同步，Agent 主动）：Agent → catalog → **provider resolve hook** → alimama → catalog → Agent
4. **佣金回流**（异步，定时）：alimama → provider → commission_ledger（不入 catalog，旁路系统）

---

## 3. 仓库与模块组织

### 3.1 仓库位置

把 `alimama-provider` 作为新的 example app 放入 OCP-Catalog monorepo：

```
e:/homework/work/OCP-Catalog/
  apps/
    examples/
      commerce-catalog-api/       (现有,需小改:resolve hook 扩展点)
      commerce-provider-api/      (现有,作为模板参考)
      alimama-provider-api/       ◀── 新建
  packages/
    ocp-schema/                   (现有,可能需新增 commission.v1 schema)
    catalog-core/                 (现有,可能需扩展 resolve hook 机制)
    alimama-sdk/                  ◀── 新建,Alimama API 客户端封装
```

理由：复用 `@ocp-catalog/ocp-schema`、`@ocp-catalog/db`、`@ocp-catalog/config`、`@ocp-catalog/auth-core`；保持架构一致。

### 3.2 `apps/examples/alimama-provider-api/` 内部结构

```
src/
  index.ts                     # 服务入口,起 HTTP server + 注册 cron
  config.ts                    # 读环境变量,校验
  http/
    resolve-hook.ts            # POST /provider/resolve_hook 端点(供 catalog 调用)
    admin.ts                   # 管理端 (可选,展示状态)
  alimama/
    client.ts                  # Alimama API 客户端(签名、调用、错误处理)
    types.ts                   # Alimama API 请求/响应 TS 类型
    sign.ts                    # taobao top sign 算法(md5/hmac-sha256)
  mapper/
    material-to-object.ts      # material.optional 响应 → CommercialObject
    privilege-to-action.ts     # privilege.get 响应 → ActionBinding
    order-to-ledger.ts         # order.get 响应 → commission_ledger rows
  workers/
    material-poller.ts         # 定时拉物料 + 同步到 catalog
    order-poller.ts            # 定时拉订单 + 写 commission_ledger
  services/
    catalog-client.ts          # 调 OCP catalog 的 /ocp/objects/sync
    adzone-allocator.ts        # 给 agent_id 分配/查找 adzone_id
    commission-ledger.ts       # commission_ledger 表的读写
  db/
    schema.ts                  # Drizzle schema:agent_adzone_mapping, commission_ledger
  tests/
    fixtures/                  # 模拟 alimama 响应的 JSON
    mapper.test.ts
    sign.test.ts
```

---

## 4. 必须改动 OCP-Catalog 的部分

这是技术上**最不能省**的一块：当前 OCP-Catalog 的 resolve 是 catalog 内部生成 ActionBinding，没有回调 Provider 的机制。

### 4.1 现状分析

[apps/examples/commerce-catalog-api/src/commerce-scenario.ts](OCP-Catalog/apps/examples/commerce-catalog-api/src/commerce-scenario.ts) 大约第 284–301 行：

```typescript
// 当前实现(伪代码)
buildResolveActions(projection: Record<string, unknown>): ActionBinding[] {
  const productUrl = projection.product_url ?? projection.source_url;
  if (!productUrl) return [];
  return [{
    action_id: 'view_product',
    action_type: 'url',
    label: 'View product',
    url: productUrl,
    method: 'GET',
  }];
}
```

这是**同步函数**，只能基于已索引在 catalog 内的字段拼接，不能回调外部服务，也拿不到当前 resolve 请求的 agent_id。

### 4.2 改造方案

**Option A — Scenario 改成 async + 引入 ResolveContext**（推荐）

修改 [packages/catalog-core/src/scenario.ts](OCP-Catalog/packages/catalog-core/src/scenario.ts) 的 Scenario interface：

```typescript
// 修改前
interface Scenario {
  buildResolveActions?(projection: Record<string, unknown>): ActionBinding[];
}

// 修改后
interface ResolveContext {
  entryId: string;
  catalogEntryRow: CatalogEntry;
  request: ResolveRequest;       // 含 agent 字段(协议扩展)
  fetch: typeof globalThis.fetch; // 注入,便于 mock
}

interface Scenario {
  buildResolveActions?(
    projection: Record<string, unknown>,
    ctx: ResolveContext
  ): Promise<ActionBinding[]> | ActionBinding[];
}
```

新增 commerce scenario 实现一个钩子版本：

```typescript
async buildResolveActions(projection, ctx) {
  const actions: ActionBinding[] = [];

  // 静态 fallback:总是带一个 view_product
  if (projection.product_url) {
    actions.push({
      action_id: 'view_product',
      action_type: 'url',
      url: projection.product_url,
      label: 'View product',
      method: 'GET',
    });
  }

  // 如果 attributes 上标记了 affiliate provider,回调
  if (projection.attributes?.requires_affiliate_resolution) {
    const providerEndpoint = projection.attributes.provider_resolve_hook_url;
    if (!providerEndpoint) return actions;

    try {
      const res = await ctx.fetch(providerEndpoint, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({
          entry_id: ctx.entryId,
          object_id: projection.sku,
          agent_id: ctx.request.agent?.agent_id,
        }),
        signal: AbortSignal.timeout(2000),  // 严格超时
      });
      if (res.ok) {
        const hookResult = await res.json();
        actions.push(...hookResult.action_bindings);
      }
    } catch (err) {
      // 降级:provider 不可达不影响 view_product 返回
      logger.warn('[resolve-hook] failed', { entryId: ctx.entryId, err });
    }
  }

  return actions;
}
```

### 4.3 协议层扩展：Resolve 请求加 `agent` 字段

修改 [packages/ocp-schema/src/index.ts](OCP-Catalog/packages/ocp-schema/src/index.ts) 的 `resolveRequestSchema`：

```typescript
// 现在
const resolveRequestSchema = z.object({
  ocp_version: z.literal('1.0').optional(),
  kind: z.literal('ResolveRequest').optional(),
  catalog_id: z.string().optional(),
  entry_id: z.string().min(1),
});

// 扩展
const resolveRequestSchema = z.object({
  ocp_version: z.literal('1.0').optional(),
  kind: z.literal('ResolveRequest').optional(),
  catalog_id: z.string().optional(),
  entry_id: z.string().min(1),
  agent: z.object({                                  // ◀── 新增,可选
    agent_id: z.string().min(1),
    manifest_url: z.string().url().optional(),
    intent: z.enum([
      'shopping_browse',
      'shopping_compare',
      'shopping_purchase',
      'research_only',
    ]).optional(),
  }).optional(),
});
```

**改动量评估**：
- `packages/ocp-schema/`：~10 行（加 agent 字段）
- `packages/catalog-core/`：~30 行（Scenario interface 改 async + ResolveContext）
- `apps/examples/commerce-catalog-api/src/commerce-scenario.ts`：~50 行（实现回调 + 错误处理）
- 都是**向后兼容**的扩展（agent 可选；旧的同步 buildResolveActions 用 Promise.resolve 包装兼容）

---

## 5. Alimama 侧的具体 API 调用

### 5.1 物料拉取（material.optional）

```typescript
// alimama/client.ts 片段
async listMaterial(opts: {
  q?: string;
  cat?: string;
  pageNo: number;
  pageSize: number;
  sort?: 'tk_rate_des' | 'price_des' | 'total_sales_des';
  hasCoupon?: boolean;
  adzoneId: string;   // PoC 阶段固定一个,后期 per-agent
  startPrice?: number;
  endPrice?: number;
}) {
  return this.callTop('taobao.tbk.dg.material.optional', {
    q: opts.q,
    cat: opts.cat,
    page_no: opts.pageNo,
    page_size: opts.pageSize,
    sort: opts.sort,
    has_coupon: opts.hasCoupon,
    adzone_id: opts.adzoneId,
    start_price: opts.startPrice,
    end_price: opts.endPrice,
  });
}
```

### 5.2 转链（privilege.get）

```typescript
// alimama/client.ts 片段
async generatePrivilegeLink(opts: {
  itemId: string;        // 商品 num_iid
  adzoneId: string;      // ★ per-agent 时换不同 adzone
  externalId?: string;   // 透传 OCP entry_id 或 session_id
  platform?: 1 | 2;      // 1 PC / 2 wireless
  relationId?: string;
}) {
  return this.callTop('taobao.tbk.privilege.get', {
    item_id: opts.itemId,
    adzone_id: opts.adzoneId,
    external_id: opts.externalId,
    platform: opts.platform ?? 2,
    relation_id: opts.relationId,
  });
}
```

### 5.3 订单回流（order.get）

```typescript
// alimama/client.ts 片段
async listOrders(opts: {
  startTime: string;    // 'YYYY-MM-DD HH:mm:ss'
  endTime: string;
  queryType: 'create_time' | 'pay_time' | 'settle_time';
  pageNo: number;
  pageSize: number;
}) {
  return this.callTop('taobao.tbk.order.get', {
    start_time: opts.startTime,
    end_time: opts.endTime,
    query_type: opts.queryType,
    page_no: opts.pageNo,
    page_size: opts.pageSize,
  });
}
```

### 5.4 签名算法（top sign）

淘宝开放平台用 md5 或 hmac-sha256 对参数按字典序拼接后签名：

```typescript
// alimama/sign.ts
import crypto from 'node:crypto';

export function topSign(params: Record<string, string>, appSecret: string): string {
  const sortedKeys = Object.keys(params).sort();
  let raw = appSecret;
  for (const k of sortedKeys) {
    raw += k + params[k];
  }
  raw += appSecret;
  return crypto.createHash('md5').update(raw, 'utf8').digest('hex').toUpperCase();
}
```

**坑**：`sign_method=md5` 是老接口，新接口常要求 `sign_method=hmac-sha256`，要按目标 API 文档选。

---

## 6. 字段映射（technical 视角）

### 6.1 material.optional 响应 → CommercialObject

参见 [alimama-ocp-catalog-research.md §8](./alimama-ocp-catalog-research.md#8-ocp-×-alimama-完整字段映射)。技术要点：

- **`reserve_price` / `zk_final_price` 是字符串**（"129.00"），mapper 要 `parseFloat`，校验 NaN
- **`small_images` 是 `{ string: string[] }` 结构**（top 风格的"包了一层"），mapper 要解包
- **`commission_rate` 是基点**（1550 = 15.5%），不要直接当百分数
- **`pict_url` 经常是无 scheme 的 `//gw.alicdn.com/...`**，mapper 要补 `https:` 前缀，否则 `z.string().url()` 校验过不去
- **`cat` 是数字 ID**，要预先维护一张映射表或调 `taobao.itemcats.get` 翻译成人类可读名

### 6.2 mapper 测试样本

把真实 `material.optional` 响应（去敏）存到 `tests/fixtures/material-optional-sample.json`，mapper 用快照测试。

```typescript
// tests/mapper.test.ts
test('maps material item to CommercialObject', () => {
  const sample = JSON.parse(fs.readFileSync('./fixtures/material-optional-sample.json', 'utf8'));
  const item = sample.tbk_dg_material_optional_response.result_list.map_data[0];
  const obj = mapMaterialToCommercialObject(item, { providerId: 'alimama_test', siteUrl: 'https://example.com' });
  expect(obj.title).toBe(item.title);
  expect(obj.descriptors).toHaveLength(4);  // core + price + inventory + commission
  expect(obj.descriptors[0].data.image_urls.every(u => u.startsWith('https://'))).toBe(true);
});
```

---

## 7. OCP Schema 扩展（commission.v1）

新增一个 descriptor pack。`packages/ocp-schema/src/index.ts` 添加：

```typescript
export const commissionPackSchema = z.object({
  affiliate_provider: z.string().min(1),
  commission_rate_bp: z.number().int().min(0).max(10000),
  estimated_commission_per_unit: z.object({
    currency: z.string().length(3),
    amount: z.number(),
    reliable: z.boolean(),
  }).optional(),
  coupon: z.object({
    info: z.string(),
    starts_at: z.string().datetime().optional(),
    ends_at: z.string().datetime().optional(),
    total_count: z.number().int().optional(),
    remain_count: z.number().int().optional(),
  }).nullable().optional(),
  settlement_policy: z.object({
    type: z.enum(['after_payment', 'after_confirmed_receipt', 'after_refund_window']),
    delay_days_p50: z.number().int().optional(),
    refund_window_days: z.number().int().optional(),
  }).optional(),
  attribution_method: z.enum(['static_pid', 'per_agent_pid', 'per_session_pid']).optional(),
}).strict();

// 在 catalog-core 的 scenario 注册时一并 register
export const COMMERCE_DESCRIPTOR_PACKS = {
  'ocp.commerce.product.core.v1': productCorePackSchema,
  'ocp.commerce.price.v1': pricePackSchema,
  'ocp.commerce.inventory.v1': inventoryPackSchema,
  'ocp.commerce.commission.v1': commissionPackSchema,    // ◀── 新增
};
```

**Catalog 端 ObjectContract 不强制要求**（保持向后兼容）。Provider 想用就发，不发也行。

---

## 8. agent_adzone_mapping 表的访问模式

### 8.1 表结构（Drizzle schema）

```typescript
// db/schema.ts
export const agentAdzoneMapping = pgTable('agent_adzone_mapping', {
  agentId: varchar('agent_id', { length: 200 }).primaryKey(),
  adzoneId: varchar('adzone_id', { length: 50 }).notNull(),
  siteId: varchar('site_id', { length: 50 }).notNull(),
  pid: varchar('pid', { length: 100 }).notNull(),                  // mm_x_y_z
  alimamaAccount: varchar('alimama_account', { length: 50 }).notNull(),
  displayName: varchar('display_name', { length: 200 }),
  status: varchar('status', { length: 20 }).default('active').notNull(),
  createdAt: timestamp('created_at').defaultNow(),
  lastUsedAt: timestamp('last_used_at'),
  metadata: jsonb('metadata').$type<Record<string, unknown>>(),
});

// 索引
export const agentAdzoneMappingIndexes = [
  uniqueIndex('agent_adzone_unique').on(agentAdzoneMapping.adzoneId),
  index('agent_adzone_status').on(agentAdzoneMapping.status),
];
```

### 8.2 分配策略

PoC 阶段：**全部 Agent 共享同一个 adzone_id**（手动在联盟后台创建一个，写死在 env）。

生产阶段：**lazy 创建**——Agent 首次 resolve 触发时，若无 mapping：
- a) 调阿里联盟后台 API 创建 adzone（如果有 API；据查阿里联盟 adzone 创建 API 不公开，可能要做网页端 RPA 或手动）
- b) 维护一个"预创建池"：人工提前在后台批量建一批 adzone 入库，分配时从池里挑空闲的

**adzone 上限**：未查到淘宝联盟单账号上限。技术方案需考虑：单账号上限 N → 多账号池 → 跨账号 adzone 池。但 OCP 协议层不需要感知账号池。

---

## 9. commission_ledger 表

```typescript
export const commissionLedger = pgTable('commission_ledger', {
  id: uuid('id').primaryKey().defaultRandom(),
  // 外部归因
  alimamaTradeId: varchar('alimama_trade_id', { length: 100 }).notNull().unique(),
  alimamaAdzoneId: varchar('alimama_adzone_id', { length: 50 }).notNull(),
  agentId: varchar('agent_id', { length: 200 }),   // join 自 agent_adzone_mapping
  // 商品
  itemId: varchar('item_id', { length: 50 }).notNull(),
  // 金额(单位:分)
  payAmount: integer('pay_amount').notNull(),
  estimatedCommission: integer('estimated_commission').notNull(),
  realCommission: integer('real_commission'),
  // 状态
  orderStatus: varchar('order_status', { length: 30 }).notNull(),
  // taobao 状态:订单付款/订单成交/订单结算/订单失效/订单维权
  // 时间
  createTime: timestamp('create_time'),
  payTime: timestamp('pay_time'),
  earningTime: timestamp('earning_time'),         // 结算时间
  // 关联
  externalId: varchar('external_id', { length: 200 }),  // 透传的 OCP entry_id
  // 元
  rawPayload: jsonb('raw_payload'),
  updatedAt: timestamp('updated_at').defaultNow(),
});
```

**关键属性**：
- `alimamaTradeId` unique → 幂等（同一订单多次拉单 upsert 不重复）
- `orderStatus` 状态机驱动：付款 → 成交 → 结算（或维权）
- 维权扣回时 status='订单维权' 且 realCommission < 0 或归零（按真实 API 行为补充）

---

## 10. 完整 resolve 链路的实现

### 10.1 时序

```
Agent
  ▼ POST /ocp/resolve { entry_id, agent: { agent_id, intent } }
Catalog
  │
  │ 1. 查 catalogEntry 行
  │ 2. 跑 scenario.buildResolveActions(projection, ctx)
  │      │
  │      │ 3. projection.attributes.requires_affiliate_resolution = true
  │      │ 4. 取 attributes.provider_resolve_hook_url
  │      │
  │      └──▶ POST <hook_url>
  │                │
  │           alimama-provider /provider/resolve_hook
  │                │
  │                │ 5. 查 agent_adzone_mapping[agent_id]
  │                │    (无则分配)
  │                │
  │                │ 6. 调 tbk.privilege.get(item_id, adzone_id, external_id=entry_id)
  │                │
  │                │ 7. 拿到 coupon_click_url
  │                │
  │                │ 8. 返 { action_bindings: [...] }
  │                ▼
  │      ◀── action_bindings
  │
  │ 9. 拼装 ResolvableReference
  ▼ 返 Agent
```

### 10.2 alimama-provider resolve_hook 端点实现

```typescript
// http/resolve-hook.ts
app.post('/provider/resolve_hook', async (req, res) => {
  const { entry_id, object_id, agent_id } = req.body;

  // 1. 分配/查找 adzone
  const mapping = await allocateAdzoneForAgent(agent_id ?? 'anonymous');

  // 2. 转链
  let privilege;
  try {
    privilege = await alimama.generatePrivilegeLink({
      itemId: object_id,
      adzoneId: mapping.adzoneId,
      externalId: entry_id,
    });
  } catch (err) {
    return res.status(200).json({ action_bindings: [] });  // 降级:不返,catalog 用 fallback
  }

  // 3. 拼 ActionBinding
  const bindings: ActionBinding[] = [];
  if (privilege.coupon_click_url) {
    bindings.push({
      action_id: 'buy_with_coupon',
      action_type: 'url',
      url: privilege.coupon_click_url,
      label: privilege.coupon_info ? `领券购买 (${privilege.coupon_info})` : '去淘宝购买',
      method: 'GET',
    });
  }
  // 同时生成淘口令(移动端备选)
  try {
    const tpwd = await alimama.createTpwd({ url: privilege.coupon_click_url, text: '推荐好物' });
    bindings.push({
      action_id: 'copy_taobao_code',
      action_type: 'url',
      url: `taobao_code://${tpwd.model.password}`,  // 协议未定义,先用自定义 scheme
      label: '复制淘口令打开手淘',
      method: 'GET',
    });
  } catch { /* swallow */ }

  res.json({ action_bindings: bindings });
});
```

---

## 11. 不需要任何 Alimama 权限就能做的 Mock 模式

为了让本地开发能脱离 Alimama 真实 API 跑通，alimama-provider 要内建 mock 模式。

```typescript
// alimama/client.ts
class AlimamaClient {
  constructor(private cfg: { mock: boolean; appKey?: string; appSecret?: string }) {}

  async listMaterial(opts) {
    if (this.cfg.mock) {
      return mockMaterialResponse(opts);  // 从 fixtures 读
    }
    return this.callTop('taobao.tbk.dg.material.optional', opts);
  }
  // ...
}
```

启用 mock：`.env` 加 `ALIMAMA_MOCK=true`，不配 AppKey 也能跑。

mock 数据来源：
- 用一次真实调用的脱敏响应，存 `tests/fixtures/`
- 或人造 10-20 个紧固件商品

这样**整个 alimama-provider + commerce-catalog-api + Agent demo 链路可以完全在本地不靠 Alimama 跑通**，验证 OCP 侧的协议契约对得齐。

---

## 12. 性能与延迟

### 12.1 关键路径预算

Agent → catalog → provider → alimama → 返回 Agent，端到端要在 **2 秒内**完成（用户体验上限），其中：

- catalog 自身处理：~50ms（DB 查 entry + scenario）
- provider resolve_hook：~80ms（DB 查 mapping）+ ~600ms（alimama API 调用）+ ~20ms（拼 binding）
- 网络往返：~100ms × 2

**alimama 调用是大头**。优化：
- adzone 查询完全走本地表（不应往 alimama 调）
- privilege.get 加 LRU 缓存（key = `${itemId}:${agentId}`，TTL 5 分钟）—— 同一 Agent 反复 resolve 同一商品不重复转链
- Catalog → provider 用 keep-alive 长连接

### 12.2 物料同步的批处理

`tbk.dg.material.optional` 单页最多 100 条，且 QPS 受限（通常 50 QPS 上限，按 app_key 维度）。
- 同步 cron 间隔 ≥ 10 分钟
- 单次同步限制 ≤ 20 页（2000 商品）/ 关键词
- POST /ocp/objects/sync 单次 ≤ 100 个对象（catalog batch 上限），客户端拆批

---

## 13. 错误处理（纯技术，非合规）

| 场景 | 处理 |
|---|---|
| alimama 限流（错误码 isv.access-limit） | 指数退避重试，3 次失败入死信队列 |
| alimama 签名错误（isv.invalid-parameter） | 不重试，告警 |
| alimama 网络超时 | 2 秒严格超时；resolve_hook 走降级返空 binding |
| Catalog 不可达 | sync job 入队列后续重试；与 betterfastener 设计中的 outbox pattern 一致 |
| privilege.get 偶发返回 coupon_click_url 为空 | fallback 到 item_url（不带 PID） |
| agent_adzone_mapping 表损坏 / 主键冲突 | 用 ON CONFLICT DO UPDATE last_used_at；不阻塞 resolve |
| commission_ledger upsert 冲突 | 用 alimama_trade_id 唯一约束 + ON CONFLICT；保留最新状态 |

---

## 14. 开发与验证路径

### 14.1 阶段一：纯本地 mock 走通（1-2 天）

- [ ] 新建 `apps/examples/alimama-provider-api/`
- [ ] 实现 `alimama/client.ts` 的 mock 模式
- [ ] 实现 `mapper/material-to-object.ts`
- [ ] 实现 `workers/material-poller.ts`（从 mock 拉 → sync 到本地 catalog）
- [ ] 启动 commerce-catalog-api + alimama-provider-api，用 mock 数据跑同步
- [ ] curl `POST /ocp/query keyword="耳机"` 能查到 mock 商品

**验证**：所有 OCP 字段映射对、batch sync 工作、CommercialObject 通过 catalog 校验。

### 14.2 阶段二：catalog resolve_hook 扩展（2-3 天）

- [ ] 改 `packages/ocp-schema/` 加 `agent` 字段到 ResolveRequest
- [ ] 改 `packages/catalog-core/` 的 Scenario interface 改 async + ResolveContext
- [ ] 改 `commerce-scenario.ts` 实现 hook 回调
- [ ] 在 alimama-provider 加 `/provider/resolve_hook`
- [ ] mapper 输出的 CommercialObject 加 `attributes.provider_resolve_hook_url`
- [ ] curl `POST /ocp/resolve { entry_id, agent: { agent_id } }` 看到 hook 调用 + 自定义 action_binding

**验证**：catalog 能正确回调 provider，provider 返的 binding 出现在 resolve 响应中；超时/降级路径正确。

### 14.3 阶段三：真实 Alimama 接入（接通官方测试号后）

- [ ] AlimamaClient mock 关闭，配真实 AppKey/AppSecret
- [ ] 跑 material.optional 真实调用，看返回结构是否与 fixture 一致（不一致就更新 mapper）
- [ ] 配一个真实 adzone_id，跑通真实 sync
- [ ] **链接 API 不开放时**：resolve_hook 跳过 privilege.get，直接用 item_url 拼 binding（验证 fallback）
- [ ] 链接 API 开放后：跑通转链 + 复制淘口令成功唤起手淘

**验证**：真实链路端到端。

### 14.4 阶段四：commission_ledger 回流（链接 API 拿到后）

- [ ] 实现 `workers/order-poller.ts`
- [ ] 配置一个真实 adzone 测试下单（拿小金额）
- [ ] 看订单数据出现在 ledger 表
- [ ] join agent_adzone_mapping 回查到 agent_id

### 14.5 阶段五：协议层扩展 commission.v1（与上面并行）

- [ ] 在 ocp-schema 注册 `ocp.commerce.commission.v1`
- [ ] mapper 加生成 commission descriptor
- [ ] catalog 校验通过、入库
- [ ] query/resolve 响应里能看到 commission 字段

---

## 15. 验证 checklist（每阶段必过）

### 协议契约
- [ ] CommercialObject 通过 OCP schema 校验
- [ ] ResolveRequest 含 agent 字段不破坏老 client
- [ ] action_bindings 的 url 都是合法 URL
- [ ] image_urls 全部 https://

### 数据正确性
- [ ] commission_rate_bp 与 alimama 返回一致
- [ ] price.amount 单位（元 vs 分）一致
- [ ] 同一商品同步两次不产生重复 catalog entry

### 性能
- [ ] resolve P95 < 2s（含 alimama 调用）
- [ ] 1000 商品 batch sync < 30s
- [ ] alimama 调用错误率 < 1%

### 降级
- [ ] alimama 完全断网：resolve 仍能返回 view_product
- [ ] catalog 断网：sync 入队列，不丢失
- [ ] adzone 表损坏：resolve 用默认 adzone

---

## 16. 不在本设计范围

- 申请 AppKey / 媒体备案 / 高级权限 → 业务/合规流程
- Agent 分润 / 商业模式 → 业务决策
- 多联盟（京东、拼多多）桥接 → 用同样架构复制，独立 design doc
- AI 万相对接 / 阿里妈妈商业合作 → 商务层
- 真实生产部署 / 高可用 → SRE 范畴

---

## 附录 A：最小 PoC 一键脚本（设想）

```bash
# 假设 OCP-Catalog Postgres 已通过 docker compose 起来
cd e:/homework/work/OCP-Catalog

# 1. 启动 catalog
bun run commerce:catalog:api &

# 2. 启动 alimama-provider(mock 模式)
ALIMAMA_MOCK=true \
OCP_CATALOG_BASE_URL=http://localhost:4000 \
OCP_PROVIDER_ID=alimama_local \
OCP_API_KEY=dev-api-key \
PROVIDER_PORT=4300 \
bun run --cwd apps/examples/alimama-provider-api start &

# 3. 触发一次同步
curl -X POST http://localhost:4300/admin/trigger-sync \
  -H "content-type: application/json" \
  -d '{"q":"无线耳机","pageSize":20}'

# 4. 查询
curl -X POST http://localhost:4000/ocp/query \
  -H "content-type: application/json" \
  -d '{"query_pack":"ocp.query.keyword.v1","query":"耳机","limit":5}'

# 5. 解析
curl -X POST http://localhost:4000/ocp/resolve \
  -H "content-type: application/json" \
  -d '{"entry_id":"<上一步返回的 entry_id>","agent":{"agent_id":"agt_test"}}'
```

预期：步骤 5 返回的 action_bindings 里看到 alimama 来源的 binding（mock URL 也行）。

---

## 附录 B：当前依然不确定、需要验证的技术点

1. **`taobao.tbk.privilege.get` 的 `external_id` 字段是否真的会出现在订单回执里**？如果会，那 OCP entry_id 可以完美追踪订单源头。需要做一次真实下单验证
2. **淘宝联盟单账号 adzone 数量上限** —— 如果 N 万个，per-agent 没问题；如果只有几百，要早做账号池
3. **adzone 创建是否有 API** —— 看似要走联盟后台 UI，没法自动化。可能要用 RPA 或人工预创建池
4. **`taobao.tbk.dg.material.optional` 的 QPS 限制和单账号每日调用量上限** —— 影响物料同步频率设计
5. **`coupon_click_url` 的 TTL** —— 是永久有效还是会过期？影响 ActionBinding 的 expires_at 字段填值
6. **淘口令的 expires_at** —— 默认多久？官方文档说可配，要确认

这些都不阻塞协议设计，但影响生产配置和监控阈值。

---

## 总结

**技术上 100% 可结合**。唯一的非平凡技术工作量是 OCP-Catalog 侧 resolve hook 的扩展（约 90 行代码，向后兼容）。其余都是常规适配器开发：

- OCP-Catalog 改动：~90 行（schema + scenario + ResolveContext）
- alimama-provider 新建：~1500-2000 行（含 mock、tests、worker）
- DB schema 新增 2 张表：~30 行
- 一份新 descriptor pack：~15 行

**总技术成本**：1.5-2 人周（不含真实 Alimama 联调）；接通真实 Alimama 后再加 1 周。
