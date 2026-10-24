# mcd-meal-advisor

麦当劳点餐顾问 Skill，给 [WorkBuddy](https://www.workbuddy.cn) 用。基于麦当劳 MCP 工具集，把"点一份麦当劳"拆成一条可核对的流程：确定场景 → 检查权益 → 选品筛选 → 试算展示 → 用户确认 → 积分兑换 → 落定 → 下单。

能力边界到**创建订单**为止，不代付、不查询支付结果。

## 它解决什么问题

点餐最难的从来不是"哪个汉堡好吃"，而是那些**看起来不用问、其实必须问**的底层参数：到店取要选哪家店、堂食还是带走，外送要送到哪个地址，以及几点取。这些值定错了，后面的菜单、优惠券、价格全都是错的，而且错得很安静——接口不会报错，只会给你另一家店的正确答案。

反过来也有坑：有些参数**看起来该问、其实不该问**。麦乐送的用户不知道哪家店能送到他家，问他"要哪家店"等于把系统该做的事外包给他，而真正缺的信息（送到哪）反而没问到。所以本 Skill 的核心约束是：**该确认的一项不少，不该问的一句不多。**

## 四条硬约束

| 约束 | 要求 |
|---|---|
| 1. 信息分级默认 | 人数、饭量可直接默认；取餐大方式必须确认；**到店型**的门店与取餐方式必须确认，**配送型（麦乐送）由配送地址决定门店、门店为系统指派只告知**；时间不得由餐次推断 |
| 2. 临期积分提醒 | 只在积分快过期时提醒，同会话只提一次；提议兑换的"预计可省"不得倒扣进总价 |
| 3. 先满足约束再优化搭配 | 明确预算/忌口是硬筛选，口味偏好只影响排序；无解时明说放宽了哪项 |
| 4. 报价带状态 | 已核价 / 待核价 / 待核实三态分立；门店或任何参数一变，价格与推荐理由一起失效 |

### 履约定位：按场景分叉

到店型和配送型的决策主体不同，不能套同一套规则。

| 取餐大方式 | 地基（必须确认） | 门店 | 取餐方式 |
|---|---|---|---|
| 到店自取 / 得来速 | 门店 + 取餐方式 | 用户选定，须确认 | 堂食／外带，须确认 |
| 麦乐送 / 团餐 | **配送地址 addressId** | **系统指派（`assigned`），只告知** | 不存在，不传 takeWayCode |

- **三种状态**：`confirmed`（用户点头）/ `candidate`（助手建议）/ `assigned`（接口按地址指派）。`assigned` 只适用于配送型门店，**不得说成"你选的店"**，也不能推广到地址或到店型门店。
- **麦乐送不问"要哪家店"**：`delivery-query-addresses` → 确认地址 → `delivery-query-stores(addressId, beType=2)` → 硬筛营业与时段 → 取首家。用户主动点名才在列表里按店名匹配。
- **外送门店择优只靠真实字段**：实测 `delivery-query-stores` **不返回** distance、配送费、预计送达时长、门店地址。所以不挑"最近/最便宜/最快"，按「营业 → 时段 → 返回顺序首家 → 缺货才换店」四步走，禁止编造理由。
- **两个闸门分场景**：闸门 1 到店型卡 storeCode、配送型卡 addressId + 已拿到可配送门店；闸门 2 到店型卡 takeWayCode，配送型无此项（改为复核地址有效 + 门店仍是 assigned + 配送费已列）。
- **费用**：配送型确认单**必须列配送费**，接口未返回就写"以支付页为准"，不得省略。
- **地址簿多个地址必须让用户选**，不得默认取第一条。

## 安装

把 `SKILL.md` 放到用户级技能目录：

```bash
# macOS / Linux
mkdir -p ~/.workbuddy/skills/mcd-meal-advisor
cp SKILL.md ~/.workbuddy/skills/mcd-meal-advisor/

# Windows (PowerShell)
New-Item -ItemType Directory -Force "$HOME\.workbuddy\skills\mcd-meal-advisor"
Copy-Item SKILL.md "$HOME\.workbuddy\skills\mcd-meal-advisor\"
```

或直接克隆本仓库到该目录：

```bash
git clone https://github.com/xiongermiao/mcd-meal-advisor.git ~/.workbuddy/skills/mcd-meal-advisor
```

## 依赖

需要在 WorkBuddy 里连接麦当劳 MCP（连接器 `mcd-mcp`），提供以下工具：

- `query-nearby-stores` / `delivery-query-stores`
- `query-meals` / `query-meal-detail` / `list-nutrition-foods`
- `query-store-coupons` / `query-my-coupons` / `query-my-account`
- `calculate-price` / `query-promotions`
- `order-list` / `query-order` / `create-order`
- 积分兑换：`mall-points-products` / `mall-product-detail` / `mall-create-order`

未接入该工具集时本 Skill 无法执行——不要凭记忆报价格或编造菜单。

## 目录

```
.
├── SKILL.md    # 技能主文件（唯一必需文件）
└── README.md
```

## 变更记录

- **2026-10-24（下午）** 履约定位按场景分叉。麦乐送/团餐不再要求用户选门店：定位地基改为配送地址 `addressId`，门店由 `delivery-query-stores` 按地址指派，新增 `assigned` 状态（只告知不确认）。外送门店择优限定为「营业 → 时段 → 返回顺序首家 → 缺货才换店」，因为实测该接口不返回距离/配送费/预计时长/门店地址。闸门 1/2 按场景拆分，确认单按场景裁剪并强制列配送费。下单前置检查由五查扩为六查。
- **2026-10-24** 新增「履约定位：可推荐，须确认」小节。原规则"未说明默认外带""无定位走收藏门店"废止，改为给候选 + 等用户确认；下单前置检查由四查扩为五查。

## 边界

- 一切实时数据（门店、时段、菜单、价格、券、积分）以接口返回为准，不缓存、不假设。
- 只创建订单，支付由用户自行完成。
- 不记录任何商品编码或价格，避免文档过期。
