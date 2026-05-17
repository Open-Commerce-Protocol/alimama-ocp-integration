# 实现技术报告：用 OCP Catalog 包阿里妈妈

> 实现导向版。读完知道**具体要写哪些代码、改哪些文件、怎么跑通**。
> 想看战略/调研背景见 [00-overview.md](00-overview.md) / [01-research.md](01-research.md)。

---

## 三方角色（实现侧）

这件事本质是一个**三层结构**，写代码前先把谁干什么钉死：

| 层 | 系统 | 实现状态 | 实现侧的责任 |
| --- | --- | --- | --- |
| **协议外壳** | OCP Catalog (`commerce-catalog-api`) | ✅ 现成 | 接受 Agent HTTP 请求；索引商品；做检索；resolve 时**回调 Provider** |
| **翻译适配** | `alimama-provider` | ❌ **要新建** | 拉阿里 API 翻译成 OCP；接 Catalog 回调；翻译 OCP `resolve` 到阿里 `privilege.get` |
| **数据源** | 阿里妈妈 / 淘宝联盟 | ✅ 现成（外部） | 提供 `material.optional` / `privilege.get` / `order.get` |

**OCP Catalog 不应该**：直接调阿里 API、知道 PID/佣金概念。
**alimama-provider 不应该**：做关键词检索、存索引（这是 Catalog 的活）。
**阿里妈妈对我们透明**：只是个 HTTP 端点，它不知道 OCP 存在。

---

## 要新建 / 修改的文件清单

### 新建（在 [`apps/examples/alimama-provider-api/`](https://github.com/Open-Commerce-Protocol/OCP-Catalog/tree/main/apps/examples/alimama-provider-api)）

```text
alimama-provider-api/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts                       # 服务入口
│   ├── config.ts                      # env 读取
│   ├── alimama/
│   │   ├── client.ts                  # 阿里 HTTP 客户端 + mock 模式
│   │   ├── sign.ts                    # top md5 签名
│   │   └── types.ts                   # 阿里 API 类型
│   ├── mapper/
│   │   └── material-to-object.ts      # 阿里字段 → OCP CommercialObject
│   ├── http/
│   │   ├── resolve-hook.ts            # POST /provider/resolve_hook(供 Catalog 调用)
│   │   └── admin.ts                   # POST /admin/sync(手动触发同步)
│   ├── workers/
│   │   └── material-poller.ts         # 定时拉物料
│   └── services/
│       └── catalog-client.ts          # 调 OCP /ocp/objects/sync
└── tests/
    ├── fixtures/
    │   ├── material-optional.json
    │   └── privilege-get.json
    └── mapper.test.ts
```

预估 1500 行。

### 修改 OCP-Catalog（90 行总改动，向后兼容）

| 文件 | 改动 | 行数 |
| --- | --- | --- |
| `packages/ocp-schema/src/index.ts` | `resolveRequestSchema` 加可选 `agent` 字段 | ~10 |
| `packages/catalog-core/src/scenario.ts` | `buildResolveActions` 改 async + 新增 `ResolveContext` | ~30 |
| `packages/catalog-core/src/resolve-service.ts` | 调用处 await + 注入 ctx | ~20 |
| `apps/examples/commerce-catalog-api/src/commerce-scenario.ts` | 实现 hook 回调逻辑 | ~30 |

---

## 关键实现片段

### 1. alimama-provider 入口

```typescript
// src/index.ts
import { Hono } from 'hono';
import { config } from './config';
import { resolveHookRouter } from './http/resolve-hook';
import { adminRouter } from './http/admin';
import { startMaterialPoller } from './workers/material-poller';

const app = new Hono();
app.route('/provider/resolve_hook', resolveHookRouter);
app.route('/admin', adminRouter);
app.get('/health', (c) => c.json({ ok: true }));

if (config.OCP_AUTO_SYNC) {
  startMaterialPoller(config);
}

console.log(`[alimama-provider] :${config.PORT}, mock=${config.ALIMAMA_MOCK}`);
export default { fetch: app.fetch, port: config.PORT };
```

### 2. Alimama HTTP 客户端（含 mock 模式）

