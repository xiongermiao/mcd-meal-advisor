# MCP 集成说明

本文件说明 `mcd-meal-advisor` 实际调用了哪些麦当劳 MCP 能力、按什么顺序调用、为什么这样调用，以及由此产生的业务价值。

## 一、接入的 MCP Server

| 项 | 值 |
|---|---|
| Server 名称 | `mcd-mcp` |
| 端点 | `https://mcp.mcd.cn`（Streamable HTTP） |
| 鉴权方式 | 请求头 `Authorization: Bearer <MCD_MCP_TOKEN>` |
| 运行载体 | WorkBuddy Skill（`SKILL.md` + `references/` 规则正文） |
| 配置示例 | [`mcp-config.example.json`](mcp-config.example.json)（仅含环境变量占位符，不含任何真实凭证） |

接入步骤见 [`USAGE.md`](USAGE.md)。

## 二、实际调用的 Tool 清单

按用途分四组，共 19 个工具。标 ⚠️ 的是**写操作**，影响用户真实资产（订单或积分），全流程受强制确认闸门约束。

### 1. 履约定位（第 1 步）

| Tool | 性质 | 用途 |
|---|---|---|
| `query-nearby-stores` | 只读 | 到店自取 / 得来速：按定位查可选门店，取得 `storeCode` |
| `delivery-query-addresses` | 只读 | 麦乐送：读取用户已有收货地址，取得 `addressId` |
| `delivery-query-stores` | 只读 | 麦乐送：由 `addressId` 反查可配送门店（门店由系统指派，非用户选择） |

### 2. 权益盘点（第 2 步）

| Tool | 性质 | 用途 |
|---|---|---|
| `query-my-account` | 只读 | 积分余额、等级、券账户总览，用于临期积分提醒 |
| `query-store-coupons` | 只读 | 当前门店可用的券与活动，决定"优惠 −X"从哪来 |

### 3. 选品、核价与展示（第 3~4 步）

| Tool | 性质 | 用途 |
|---|---|---|
| `query-meals` | 只读 | 按门店与餐次拉取菜单（含套餐与单品） |
| `list-nutrition-foods` | 只读 | 营养库，按热量 / 蛋白质 / 钠做硬约束筛选 |
| `query-meal-detail` | 只读 | 拆套餐结构、读取可换配项与特调属性 |
| `calculate-price` | 只读 | 官方核价，产出原价 / 优惠 / 应付三态报价 |
| `query-promotions` | 只读 | 门店级促销叠加校验 |
| `query-meal-assistance` | 只读 | 单人餐 / 多人餐辅助搭配参考 |

### 4. 积分兑换与下单（第 6、8 步）

| Tool | 性质 | 用途 |
|---|---|---|
| `mall-points-products` | 只读 | 积分商城可兑商品列表 |
| `mall-product-detail` | 只读 | 兑换商品详情与所需积分 |
| `mall-create-order` | ⚠️ 写 | 扣积分兑券；成功后券回流，必须回第 4 步重新核价 |
| `mall-order-detail` | 只读 | 兑换单结果核对 |
| `create-order` | ⚠️ 写 | 创建餐品订单（**止步于此，不代付**） |
| `query-order` | 只读 | 回查已建订单状态 |
| `cancel-order` | ⚠️ 写 | 用户明确要求时撤销订单 |
| `auto-bind-coupons` | ⚠️ 写 | 用户明确要求时一键领券 |

> 另有 `party-order-create`（企业团餐）与 `mall-order-list` / `party-*` 系列作为团餐场景的扩展路径，主流程默认不使用。

## 三、调用流程

```
                    ┌─ query-nearby-stores ──────┐  到店型
第 1 步 履约定位 ────┤                            ├──→ storeCode / addressId
                    └─ delivery-query-addresses ─┴─→ delivery-query-stores   配送型

                    ┌─ query-my-account ─────────┐
第 2、3 步 ★同批 ───┤                            ├──→ 券 / 积分 / 菜单 / 营养
                    └─ query-store-coupons ─────┤
                      query-meals ──────────────┤
                      list-nutrition-foods ─────┘

第 4 步 试算展示 ── query-meal-detail ─→ calculate-price ─→ query-promotions

第 5 步 确认闸门 ── 纯对话，不调工具

第 6 步 积分兑换 ── mall-points-products → mall-product-detail → ⚠️ mall-create-order → 回第 4 步

第 7 步 最终确认单 ── 纯对话，不调工具

第 8 步 ⚠️ create-order ── 用户明确下单指令后，单独一发
```

