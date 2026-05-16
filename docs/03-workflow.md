# 执行 Workflow：真要做的话，从今天起一步一步怎么走

> 这是 [02-technical-design.md](02-technical-design.md) 的**执行版本**，按时间线展开。
> 每个阶段都有：动作清单 + 验证标准 + 决策点 + 卡点处理。
> 假设你已经决定走 OCP-Catalog 改造 + 自建 alimama-provider 路线。

---

## 总览：六个阶段，约 4–6 周

```
Day 0          Week 1          Week 2          Week 3          Week 4          Week 5+
  │              │               │               │               │               │
  ├─ 决策签字   ├─ 阶段一       ├─ 阶段二       ├─ 阶段三       ├─ 阶段四       ├─ 阶段五
  │  + 申请账号 │  Mock 跑通    │  Catalog       │  真实 Alimama  │  佣金回流     │  协议扩展
  │             │  (无凭据)     │  扩展          │  接入          │  + 监控       │  + 多联盟桥接
  │             │               │  +ResolveHook  │               │               │
```

每个阶段独立可交付。中途任何阶段终止，已交付的部分仍然可演示。

---

## Day 0 — 决策与准备（半天）

### 必须做的

1. **拉齐 [04-decisions.md](04-decisions.md) 里的前三项决策**（运营主体、agent 合规边界、是否走二房东）
   - 走法务 / 业务 / 技术三方各 1 人快速评审
   - 输出：每项 ✅ 或 ❌ 加签字时间

2. **注册测试用淘宝联盟账号**（5 分钟）
   - 进 https://pub.alimama.com/，用任意淘宝账号登录即可成为淘客
   - 完成最低门槛"媒体备案"（个人/App/网站 三选一，PoC 阶段选哪个都行）
   - 申请基础 AppKey（自动审核，1-3 工作日，但通常 1 天内）

3. **后台手动创建 1 个 adzone**（推广位）
   - 路径：阿里妈妈后台 → 推广管理 → 推广位管理 → 新建
   - 记下 PID（`mm_x_y_z` 格式）和 `adzone_id`（PID 最后一段）
   - PoC 阶段全部 Agent 共用这一个 adzone

### 卡点处理

- AppKey 审核没过 → 阶段一仍然能做（mock 模式不需要真实 AppKey）
- 法务说"不能走"→ 看 [04-decisions.md](04-decisions.md) 的退路（C：私有/封闭 Agent 模式）

### 完成判据

- [ ] 决策清单前三项有签字
- [ ] 拿到测试 AppKey + AppSecret + 至少 1 个 adzone_id
- [ ] 这些凭据安全存好（不入 git，建议 1Password / 公司 vault）

---

## 阶段一 — Mock 模式跑通（Week 1，1-2 天有效工时）

**目标**：完全脱离真实 Alimama，验证 OCP 协议字段映射 + sync 链路 + provider 服务骨架可工作。

### 工程任务

1. 在 OCP-Catalog 仓库新建 `apps/examples/alimama-provider-api/`
   - 复制 `commerce-provider-api` 结构作为模板
   - 替换业务逻辑为 Alimama 适配

2. 实现 `alimama/client.ts` 的 mock 模式
   ```typescript
   if (process.env.ALIMAMA_MOCK === 'true') {
     return JSON.parse(fs.readFileSync('./fixtures/material-optional-sample.json', 'utf8'));
   }
   ```

