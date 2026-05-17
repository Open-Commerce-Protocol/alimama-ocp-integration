# 代码实现流程：从空白到能跑的 Demo

> 这是**实施圣经**。打开 VS Code 跟着这份文档从 Step 1.1 开始写，7 个工作日能交付完整的 mock-mode demo。
> 比 [05-implementation.md](05-implementation.md) 更细：精确到"先写哪个文件、写什么、怎么验证"。

---

## 实际执行状态（2026-05-15 更新）

```text
✅ Day 0   前置准备                跑通基线
✅ Day 1   Foundation              50 测试通过(types/sign/mapper)
✅ Day 2   Adapter                 累计 50(client/config/registration)
✅ Day 3   Server + ⭐ Checkpoint A   query/sync 端到端
✅ Day 4   OCP-Catalog 协议扩展      ⭐ Checkpoint B(正向+负向双向)
✅ Day 5   Resolve Hook 真实实现     ⭐ Checkpoint C(3 个 ActionBinding)
✅ Day 6   cron + ledger             ⭐ Checkpoint D(自动同步 + 佣金账本)
⏳ Day 7   真实 Alimama 联调          等 AppKey,见 07-接入申请清单.md
```

**累计**：79 测试 / 226 expect / typecheck 零错。

详细复盘见 Day 3、Day 4、Day 6 的对话产物;归档 patches 见 [../patches/](../patches/)。

---

## 总览（原方案,Day 1-Day 6 已经按此执行）

```text
Day 0 (0.5d)  ─── 前置准备:跑通 OCP-Catalog 自身
Day 1 (1d)    ─── Foundation: fixture + types + sign + mapper (纯函数,无 I/O)
Day 2 (1d)    ─── Adapter: config + Alimama 客户端 + OCP 客户端
Day 3 (1d)    ─── Server + ⭐ Checkpoint A: mock sync → query 跑通
Day 4 (1d)    ─── OCP-Catalog 协议扩展 + ⭐ Checkpoint B(resolve hook 双向)
Day 5 (1d)    ─── 真转链 + ⭐ Checkpoint C(3 个 ActionBinding)
Day 6 (1d)    ─── cron worker + ledger + ⭐ Checkpoint D
Day 7+ (?d)   ─── 真实 Alimama 接入 + 真机测试
```

每天结束有可演示的产出。中途任何一天停下，前面已完成的部分都可单独 demo。

---

## 实施过程中踩到的两个真实坑（值得记下来）

### 坑 1 — OCP catalog 校验要求 `catalog_id`（Day 3 中途）

`ProviderRegistration` 和 `ObjectSyncRequest` 都需要 `catalog_id` 字段（OCP-Catalog `.env` 的 `CATALOG_ID=cat_local_dev`）。
我的 builder 最初漏了它，结果 400 校验失败。

**修复**：[config.ts](../../OCP-Catalog/apps/examples/alimama-provider-api/src/config.ts) 加 `OCP_CATALOG_ID` env、[registration.ts](../../OCP-Catalog/apps/examples/alimama-provider-api/src/services/registration.ts) 和 [admin.ts](../../OCP-Catalog/apps/examples/alimama-provider-api/src/http/admin.ts) 都加 `catalog_id`。

**调试技巧**：`onError` 透传 upstream `payload`/`details`/`subCode` 让第一次 curl 就能看到 catalog 的具体校验错误，省一轮排查。

### 坑 2 — `buildSearchProjection` 默认丢弃 `attributes`（Day 4 隐蔽 bug）

Day 4 写完 4 个 patch 后 hook 死活不触发。根因：OCP-Catalog 的 `buildSearchProjection` 按白名单字段提进 projection,**`attributes` 字段直接被丢**。所以 catalog resolve 时看不到我们的 `requires_affiliate_resolution` 旗标。

**修复**：在 4 个原 patch 之外补 **Patch 005** —— 让 `buildSearchProjection` 把原始 `attributes` 也带过去。

**经验**：Provider 想往 `attributes` 塞自定义旗标控制 catalog 行为时,**必须验证 catalog 真把 attributes 透到了 projection**。grep `buildSearchProjection`/projection 相关代码,看输出对象里有没有你想要的 key。

---

## Day 0：前置准备（半天）

### 0.1 把 OCP-Catalog 自己跑通（先验证基线）

```bash
# clone: git clone https://github.com/Open-Commerce-Protocol/OCP-Catalog.git
cd OCP-Catalog
docker ps | grep ocp-catalog-postgres-wsl   # 应该看到 healthy 容器
bun install
bun run db:migrate
bun run commerce:catalog:api &              # :4000
curl http://localhost:4000/health           # 应返 {"ok":true,"service":"commerce-catalog-api",...}
```

如果 health 没通：先看 [.env](../../OCP-Catalog/.env) `DATABASE_URL=postgres://ocp:ocp@localhost:55432/ocp_catalog` 是不是这个值。

### 0.2 选编辑器 + VS Code 插件

- TypeScript 工作区：根目录用克隆后的 `OCP-Catalog/` 打开（不是 alimama-ocp-integration），让 TS 能解析 workspace 包
- 装 Biome 或 Prettier 插件按项目格式

### 0.3 决定 mock 模式还是真实 AppKey 起步

**默认选 mock 模式**——快、可控、不依赖审核。整个 Day 1-6 都不需要真实 Alimama 凭据。

> Day 0 完成判据：OCP-Catalog 的 `/health` 通；本地能跑测试 `bun test`；编辑器能跳转 `@ocp-catalog/ocp-schema` 类型定义。

---

## Day 1：Foundation 层（纯函数，无 I/O，全单测）

**今天产出**：在 `apps/examples/alimama-provider-api/` 里有 fixture、types、sign、mapper 四件套，跑 `bun test` 全绿。

### Step 1.1 创建 workspace 骨架（30 分钟）

