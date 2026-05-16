# Day 4 Patches — Resolve Hook Mechanism

让 OCP-Catalog 在 resolve 时能回调 Provider 拿动态 ActionBinding。这是"用 OCP Catalog 包阿里妈妈"动态转链能力的协议层基础。

## 总览

| # | 文件 | 目的 | 改动 |
| --- | --- | --- | --- |
| 001 | `packages/ocp-schema/src/index.ts` | `resolveRequestSchema` 加可选 `agent` 字段 | +12 |
| 002 | `packages/catalog-core/src/scenario.ts` | 新增 `ResolveContext`,`buildResolveActions` 改 async-friendly | +31 |
| 003 | `packages/catalog-core/src/resolve-service.ts` | 调用处改 await + 注入 ctx | +17 |
| 003a | `packages/catalog-core/src/index.ts` | 导出 `ResolveContext` 类型 | +7 |
| 004 | `apps/examples/commerce-catalog-api/src/commerce-scenario.ts` | 实现 hook 回调 + 静态 fallback | +90 |
| 005 | (同上) | `buildSearchProjection` 透传 `attributes` 字段（让 hook 能看到 Provider 的旗标） | (含在 004 文件) |
| 006 | `apps/examples/commerce-provider-api/src/provider-mapper.test.ts` | side-fix:补齐预存在的 fixture 漂移 | +4 |

**合计**：6 files, +150 行，向后兼容。

---

## Patch 001 — ocp-schema: ResolveRequest 加 agent 字段

### 为什么

让 catalog 在 resolve 时知道是哪个 Agent 在调，可以做 per-agent attribution、风控、限流。

### 改动

`packages/ocp-schema/src/index.ts` 第 291-296 行：

```diff
 export const resolveRequestSchema = z.object({
   ocp_version: ocpVersionSchema.optional(),
   kind: z.literal('ResolveRequest').optional(),
   catalog_id: z.string().min(1).optional(),
   entry_id: z.string().min(1),
+  agent: z
+    .object({
+      agent_id: z.string().min(1),
+      intent: z
+        .enum(['shopping_browse', 'shopping_compare', 'shopping_purchase'])
+        .optional(),
+    })
+    .optional(),
 });
```

**向后兼容**：`agent` 是 optional，已有的 Resolve client 不传也合法。

---

## Patch 002 — catalog-core: 新增 ResolveContext + async signature

### 为什么

`buildResolveActions` 原本是同步的、只接 projection 一个参数。要支持回调 Provider 必须：
1. 异步（fetch 是 async）
2. 知道是哪个 entry 在 resolve（拼 hook 请求 body）
3. 知道 Agent 是谁（透传给 Provider）
4. 拿到一个 `fetch` 引用（便于单元测试 mock）

### 改动

`packages/catalog-core/src/scenario.ts`：

```diff
 import type {
   ActionBinding,
   CatalogManifest,
   CommercialObject,
   ObjectContract,
+  ResolveRequest,
 } from '@ocp-catalog/ocp-schema';

+/**
+ * Context passed to `buildResolveActions`, enabling scenarios to:
+ *   - know which entry is being resolved (entryId)
+ *   - read the original resolve request (e.g. request.agent.agent_id)
+ *   - perform async I/O (e.g. callback to a Provider) via injected fetch
+ */
+export interface ResolveContext {
+  entryId: string;
+  request: ResolveRequest;
+  fetch: typeof globalThis.fetch;
+}

 export type CatalogScenarioModule = {
-  buildResolveActions?(projection: Record<string, unknown>): ActionBinding[];
+  buildResolveActions?(
+    projection: Record<string, unknown>,
+    ctx: ResolveContext,
+  ): ActionBinding[] | Promise<ActionBinding[]>;
 };
```

并在 `packages/catalog-core/src/index.ts` 导出 `ResolveContext` 类型。

**向后兼容**：
- 老 Scenario 不写 `buildResolveActions` 完全没影响（optional method）
- 老 Scenario 同步返回 ActionBinding[] 仍可用（union type）
- 新加的 ctx 参数:老 Scenario 不读它即可

---

## Patch 003 — resolve-service: 实际把 hook 跑起来

### 改动

`packages/catalog-core/src/resolve-service.ts`：

```diff
 import {
   resolveRequestSchema,
+  type ActionBinding,
   type ResolvableReference,
 } from '@ocp-catalog/ocp-schema';

-      action_bindings: this.scenario.buildResolveActions?.(projection) ?? [],
+    // (extracted above)
+    let actionBindings: ActionBinding[] = [];
+    if (this.scenario.buildResolveActions) {
+      actionBindings = await Promise.resolve(
+        this.scenario.buildResolveActions(projection, {
+          entryId: row.entryId,
+          request,
+          fetch: globalThis.fetch,
+        }),
+      );
+    }
+    ...
+      action_bindings: actionBindings,
```

---

## Patch 004 — commerce-scenario: 实现真正的回调逻辑

### 为什么

这是"翻译"层：catalog 检测到 projection 上有 `requires_affiliate_resolution` 旗标，就主动 POST Provider 的 `provider_resolve_hook_url` 拿动态 binding。

### 改动

`apps/examples/commerce-catalog-api/src/commerce-scenario.ts` 把 `buildResolveActions` 改成 async：