```typescript
// src/alimama/client.ts
export class AlimamaClient {
  constructor(private cfg: { appKey: string; appSecret: string; mock: boolean }) {}

  async listMaterial(opts: { q?: string; pageNo: number; pageSize: number; adzoneId: string }) {
    if (this.cfg.mock) return loadFixture('material-optional.json');
    return this.callTop('taobao.tbk.dg.material.optional', {
      q: opts.q,
      page_no: opts.pageNo,
      page_size: opts.pageSize,
      adzone_id: opts.adzoneId,
    });
  }

  async generatePrivilegeLink(opts: { itemId: string; adzoneId: string; externalId?: string }) {
    if (this.cfg.mock) return loadFixture('privilege-get.json');
    return this.callTop('taobao.tbk.privilege.get', {
      item_id: opts.itemId,
      adzone_id: opts.adzoneId,
      external_id: opts.externalId,
    });
  }

  private async callTop(method: string, params: Record<string, any>) {
    const sysParams = {
      method, app_key: this.cfg.appKey, v: '2.0', format: 'json',
      sign_method: 'md5',
      timestamp: new Date().toISOString().replace('T', ' ').slice(0, 19),
    };
    const all = { ...sysParams, ...params };
    all.sign = topSign(all, this.cfg.appSecret);
    const res = await fetch('https://gw.api.taobao.com/router/rest', {
      method: 'POST',
      headers: { 'content-type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams(all).toString(),
    });
    return res.json();
  }
}
```

### 3. 字段映射（阿里 → OCP）

```typescript
// src/mapper/material-to-object.ts
export function mapMaterialToCommercialObject(m: AlimamaMaterial, ctx: MapperCtx): CommercialObject {
  return {
    ocp_version: '1.0',
    kind: 'CommercialObject',
    id: `obj_${ctx.providerId}_${m.num_iid}`,
    object_id: String(m.num_iid),
    object_type: 'product',
    provider_id: ctx.providerId,
    title: m.title,
    status: 'active',
    source_url: absolutize(m.item_url),
    descriptors: [
      {
        pack_id: 'ocp.commerce.product.core.v1',
        data: {
          title: m.title,
          brand: m.shop_title,
          category: String(m.category_id ?? m.cat ?? ''),
          sku: String(m.num_iid),
          product_url: absolutize(m.item_url),
          image_urls: [m.pict_url, ...(m.small_images?.string ?? [])].map(absolutize),
          attributes: {
            platform: m.user_type === 1 ? 'tmall' : 'taobao',
            sales_volume_30d: m.volume,
            commission_rate_bp: m.commission_rate,
            coupon: m.coupon_info ? {
              info: m.coupon_info,
              ends_at: m.coupon_end_time,
              remain_count: m.coupon_remain_count,
            } : null,
            // ★ 让 catalog 知道 resolve 时要回调谁
            requires_affiliate_resolution: true,
            provider_resolve_hook_url: `${ctx.providerBaseUrl}/provider/resolve_hook`,
            affiliate_provider: 'alimama_taobao_union',
          },
        },
      },
      {
        pack_id: 'ocp.commerce.price.v1',
        data: {
          currency: 'CNY',
          amount: parseFloat(m.zk_final_price),
          list_amount: parseFloat(m.reserve_price),
          price_type: 'fixed',
        },
      },
      {
        pack_id: 'ocp.commerce.inventory.v1',
        data: { availability_status: 'unknown' },
      },
    ],
  };
}

function absolutize(url: string): string {
  if (!url) return url;
  if (url.startsWith('//')) return 'https:' + url;
  if (url.startsWith('http')) return url;
  return 'https://' + url;
}
```

### 4. Resolve 回调端点