```bash
cd OCP-Catalog
mkdir -p apps/examples/alimama-provider-api/src/{alimama,mapper,http,workers,services}
mkdir -p apps/examples/alimama-provider-api/tests/fixtures
cd apps/examples/alimama-provider-api
```

创建 `package.json`（**完全拷贝 commerce-provider-api 的结构**，改名字）：

```jsonc
{
  "name": "@ocp-catalog/alimama-provider-api",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "cd ../../.. && bun --watch apps/examples/alimama-provider-api/src/index.ts",
    "start": "cd ../../.. && bun apps/examples/alimama-provider-api/src/index.ts",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "test": "bun test --pass-with-no-tests"
  },
  "dependencies": {
    "@elysiajs/cors": "^1.3.3",
    "@ocp-catalog/auth-core": "workspace:*",
    "@ocp-catalog/config": "workspace:*",
    "@ocp-catalog/ocp-schema": "workspace:*",
    "@ocp-catalog/shared": "workspace:*",
    "elysia": "^1.3.22",
    "zod": "^4.1.12"
  },
  "devDependencies": {
    "@types/bun": "^1.3.1",
    "typescript": "^5.9.3"
  }
}
```

创建 `tsconfig.json`（直接抄 commerce-provider-api）：

```json
{
  "extends": "../../../tsconfig.base.json",
  "include": ["src/**/*.ts", "../../../packages/*/src/**/*.ts"]
}
```

回根目录跑：

```bash
cd OCP-Catalog
bun install   # workspace 会自动 link
```

**验证**：`bun run --cwd apps/examples/alimama-provider-api typecheck` 不报错（虽然 src/ 还是空的）。

### Step 1.2 造 fixture JSON（1 小时）

[fixtures/](../fixtures/) 写一份 `material-optional-sample.json`，放进 `apps/examples/alimama-provider-api/tests/fixtures/`。

**重点**：故意造覆盖边界场景的数据：
- 至少 5 个 item
- 一个 `pict_url` 是 `//gw.alicdn.com/xxx` 无 scheme
- 一个 `small_images` 为 `null`、一个为空对象、一个有 3 张图
- 一个有券、一个无券
- 价格用字符串 `"129.00"` 格式
- 含天猫（user_type=1）和淘宝（user_type=0）混合

