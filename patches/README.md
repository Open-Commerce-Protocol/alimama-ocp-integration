# patches/

记录对 OCP-Catalog 仓库的所有修改（PR 形式或独立 commit），按 Day 编号组织。

## 应用状态

| Day | Patches | 文件 | 状态 |
| --- | --- | --- | --- |
| Day 4 | 5 个核心 + 1 个 side-fix | [day-4-changes.md](day-4-changes.md) + [day-4-all-changes.patch](day-4-all-changes.patch) | ✅ 应用到 OCP-Catalog 工作树（未 commit） |

## Day 4 总览

让 OCP-Catalog 在 resolve 时支持回调 Provider 拿动态 ActionBinding。

```text
6 files changed, 150 insertions(+), 11 modifications
向后兼容(老的同步 Scenario 仍可工作)
全部 18 packages typecheck 通过
```

详细见 [day-4-changes.md](day-4-changes.md)。

## 应用 / 回滚

```bash
cd e:/homework/work/OCP-Catalog

# 看当前 working tree 改动
git diff HEAD -- packages/ocp-schema/src/index.ts packages/catalog-core/ apps/examples/commerce-catalog-api/src/commerce-scenario.ts apps/examples/commerce-provider-api/src/provider-mapper.test.ts

# 回滚全部 Day 4 改动(注意:会清掉所有修改)
git checkout HEAD -- packages/ocp-schema/src/index.ts packages/catalog-core/ apps/examples/commerce-catalog-api/src/commerce-scenario.ts apps/examples/commerce-provider-api/src/provider-mapper.test.ts

# 重新应用(已保存的 unified diff)
git apply e:/homework/work/alimama-ocp-integration/patches/day-4-all-changes.patch
```

## 最终去向

完成 PoC 阶段后,这些 patch 会以 PR 形式提到 OCP-Catalog 仓库
（或保留在 fork 里维护）。本目录持续作为变更日志。