3. 准备 fixture 数据
   - 在 [fixtures/](../fixtures/) 放 1 份手造的 material.optional 响应（10-20 个商品）
   - 字段参考 [02-technical-design.md §6](02-technical-design.md#6-字段映射technical-视角)

4. 实现 `mapper/material-to-object.ts`
   - 单元测试用 fixture 输入做快照测试
   - 重点测试边界：相对图片 URL → 绝对化、category ID → 文本、价格字符串 → number

5. 实现 `workers/material-poller.ts`
   - 从 mock 拉 → 映射 → POST /ocp/objects/sync
   - 跑一次后退出（cron 留到后面）

6. 实现 `services/catalog-client.ts`
   - 调 commerce-catalog-api 的 `/ocp/providers/register` 和 `/ocp/objects/sync`

### 启动命令

```bash
cd e:/homework/work/OCP-Catalog
bun install
bun run db:migrate
bun run commerce:catalog:api &

ALIMAMA_MOCK=true \
OCP_CATALOG_BASE_URL=http://localhost:4000 \
OCP_API_KEY=dev-api-key \
OCP_PROVIDER_ID=alimama_local \
PROVIDER_PORT=4300 \
bun run --cwd apps/examples/alimama-provider-api start
```

### 验证

```bash
# 触发一次同步
curl -X POST http://localhost:4300/admin/trigger-sync \
  -H "content-type: application/json" \
  -d '{"q":"无线耳机","pageSize":20}'

# 查询(应该命中 mock 数据)
curl -X POST http://localhost:4000/ocp/query \
  -H "content-type: application/json" \
  -d '{"query_pack":"ocp.query.keyword.v1","query":"耳机","limit":5}'
# 期望:返回 items[],含 mock 商品的 title、image
```

### 完成判据

- [ ] alimama-provider 能起来、bind 到端口 4300
- [ ] 触发同步后 catalog 数据库里能看到 commercialObjects 表新增行
- [ ] OCP query 能查到 mock 商品
- [ ] mapper 单元测试覆盖：URL 绝对化、价格解析、commission_rate 基点、空 small_images
- [ ] 整个流程不需要任何真实 Alimama 凭据

---

## 阶段二 — Catalog 扩展 + ResolveHook（Week 2，2-3 天）

**目标**：让 catalog 在 resolve 时能回调 provider 拿动态 ActionBinding。这是**最关键的协议级改动**。

### 工程任务

#### Catalog 改动（OCP-Catalog 仓库）

1. **修改 ocp-schema**：在 `packages/ocp-schema/src/index.ts` 给 ResolveRequest 加可选 `agent` 字段
   - 参考 [02-technical-design.md §4.3](02-technical-design.md#43-协议层扩展resolve-请求加-agent-字段)
   - 向后兼容（agent 可选）

2. **修改 catalog-core Scenario interface**：在 `packages/catalog-core/src/scenario.ts`
   - 把 `buildResolveActions(projection)` 改成 `buildResolveActions(projection, ctx): Promise<ActionBinding[]> | ActionBinding[]`
   - 新增 `ResolveContext` interface（含 `request`、`fetch`、`entryId`、`catalogEntryRow`）

3. **修改 ResolveService**：在 `packages/catalog-core/src/resolve-service.ts`
   - 把对 `buildResolveActions` 的调用改成 `await Promise.resolve(scenario.buildResolveActions(projection, ctx))`
   - 注入 ctx

4. **修改 commerce-scenario.ts**：实现回调
   - 检测 `projection.attributes.requires_affiliate_resolution`
   - 取 `attributes.provider_resolve_hook_url`
   - POST 该 URL，传 `{ entry_id, object_id, agent_id }`
   - 严格超时（2s），失败降级到不返 affiliate binding
   - 参考 [02-technical-design.md §4.2 Option A](02-technical-design.md#42-改造方案)

5. **把这些 diff 整理到 [../patches/](../patches/)**
   - 文件名：`patch-001-resolve-hook.md`（描述 + diff）
   - 后续可作为对 OCP-Catalog 仓库的 PR

#### Provider 改动

6. 在 alimama-provider 加 `/provider/resolve_hook` POST 端点
   - 暂时返回静态 mock：`{ action_bindings: [{ action_id: 'buy_with_coupon', action_type: 'url', url: 'https://s.click.taobao.com/mock', label: '去淘宝购买', method: 'GET' }] }`
   - 真实转链留到阶段三

7. mapper 输出的 CommercialObject 加 `attributes`：
   ```typescript
   attributes: {
     requires_affiliate_resolution: true,
     provider_resolve_hook_url: 'http://localhost:4300/provider/resolve_hook',
     affiliate_provider: 'alimama_taobao_union',
     // ...
   }
   ```

### 验证

```bash
# resolve 一个 entry,带 agent
curl -X POST http://localhost:4000/ocp/resolve \
  -H "content-type: application/json" \
  -d '{
    "entry_id":"<某个 mock 商品 entry>",
    "agent":{"agent_id":"agt_test_001","intent":"shopping_compare"}
  }'

# 期望:action_bindings 里看到 mock 的 affiliate URL
# 同时 alimama-provider 日志能看到 /provider/resolve_hook 被调用,带上正确的 entry_id 和 agent_id
```

### 负向验证

```bash
# 把 alimama-provider 停掉,再 resolve 一次
# 期望:catalog 仍然返回 view_product binding(降级路径),不抛 500
```

### 完成判据

- [ ] resolve 请求带 agent 字段不会被 schema 校验拒绝
- [ ] catalog 收到 resolve 后日志里能看到 hook 调用
- [ ] provider /provider/resolve_hook 收到正确 payload（entry_id + object_id + agent_id）
- [ ] provider 返的 binding 出现在 resolve 响应里
- [ ] provider 不可达时 catalog 返回退化结果，不阻塞
- [ ] 4 个 patch（schema + scenario + resolve-service + commerce-scenario）整理在 [../patches/](../patches/)

---

## 阶段三 — 真实 Alimama 接入（Week 3，3-5 天）

**目标**：把 alimama-provider 从 mock 切到真实 Alimama API。

### 前置

- 阶段一二跑通
- AppKey 已审核通过
- 至少 1 个 adzone_id 可用
- **看链接 API 权限有没有批**：
  - 如果批了 → 走 3.A 路线
  - 如果没批 → 走 3.B 路线（二房东兜底）或 3.C 路线（不转链）

### 工程任务

#### 通用部分

1. 实现 `alimama/sign.ts`（top md5 / hmac-sha256 签名）
2. 实现 `alimama/client.ts` 真实 HTTP 调用 `gw.api.taobao.com/router/rest`
3. 把 fixture 替换：用一次真实 `material.optional` 调用的脱敏响应（人造的可能与真实有出入）
4. 跑 mapper 单测，看真实数据是否打破任何假设
5. 调整 mapper 至全部测试过

#### 3.A — 有链接 API 权限

6.A 实现 `taobao.tbk.privilege.get` 调用
7.A `resolve_hook` 改为：查 adzone_id（PoC 阶段固定一个）→ 调 privilege.get → 拼 binding（含 coupon_click_url）
8.A 加 `taobao.tbk.tpwd.create` 作为备选 binding（淘口令）

#### 3.B — 无链接 API 权限，走二房东

6.B 评估 [订单侠](https://www.dingdanxia.com/price)、维易、淘口令网 三家
   - 看接口稳定性、定价、合规说明
   - 决策放进 [04-decisions.md](04-decisions.md)
7.B 在 `alimama/client.ts` 加 `generatePrivilegeLink` 路由到二房东 HTTP API
8.B `resolve_hook` 与 3.A 一致，底层 client 不同

#### 3.C — 不转链，纯展示

6.C `resolve_hook` 直接返 `view_product` binding，URL 用 `item_url`（原始非 affiliate）
   - 不能归因佣金，但链路是完整的
   - 适合演示协议层，不适合上线赚钱

### 验证

```bash
# 真实关键词跑同步
curl -X POST http://localhost:4300/admin/trigger-sync \
  -H "content-type: application/json" \
  -d '{"q":"无线耳机","pageSize":20}'

# 看 catalog 数据库:title 是真实商品标题,image_urls 是 alicdn 真实图,item_url 是 detail.tmall.com 或 detail.taobao.com

# resolve
curl -X POST http://localhost:4000/ocp/resolve \
  -H "content-type: application/json" \
  -d '{"entry_id":"<真实 entry>","agent":{"agent_id":"agt_test_001"}}'
# 3.A/3.B:binding.url 应是 s.click.taobao.com/* 短链
# 3.C:binding.url 是普通商品页

# 真机测试(关键)
# 把 binding.url 复制到手机浏览器打开,应该自动唤起手淘 App
# 3.A/3.B:手淘里能看到券、能下单(下完取消即可,验证归因)
```

### 完成判据

- [ ] 真实 `material.optional` 调用成功，返回数据被 mapper 正确处理
- [ ] mapper 单测全部通过（包括边界场景）
- [ ] 3.A 或 3.B：resolve 拿到带 PID 的真实短链
- [ ] 手机点开短链能唤起手淘
- [ ] 真实下一单测试：下单成功，订单状态在联盟后台可见

---

## 阶段四 — 佣金回流 + 监控（Week 4，3-4 天）

**目标**：把订单数据拉回来，按 agent_id 聚合，建监控。

### 前置

- 阶段三的真实链路跑通至少 3 天
- 真实产生过 ≥ 1 单测试订单（自购或邀请测试）

### 工程任务

1. 实现 `db/schema.ts` 加 `commission_ledger` 表
   - 字段参考 [02-technical-design.md §9](02-technical-design.md#9-commission_ledger-表)
   - 跑 drizzle 迁移

2. 实现 `workers/order-poller.ts`
   - 每天定时调 `taobao.tbk.order.get`
   - 时间窗口：前 7 天（覆盖状态变化）
   - upsert 到 `commission_ledger`（按 `alimama_trade_id` 唯一约束）

3. 实现 `mapper/order-to-ledger.ts`
   - 状态映射：alimama 的"订单付款/订单成交/订单结算/订单失效/订单维权" → ledger.order_status
   - 维权扣回时的处理

4. 实现 `services/commission-ledger.ts` 的 join 视图
   - `agent_adzone_mapping` JOIN `commission_ledger` ON adzone_id
   - 按 agent_id 聚合：GMV、估算佣金、实际佣金、净额

5. 简单 admin UI（可选）
   - 一个 HTML 页面展示按 agent 的日 / 月聚合
   - 不需要复杂前端，Server-rendered table 就行

### 验证

```bash
# 触发一次回流
curl -X POST http://localhost:4300/admin/trigger-order-sync

# 看 commission_ledger 表
docker exec -it ocp-catalog-postgres-wsl psql -U ocp -d ocp_catalog -c \
  "SELECT order_status, count(*), sum(pay_amount), sum(estimated_commission) FROM commission_ledger GROUP BY order_status;"

# 期望:看到测试订单按状态分组的金额
```

### 完成判据

- [ ] 真实订单出现在 ledger 表，幂等（重复回流不产生重复行）
- [ ] 订单状态变化能正确反映（付款 → 成交 → 结算）
- [ ] 维权订单能正确扣回
- [ ] 能按 agent_id 聚合查询
- [ ] 监控告警接好（至少：调用失败率 > 5%、回流断流 > 24h）

---

## 阶段五 — 协议扩展 + 多联盟桥接（Week 5+）

**目标**：把这次的实践成果反哺到 OCP 协议本身。

### 工程任务

1. **正式提案 `ocp.commerce.commission.v1`**
   - 写 schema 进 `packages/ocp-schema/src/index.ts`
   - 写 RFC 文档放 OCP-Catalog `docs/rfcs/`
   - 走 OCP-Catalog 仓库的 PR 流程

2. **正式提案 ResolveContext / async Scenario**
   - 同上路径

3. **per-agent adzone 池**（如果阶段四数据证明单 adzone 风控有压力）
   - 实现 `services/adzone-allocator.ts`：lazy 创建 / 池中分配
   - 注意：adzone 创建可能没 API，要 RPA 或人工预创建池
   - 真要做之前先调研一次"adzone 单账号上限"

4. **多联盟桥接**（按 [01-research.md §6](01-research.md#6-跨联盟参照京东联盟--拼多多--抖音) 设计）
   - 在 alimama-provider 之外新建 `jd-provider`, `pdd-provider`
   - 复用 alimama-provider 的目录结构和 OCP 适配模式
   - 抽公共部分到 `packages/affiliate-provider-common/`

### 完成判据

- [ ] OCP-Catalog 主仓接受 commission.v1 + ResolveContext 提案
- [ ] adzone 池在生产能稳定支撑 ≥ 100 个 agent 并发
- [ ] 至少接入第二个联盟（JD 或 PDD）

---

## 卡点处理速查

| 现象 | 阶段 | 处理 |
|---|---|---|
| AppKey 审核迟迟不批 | Day 0 | 进阶段一（mock 模式不需要） |
| 链接 API 拿不到 | 阶段三 | 走二房东（3.B）或暂时不转链（3.C） |
| alimama 限流 isv.access-limit | 阶段三/四 | 退避 + 监控；考虑多 AppKey 池 |
| Catalog resolve P95 > 3s | 阶段二/三 | 加 privilege.get 结果缓存（key=itemId:agentId，TTL 5min） |
| 单 adzone 被风控冻结 | 阶段四 | 立刻切到备用 adzone；调查异常调用模式 |
| mapper 校验 image_urls 失败 | 阶段一/三 | 检查 `pict_url` 是否缺 https: 前缀；强制 absolutize |
| 维权扣回不反映在 ledger | 阶段四 | 拉单窗口拉长到 30 天；检查状态映射 |
| OCP-Catalog 主仓拒绝 commission.v1 提案 | 阶段五 | 自己 fork 维护；descriptor pack 本来就是开放扩展点，不阻塞使用 |

---

## 每周交付物（Demo 友好）

| 周末 | 能演示什么 |
|---|---|
| Week 1 | "Agent 用 OCP 查到了我的商品库" —— mock 数据，全本地 |
| Week 2 | "Agent resolve 时 catalog 回调 provider 动态生成了 binding" —— 协议级 demo |
| Week 3 | "用户点 binding 跳手淘，能下单" —— 真实商品 + 真实链接 |
| Week 4 | "我能看到哪个 agent 带来了多少 GMV" —— Dashboard demo |
| Week 5+ | "OCP 接了第二个联盟" —— 桥接架构印证 |

---

## 如果只能做一周

只做阶段一 + 阶段二的前两步（schema + scenario 改造）：

- 写 alimama-provider 骨架
- mock 模式跑通同步
- catalog 加 ResolveContext 但 commerce-scenario 不改

这个最小版本能证明**协议可扩展性 + 适配器模式可行**，作为内部技术评审材料足够。

---

## 决策依赖

执行前必须完成 [04-decisions.md](04-decisions.md) 顶部三项决策；其余决策可以延后到对应阶段。