```typescript
// src/http/resolve-hook.ts
export const resolveHookRouter = new Hono();

resolveHookRouter.post('/', async (c) => {
  const { entry_id, object_id, agent_id } = await c.req.json();

  // PoC 阶段所有 Agent 共用同一个 adzone(env 配)
  const adzoneId = config.ALIMAMA_ADZONE_ID;

  try {
    const result = await alimama.generatePrivilegeLink({
      itemId: object_id,
      adzoneId,
      externalId: entry_id,  // 透传,用于订单归因
    });

    const data = result.tbk_privilege_get_response?.result?.data;
    const url = data?.coupon_click_url ?? data?.item_url;
    if (!url) return c.json({ action_bindings: [] });

    return c.json({
      action_bindings: [{
        action_id: 'buy_with_coupon',
        action_type: 'url',
        url,
        label: data.coupon_info ? `领券购买 (${data.coupon_info})` : '去淘宝购买',
        method: 'GET',
      }],
    });
  } catch (err) {
    // 降级:返空,让 catalog 用静态 view_product
    return c.json({ action_bindings: [] });
  }
});
```

### 5. OCP-Catalog 那 90 行 patch（核心：让 commerce-scenario 能回调 provider）

```typescript
// patch-001: packages/ocp-schema/src/index.ts (+10)
export const resolveRequestSchema = z.object({
  // ...existing fields...
  agent: z.object({                                  // ◀── 新增,可选
    agent_id: z.string().min(1),
    intent: z.enum(['shopping_browse', 'shopping_compare', 'shopping_purchase']).optional(),
  }).optional(),
});
```

```typescript
// patch-002: packages/catalog-core/src/scenario.ts (+30)
export interface ResolveContext {
  entryId: string;
  catalogEntryRow: CatalogEntry;
  request: ResolveRequest;     // 含 agent 字段
  fetch: typeof globalThis.fetch;
}

export interface Scenario {
  // 改成 async + 注入 ctx
  buildResolveActions?(
    projection: Record<string, unknown>,
    ctx: ResolveContext,
  ): Promise<ActionBinding[]> | ActionBinding[];
}
```

```typescript
// patch-003: packages/catalog-core/src/resolve-service.ts (+20)
// 在 resolve 方法内:
const actions = scenario.buildResolveActions
  ? await Promise.resolve(
      scenario.buildResolveActions(projection, {
        entryId: entry.id,
        catalogEntryRow: entry,
        request: req,
        fetch: globalThis.fetch,
      }),
    )
  : [];
```

```typescript
// patch-004: apps/examples/commerce-catalog-api/src/commerce-scenario.ts (+30)
async buildResolveActions(projection, ctx) {
  const actions: ActionBinding[] = [];

  // 静态 fallback
  if (projection.product_url) {
    actions.push({
      action_id: 'view_product', action_type: 'url',
      url: projection.product_url, label: 'View product', method: 'GET',
    });
  }

  // 动态 hook
  const attrs = (projection.attributes ?? {}) as any;
  if (attrs.requires_affiliate_resolution && attrs.provider_resolve_hook_url) {
    try {
      const res = await ctx.fetch(attrs.provider_resolve_hook_url, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({
          entry_id: ctx.entryId,
          object_id: projection.sku,
          agent_id: ctx.request.agent?.agent_id,
        }),
        signal: AbortSignal.timeout(2000),
      });
      if (res.ok) {
        const data = await res.json();
        actions.push(...(data.action_bindings ?? []));
      }
    } catch (err) {
      console.warn('[resolve-hook] failed', err);
    }
  }

  return actions;
}
```

---

## 启动命令（Week 1 mock 跑通版本）

```bash
# Postgres 已经在跑(之前 docker compose 起的 ocp-catalog-postgres-wsl on :55432)

cd OCP-Catalog

# 起 commerce-catalog-api
bun run commerce:catalog:api &      # :4000

# 起 alimama-provider (mock 模式,不需要真实 AppKey)
ALIMAMA_MOCK=true \
ALIMAMA_ADZONE_ID=mock_adzone_001 \
OCP_CATALOG_BASE_URL=http://localhost:4000 \
OCP_API_KEY=dev-api-key \
OCP_PROVIDER_ID=alimama_local \
OCP_PROVIDER_BASE_URL=http://localhost:4300 \
OCP_AUTO_SYNC=false \
PROVIDER_PORT=4300 \
bun run --cwd apps/examples/alimama-provider-api start &
```

---

## 验证流程（端到端 4 个 curl）

### 1. 触发一次同步（mock 数据进 OCP）

