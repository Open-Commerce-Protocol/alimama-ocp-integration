# 阿里妈妈推广链接 × OCP Catalog 深度调研

> 调研对象：阿里巴巴旗下"阿里妈妈"（核心产品线：**淘宝联盟 / 淘宝客**）的推广链接体系，及其与 OCP Catalog 协议结合的可能性、合规边界和战略空间。
> 调研日期：2026-05-13
> 状态：研究稿，含若干需要业务/法务/运营拍板的决策点

---

## 0. TL;DR

1. "推广链接"是**淘宝联盟**的 CPS 联盟营销链路：用户点 `s.click.taobao.com/xxx` → 跳转商品 → 下单 → 联盟按 PID 把佣金结算给对应的"淘客"。PID 是 `mm_<memberid>_<site_id>_<adzone_id>` 三段结构，**`adzone_id`（推广位 ID）是订单归因的唯一原子**。
2. 与 OCP Catalog 最自然的耦合点是 **resolve 阶段动态生成 affiliate URL**——每个发起 resolve 的 Agent 可以分配一个独立 adzone_id，自然实现"千 Agent 千归因"。
3. **三道合规红线**严重约束玩法：(a) API 权限"不得擅自开放给第三方调用"；(b) 链接 API 是邀请制（需流量证明）；(c) PID 异常复用被风控。最严苛地解读，**OCP 必须把"Agent"定义为同一主体下的内部渠道**，而不是任意第三方。
4. **2026 年全球已经在打 Agentic Commerce 协议战**：OpenAI ACP、Google UCP、Klarna APP、Visa TAP、Mastercard Agent Pay 等八家并立。**国内的 OCP Catalog 在中文电商生态有结构性空间**（ChatGPT/UCP 在中国可达性受限），但需要趁早做出标杆案例和与阿里妈妈/京东联盟/拼多多多多客的桥接。
5. 建议分三阶段：**短期 PoC（基础 API 起步，不依赖高级权限）→ 中期主体准入（接链接 API 走单一 PID）→ 长期"每 Agent 一 PID"+ 协议层加 `ocp.commerce.commission.v1` descriptor 扩展**。

---

## 1. 阿里妈妈 vs 淘宝联盟：清场

很多人把这两者混着用，但调研要做区分。

| 维度 | 阿里妈妈 | 淘宝联盟 |
|---|---|---|
| 定位 | 阿里巴巴集团**全域营销平台**，面向商家 | 阿里妈妈下面的**CPS 联盟营销**子平台，面向流量主/淘客 |
| 入口 | https://www.alimama.com/ | https://www.alimama.com/aff/ , https://pub.alimama.com/ |
| 收入模型 | 商家付费投放（CPC / CPM / 品牌广告） | 成交后按佣金分润（CPS） |
| 主要产品 | 万相台无界版、UniDesk、引力魔方、直通车、品牌专区、**AI 万相** | 淘宝客（流量主侧）+ 阿里妈妈联盟（商家侧设置佣金） |
| 是否相关 | 🔴 不相关（与 Agent 流量关系小） | 🟢 **本次调研核心** |
| API 集合 | 阿里妈妈 API（全集） | 淘宝客 API（子集，`taobao.tbk.*` 命名空间） |