参考 [taobao.tbk.dg.material.optional 字段说明](https://www.taokeshow.com/51162.html) 的响应结构。

**验证**：`cat tests/fixtures/material-optional-sample.json | jq '.tbk_dg_material_optional_response.result_list.map_data | length'` 输出 ≥ 5。

### Step 1.3 写 TypeScript types（1 小时）

`src/alimama/types.ts`：

```typescript
// 阿里妈妈 material.optional 响应的类型(只列我们用到的字段)
export interface AlimamaMaterialItem {
  num_iid: number;
  title: string;
  pict_url: string;
  small_images?: { string: string[] } | null;
  item_url: string;
  reserve_price: string;       // "129.00"
  zk_final_price: string;
  user_type: 0 | 1;            // 0 淘宝 / 1 天猫
  shop_title?: string;
  seller_id?: number;
  category_id?: number;
  cat?: string;
  volume?: number;
  commission_rate?: number;    // 基点
  coupon_info?: string | null;
  coupon_start_time?: string | null;
  coupon_end_time?: string | null;
  coupon_remain_count?: number | null;
  coupon_total_count?: number | null;
}

export interface AlimamaMaterialResponse {
  tbk_dg_material_optional_response: {
    result_list: { map_data: AlimamaMaterialItem[] };
    total_results: number;
  };
}

export interface AlimamaPrivilegeData {
  coupon_click_url?: string;
  item_url?: string;
  coupon_info?: string;
  coupon_end_time?: string;
  max_commission_rate?: string;
}

export interface AlimamaPrivilegeResponse {
  tbk_privilege_get_response: {
    result: { data: AlimamaPrivilegeData };
  };
}
```

### Step 1.4 写 sign + 单测（1 小时）

`src/alimama/sign.ts`：

```typescript
import { createHash, createHmac } from 'node:crypto';

export function topSign(
  params: Record<string, string>,
  appSecret: string,
  method: 'md5' | 'hmac-sha256' = 'md5',
): string {
  const sortedKeys = Object.keys(params).sort();
  const concat = sortedKeys.map((k) => k + params[k]).join('');

  if (method === 'md5') {
    return createHash('md5')
      .update(appSecret + concat + appSecret, 'utf8')
      .digest('hex')
      .toUpperCase();
  }
  return createHmac('sha256', appSecret).update(concat, 'utf8').digest('hex').toUpperCase();
}
```

`tests/sign.test.ts`：

```typescript
import { describe, expect, test } from 'bun:test';
import { topSign } from '../src/alimama/sign';

describe('topSign', () => {
  test('md5 签名稳定', () => {
    const sig = topSign({ method: 'taobao.tbk.foo', app_key: 'k1', timestamp: '2026-05-13 10:00:00' }, 's3cr3t');
    expect(sig).toMatch(/^[A-F0-9]{32}$/);
  });

  test('参数顺序不影响结果', () => {
    const a = topSign({ method: 'foo', app_key: 'k1' }, 'sec');
    const b = topSign({ app_key: 'k1', method: 'foo' }, 'sec');
    expect(a).toEqual(b);
  });
});
```

**验证**：`bun test tests/sign.test.ts` 两条 pass。

### Step 1.5 写 mapper + 单测（2-3 小时，今天最重要）

`src/mapper/material-to-object.ts`：见 [05-implementation.md §3](05-implementation.md#3-字段映射阿里--ocp) 的完整代码。**关键不变量**：

- 所有 image_urls 经过 `absolutize()` 处理
- `reserve_price` / `zk_final_price` 用 `parseFloat`，失败用 `0`
- `attributes.requires_affiliate_resolution = true`
- `attributes.provider_resolve_hook_url` 来自参数注入

`tests/mapper.test.ts`：

```typescript
import { describe, expect, test } from 'bun:test';
import { readFileSync } from 'node:fs';
import { mapMaterialToCommercialObject } from '../src/mapper/material-to-object';

const sample = JSON.parse(readFileSync('./tests/fixtures/material-optional-sample.json', 'utf8'));
const items = sample.tbk_dg_material_optional_response.result_list.map_data;

describe('mapMaterialToCommercialObject', () => {
  const ctx = {
    providerId: 'alimama_test',
    siteUrl: 'http://localhost:4300',
    providerBaseUrl: 'http://localhost:4300',
  };

  test('生成的 image_urls 全部是 https://', () => {
    for (const item of items) {
      const obj = mapMaterialToCommercialObject(item, ctx);
      const urls = obj.descriptors[0].data.image_urls as string[];
      for (const u of urls) expect(u).toMatch(/^https:\/\//);
    }
  });

  test('包含三个 descriptor pack', () => {
    const obj = mapMaterialToCommercialObject(items[0], ctx);
    const packIds = obj.descriptors.map((d) => d.pack_id);
    expect(packIds).toContain('ocp.commerce.product.core.v1');
    expect(packIds).toContain('ocp.commerce.price.v1');
    expect(packIds).toContain('ocp.commerce.inventory.v1');
  });

  test('attributes 含 resolve_hook_url 和 affiliate 标记', () => {
    const obj = mapMaterialToCommercialObject(items[0], ctx);
    const attrs = obj.descriptors[0].data.attributes as Record<string, unknown>;
    expect(attrs.requires_affiliate_resolution).toBe(true);
    expect(attrs.provider_resolve_hook_url).toBe('http://localhost:4300/provider/resolve_hook');
    expect(attrs.affiliate_provider).toBe('alimama_taobao_union');
  });

  test('price 是 number 不是 string', () => {
    const obj = mapMaterialToCommercialObject(items[0], ctx);
    const price = obj.descriptors.find((d) => d.pack_id === 'ocp.commerce.price.v1')!.data;
    expect(typeof price.amount).toBe('number');
    expect(typeof price.list_amount).toBe('number');
  });

  test('user_type=1 映射 tmall, =0 映射 taobao', () => {
    const tmall = items.find((i) => i.user_type === 1);
    const taobao = items.find((i) => i.user_type === 0);
    if (tmall) {
      const attrs = mapMaterialToCommercialObject(tmall, ctx).descriptors[0].data.attributes as any;
      expect(attrs.platform).toBe('tmall');
    }
    if (taobao) {
      const attrs = mapMaterialToCommercialObject(taobao, ctx).descriptors[0].data.attributes as any;
      expect(attrs.platform).toBe('taobao');
    }
  });
});
```

**验证**：`bun test tests/mapper.test.ts` 全绿。这是 Day 1 的核心交付。

> **Day 1 完成判据**：fixture 在；types 完整；sign 通过；mapper 通过 5 条单测。**全程零 I/O，零网络**。

---

## Day 2：Adapter 层

**今天产出**：能调阿里 API（mock + 真实双模式）、能调 OCP catalog 的 `/ocp/objects/sync`。

### Step 2.1 写 config（30 分钟）

`src/config.ts`：

```typescript
import { z } from 'zod';

const configSchema = z.object({
  // OCP 相关
  OCP_CATALOG_BASE_URL: z.string().url(),
  OCP_PROVIDER_ID: z.string().min(1),
  OCP_API_KEY: z.string().min(1),
  OCP_PROVIDER_BASE_URL: z.string().url(),
  PROVIDER_PORT: z.coerce.number().int().default(4300),

  // Alimama 相关
  ALIMAMA_MOCK: z.coerce.boolean().default(true),
  ALIMAMA_APP_KEY: z.string().optional(),
  ALIMAMA_APP_SECRET: z.string().optional(),
  ALIMAMA_ADZONE_ID: z.string().default('mock_adzone_001'),

  // 其他
  OCP_AUTO_SYNC: z.coerce.boolean().default(false),
});

export function loadAlimamaConfig() {
  const parsed = configSchema.parse(process.env);
  if (!parsed.ALIMAMA_MOCK && (!parsed.ALIMAMA_APP_KEY || !parsed.ALIMAMA_APP_SECRET)) {
    throw new Error('ALIMAMA_MOCK=false 时必须提供 ALIMAMA_APP_KEY / ALIMAMA_APP_SECRET');
  }
  return parsed;
}

export type AlimamaConfig = ReturnType<typeof loadAlimamaConfig>;
```

### Step 2.2 写 AlimamaClient（2 小时）

`src/alimama/client.ts`：

```typescript
import { readFileSync } from 'node:fs';
import { join } from 'node:path';
import type { AlimamaConfig } from '../config';
import { topSign } from './sign';
import type { AlimamaMaterialResponse, AlimamaPrivilegeResponse } from './types';

const FIXTURE_DIR = new URL('../../tests/fixtures', import.meta.url).pathname;

function loadFixture<T>(name: string): T {
  return JSON.parse(readFileSync(join(FIXTURE_DIR, name), 'utf8'));
}

export class AlimamaClient {
  constructor(private readonly cfg: AlimamaConfig) {}

  async listMaterial(opts: {
    q?: string;
    cat?: string;
    pageNo: number;
    pageSize: number;
    adzoneId?: string;
  }): Promise<AlimamaMaterialResponse> {
    if (this.cfg.ALIMAMA_MOCK) return loadFixture('material-optional-sample.json');
    return this.callTop('taobao.tbk.dg.material.optional', {
      q: opts.q,
      cat: opts.cat,
      page_no: String(opts.pageNo),
      page_size: String(opts.pageSize),
      adzone_id: opts.adzoneId ?? this.cfg.ALIMAMA_ADZONE_ID,
    });
  }

  async generatePrivilegeLink(opts: {
    itemId: string;
    adzoneId?: string;
    externalId?: string;
  }): Promise<AlimamaPrivilegeResponse> {
    if (this.cfg.ALIMAMA_MOCK) return loadFixture('privilege-get-sample.json');
    return this.callTop('taobao.tbk.privilege.get', {
      item_id: opts.itemId,
      adzone_id: opts.adzoneId ?? this.cfg.ALIMAMA_ADZONE_ID,
      ...(opts.externalId ? { external_id: opts.externalId } : {}),
    });
  }

  private async callTop<T>(method: string, params: Record<string, string | undefined>): Promise<T> {
    const cleanParams = Object.fromEntries(
      Object.entries(params).filter(([, v]) => v !== undefined),
    ) as Record<string, string>;

    const sysParams = {
      method,
      app_key: this.cfg.ALIMAMA_APP_KEY!,
      v: '2.0',
      format: 'json',
      sign_method: 'md5',
      timestamp: new Date().toISOString().replace('T', ' ').slice(0, 19),
    };

    const all = { ...sysParams, ...cleanParams };
    const all_with_sign = { ...all, sign: topSign(all, this.cfg.ALIMAMA_APP_SECRET!, 'md5') };

    const res = await fetch('https://gw.api.taobao.com/router/rest', {
      method: 'POST',
      headers: { 'content-type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams(all_with_sign).toString(),
      signal: AbortSignal.timeout(5000),
    });

    if (!res.ok) throw new Error(`Alimama API failed: ${res.status}`);
    return res.json();
  }
}
```

**验证**：写一个最小测试：

```typescript
// tests/client.test.ts
import { test, expect } from 'bun:test';
import { AlimamaClient } from '../src/alimama/client';

test('mock 模式不调网络', async () => {
  const client = new AlimamaClient({
    ALIMAMA_MOCK: true,
    ALIMAMA_ADZONE_ID: 'test',
  } as any);
  const res = await client.listMaterial({ pageNo: 1, pageSize: 10 });
  expect(res.tbk_dg_material_optional_response.result_list.map_data.length).toBeGreaterThan(0);
});
```

### Step 2.3 写 OcpCatalogClient（1.5 小时）

`src/services/catalog-client.ts`：

```typescript
import type { AlimamaConfig } from '../config';

export class OcpCatalogClient {
  constructor(private readonly cfg: AlimamaConfig) {}

  async registerProvider(registration: Record<string, unknown>) {
    return this.post('/ocp/providers/register', registration);
  }

  async syncObjects(request: Record<string, unknown>) {
    return this.post('/ocp/objects/sync', request);
  }

  private async post(path: string, body: unknown) {
    const url = `${this.cfg.OCP_CATALOG_BASE_URL}${path}`;
    const res = await fetch(url, {
      method: 'POST',
      headers: {
        'content-type': 'application/json',
        'x-api-key': this.cfg.OCP_API_KEY,
      },
      body: JSON.stringify(body),
    });
    const data = await res.json().catch(() => ({}));
    if (!res.ok) throw new Error(`OCP catalog ${res.status}: ${JSON.stringify(data)}`);
    return data;
  }
}
```

### Step 2.4 写 ProviderRegistration builder（30 分钟）

`src/services/registration.ts`：

```typescript
import type { AlimamaConfig } from '../config';

export function buildProviderRegistration(cfg: AlimamaConfig, version: number) {
  return {
    ocp_version: '1.0' as const,
    kind: 'ProviderRegistration' as const,
    id: `reg_${cfg.OCP_PROVIDER_ID}_${version}`,
    registration_version: version,
    updated_at: new Date().toISOString(),
    provider: {
      provider_id: cfg.OCP_PROVIDER_ID,
      entity_type: 'merchant' as const,
      display_name: 'Alimama (Taobao Union Adapter)',
      homepage: cfg.OCP_PROVIDER_BASE_URL,
      domains: [new URL(cfg.OCP_PROVIDER_BASE_URL).hostname],
    },
    object_declarations: [
      {
        guaranteed_fields: [
          'ocp.commerce.product.core.v1#/title',
          'ocp.commerce.product.core.v1#/product_url',
          'ocp.commerce.price.v1#/currency',
          'ocp.commerce.price.v1#/amount',
        ],
        optional_fields: [
          'ocp.commerce.product.core.v1#/brand',
          'ocp.commerce.product.core.v1#/image_urls',
          'ocp.commerce.inventory.v1#/availability_status',
        ],
        sync: {
          preferred_capabilities: ['ocp.push.batch'],
          avoid_capabilities_unless_necessary: [],
          provider_endpoints: {},
        },
      },
    ],
  };
}
```

> **Day 2 完成判据**：types 全编译过；mock 模式下 AlimamaClient 能返 fixture；OcpCatalogClient + registration builder 写完（暂未跑真实调用）。

---

## Day 3：Server + ⭐ Checkpoint A

**今天产出**：能起 HTTP 服务、能 POST `/admin/sync` 把 mock 商品同步到 OCP catalog、Agent 能 query 到。

### Step 3.1 HTTP server scaffold (Elysia)（1 小时）

`src/index.ts`：

```typescript
import { cors } from '@elysiajs/cors';
import { Elysia } from 'elysia';
import { ZodError } from 'zod';
import { loadAlimamaConfig } from './config';
import { AlimamaClient } from './alimama/client';
import { OcpCatalogClient } from './services/catalog-client';
import { adminRouter } from './http/admin';
import { resolveHookRouter } from './http/resolve-hook';

const cfg = loadAlimamaConfig();
const alimama = new AlimamaClient(cfg);
const catalog = new OcpCatalogClient(cfg);

const app = new Elysia()
  .use(cors())
  .decorate('alimama', alimama)
  .decorate('catalog', catalog)
  .decorate('cfg', cfg)
  .onError(({ error, set }) => {
    if (error instanceof ZodError) {
      set.status = 400;
      return { error: { code: 'validation_error', details: error.issues } };
    }
    set.status = 500;
    return { error: { code: 'internal_error', message: error instanceof Error ? error.message : 'unknown' } };
  })
  .get('/health', () => ({
    ok: true,
    service: 'alimama-provider-api',
    provider_id: cfg.OCP_PROVIDER_ID,
    mock: cfg.ALIMAMA_MOCK,
  }))
  .use(adminRouter)
  .use(resolveHookRouter)
  .listen(cfg.PROVIDER_PORT);

console.log(`Alimama Provider API listening on :${cfg.PROVIDER_PORT}, mock=${cfg.ALIMAMA_MOCK}`);
```

### Step 3.2 写 /admin/sync 端点（2 小时）

`src/http/admin.ts`：

```typescript
import { Elysia, t } from 'elysia';
import { mapMaterialToCommercialObject } from '../mapper/material-to-object';
import { buildProviderRegistration } from '../services/registration';

export const adminRouter = new Elysia({ prefix: '/admin' })
  .post(
    '/register',
    async ({ catalog, cfg }: any) => {
      const registration = buildProviderRegistration(cfg, 1);
      const result = await catalog.registerProvider(registration);
      return result;
    },
  )
  .post(
    '/sync',
    async ({ body, alimama, catalog, cfg }: any) => {
      const { q, pageSize = 20 } = body;

      // 1. 拉物料
      const mat = await alimama.listMaterial({ q, pageNo: 1, pageSize });
      const items = mat.tbk_dg_material_optional_response?.result_list?.map_data ?? [];

      // 2. 映射
      const ctx = {
        providerId: cfg.OCP_PROVIDER_ID,
        siteUrl: cfg.OCP_PROVIDER_BASE_URL,
        providerBaseUrl: cfg.OCP_PROVIDER_BASE_URL,
      };
      const objects = items.map((item: any) => mapMaterialToCommercialObject(item, ctx));

      // 3. 推送(单批 100 上限)
      const batches: any[][] = [];
      for (let i = 0; i < objects.length; i += 100) {
        batches.push(objects.slice(i, i + 100));
      }

      const results = [];
      for (const batch of batches) {
        const res = await catalog.syncObjects({
          ocp_version: '1.0',
          kind: 'ObjectSyncRequest',
          provider_id: cfg.OCP_PROVIDER_ID,
          registration_version: 1,
          batch_id: `batch_${Date.now()}`,
          objects: batch,
        });
        results.push(res);
      }

      return {
        total: objects.length,
        batches: results.length,
        results,
      };
    },
    {
      body: t.Object({
        q: t.Optional(t.String()),
        pageSize: t.Optional(t.Number()),
      }),
    },
  );
```

### Step 3.3 临时 stub /provider/resolve_hook（避免 Day 5 前 import 报错）

`src/http/resolve-hook.ts`：

```typescript
import { Elysia } from 'elysia';

// Day 5 才完整实现,这里先 stub
export const resolveHookRouter = new Elysia({ prefix: '/provider/resolve_hook' })
  .post('/', () => ({ action_bindings: [] }));
```

### Step 3.4 ⭐ Checkpoint A：跑通 mock sync → query

```bash
# Terminal 1: commerce-catalog-api (应该已经在跑)
cd OCP-Catalog
bun run commerce:catalog:api &

# Terminal 2: alimama-provider
ALIMAMA_MOCK=true \
OCP_CATALOG_BASE_URL=http://localhost:4000 \
OCP_PROVIDER_ID=alimama_local \
OCP_API_KEY=dev-api-key \
OCP_PROVIDER_BASE_URL=http://localhost:4300 \
PROVIDER_PORT=4300 \
bun run --cwd apps/examples/alimama-provider-api dev

# Terminal 3: 验证
curl http://localhost:4300/health
# {"ok":true,"service":"alimama-provider-api","mock":true,...}

curl -X POST http://localhost:4300/admin/register \
  -H 'content-type: application/json' \
  -d '{}'
# 期望: {"status":"accepted_full",...} 或类似

curl -X POST http://localhost:4300/admin/sync \
  -H 'content-type: application/json' \
  -d '{"q":"无线耳机","pageSize":10}'
# 期望: {"total":10,"batches":1,"results":[{...accepted_count:10}]}

curl -X POST http://localhost:4000/ocp/query \
  -H 'content-type: application/json' \
  -d '{"query_pack":"ocp.query.keyword.v1","query":"耳机","limit":5}'
# 期望: items[] 里能看到 mock 商品(title/image_urls/amount 完整)
```

> **Day 3 完成判据 = Checkpoint A**：
> - [ ] alimama-provider 在 :4300 起来、health 通
> - [ ] `/admin/register` 成功（Catalog 接受 Provider 注册）
> - [ ] `/admin/sync` 把 mock 数据成功 sync 到 catalog
> - [ ] OCP `/ocp/query` 能搜到 mock 商品
> - [ ] 整个过程零 Alimama 真实调用

**到这一步可以演示给领导：AI Agent 用 OCP 标准协议，就能查到淘宝风格商品。**

---

## Day 4：OCP-Catalog 90 行 Patch

**今天产出**：commerce-catalog-api 支持 resolve 时回调 provider 拿动态 binding。

### Step 4.1 Patch-001: ocp-schema 加 agent 字段（30 分钟）

修改 `packages/ocp-schema/src/index.ts` 的 `resolveRequestSchema`。

注意：找到现有的 `resolveRequestSchema` 定义（约 291-296 行），在 entry_id 后加：

```typescript
agent: z.object({
  agent_id: z.string().min(1),
  intent: z.enum(['shopping_browse', 'shopping_compare', 'shopping_purchase']).optional(),
}).optional(),
```

**验证**：

```bash
bun run --cwd packages/ocp-schema typecheck
# 跑现有测试,看老的 resolve 请求(没 agent 字段)还能通过
bun test packages/ocp-schema
```

### Step 4.2 Patch-002: catalog-core scenario interface（45 分钟）

找到 `packages/catalog-core/src/scenario.ts`（或类似文件）的 `Scenario` interface：

```typescript
// 新增
export interface ResolveContext {
  entryId: string;
  catalogEntryRow: unknown;       // 用现有 type
  request: unknown;                // ResolveRequest
  fetch: typeof globalThis.fetch;
}

// 修改
export interface Scenario {
  buildResolveActions?(
    projection: Record<string, unknown>,
    ctx: ResolveContext,
  ): Promise<ActionBinding[]> | ActionBinding[];   // 加 Promise<>
}
```

### Step 4.3 Patch-003: resolve-service 调用处（30 分钟）

找到 `packages/catalog-core/src/resolve-service.ts` 中调用 `buildResolveActions` 的地方：

```typescript
// 原代码大约:
// const actions = scenario.buildResolveActions?.(projection) ?? [];

// 改成:
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

### Step 4.4 Patch-004: commerce-scenario hook 实现（1 小时）

找到 `apps/examples/commerce-catalog-api/src/commerce-scenario.ts` 的 `buildResolveActions`，整段替换为：

```typescript
async buildResolveActions(
  projection: Record<string, unknown>,
  ctx: ResolveContext,
): Promise<ActionBinding[]> {
  const actions: ActionBinding[] = [];

  // 1. 静态 fallback
  if (projection.product_url) {
    actions.push({
      action_id: 'view_product',
      action_type: 'url',
      url: projection.product_url as string,
      label: 'View product',
      method: 'GET',
    });
  }

  // 2. 动态 hook
  const attrs = (projection.attributes ?? {}) as Record<string, any>;
  if (attrs.requires_affiliate_resolution && attrs.provider_resolve_hook_url) {
    try {
      const res = await ctx.fetch(attrs.provider_resolve_hook_url, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({
          entry_id: ctx.entryId,
          object_id: projection.sku,
          agent_id: (ctx.request as any)?.agent?.agent_id,
        }),
        signal: AbortSignal.timeout(2000),
      });
      if (res.ok) {
        const data = await res.json();
        actions.push(...(data.action_bindings ?? []));
      }
    } catch (err) {
      console.warn('[commerce-scenario] resolve hook failed:', err);
    }
  }

  return actions;
},
```

### Step 4.5 把 4 个 patch 归档到 patches/

在 [../patches/](../patches/) 写四份 `patch-001-*.md`，每份含：目的、文件、diff、验证、回滚。

### Step 4.6 跑 OCP-Catalog 现有测试

```bash
cd OCP-Catalog
bun test
# 期望:所有现有测试 pass(因为是向后兼容扩展)
```

> **Day 4 完成判据**：
> - [ ] 4 个 patch 都改完且 typecheck 通过
> - [ ] OCP-Catalog 现有测试全绿
> - [ ] patches/ 下有 4 个 md 文档归档

---

## Day 5：Resolve Hook + ⭐ Checkpoint B

**今天产出**：完整的 Agent resolve → Catalog → Provider → Alimama → Agent 链路打通。

### Step 5.1 完整实现 /provider/resolve_hook（1.5 小时）

替换掉 Day 3 的 stub：

```typescript
// src/http/resolve-hook.ts
import { Elysia, t } from 'elysia';

export const resolveHookRouter = new Elysia({ prefix: '/provider/resolve_hook' })
  .post(
    '/',
    async ({ body, alimama, cfg }: any) => {
      const { entry_id, object_id, agent_id } = body;

      try {
        const result = await alimama.generatePrivilegeLink({
          itemId: String(object_id),
          adzoneId: cfg.ALIMAMA_ADZONE_ID,
          externalId: entry_id,
        });

        const data = result.tbk_privilege_get_response?.result?.data;
        const url = data?.coupon_click_url ?? data?.item_url;

        if (!url) {
          console.warn(`[resolve_hook] no url for ${object_id}`);
          return { action_bindings: [] };
        }

        return {
          action_bindings: [
            {
              action_id: 'buy_with_coupon',
              action_type: 'url',
              url,
              label: data?.coupon_info ? `领券购买 (${data.coupon_info})` : '去淘宝购买',
              method: 'GET',
            },
          ],
        };
      } catch (err) {
        console.error('[resolve_hook] failed:', err);
        return { action_bindings: [] };
      }
    },
    {
      body: t.Object({
        entry_id: t.String(),
        object_id: t.Union([t.String(), t.Number()]),
        agent_id: t.Optional(t.String()),
      }),
    },
  );
```

### Step 5.2 造 privilege.get 的 fixture（30 分钟）

写一份 `tests/fixtures/privilege-get-sample.json`：

```json
{
  "tbk_privilege_get_response": {
    "result": {
      "data": {
        "coupon_click_url": "https://s.click.taobao.com/mock_coupon_xyz",
        "item_url": "https://s.click.taobao.com/mock_item_xyz",
        "coupon_info": "满 99 减 10",
        "coupon_end_time": "2026-12-31",
        "max_commission_rate": "1550"
      }
    }
  }
}
```

### Step 5.3 ⭐ Checkpoint B：跑通完整 resolve 链路

```bash
# 重启 alimama-provider 加载新代码

# 1. 同步(用 Day 3 的命令)
curl -X POST http://localhost:4300/admin/sync \
  -H 'content-type: application/json' \
  -d '{"q":"无线耳机","pageSize":5}'

# 2. 查询拿到 entry_id
curl -X POST http://localhost:4000/ocp/query \
  -H 'content-type: application/json' \
  -d '{"query_pack":"ocp.query.keyword.v1","query":"耳机","limit":1}' | jq

# 3. resolve(关键!)
ENTRY_ID="<上面返回的 entry_id>"
curl -X POST http://localhost:4000/ocp/resolve \
  -H 'content-type: application/json' \
  -d "{\"entry_id\":\"$ENTRY_ID\",\"agent\":{\"agent_id\":\"agt_test_001\"}}" | jq

# 期望:
# - action_bindings 数组里至少有 2 个 binding:
#   - "view_product"(静态 fallback)
#   - "buy_with_coupon"(从 provider 来的,url 是 s.click.taobao.com/mock_coupon_xyz)
# - alimama-provider 日志能看到 /provider/resolve_hook POST 被调用
# - commerce-catalog-api 日志能看到 hook 调用的尝试

# 4. 负向验证(降级)
# 关闭 alimama-provider,重新 resolve
# 期望:只剩 view_product binding,不报 500
```

### Step 5.4 协议级 demo 录屏（30 分钟，可选）

录一段 30 秒的 curl 演示视频，发群里："AI Agent 用 OCP 协议就能拿到淘宝带 PID 链接"。

> **Day 5 完成判据 = Checkpoint B**：
> - [ ] resolve 请求带 agent 字段不被拒绝
> - [ ] action_bindings 含 provider 返的 affiliate binding
> - [ ] provider 不可达时 catalog 返回退化结果不崩
> - [ ] 端到端链路：mock sync → query → resolve → action_binding 全通

**到这一步，协议层的"用 OCP 包阿里妈妈"已经完整跑通（mock 数据）。**

---

## Day 6：加固

**今天产出**：错误处理 / 日志 / cron worker / 单测覆盖率 ≥ 70%。

### Step 6.1 Error handling 完善（1.5 小时）

`src/alimama/client.ts` 加：

```typescript
class AlimamaApiError extends Error {
  constructor(public code: string, message: string, public details?: unknown) {
    super(message);
  }
}

// callTop 内部:
if (data.error_response) {
  throw new AlimamaApiError(
    data.error_response.sub_code ?? 'unknown',
    data.error_response.sub_msg ?? 'Alimama API returned error',
    data.error_response,
  );
}

// 限流的特殊处理:
if (data.error_response?.sub_code === 'isv.access-limit') {
  // 抛出来让上层退避重试
  throw new AlimamaApiError('rate_limited', 'Rate limited', data.error_response);
}
```

### Step 6.2 结构化日志（30 分钟）

`src/lib/logger.ts`：

```typescript
type Level = 'debug' | 'info' | 'warn' | 'error';
export function log(level: Level, msg: string, meta?: Record<string, unknown>) {
  console.log(JSON.stringify({ ts: new Date().toISOString(), level, msg, ...meta }));
}
```

替换 Day 1-5 里所有 `console.log` / `console.error`。

### Step 6.3 material-poller worker（1.5 小时）

`src/workers/material-poller.ts`：

```typescript
import type { AlimamaConfig } from '../config';
import { mapMaterialToCommercialObject } from '../mapper/material-to-object';
import type { AlimamaClient } from '../alimama/client';
import type { OcpCatalogClient } from '../services/catalog-client';
import { log } from '../lib/logger';

const KEYWORDS = ['耳机', '充电宝', '手机壳'];   // 改成实际要拉的关键词列表

export function startMaterialPoller(
  cfg: AlimamaConfig,
  alimama: AlimamaClient,
  catalog: OcpCatalogClient,
) {
  const intervalMs = 30 * 60 * 1000;   // 30 分钟一轮
  log('info', 'material-poller started', { intervalMs });

  const tick = async () => {
    for (const q of KEYWORDS) {
      try {
        const mat = await alimama.listMaterial({ q, pageNo: 1, pageSize: 50 });
        const items = mat.tbk_dg_material_optional_response?.result_list?.map_data ?? [];
        const ctx = {
          providerId: cfg.OCP_PROVIDER_ID,
          siteUrl: cfg.OCP_PROVIDER_BASE_URL,
          providerBaseUrl: cfg.OCP_PROVIDER_BASE_URL,
        };
        const objects = items.map((it: any) => mapMaterialToCommercialObject(it, ctx));
        await catalog.syncObjects({
          ocp_version: '1.0',
          kind: 'ObjectSyncRequest',
          provider_id: cfg.OCP_PROVIDER_ID,
          registration_version: 1,
          batch_id: `poller_${Date.now()}_${q}`,
          objects,
        });
        log('info', 'poller synced', { q, count: objects.length });
      } catch (err) {
        log('error', 'poller failed', { q, err: String(err) });
      }
    }
  };

  setInterval(tick, intervalMs);
  void tick();   // 立即跑一次
}
```

在 `src/index.ts` 末尾启用：

```typescript
if (cfg.OCP_AUTO_SYNC) {
  const { startMaterialPoller } = await import('./workers/material-poller');
  startMaterialPoller(cfg, alimama, catalog);
}
```

### Step 6.4 单测覆盖（剩余时间）

- 至少给 `client.ts` 的 mock 模式补 3 个 test
- 给 `/admin/sync` 加一个集成测试（用本地 stub catalog）
- 跑 `bun test --coverage` 看覆盖率

> **Day 6 完成判据**：
> - [ ] 限流 / 错误码识别正确
> - [ ] 所有日志走 structured logger
> - [ ] material-poller cron 每 30 分钟自动同步
> - [ ] `bun test` 全绿，覆盖率 ≥ 70%

---

## Day 7+：⭐ Checkpoint C - 真实 Alimama 接入

**今天产出**：从 mock 切真实，看真实链路是否符合所有假设。

### Step 7.1 申请测试 AppKey（异步等待，1-3 天）

不阻塞其他工作。等审核期间继续完善代码。

### Step 7.2 创建一个测试 adzone

阿里妈妈后台 → 推广管理 → 推广位 → 新建。记下 PID（`mm_x_y_z`），`adzone_id` 是最后一段。

### Step 7.3 切真实 + 跑 sync

```bash
ALIMAMA_MOCK=false \
ALIMAMA_APP_KEY=<真实> \
ALIMAMA_APP_SECRET=<真实> \
ALIMAMA_ADZONE_ID=<真实> \
... \
bun run --cwd apps/examples/alimama-provider-api start
```

```bash
curl -X POST http://localhost:4300/admin/sync \
  -H 'content-type: application/json' \
  -d '{"q":"无线耳机","pageSize":5}'
```

**这一步会暴露所有"fixture vs 真实数据"的不一致**。常见问题：

| 现象 | 原因 | 修复 |
| --- | --- | --- |
| `image_urls` 校验失败 | `pict_url` 是 `//gw.alicdn.com/...` 无 scheme | mapper 已经 `absolutize()`，再确认下 |
| 价格变成 `NaN` | 某商品 `zk_final_price` 是空字符串 | mapper 加 `isNaN` 检查，fallback 到 `reserve_price` |
| `category` 为空 | 真实返回里没有 `category_id` | mapper fallback 到 `cat` 字段 |
| `commission_rate` 是字符串 | 类型描述错了 | 修 types |
| 签名错误 isv.invalid-signature | `sign_method` 不匹配 | 试 `hmac-sha256` |

### Step 7.4 真机测试：链接到底能不能跳手淘

```bash
# 拿一个 resolve 返回的 binding url
URL="<某个 s.click.taobao.com 短链>"
# 复制到手机微信发给自己 -> 点开 -> 应该自动唤起手淘
```

如果链接 API 权限没拿到，这一步用 `item_url`（原始链接，无 PID），跳转能跳但没归因。

### Step 7.5 真实下一单（可选）

- 用 PoC 链接下个小金额测试单
- 等 1-2 小时后调 `taobao.tbk.order.get` 看订单是否带正确 `adzone_id`
- 验证 `external_id` 是否回流（决定 OCP entry_id → 订单的追踪能力）

> **Day 7+ 完成判据 = Checkpoint C**：
> - [ ] 真实 material.optional 调用成功，mapper 不崩
> - [ ] 真实 privilege.get 返回 s.click.taobao.com 短链
> - [ ] 手机点链接跳手淘成功
> - [ ] 真实订单数据能从 order.get 拉到

---

## 完成态

跑到 Day 7 Checkpoint C 之后，你拥有：

- ✅ **可演示的完整 demo**：AI Agent 标准 OCP 协议查淘宝商品、拿带 PID 链接
- ✅ **OCP-Catalog 协议扩展**：4 个 patch 等待提 PR（agent 字段、async scenario、ResolveContext、hook 回调）
- ✅ **alimama-provider 服务**：~1500 行代码 + 单测覆盖 70%+
- ✅ **mock 模式与真实模式双轨**：本地开发不依赖真实 API
- ✅ **commission_ledger 等扩展可选项**：基础设施就位，后续扩展容易

接下来可以做的方向（见 [03-workflow.md 阶段五](03-workflow.md#阶段五--协议扩展--多联盟桥接week-5)）：
- 给 OCP-Catalog 主仓提 PR 把 4 个 patch 合入
- 实现 `commission_ledger` 表 + order-poller
- 接入第二个联盟（京东 / 拼多多）验证桥接架构
- 申请官方链接 API 高级权限

---

## 卡点速查

| 卡点 | 阶段 | 处理 |
| --- | --- | --- |
| `bun install` 在 Windows 失败 | Day 0 | `bun install --backend=hardlink` |
| TypeScript 找不到 `@ocp-catalog/*` | Day 1 | 回根目录 `bun install`，让 workspace symlinks 重建 |
| Elysia 类型推断报错 | Day 3 | route handler 参数加 `({ body, ... }: any)` 临时绕过；后期补 generic |
| OCP catalog 拒绝注册 `provider_id` 重复 | Day 3 | 删 catalog DB 里 `providers` 表对应行重来，或换 `OCP_PROVIDER_ID` |
| OCP catalog `/ocp/objects/sync` 返 schema 错 | Day 3 | 看错误里的 `path`，对照 [ocp-schema commercialObjectSchema](../../OCP-Catalog/packages/ocp-schema/src/index.ts) 修 mapper |
| Resolve hook 不被回调 | Day 5 | 检查 `attributes.requires_affiliate_resolution` 有没有同步进 catalog；可能 mapper 漏注入 |
| 真实 Alimama 返签名错误 | Day 7 | `sign_method` 改 `hmac-sha256` 重试；确认 `app_secret` 没空格 |
| `external_id` 不出现在订单回执 | Day 7+ | 阿里官方文档没保证，需要真实下单测试 |

---

## 想找代码片段对照

| 找什么 | 去哪 |
| --- | --- |
| Provider 服务的参考模板 | [`apps/examples/commerce-provider-api/src/index.ts`](../../OCP-Catalog/apps/examples/commerce-provider-api/src/index.ts) |
| Provider → Catalog 调用模式 | [`apps/examples/commerce-provider-api/src/catalog-client.ts`](../../OCP-Catalog/apps/examples/commerce-provider-api/src/catalog-client.ts) |
| Provider 注册请求 schema | [`apps/examples/commerce-provider-api/src/provider-mapper.ts`](../../OCP-Catalog/apps/examples/commerce-provider-api/src/provider-mapper.ts) |
| OCP schema 全集 | [`packages/ocp-schema/src/index.ts`](../../OCP-Catalog/packages/ocp-schema/src/index.ts) |
| commerce-catalog-api 入口 | [`apps/examples/commerce-catalog-api/src/index.ts`](../../OCP-Catalog/apps/examples/commerce-catalog-api/src/index.ts) |
| commerce scenario 类（要改的） | [`apps/examples/commerce-catalog-api/src/commerce-scenario.ts`](../../OCP-Catalog/apps/examples/commerce-catalog-api/src/commerce-scenario.ts) |

---

## 文档体系导航

| 你在哪 | 下一步看哪 |
| --- | --- |
| 想理解为什么做这事 | [00-overview.md](00-overview.md) |
| 想懂技术上为什么这么设计 | [05-implementation.md](05-implementation.md) |
| 想看完整 4-6 周路线图 | [03-workflow.md](03-workflow.md) |
| 想知道还要拍板什么 | [04-decisions.md](04-decisions.md) |
| **想开始写代码** | **就是你正在看的这份** |