```bash
curl -X POST http://localhost:4300/admin/sync \
  -H 'content-type: application/json' \
  -d '{"q":"无线耳机","pageSize":10}'
# 期望: {"synced": 10, "accepted": 10, "rejected": 0}
```

### 2. Agent 查询

```bash
curl -X POST http://localhost:4000/ocp/query \
  -H 'content-type: application/json' \
  -d '{"query_pack":"ocp.query.keyword.v1","query":"耳机","limit":5}'
# 期望: items[] 含 mock 商品,带 title / image / amount
```

### 3. Agent resolve（关键：会触发回调）

```bash
ENTRY_ID="<上一步返回的 entry_id>"

curl -X POST http://localhost:4000/ocp/resolve \
  -H 'content-type: application/json' \
  -d "{\"entry_id\":\"$ENTRY_ID\",\"agent\":{\"agent_id\":\"agt_test_001\"}}"
# 期望: action_bindings 里有 alimama-provider 返的 binding,url 是 mock 短链
# 同时 alimama-provider 日志能看到 /provider/resolve_hook 被调用
```

### 4. 负向验证（provider 不可达时降级）

```bash
# kill alimama-provider 进程,再 resolve 一次
# 期望: catalog 仍返回 view_product binding(降级),不抛 500
```

---

## Week 2-3 切真实阿里要做的事

1. **mock 切真实**：`ALIMAMA_MOCK=false` + 配真实 `APP_KEY`、`APP_SECRET`、`ALIMAMA_ADZONE_ID`
2. **跑一次真实 `material.optional`，看响应是否与 fixture 一致**（不一致就更新 mapper）
3. **链接 API 权限**：
   - 拿到 → 直接用 `privilege.get`，返带 PID 短链
   - 没拿到 → 选 a) 用 `item_url` 不归因（演示用） b) 接 [订单侠 API](https://www.dingdanxia.com/price)（按调用付费）
4. **真机测试**：复制 binding.url 到手机浏览器 → 自动唤起手淘 → 下个测试订单 → 看联盟后台有归因

---

## 风险与降级

| 场景 | 影响 | 处理 |
| --- | --- | --- |
| 阿里限流（isv.access-limit） | 同步/resolve 失败 | 指数退避 + 降级到不带 PID 链接 |
| 阿里超时 > 2s | resolve 拖慢 Agent | 严格超时 + 返空 binding 让 catalog 用 view_product fallback |
| Catalog → provider 网络不通 | resolve 拿不到 affiliate binding | catalog 已有降级路径，返 view_product |
| 链接 API 权限被收回 | 不能转链 | provider 退回到二房东 / 不带 PID 链接 |
| pict_url 是 `//gw.alicdn.com/...` 无 scheme | OCP schema URL 校验失败 | mapper 已绝对化（`absolutize` 函数） |
| `reserve_price` 是字符串"129.00" | parseFloat 后 NaN | mapper 处理：parseFloat + isNaN 检查 |

---

## 完成判据（Week 1 结束）

- [ ] alimama-provider 在 :4300 起来，health 接口通
- [ ] `POST /admin/sync` 能把 10 条 mock 数据推进 OCP catalog 数据库
- [ ] OCP `/ocp/query` 能查到 mock 商品，字段全（title/images/amount）
- [ ] OCP `/ocp/resolve` 触发 provider 回调（日志能看到）
- [ ] 回调失败时 catalog 不崩，返 view_product fallback
- [ ] mapper 单测覆盖：URL 绝对化、价格字符串→number、commission_rate 基点、空 small_images

---

## 紧接着的下一步（按优先级）

1. **造 fixture 数据**（最高，半天）：在 [../fixtures/](../fixtures/) 写 `material-optional.json` + `privilege-get.json`
2. **写 90 行 OCP-Catalog patch**（关键，1 天）：协议层扩展先做完
3. **写 alimama-provider 骨架**（核心，3-5 天）：按上面文件清单
4. **跑通 Week 1 端到端验证**（半天）

完整执行流程见 [03-workflow.md](03-workflow.md)。决策依赖见 [04-decisions.md](04-decisions.md)。
