# mcd-meal-advisor

麦当劳点餐顾问 Skill，给 [WorkBuddy](https://www.workbuddy.cn) 用。基于麦当劳 MCP 工具集，把"点一份麦当劳"拆成一条可核对的流程：确定场景 → 检查权益 → 选品筛选 → 试算展示 → 用户确认 → 积分兑换 → 落定 → 下单。

能力边界到**创建订单**为止，不代付、不查询支付结果。

## 它解决什么问题

点餐最难的从来不是"哪个汉堡好吃"，而是那些**看起来不用问、其实必须问**的底层参数：在哪家店取、堂食还是带走、几点取。这些值定错了，后面的菜单、优惠券、价格全都是错的，而且错得很安静——接口不会报错，只会给你另一家店的正确答案。

所以本 Skill 的核心约束是：**可推荐，须确认**。

## 四条硬约束

| 约束 | 要求 |
|---|---|
| 1. 信息分级默认 | 人数、饭量可直接默认；**门店、取餐大方式、取餐方式必须用户确认**；时间不得由餐次推断 |
| 2. 临期积分提醒 | 只在积分快过期时提醒，同会话只提一次；提议兑换的"预计可省"不得倒扣进总价 |
| 3. 先满足约束再优化搭配 | 明确预算/忌口是硬筛选，口味偏好只影响排序；无解时明说放宽了哪项 |
| 4. 报价带状态 | 已核价 / 待核价 / 待核实三态分立；门店或任何参数一变，价格与推荐理由一起失效 |

### 履约定位：门店 / 取餐大方式 / 取餐方式

三项各有 `confirmed` 与 `candidate` 两态，`candidate` 只表示"我建议这个"，不等于已定。

- **推荐来源**：本次明确表达 > 最近点单 > 收藏。`order-list` 无参可调，能给出历史门店、`beType`；再调 `query-order` 可拿到门店地址与 `takeWay`。
- **"常点"不在此列**：`query-meals` 必传 `storeCode`，`data.frequent` 在选店前拿不到，它属于选品维度。
- **两个闸门**：门店或取餐大方式未确认 → 禁止查菜单/核价；取餐方式未确认 → 禁止出确认单、禁止下单。
- **取餐方式两段式**：核价前确认"意图"（堂食/外带），核价后用 `takeWayList[].code` 映射成实际码值，映射不到就回报重选，不得静默替换。
- **展示三义务**：标来源（哪一单、哪家收藏）、报地址不报距离、城市一致性检查。

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

- **2026-10-24** 新增「履约定位：可推荐，须确认」小节。原规则"未说明默认外带""无定位走收藏门店"废止，改为给候选 + 等用户确认；下单前置检查由四查扩为五查。

## 边界

- 一切实时数据（门店、时段、菜单、价格、券、积分）以接口返回为准，不缓存、不假设。
- 只创建订单，支付由用户自行完成。
- 不记录任何商品编码或价格，避免文档过期。
