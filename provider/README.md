# provider/

> ⚠️ **真正的代码不在这里**。

`alimama-provider-api` 服务的代码住在 OCP-Catalog monorepo 里：

```text
e:/homework/work/OCP-Catalog/apps/examples/alimama-provider-api/
```

之所以不放在本目录，是为了复用 OCP-Catalog 的 workspace 链接（`@ocp-catalog/ocp-schema` 等）、Drizzle 配置、`bun run` 工程入口。

## 当前真实代码概况（Day 1-Day 6 完成态）

```text
apps/examples/alimama-provider-api/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts                       Elysia 服务入口 + cron 启动
│   ├── config.ts                      zod 校验 env(11 个字段)
│   ├── alimama/
│   │   ├── client.ts                  AlimamaClient(mock + real 双模式)
│   │   ├── sign.ts                    top md5 签名
│   │   └── types.ts                   完整 TS 类型 + isAlimamaError guard
│   ├── mapper/
│   │   ├── material-to-object.ts      物料 → CommercialObject
│   │   ├── privilege-to-action.ts     转链 → ActionBinding[]
│   │   └── order-to-ledger.ts         订单 → LedgerEntry
│   ├── http/
│   │   ├── admin.ts                   /admin/register、/sync、/sync-orders、/stats、/ledger
│   │   └── resolve-hook.ts            /provider/resolve_hook(被 catalog 回调)
│   ├── workers/
│   │   ├── material-poller.ts         cron 拉物料
│   │   └── order-poller.ts            cron 拉订单
│   └── services/
│       ├── catalog-client.ts          OCP Catalog HTTP 客户端
│       ├── registration.ts            ProviderRegistration builder
│       └── commission-ledger.ts       内存 Map ledger(Day 7+ 迁 Postgres)
└── tests/
    ├── fixtures/
    │   ├── material-optional-sample.json (6 商品,14 边界覆盖)
    │   ├── privilege-get-sample.json
    │   └── order-get-sample.json (4 订单,4 状态)
    ├── sign.test.ts                   (8)
    ├── mapper.test.ts                 (21)
    ├── config.test.ts                 (7)
    ├── client.test.ts                 (7)
    ├── registration.test.ts           (7)
    ├── privilege-to-action.test.ts    (9)
    ├── resolve-hook.test.ts           (7)
    ├── order-to-ledger.test.ts        (7)
    └── commission-ledger.test.ts      (8)
                                       共 79 个测试,全部通过
```

## 跑起来

```bash
# 1. 起 Postgres 容器 + OCP catalog
cd e:/homework/work/OCP-Catalog
docker compose up -d postgres        # 或 docker start ocp-catalog-postgres-wsl
bun run commerce:catalog:api &       # :4000

# 2. 起 provider(mock 模式,默认)
ALIMAMA_MOCK=true \
OCP_CATALOG_BASE_URL=http://localhost:4000 \
OCP_CATALOG_ID=cat_local_dev \
OCP_PROVIDER_ID=alimama_local \
OCP_API_KEY=dev-api-key \
OCP_PROVIDER_BASE_URL=http://localhost:4300 \
PROVIDER_PORT=4300 \
bun run --cwd apps/examples/alimama-provider-api start
```

## 为什么本目录还留着

未来如果决定把 `alimama-provider` 抽成独立仓库（脱离 OCP-Catalog monorepo），可以作为 staging area。

设计文档参考 [../docs/05-implementation.md §3.2](../docs/05-implementation.md)。