来源：[阿里妈妈 API 与淘宝联盟 API 区别](https://developer.aliyun.com/article/1678436)、[淘宝联盟·生态伙伴](https://pub.alimama.com/)

**结论**："推广链接"几乎专指淘宝联盟域名 `s.click.taobao.com` 体系（淘客链接），所以本报告余下的所有"推广链接"等同于"淘宝联盟推广链接"。

---

## 2. PID 归因模型（最核心的一节）

### 2.1 PID 三段结构

```
PID = mm_<memberid>_<site_id>_<adzone_id>
       │      │           │           │
       │      │           │           └── 推广位 ID：渠道级归因原子
       │      │           └────────────── 媒体 ID：一个站/App 一个
       │      └────────────────────────── 联盟账号 ID：实体级
       └───────────────────────────────── 固定前缀
```

来源：[淘宝联盟 PID 详解](https://www.dingdanxia.com/article/5.html)

### 2.2 归因和结算

- "结算归属**完全依赖**推广 PID，一旦发起佣金结算，联盟会以**订单成交时**对应的 PID 为准。"
- 用错 PID → **佣金清零**，无法追溯
- API 调用时 `adzone_id` + `site_id` + `app_key` 三者必须配套，错配报 `isv.invalid-parameter-adzone`

### 2.3 PID 优先级规则（用户多重身份时）

当用户同时被多个 PID "标记"时（例如先在站外点了 A 链接、又在站内点了 B 链接），归因优先级：

1. **站外 PID > 站内 PID**
2. **非淘宝用户的 PID > 淘宝用户的 PID**
3. 同等条件下，**后点击的 PID > 先点击的 PID**

### 2.4 一个 Agent 一个 adzone_id：天然映射

由于 adzone_id 是细粒度归因的最小单元、平台支持批量创建多个 adzone，**OCP × Alimama 场景的最佳实践就是**：

```
Agent.agent_id  ←(1:1 映射)→  Alimama.adzone_id
```

好处：
- 每个 Agent 带来的成交可以独立统计 GMV / 佣金
- 跨 Agent 比较"导购质量"成为可能
- 风控视角：每个 Agent 的流量隔离，单个 Agent 异常不影响其他

代价：
- 需要管理一张 `agent_id → adzone_id → adzone metadata` 表
- 触发 API 调用配套校验（adzone_id ↔ site_id ↔ app_key）

---

## 3. 淘宝客 API 表面（按权限分层）

### 3.1 三个权限层

来源：[淘宝联盟 API 申请流程 2025](https://blog.csdn.net/api_open/article/details/149245815)、[阿里妈妈 API 与淘宝联盟 API 区别](https://developer.aliyun.com/article/1678436)

| 等级 | 包含 API | 准入 |
|---|---|---|
| **初级权限包**（自动） | 商品搜索 / 详情 / 店铺查询 / 关联推荐 / 定向招商 / 联盟选品库 | 媒体备案 + AppKey 申请，1–3 工作日审核 |
| **链接 API**（邀请制） | `taobao.tbk.privilege.get`、`taobao.tbk.item.convert`、`taobao.tbk.spread.get`、`taobao.tbk.tpwd.create` | **月日均点击 ≥ 5 万** OR **月日均 alipay ≥ 10 万** |
| **三方 API**（邀请制） | 三方分成转化（向二级 publisher 分佣） | 邀请制 |

### 3.2 核心 API 速查表

| API | 类别 | 功能 | OCP 角色 |
|---|---|---|---|
| `taobao.tbk.dg.material.optional` / `.upgrade` | 基础 | **物料/商品搜索**：按关键词、类目、价格区间、佣金率排序取候选商品 | **Provider → Catalog 同步数据源** |
| `taobao.tbk.item.info.get` | 基础 | 商品详情批量查询 | Provider 拉详情/refresh |
| `taobao.tbk.dg.material.recommend` | 基础 | 关联/相似商品推荐 | Provider 富化 |
| `taobao.tbk.coupon.get` | 基础 | 优惠券信息查询 | Provider enrichment |
| `taobao.tbk.privilege.get` | **链接** | **单品券高效转链**：商品 ID + adzone_id → 带 PID 的 affiliate URL（含券）| **Resolve 时动态生成 ActionBinding** |
| `taobao.tbk.item.convert` | **链接** | 普通商品链接转 affiliate | 无券场景的转链 |
| `taobao.tbk.spread.get` | **链接** | 长链转短链 | 可选美化 |
| `taobao.tbk.tpwd.create` | **链接** | 生成"淘口令"（13 字符分享码） | 移动端唤起手淘的辅助 action |
| `taobao.tbk.tpwd.convert` | **链接** | 淘口令解析 | 可选反向解析 |
| `taobao.tbk.order.get` | 基础 | 推广订单查询 | **佣金回流 / Agent attribution** |

### 3.3 物料搜索（material.optional）字段表

来源：[taobao.tbk.dg.material.optional 字段说明](https://www.taokeshow.com/51162.html)

返回结构：`tbk_dg_material_optional_response.result_list.map_data[]`，单个商品字段：

**商品基础**
- `num_iid`：商品 ID（主键）
- `title`：标题
- `pict_url`：主图 URL
- `small_images.string[]`：副图列表
- `item_url`：商品链接（**非 affiliate**）
- `provcity`：所在地
- `volume`：30 天销量
- `cat`：类目（数字 ID）
- `user_type`：卖家类型（0 = 淘宝、1 = 天猫）

**价格**
- `reserve_price`：一口价（吊牌价）
- `zk_final_price`：折扣价

**佣金/淘客信息**
- `commission_rate`：佣金率（基点形式，`1550` = 15.5%）
- `tk_total_sales`：淘客 30 天销量
- `tk_total_commi`：淘客 30 天佣金

**店铺**
- `seller_id`：卖家 ID
- `shop_title`：店铺名

**优惠券**
- `coupon_id` / `coupon_info` / `coupon_start_time` / `coupon_end_time` / `coupon_total_count` / `coupon_remain_count`
- `coupon_click_url`：券链接

**升级版 `.upgrade` 还多**：
- **预估到手价**（含优惠路径）
- **预热到手价**
- **预估凑单价**
- **更多优惠**（其他优惠信息）
- **商品利益点标签**（88VIP / 花呗免息 / 猫超买返 等）

升级版对 Agent 比"原价"更有用——Agent 用户问"多少钱"，应该回到手价不是吊牌价。

### 3.4 `taobao.tbk.privilege.get`：转链 API 的输入输出

来源：[taobao.tbk.privilege.get 文档](https://developer.alibaba.com/docs/api.htm?apiId=28625)

**关键入参**：
- `item_id`：商品 ID（必填）
- `adzone_id`：**推广位 ID（必填，决定归因）**
- `site_id`：媒体 ID
- `platform`：1 = PC，2 = 无线
- `promotion_type`：推广类型
- `relation_id`：渠道关系 ID（可选，用于三方分成）
- `external_id`：**外部业务 ID（可选）→ 这是把 OCP entry_id / session_id 透传进归因系统的钩子**
- `xid`：跨平台 ID
- `special_id`：特殊渠道 ID

**关键出参**（`tbk_privilege_get_response.result.data`）：
- `coupon_click_url`：**带 PID 的券链接 → 这就是 ActionBinding 的 URL**
- `item_url`：商品深链
- `coupon_info`、`coupon_start_time`、`coupon_end_time`、`coupon_remain_count`
- `max_commission_rate`：最大佣金率
- `mm_coupon_*`：阿里妈妈专属券信息

**重要属性**：`external_id` 字段允许把 OCP 的 `entry_id` 或 `agent_id` 透传进归因系统，这是构建 Agent 级 attribution 的关键钩子（具体作用机制待业务确认）。

---

## 4. 佣金结算与退款的完整生命周期

来源：[淘宝联盟佣金结算解析](https://www.alimama.com/news_detail.htm?contentId=1173)、[淘宝联盟结算流程详解](https://www.taokeshow.com/18992.html)、[淘宝客佣金结算规则](https://www.kaitao.cn/article/20250731131017.htm)

### 4.1 状态机

```
[拍下付款]
    ↓
[预估收入]  ← 实时显示在淘客后台
    ↓
[买家确认收货 / 自动确认]
    ↓
[当月已确认订单池]
    ↓
[次月 20 号月结]  ←  扣除技术服务费(佣金的10%) + 税
    ↓
[结算到账]
    ↓
（可能） [4 个月内维权退款]  →  补扣已发佣金
```

### 4.2 几条对 OCP attribution 至关重要的事实

1. **结算依据是"确认收货时间"，不是"下单时间"**——这意味着 Agent attribution 的延迟可能长达 1–3 个月
2. **退款会扣回佣金**——OCP 的 attribution metric 必须支持"撤销"，不能记一笔确认一笔
3. **技术服务费 = 佣金 × 10%**，税 = 平台补贴 × 20%（天猫订单）——净收益要从总佣金里扣
4. **4 个月维权窗口**——长尾退款。OCP commission ledger 至少要保留 5 个月历史可改
5. **每月 20 号月结**——传统淘客是月结，OCP 想做实时榜单只能取"预估收入"，最终结算要等月结回执

### 4.3 这对 OCP `commission.v1` 扩展的设计含义

OCP 现有协议没有 commission descriptor。若要扩展，至少需要：

```typescript
{
  pack_id: 'ocp.commerce.commission.v1',
  data: {
    affiliate_provider: 'alimama_taobao_union' | 'jd_union' | ...,
    commission_rate_bp: number,          // 基点
    estimated_commission: { 
      currency: 'CNY', 
      amount: number,                     // 基于单价 + 数量估算
      reliable: boolean,                   // false 表示要等真实结算
    },
    settlement_policy: {
      type: 'after_confirmed_receipt',    // 确认收货后
      delay_days_p50: 30,                 // 中位结算周期
      refund_window_days: 120,            // 维权回扣窗口
    },
    deductions: {                          // 净收入要扣
      tech_service_fee_pct: 10,           // 佣金的 10%
      tax_pct_for_subsidy: 20,            // 平台补贴部分扣 20%
    },
  },
}
```

---

## 5. 合规深度分析：三道红线

### 5.1 红线一："不得擅自开放给第三方调用"

原文（来自[阿里妈妈 API 与淘宝联盟 API 区别](https://developer.aliyun.com/article/1678436)）：

> 申请的 api 权限**不得用于非申请时提供的网站或 APP 渠道内**，或**未经阿里妈妈事先同意擅自开放给第三方进行调用**。

**精确解读**：

- ✅ 允许：deeplumen 主体下，旗下多个内部 Agent / 内部 App 使用同一个 AppKey 调 API
- 🟡 模糊：deeplumen 暴露 OCP Catalog，下游 Agent 不知道幕后是 Alimama
- 🔴 违规：deeplumen 直接把 Alimama AppKey/AppSecret 给第三方 Agent，让它自己调

**OCP 上的合规设计**：
- OCP 协议层上的 Agent 不直接接触 Alimama 凭据
- Alimama API 调用全部由 deeplumen 自营的 `alimama-provider` 服务封装
- Agent 通过 OCP 协议看到的，是 OCP 标准的 query / resolve 接口；底层是 Alimama 还是 JD 还是 PDD，对 Agent 透明
- 这种模式实际上是一种"代客购物 SaaS"——和 dingdanxia 那种二房东本质相同，只是表面披了 OCP 协议外衣

### 5.2 红线二：链接 API 是邀请制

需 **月日均点击 ≥ 5 万** OR **月日均 alipay ≥ 10 万**

**含义**：
- 启动阶段流量不够 → 拿不到 `taobao.tbk.privilege.get` → 不能转链 → ActionBinding 拿不到带 PID 的链接 → 佣金归不到自己
- 短期解法：通过 [订单侠](https://www.dingdanxia.com/price)、维易、淘口令网 等二房东 API（他们持有官方高级权限，按调用次数收费）
- 长期解法：积累流量到达标线，正式申请

### 5.3 红线三：PID 异常复用被风控

"部分平台对异常大量重复使用同一 pid 有**风险监控**"

**含义**：
- 单一 adzone_id 给所有 Agent 共用，调用量上去就被风控
- 解法只能是"一 Agent 一 adzone"——但要在自己主体下批量创建 adzone（这个是允许且方便的）

---

## 6. 跨联盟参照：京东联盟 / 拼多多 / 抖音

来源：[京东联盟开放平台](https://union.jd.com/openplatform/api)、[订单侠开放平台](https://www.dingdanxia.com/price)

### 6.1 京东联盟（JD Union）

| 维度 | 京东联盟 | 淘宝联盟 |
|---|---|---|
| 子渠道归因 | `subUnionId`（**日均订单 500 单门槛**） | `adzone_id`（门槛较低） |
| 推广位 | `positionId`（接口创建不显示在后台） | `adzone_id` |
| 转链能力 | 商品 / 店铺 / 活动 / 红包 / **礼金二合一** | 商品 + 优惠券二合一 |
| 特色功能 | **礼金推广**（需充值） | 淘礼金、淘口令 |
| 跟单机制 | `positionId` + `subUnionId` | `adzone_id` + `relation_id` |

**对 OCP 含义**：
- 京东的 `subUnionId` **比淘宝的 adzone_id 门槛高得多**（日均 500 单才能持续维持）
- 京东更适合"中大流量主"，淘宝更适合个人/小流量起步
- 如果 OCP 想做**多联盟聚合**，每个联盟的 attribution unit 不同（adzone_id / positionId / 别的），需要在 OCP `commission.v1` 里抽象出统一的 `attribution_token` 字段

### 6.2 抖音抖客 / 拼多多多多客

两者都是类似 CPS 联盟模式，API 形态都跟着淘宝学，但都有自己的：
- 资质门槛
- 邀请制权限
- 风控规则
- 结算周期

二房东（[订单侠](https://www.dingdanxia.com/price)、维易等）通常**一站接入淘宝 + 京东 + 拼多多 + 抖音 + 唯品会**，这是后续做"多联盟聚合 OCP Provider"最实用的工程姿态。

---

## 7. 战略背景：2026 年全球 Agentic Commerce 协议格局

**这是这次补充调研最重要的发现**——OCP Catalog 不是在协议真空里设计的，2026 年全球已经有八家在卷 Agentic Commerce 协议。

来源：[Making Sense of the AI Shopping Protocol Moment - PayPal Newsroom](https://newsroom.paypal-corp.com/2026-01-22-Making-Sense-of-the-AI-Shopping-Protocol-Moment)、[Agentic Commerce in 2026 - Opascope](https://opascope.com/insights/ai-shopping-assistant-guide-2026-agentic-commerce-protocols/)、[Perplexity Buy with Pro - Stellagent](https://stellagent.ai/insights/perplexity-shopping-buy-with-pro)、[Shopify Agentic Commerce Guide 2026](https://askphill.com/blogs/blog/agentic-commerce-for-shopify-protocols-platforms-and-what-to-prioritize-in-2026)、[Amazon vs Perplexity case](https://www.realinternetsales.com/amazon-wins-court-order-blocking-perplexity-s-ai-shopping-agent-what-it-means-fo/)

### 7.1 主要玩家与协议

| 协议 / 平台 | 主导方 | 关键特征 | 中国可达 |
|---|---|---|---|
| **OpenAI ACP**（Agentic Commerce Protocol） | OpenAI / ChatGPT | 4% 交易费、2026 年 3 月已大幅缩减 | 🔴 不可用 |
| **Google UCP**（Universal Commerce Protocol） | Google AI Mode | Cart / Catalog / Identity Linking；Shopping Graph 50 亿 SKU、每小时刷 20 亿；75M+ DAU | 🔴 不可用 |
| **Microsoft Copilot Checkout** | Microsoft + Shopify/PayPal/Etsy | 53% 多购买、33% 短路径 | 🟡 部分可达 |
| **Perplexity Buy with Pro** | Perplexity + PayPal | 商家 0 交易费、靠订阅和未来 affiliate revenue | 🟡 部分可达 |
| **Klarna Agentic Product Protocol** | Klarna | 北欧/欧美购物链路 | 🔴 |
| **Visa Trusted Agent Protocol** + **Mastercard Agent Pay** + **PayPal AP2** | 卡组织 | 支付/信任层 | 🟡 |
| **Shopify** | Shopify | 商家侧抽象层、各协议都接 | 🟡 |

### 7.2 市场规模

- **eMarketer 2025-12**：AI 平台 2026 年带动零售支出 **209 亿美元**（2025 年的近 4 倍）
- **McKinsey**：到 2030 年达 **3–5 万亿美元**

### 7.3 战略格局判断

1. **协议碎片化是事实**，没有人能一统天下
2. **支付/信任层**（PayPal AP2、Visa TAP）和**商品/购物层**（UCP、ACP、APP）分开演化
3. **法律风险已经显现**：2026-03-10 美国法院对 Perplexity Comet AI 浏览器下达临时禁令（Amazon 起诉胜诉），裁定 AI Agent 未经平台明确许可访问 e-commerce 是侵权——**这意味着 Agent-friendly 协议必须有平台官方授权背书**
4. **中国市场是空地**——OpenAI ACP / Google UCP / Klarna APP 在中国实际不可用；阿里妈妈"AI 万相"是商家侧 AI 工具，**不是 Agent 侧协议**

### 7.4 阿里妈妈"AI 万相"是什么、不是什么

来源：[阿里妈妈 AI 万相发布](https://www.time-weekly.com/post/328244)、[618 新策见面会](https://www.ithome.com/0/943/171.htm)

**是**：商家经营智能体引擎，四大 Agent 协同（**万相智识** 意图识别、**万相智品** 商品理解、**万相智造** AIGC 创意、**万相智投** 投放优化），落地在百灵、万相台 AI 无界、UD 智汇投等商家工具背后。

**不是**：面向第三方 AI Agent 的 commerce 协议。AI 万相是**商家 → 阿里**侧的，Agent → 商家侧目前没有官方协议出口。

### 7.5 OCP Catalog 的战略空间

基于上面格局：

| 选项 | 描述 | 可行性 | 上限 |
|---|---|---|---|
| **A. 中国版 UCP** | OCP 成为中文电商生态的 Agentic Commerce 标准 | 🟡 难（要阿里/京东/拼多多支持） | 极高（年规模可达数百亿） |
| **B. 多联盟聚合协议** | OCP 不是协议标准，而是**桥接层**——后端聚合淘宝 / 京东 / 拼多多 / 抖音联盟，前端对 Agent 暴露统一 OCP 接口 | 🟢 现实 | 中高（取决于流量） |
| **C. 垂直商品索引** | OCP 只做特定品类（比如紧固件、医疗器械），不做联盟桥接 | 🟢 容易 | 低（小生意） |
| **D. 协议供应商** | OCP 把协议开源，被其他公司部署、自己不接联盟 | 🟡 | 中（看渗透） |

**Alimama 集成的战略价值取决于 OCP 走 A / B / C / D 的哪条路**：
- 走 A：Alimama 是关键合作方（PR & 共建）
- 走 B：Alimama 是**第一个**桥接的联盟，紧接着是京东 / 拼多多
- 走 C：Alimama 仅作为商品池数据源，contribute 少
- 走 D：Alimama 不在路径上

---

## 8. OCP × Alimama 完整字段映射

把 Alimama material.optional 的字段映射到 OCP 三个 descriptor pack + 提议的 `commission.v1` 扩展。

### 8.1 product.core.v1

```typescript
{
  pack_id: 'ocp.commerce.product.core.v1',
  data: {
    title: m.title,                                     // 直接映射
    summary: undefined,                                  // material API 不给 summary
    brand: m.shop_title,                                 // 用店铺名当 brand
    category: lookupCategoryName(m.cat),                 // 数字 cat → 中文/英文名
    sku: String(m.num_iid),                              // 商品 ID 当 SKU
    product_url: m.item_url,                             // 注意:非 affiliate URL
    image_urls: [m.pict_url, ...(m.small_images?.string ?? [])],
    video_urls: [],
    attributes: {
      // 平台属性
      platform: m.user_type === 1 ? 'tmall' : 'taobao',
      seller_id: m.seller_id,
      provcity: m.provcity,
      // 销售信号
      sales_volume_30d: m.volume,
      tk_sales_30d: m.tk_total_sales,
      tk_commission_30d: m.tk_total_commi,
      // 价格信号
      list_price: m.reserve_price,
      // 升级版 .upgrade 才有
      estimated_final_price: m.real_final_price,
      benefit_tags: m.itemtag,
      // 关键标记
      affiliate_provider: 'alimama_taobao_union',
      requires_affiliate_resolution: true,
      raw_affiliate_id: String(m.num_iid),               // 用于 resolve 转链
    },
  },
}
```

### 8.2 price.v1

```typescript
{
  pack_id: 'ocp.commerce.price.v1',
  data: {
    currency: 'CNY',
    amount: parseFloat(m.zk_final_price),                // 折扣价
    list_amount: parseFloat(m.reserve_price),            // 原价
    price_type: 'fixed',
  },
}
```

### 8.3 inventory.v1

```typescript
{
  pack_id: 'ocp.commerce.inventory.v1',
  data: {
    availability_status: 'unknown',                       // material API 不给库存
    // 推断:有券且券剩余 > 0 → 可视为 in_stock
  },
}
```

### 8.4 `commission.v1`（OCP 协议扩展提案）

```typescript
{
  pack_id: 'ocp.commerce.commission.v1',                  // 提案,尚未在 OCP 现有 schema 中
  data: {
    affiliate_provider: 'alimama_taobao_union',
    commission_rate_bp: m.commission_rate,                // 基点 (1550 = 15.5%)
    estimated_commission_per_unit: {
      currency: 'CNY',
      amount: parseFloat(m.zk_final_price) * m.commission_rate / 10000,
      reliable: false,                                     // 要等结算
    },
    coupon: m.coupon_info ? {
      info: m.coupon_info,
      starts_at: m.coupon_start_time,
      ends_at: m.coupon_end_time,
      total_count: m.coupon_total_count,
      remain_count: m.coupon_remain_count,
    } : null,
    settlement_policy: {
      type: 'after_confirmed_receipt',
      delay_days_p50: 30,
      refund_window_days: 120,
    },
    deductions: {
      tech_service_fee_pct: 10,
      tax_pct_for_tmall_subsidy: 20,
    },
    attribution_method: 'per_agent_adzone',                // OCP 自定义
  },
}
```

---

## 9. 完整调用链路（Sequence Diagram）

### 9.1 商品同步（离线，定时触发）

```
[cron 每小时]                          [alimama-provider]               [OCP catalog-api]
       │                                       │                                │
       │ trigger sync                          │                                │
       │──────────────────────────────────────▶│                                │
       │                                       │ taobao.tbk.dg.material.optional│
       │                                       │  (or .upgrade)                 │
       │                                       │  q="无线耳机", pageNo=1..N    │
       │                                       │──────────▶ Alimama Open Gateway│
       │                                       │◀────────── result_list[]       │
       │                                       │                                │
       │                                       │ map to CommercialObject[]      │
       │                                       │ split into ≤100/batch          │
       │                                       │                                │
       │                                       │ POST /ocp/objects/sync         │
       │                                       │────────────────────────────────▶
       │                                       │◀──── ObjectSyncResult          │
       │                                       │                                │
       │                                       │ persist last_sync_at,          │
       │                                       │ cursor for incremental         │
```

### 9.2 Agent 发现 + Resolve + 用户点击成交

```
[Agent]              [OCP center/catalog]              [alimama-provider]              [Alimama Open]              [淘宝 App / 浏览器]              [淘宝商品页]              [用户支付]
   │                          │                                │                              │                              │                              │                          │
   │  query                   │                                │                              │                              │                              │                          │
   │ keyword="无线耳机"       │                                │                              │                              │                              │                          │
   │ agent_id="agt_abc"       │                                │                              │                              │                              │                          │
   │─────────────────────────▶│                                │                              │                              │                              │                          │
   │                          │ keyword + filter search        │                              │                              │                              │                          │
   │                          │ (返回 CatalogEntries[])        │                              │                              │                              │                          │
   │◀─────────────────────────│                                │                              │                              │                              │                          │
   │  resolve                 │                                │                              │                              │                              │                          │
   │ entry_id="ent_xyz"       │                                │                              │                              │                              │                          │
   │ agent_id="agt_abc"       │                                │                              │                              │                              │                          │
   │─────────────────────────▶│                                │                              │                              │                              │                          │
   │                          │ POST resolve hook              │                              │                              │                              │                          │
   │                          │ (entry, agent_id)              │                              │                              │                              │                          │
   │                          │───────────────────────────────▶│                              │                              │                              │                          │
   │                          │                                │ lookup adzone_id            │                              │                              │                          │
   │                          │                                │ for agent_id                │                              │                              │                          │
   │                          │                                │ (create if missing)          │                              │                              │                          │
   │                          │                                │                              │                              │                              │                          │
   │                          │                                │ taobao.tbk.privilege.get    │                              │                              │                          │
   │                          │                                │ item_id, adzone_id,         │                              │                              │                          │
   │                          │                                │ external_id="ent_xyz"       │                              │                              │                          │
   │                          │                                │─────────────────────────────▶│                              │                              │                          │
   │                          │                                │◀──── coupon_click_url       │                              │                              │                          │
   │                          │                                │                              │                              │                              │                          │
   │                          │ ResolvableReference            │                              │                              │                              │                          │
   │                          │ + ActionBinding{url=...}       │                              │                              │                              │                          │
   │                          │◀───────────────────────────────│                              │                              │                              │                          │
   │◀─────────────────────────│                                │                              │                              │                              │                          │
   │                          │                                │                              │                              │                              │                          │
   │  show product card +     │                                │                              │                              │                              │                          │
   │  "去淘宝看看" 按钮       │                                │                              │                              │                              │                          │
   │  to user                 │                                │                              │                              │                              │                          │
   │                                                                                          │                              │                              │                          │
   │                                                          [user click]                    │                              │                              │                          │
   │                                                          ──────────────────────────────▶ │ s.click.taobao.com/xxx       │                              │                          │
   │                                                                                          │ 302 redirect with PID cookie │                              │                          │
   │                                                                                          │──────────────────────────────▶                              │                          │
   │                                                                                          │                              │ commodity detail             │                          │
   │                                                                                          │                              │                              │ click "立即购买"         │
   │                                                                                          │                              │                              │─────────────────────────▶│
   │                                                                                          │                              │                              │                          │
   │                                                                                          │                              │                              │                  pay
```

### 9.3 异步佣金回流

```
[cron 每天]                  [alimama-provider]                       [Alimama Open]                   [OCP Metrics Service / DB]
     │                              │                                       │                                  │
     │ trigger daily report         │                                       │                                  │
     │─────────────────────────────▶│                                       │                                  │
     │                              │ taobao.tbk.order.get                  │                                  │
     │                              │ (start_time = yesterday)              │                                  │
     │                              │──────────────────────────────────────▶│                                  │
     │                              │◀───── order_list[]                    │                                  │
     │                              │       (含 adzone_id, GMV, commission, │                                  │
     │                              │        status: 已付款/已收货/已结算   │                                  │
     │                              │        /已维权)                       │                                  │
     │                              │                                       │                                  │
     │                              │ join adzone_id → agent_id             │                                  │
     │                              │                                       │                                  │
     │                              │ upsert into commission_ledger         │                                  │
     │                              │ (agent_id, date, gmv,                 │                                  │
     │                              │  estimated_commission, status)        │                                  │
     │                              │─────────────────────────────────────────────────────────────────────────▶│
     │                              │                                       │                                  │
     │                              │ for 已维权 orders:                   │                                  │
     │                              │   mark 'refund_pending' or            │                                  │
     │                              │   adjust estimated_commission         │                                  │
     │                              │─────────────────────────────────────────────────────────────────────────▶│
```

---

## 10. 架构方案对比

### 10.1 关键问题：alimama-provider 服务放在哪？

| 选项 | 优点 | 缺点 |
|---|---|---|
| **A. OCP-Catalog monorepo 内新增 `apps/examples/alimama-provider-api/`** | 复用 `@ocp-catalog/ocp-schema`、`@ocp-catalog/db`、`@ocp-catalog/auth-core`；架构一致 | OCP-Catalog 仓库变重；商业敏感的 AppKey/AppSecret 进了协议参考实现仓库 |
| **B. 独立 `alimama-provider` 仓库**，用 OCP-Catalog 作为 npm 包依赖 | 商业凭据隔离；可以闭源 | 需要把 OCP-Catalog 的 schema 包发到 npm；版本管理麻烦 |
| **C. 嵌入 OCP Catalog Node 内部**（catalog 直接和 alimama API 对话） | 链路最短；resolve 时不用走 Provider 中转 | 违反 OCP 协议分层（Catalog 不应该是 Provider）；扩展到 JD/拼多多时一团乱 |

推荐：**A 起步、长期演化到 B**。

### 10.2 关键问题：每 Agent 一 adzone_id 的存储

```sql
CREATE TABLE agent_adzone_mapping (
  agent_id           VARCHAR(100) PRIMARY KEY,
  adzone_id          VARCHAR(50) NOT NULL,
  site_id            VARCHAR(50) NOT NULL,
  pid                VARCHAR(100) NOT NULL,       -- 'mm_<member>_<site>_<adzone>'
  alimama_account    VARCHAR(50) NOT NULL,         -- 所属联盟账号(支持多账号池)
  display_name       VARCHAR(200),
  created_at         TIMESTAMP DEFAULT now(),
  last_used_at       TIMESTAMP,
  status             VARCHAR(20) DEFAULT 'active', -- active / suspended / closed
  metadata           JSONB                          -- 额外标签:agent platform、tier 等
);

CREATE INDEX ON agent_adzone_mapping (adzone_id);
CREATE INDEX ON agent_adzone_mapping (status);
```

**冷启动策略**：
- Agent 首次 resolve 时，若无 mapping → 调阿里妈妈后台 API（或 alimama 网页端自动化）创建 adzone → 写入 mapping
- 或预先批量创建一批"待分配池"，Agent 上线时分配

**adzone 上限**：暂未查到淘宝联盟单账号 adzone 数量上限。需要业务确认，否则做多账号轮换池。

### 10.3 关键问题：Agent 身份标识从哪传

OCP 现有协议中，query 和 resolve 请求没有显式 `agent_id` 字段。两种扩展姿态：

**方案 1：HTTP header 透传**

```http
POST /ocp/resolve
X-OCP-Agent-Id: agt_abc
X-OCP-Agent-Manifest-Url: https://agent-host/manifest
```

优点：协议层零侵入
缺点：proxy 链路可能丢 header；不在 audit log 的 body 里

**方案 2：协议层 query/resolve request body 加 `agent` 字段**

```typescript
{
  ocp_version: '1.0',
  kind: 'ResolveRequest',
  entry_id: 'ent_xyz',
  agent: {                           // 协议扩展
    agent_id: 'agt_abc',
    manifest_url: 'https://agent-host/manifest',
    intent: 'shopping_compare',     // 可选用途声明
  }
}
```

推荐：**方案 2**，符合 OCP "Purpose-Based Access" 原则（设计文档第 14.2 节）

---

## 11. 风险与异常处理目录

| 风险类别 | 具体场景 | 处理 |
|---|---|---|
| **归因失败** | 用户点 affiliate 链接，但已有更高优先级 PID cookie | 不可控；监控订单回执，记录归因率 |
| | 用户在淘宝 App 内被切到另一个内嵌 H5 | 不可控；记录 |
| **APP 唤起失败** | 用户没装手淘 | ActionBinding 提供两个 url：affiliate URL（首选）+ tpwd 淘口令（备选） |
| **API 调用失败** | Alimama API 限流、超时 | retry + circuit breaker；fallback 到不带 PID 的 item_url（不能归因但不阻塞用户） |
| **Provider API 限流** | 单 AppKey QPS 上限 | 多 AppKey 池 + 配额管理 |
| **退款回扣** | 已结算订单 4 个月内被维权 | commission_ledger 必须支持反向调整；保留 5 个月历史 |
| **adzone 被风控** | 单 adzone 异常被冻结 | mapping 表 status 字段标记；触发时给 agent 切换备用 adzone |
| **链接 API 权限被收回** | 流量下降跌破门槛 | 退回二房东兜底；保持基础 API 同步可用 |
| **法律风险** | 类似 Amazon vs Perplexity，平台起诉 | **关键**：Alimama 是合作方而非被爬，原则上风险低；但 Agent 跨向 JD/拼多多时要逐个评估 |
| **合规审计** | 监管检查我们如何处理用户/Agent 关系 | 留好审计日志（OCP 协议自带 audit support） |
| **PID 优先级冲突** | 用户在我们站外又点了别人的 PID | 不可控；统计回执即知 |
| **跨设备购买** | 用户在 Agent 看完，几天后在另一台设备买 | 淘宝 PID cookie 多设备失效；归因丢失 |

---

## 12. 商业模式与利益分配

```
[商家]                           [Alimama 平台]                         [OCP Provider (我们)]                    [Agent 平台]
  │                                    │                                          │                                       │
  │ 报名联盟 + 设佣金率                │                                          │                                       │
  │ (例如 15%)                         │                                          │                                       │
  │ ──────────────────────────────────▶│                                          │                                       │
  │                                    │                                          │                                       │
  │                                    │ ── 我们注册成淘客 ─────────────────────▶│                                       │
  │                                    │                                          │                                       │
  │                                    │                                          │ 提供 OCP 接口给 Agent                │
  │                                    │                                          │ ─────────────────────────────────────▶│
  │                                    │                                          │                                       │
  │                                    │                                          │                                       │ Agent 推荐
  │                                    │                                          │                                       │ 给用户
  │                                    │                                          │                                       │
  │                                    │                                          │                                       │
  │                                                  [用户成交,100 元]                                                    │
  │                                    │                                          │                                       │
  │ ◀── 收到 100 元(扣平台抽成) ──────│                                          │                                       │
  │                                    │ ── 扣 15 元佣金给淘客 ──────────────────▶│                                       │
  │                                    │ ── 留 1.5 元技术服务费 ─                │                                       │
  │                                    │                                          │ 收到 13.5 元(净)                      │
```

### 12.1 OCP Provider 端的收入来源

如果 OCP Provider 是我们（deeplumen），那么我们的收入 = **联盟佣金净额**。

100 元订单、15% 佣金、10% 技术服务费 ≈ **13.5 元**到账。

**这笔钱怎么和 Agent 分？** 几种模式：

| 模式 | 分配规则 | 优缺点 |
|---|---|---|
| A. **全留**（封闭 Agent） | 100% 归 OCP Provider | Agent 没动力推；但如果 Agent 也是 OCP Provider 自己的，无冲突 |
| B. **rev-share** | 比如 50/50 给 Agent | Agent 有动力做转化；要做 settlement 系统 |
| C. **CPM/CPC 模式付给 Agent** | 不按佣金分，按曝光/点击付费 | Agent 收入稳定；OCP Provider 风险高 |
| D. **平台抽成** | OCP Provider 抽 X%，剩下给 Agent | 类似多多客 |

### 12.2 关键合规问题：**红线一（"不得开放给第三方"）的解读决定 B/C/D 是否可行**

- 如果"第三方 Agent"被算作违规 → 只能走 A（封闭生态）
- 如果"已签合作协议的 Agent"算作合规渠道 → B/C/D 都可
- 需要法务和 Alimama 沟通确认

---

## 13. 推荐的分阶段 Roadmap

### Phase 0 — 准备工作（1 周）

- [ ] 注册一个**测试淘宝联盟账号**（用 deeplumen 主体或个人都行，做 PoC）
- [ ] 完成媒体备案（最低门槛）
- [ ] 申请基础 AppKey
- [ ] 调用 `taobao.tbk.dg.material.optional` 跑通 hello-world

### Phase 1 — PoC：基础 API 起步（2 周，无高级权限依赖）

- [ ] 在 OCP-Catalog monorepo 新建 `apps/examples/alimama-provider-api/`
- [ ] 实现 `material.optional` 拉取 → 映射到三 descriptor pack → 同步到 commerce-catalog-api
- [ ] ActionBinding 暂时用**未转链的 `item_url`**（不能归因但跑通链路）
- [ ] 录一个 demo：Agent 用 OCP 查"无线耳机" → 拿到天猫商品 → 点链接跳手淘
- [ ] **验证点**：OCP 协议、字段映射、batch sync、增量更新都跑通

### Phase 2 — 静态转链 + 二房东兜底（3 周）

- [ ] 接 [订单侠 API](https://www.dingdanxia.com/price) 作为转链兜底（拿不到官方高级权限时用）
- [ ] ActionBinding URL 改成转链后的 `coupon_click_url`
- [ ] 实现 `taobao.tbk.order.get` 拉单 → 写 `commission_ledger`
- [ ] 做一个 admin dashboard 展示 daily GMV / 净佣金 / 维权情况
- [ ] **验证点**：能产生真实佣金、统计准确

### Phase 3 — 流量积累 + 申请官方链接 API（视 Phase 2 数据）

- [ ] 业务侧：让若干 Agent 实际有用户使用，把流量做到月日均 5 万点击 OR 月日均 alipay 10 万
- [ ] 向 Alimama 申请高级权限包
- [ ] 切换到官方 `taobao.tbk.privilege.get`
- [ ] **验证点**：拿到官方高级权限、不再依赖二房东

### Phase 4 — 每 Agent 一 adzone（4 周）

- [ ] 实现 `agent_adzone_mapping` 表 + 自动分配逻辑
- [ ] resolve 协议层引入 `agent` 字段（向 OCP 协议提扩展提案）
- [ ] 实现按 agent 分发不同 PID 的 ActionBinding
- [ ] Commission ledger 按 agent_id 聚合
- [ ] **验证点**：千 Agent 千归因正确性 ≥ 95%

### Phase 5 — 协议扩展 + 多联盟桥接（持续）

- [ ] 起草 `ocp.commerce.commission.v1` descriptor pack 草案，正式提案给 OCP 仓库
- [ ] 评估接入京东联盟（subUnionId 模型）
- [ ] 评估接入拼多多多多客
- [ ] **里程碑**：OCP × 多联盟桥接成型，对外宣传"中国 Agentic Commerce 协议桥"

---

## 14. 需要决策的开放问题

### 业务/法务

1. **运营主体**：用 deeplumen 公司主体注册淘宝联盟账号？还是只做技术 PoC？
2. **OCP 战略路线 A/B/C/D**（见 §7.5）：要走哪条路？
3. **"第三方 Agent"合规边界**：什么样的 Agent 可以算"deeplumen 内部渠道"？需法务和 Alimama 商务确认
4. **rev-share 模式**：佣金怎么和 Agent 分？

### 技术

5. **`agent_id` 协议位置**：放 HTTP header 还是放 request body？要不要正式向 OCP 协议提扩展？
6. **adzone 上限**：淘宝联盟单账号 adzone 上限是多少？要不要做多账号池？
7. **Provider 仓库位置**：放 OCP-Catalog monorepo 还是独立仓库？
8. **二房东依赖**：短期接订单侠是必要的，但要不要约束依赖期限？

### 协议层

9. **`ocp.commerce.commission.v1`**：是否正式向 OCP 协议提扩展？
10. **`PurposeDeclaration` 与 attribution**：OCP 设计文档第 14.2 节提到 purpose-based access，attribution 算不算一种 purpose？

---

## 15. 后续可深挖的方向

- **官方申请材料清单**：媒体备案 ICP 要求、企业资质、面试问题、过审窍门
- **二房东 API 详细评测**：订单侠 vs 维易 vs 淘口令网，定价、稳定性、合规边界
- **OCP × Google UCP 兼容性研究**：能否把 OCP query/resolve 翻译到 UCP catalog/cart 接口
- **PayPal AP2 / Visa TAP 集成**：跨境场景下的信任层接入
- **Amazon vs Perplexity 判例的实务影响**：对中国 Agent ecosystem 的潜在传染
- **AI 万相"四 Agent 协同"的接口**：是否有可对接的开放 API（目前应该没有，但值得跟踪）

---

## 16. 参考资料

### 阿里妈妈 / 淘宝联盟官方
- [淘宝联盟开放平台](https://open.alimama.com/)
- [淘宝联盟·生态伙伴](https://pub.alimama.com/)
- [淘宝联盟（aff.alimama.com）](https://aff.alimama.com/)
- [taobao.tbk.privilege.get API 文档](https://developer.alibaba.com/docs/api.htm?apiId=28625)
- [淘宝客佣金结算和额外奖励解析](https://www.alimama.com/news_detail.htm?contentId=1173)

### 技术文档与字段说明
- [阿里妈妈 API 与淘宝联盟 API 区别](https://developer.aliyun.com/article/1678436)
- [淘宝联盟 PID / adzone_id 详解](https://www.dingdanxia.com/article/5.html)
- [taobao.tbk.dg.material.optional 字段说明](https://www.taokeshow.com/51162.html)
- [淘宝开放平台应用申请全流程指南 2025](https://blog.csdn.net/api_open/article/details/149245815)
- [淘宝联盟结算流程详解](https://www.taokeshow.com/18992.html)
- [淘宝客佣金结算规则](https://www.kaitao.cn/article/20250731131017.htm)
- [GitHub - ennnnny/tbk SDK](https://github.com/ennnnny/tbk)
- [GitHub - makelove/Taobao_topsdk](https://github.com/makelove/Taobao_topsdk)

### 跨联盟参照
- [京东联盟开放平台](https://union.jd.com/openplatform/api)
- [京东联盟高级 API 文档](https://www.dingdanxia.com/jd)
- [订单侠开放平台（多联盟二房东）](https://www.dingdanxia.com/price)

### 阿里妈妈 AI 战略
- [阿里妈妈 AI 万相发布会](https://www.time-weekly.com/post/328244)
- [阿里妈妈 618 新策见面会](https://www.ithome.com/0/943/171.htm)
- [阿里妈妈释放 2025 重磅利好](https://finance.sina.com.cn/tech/roll/2025-02-13/doc-inekizxx1609206.shtml)
- [AI 智能体接管电商生态（新浪 2026-04）](https://finance.sina.cn/stock/jdts/2026-04-24/detail-inhvpkaq4816049.d.html)

### 全球 Agentic Commerce 协议格局
- [Making Sense of the AI Shopping Protocol Moment (PayPal, 2026-01)](https://newsroom.paypal-corp.com/2026-01-22-Making-Sense-of-the-AI-Shopping-Protocol-Moment)
- [AI Shopping Assistant Guide 2026 - Opascope](https://opascope.com/insights/ai-shopping-assistant-guide-2026-agentic-commerce-protocols/)
- [Perplexity Buy with Pro - Stellagent](https://stellagent.ai/insights/perplexity-shopping-buy-with-pro)
- [Shopify Agentic Commerce Guide 2026](https://askphill.com/blogs/blog/agentic-commerce-for-shopify-protocols-platforms-and-what-to-prioritize-in-2026)
- [Amazon vs Perplexity case 影响分析](https://www.realinternetsales.com/amazon-wins-court-order-blocking-perplexity-s-ai-shopping-agent-what-it-means-fo/)
- [Perplexity Shopping 优化指南 - Shopify](https://www.shopify.com/blog/perplexity-shopping)
- [2026 年电商 AI 导购趋势 - TMO Group](https://www.tmogroup.com.cn/insights/ai-shopping-assistant/)

---

*报告生成于 2026-05-13。Alimama / 淘宝联盟政策、API 与佣金规则随时间变化，关键决策前请向官方/法务复核。*