### 三条调用纪律

1. **第 2、3 步必须同批发。** 拿到 `storeCode` / `addressId` 之后，`query-my-account`、`query-store-coupons`、`query-meals`、`list-nutrition-foods` 四者入参互不依赖，逐个串行调用实测白等 27 秒。逐个串行即判不合格。
2. **写操作永不并行。** `create-order`、`mall-create-order`、`cancel-order`、`auto-bind-coupons` 一次只发一个，必须等上一个有结论再发下一个。
3. **并行中任一失败不得静默空值。** 其余结果照用，但展示时必须点名缺了哪一项（"券信息没取到，本次按无券核价"）——"没取到"和"没有券"在展示上是同一句话，却是完全不同的钱。

### 两个不可越过的语义边界

- **第 5 步是唯一确认闸门**：候选方案卡给用户看完并得到明确选择之前，不得发第 6、8 步的任何写调用。
- **第 7 步与第 8 步必须分成两个回合**：展示确认单的那条消息里不得出现"已下单"字样；`create-order` 只能由用户**另外**明确说"下单 / 创建订单"触发。下单后把支付入口交还用户，不代付、不查询支付结果、不宣称支付成功。

## 四、业务价值

### 1. 把"接口不报错但结果错"的静默失败变成显性确认

点餐真正的难点不是"哪个汉堡好吃"，而是履约定位那几个**看起来不用问、其实必须问**的参数：到店自取选哪家店、堂食还是带走、麦乐送送到哪个地址、几点取。这些值定错了，`query-meals` 不会报错，只会返回另一家店的正确答案——菜单、券、营养、价格全错，且错得很安静。

本 Skill 用分场景硬约束解决它：到店型的门店与取餐方式**必须用户确认**；配送型则相反，门店由系统指派（`assigned`）只告知，只确认配送地址——因为麦乐送用户根本不知道哪家店能送他家，问他"要哪家店"等于把系统该做的事外包给用户。**该确认的一项不少，不该问的一句不多。**

### 2. 让每一分钱都可核对、可失效

`calculate-price` 有原价 / 优惠 / 应付三种状态，SKILL 规定报价必须带状态标注（已核价 / 待核价 / 待核实），且**门店或任一参数变化即整张报价失效**，不允许沿用旧价只改一个数字；接口未返回原价或优惠额时禁止倒推。券换来的单项标「券兑换」，提议兑换的"预计可省"不得倒扣进总价。这直接压制了餐饮类助手最常见的错误：用估算数字冒充官方价格。

### 3. 营养约束前置，而不是事后点评

`list-nutrition-foods` 在第 3 步就参与硬过滤，预算与忌口是筛选条件而非排序权重，口味偏好只影响排序。无解时明说放宽了哪一项，而不是硬凑一个超标的方案。对减脂 / 控钠人群，这是可执行与不可执行的区别。

### 4. 积分与券当资产管，不当营销素材

`query-my-account` + `query-store-coupons` 用于盘点真实权益：只在积分临期时提醒且同会话只提一次；`mall-create-order` 扣分兑券后强制回第 4 步重算，避免"用积分换了券却按没券的价格报账"。写操作全部落在用户明确指令之后。

### 5. 消费决策的边界感

能力边界明确止于 `create-order`：不代付、不查支付结果、不宣称支付成功。用户始终持有支付这一步的最终控制权——这是把 MCP 写能力交给 AI 使用时最需要守住的线。

## 五、验证方式

- 12 条实测用例与结论见 [`docs/test-report.md`](docs/test-report.md)，其中主动判定 3 条不合格并记录原因。
- 三个零风险验证用例（不建单、不扣积分）见 [`USAGE.md`](USAGE.md)。
- 性能基线（串行 vs 同批的实测耗时）见 [`docs/perf-baseline.md`](docs/perf-baseline.md)。
