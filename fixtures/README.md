# fixtures/

> ⚠️ **真正的 fixtures 不在这里**。

3 份 mock 数据样本住在 OCP-Catalog monorepo 里：

```text
https://github.com/Open-Commerce-Protocol/OCP-Catalog/tree/main/apps/examples/alimama-provider-api/tests/fixtures/
├── material-optional-sample.json   ← 6 个商品,14 边界场景全覆盖
├── privilege-get-sample.json       ← 转链响应(含淘宝券 + 天猫专属券)
└── order-get-sample.json           ← 4 个订单,4 种状态(paid / settled / invalid / under_dispute)
```

之所以不放本目录，是因为它们和 mapper 测试紧耦合，放一起更直观。

## 已覆盖的边界场景

### material-optional-sample.json

| Item | 平台 | 图片场景 | 券 | 备注 |
| --- | --- | --- | --- | --- |
| #1 | 天猫 user_type=1 | `pict_url` 无 scheme + 3 张 small_images 也无 scheme | 满 199-50 | 高佣 15.5% |
| #2 | 淘宝 user_type=0 | `pict_url` `https://` + small_images `null` | 无 | 中佣 8% |
| #3 | 天猫 | `pict_url` 无 scheme + `small_images: {string: []}` 空数组 | 满 399-30 | |
| #4 | 淘宝 | 1 张 small_image + 券剩 12 张快用完 | 满 10-2 | |
| #5 | 天猫 | 2 张 small_images | 无 | 超高佣 25% |
| #6 | 淘宝 | **缺 small_images 字段、缺 shop_title、缺 commission_rate** | 无 | 测 mapper 鲁棒性 |

### order-get-sample.json

| trade_id | item | tk_status | 含义 | 金额 |
| --- | --- | --- | --- | --- |
| 9001234567890001 | Tmall 蓝牙耳机 | **13 settled** | 已结算,佣金到账 27.77 元 | 付 199 元 |
| 9001234567890002 | Taobao 充电宝 | **12 paid** | 用户付款,佣金预估 | 付 59 元 |
| 9001234567890003 | Tmall 电竞鼠标 | **14 invalid** | 退款/取消 | 付 249 元 |
| 9001234567890004 | Taobao 数据线 | **15 under_dispute** | 维权中 | 付 25.80 元 ×2 |

### privilege-get-sample.json

含 `coupon_click_url`、`mm_coupon_click_url`（天猫专属券，URL 不同）、`coupon_info` 等，用于测试 mapper 同时产出 2 个独立 binding。

## 真实脱敏指引（拿到真实 AppKey 后）

```bash
# 假设 fixtures/raw-material.json 是真实响应,要脱敏后替换 fixture:
jq '
  .tbk_dg_material_optional_response.result_list.map_data[].seller_id = 999999
  | .tbk_dg_material_optional_response.result_list.map_data[].shop_title = "测试店铺"
' fixtures/raw-material.json > fixtures/material-optional-sample.json
```