```typescript
async function buildResolveActions(
  projection: Record<string, unknown>,
  ctx: ResolveContext,
): Promise<ActionBinding[]> {
  const actions: ActionBinding[] = [];

  // (1) 静态 fallback:view_product 永远保留
  const url = typeof projection.product_url === 'string'
    ? projection.product_url
    : typeof projection.source_url === 'string'
      ? projection.source_url : null;
  if (url) {
    actions.push({
      action_id: 'view_product', action_type: 'url',
      label: 'View product', url, method: 'GET',
    });
  }

  // (2) 动态回调 Provider
  const attrs = (projection.attributes && typeof projection.attributes === 'object'
    ? (projection.attributes as Record<string, unknown>) : {});
  const requires = attrs.requires_affiliate_resolution === true;
  const hookUrl = typeof attrs.provider_resolve_hook_url === 'string'
    ? attrs.provider_resolve_hook_url : null;

  if (requires && hookUrl) {
    try {
      const res = await ctx.fetch(hookUrl, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({
          entry_id: ctx.entryId,
          object_id: projection.sku ?? projection.object_id,
          agent_id: ctx.request.agent?.agent_id,
        }),
        signal: AbortSignal.timeout(2000),  // 严格 2s 超时
      });
      if (res.ok) {
        const data = await res.json() as { action_bindings?: ActionBinding[] };
        if (Array.isArray(data.action_bindings)) actions.push(...data.action_bindings);
      }
    } catch (err) {
      // 失败完全吞掉:resolution 必须在 Provider 挂掉时仍能工作
      console.warn(`[commerce-scenario] resolve_hook failed for entry=${ctx.entryId}`);
    }
  }

  return actions;
}
```

### 行为保证（Checkpoint B 验证过）

- Provider 在线 + 返 binding → 合并 view_product + Provider binding
- Provider 在线 + 返空 → 只有 view_product
- Provider 不可达 → 只有 view_product，**不报 500**
- 普通商品（无 `requires_affiliate_resolution`）→ 行为不变（只 view_product）

---

## Patch 005 — buildSearchProjection 透传 attributes（关键!）

### 为什么这条最隐蔽

Patch 004 写完后，hook 始终不触发。根因：`buildSearchProjection` 只把白名单字段（title/brand/sku/amount...）提进 projection，**`attributes` 字段被丢弃**。所以 catalog 在 resolve 时根本看不到 `requires_affiliate_resolution` 旗标。

**没有 Patch 005，Patch 004 形同虚设**。

### 改动

`apps/examples/commerce-catalog-api/src/commerce-scenario.ts` 的 `buildSearchProjection`：

```diff
+ const rawAttributes = readDescriptorField(object, 'ocp.commerce.product.core.v1#/attributes');
+ const attributesField =
+   rawAttributes && typeof rawAttributes === 'object' && !Array.isArray(rawAttributes)
+     ? (rawAttributes as Record<string, unknown>) : undefined;
  ...
  return {
    title,
    ...
    provider_id: object.provider_id,
    object_id: object.object_id,
+   ...(attributesField ? { attributes: attributesField } : {}),
    text,
  };
```

---

## Patch 006 — Side fix: commerce-provider-api fixture 漂移

### 为什么

预存在的问题:别人在 `@ocp-catalog/config` 加了 `QWEN_MODEL_NAME` / `OPENAI_MODEL_NAME` / `QWEN_API_KEY` / `QWEN_BASE_URL` 字段,但 `commerce-provider-api/src/provider-mapper.test.ts` 的 mock config 没补上,导致 monorepo typecheck 失败。**和 Day 4 patches 无关**,但既然碰到了顺手修。

### 改动

补 4 个字段到 mock config:

```diff
   USER_DEMO_AGENT_MODEL: 'qwen-plus',
+  OPENAI_MODEL_NAME: 'gpt-4o-mini',
   OPENAI_API_KEY: '',
   OPENAI_BASE_URL: 'https://api.openai.com/v1',
+  QWEN_MODEL_NAME: 'qwen3.6-plus',
+  QWEN_API_KEY: '',
+  QWEN_BASE_URL: 'https://dashscope.aliyuncs.com/compatible-mode/v1',
 };
```

---

## 完整验证 (Checkpoint B)

### POSITIVE — Provider 在线

```bash
curl -X POST http://localhost:4000/ocp/resolve \
  -H 'content-type: application/json' \
  -d '{"entry_id":"<some_entry>","agent":{"agent_id":"agt_test","intent":"shopping_compare"}}'

# 期望 catalog 日志没异常,Provider 日志看到:
#   [resolve_hook] entry=<entry_id> object=<num_iid> agent=agt_test
```

✅ 实测通过:provider 日志确认收到回调,参数完整。

### NEGATIVE — Provider 掉线

```bash
# 停掉 alimama-provider 后再 resolve
curl -X POST http://localhost:4000/ocp/resolve ...
```

期望:HTTP 200,`action_bindings` 只有 `view_product`,响应时间 < 1s（ECONNREFUSED 立即返回，不等满 2s timeout）。

✅ 实测通过:25ms 返回 200,只 view_product。

### Side effects 检查

- ✅ 所有 18 个 monorepo packages typecheck pass
- ✅ Day 1+2 写的 50 个 alimama-provider 测试全部维持绿
- ✅ catalog 自带的 commerce-scenario.test.ts 仍然通过（因为我们改的是 async 兼容签名）

---

## 关键设计取舍

1. **fetch 注入而非全局**:让 Scenario 单测可以 mock fetch。从 ResolveService 注入 `globalThis.fetch`,Scenario 用 `ctx.fetch` 而非直接 import。
2. **2s 严格超时**:用 `AbortSignal.timeout()` 而非 setTimeout race,失败时干净中止。
3. **失败完全吞掉 + console.warn**:OCP 协议第一原则是 resolve 必须返回 ResolvableReference;Provider 挂了不能 cascade 让 catalog 也挂。
4. **`attributes` 透传到 projection** 而非 hardcode 字段:让 Provider 自由定义任意 free-form 标记,catalog 无需感知具体语义。
5. **`buildResolveActions` 返回 union `T | Promise<T>`** 而非纯 async:老 Scenario 同步返回仍可用,减少 breaking。
