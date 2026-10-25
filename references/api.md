# 接口要点与数据形态

**何时读**：需要核对具体参数名、返回字段、数据形态，或要按品类给搭配策略时。纯流程推进不必读。

主文件第八章只留「每次调用都要遵守」的几条；**本文件是查手册性质**，遇到"这个字段叫什么""返回里有没有 X""这单为什么拼错参数"再翻。

## 一、商品与券

- **两套商品体系**：常规菜单商品 vs 券虚拟商品，券商品不在普通 query-meals 列表里。用券时必须传券自带的 `products[].productCode` + couponId + couponCode，不能拿普通菜单的 code 硬配。
- **query-my-coupons 只能展示、不能算价**（不返回 couponId/couponCode）。凡涉及金额一律走 query-store-coupons 拿可算价的券。
- **套餐默认直接传 productCode 即可算价**，不必拆 roundList。用得到 `query-meal-detail` 的 rounds 结构只有两种情形：**换套餐内小食/饮料**，以及**算整餐营养**（营养库无套餐，必须拆到单品加总，见 `nutrition.md`）。
- **随单购是陷阱**：discountType 标"随单购麦金卡优惠"的商品，优惠价前提是先买麦金卡（withOrder.membershipCode / membershipSpecId），不得当普通优惠推荐。
- **参数名不一致**：query-meals 返回 `withOrder.specId`，calculate-price 入参是 `membershipSpecId`。
- **特调 key**：含 `unselectedKey` 的组，组内所有项都要传，选中用 `selectedKey`、未选中用 `unselectedKey`，漏传会导致价格或商品不一致。特调组 `minValues=1 / maxValues=1` 且只有 1 项时是必选，不许给用户"不选"的选项。
- **query-meals 的餐品对象只有这 7 个字段**（实测）：`name` / `image` / `currentPrice` / `originalPrice` / `discountType` / `canWithOrder` / `withOrder`。**没有营养数据、没有规格列表、没有 roundList**——营养走 `list-nutrition-foods`，规格与套餐构成走 `query-meal-detail`。
- **`image` 字段真实存在**（`https://menu-img.mcd.cn/...`，统一 320px，PNG 与 GIF 混用）。但聊天窗口的渲染白名单不含该域名，**图片在文字回复里显示不出来**，不要因为接口有 URL 就往回复里贴 `<img>`；要放图只能用可视化渲染容器，且实测其 CSP 同样拦截。
- **换套餐内小食/饮料时**，roundList 的 `round` 字段填 `rounds[].id` 的**字符串**形式（如 "1"、"2"），不是数字。
- **积分兑换**：mall-create-order 仅支持 shopId=2；spuCategory=1 虚拟券、2 实物（实物需 addressId）。派对类活动走 party-order-create，不走此接口。

## 二、配送型接口细节

- **麦乐送定位链（不可跳步）**：`delivery-query-addresses`（拿 addressId / contactName / fullAddress / phone）→ **用户确认地址** → `delivery-query-stores(addressId, beType=2)` → 硬筛营业与时段 → 取首家 → `query-meals(storeCode, orderType=2, beType=2, beCode)` → `calculate-price` 同参 → `create-order` 额外传 `addressId`。**全程不需要用户选门店。**
- **`delivery-query-stores` 实际返回字段（实测）**：`storeCode`、`beCode`、`storeName`、`reservation`、`reservationTimeOptions`、`businessStatus`、`businessStartTime`、`businessEndTime`。**没有 distance、没有配送费、没有预计送达时长、也没有门店地址**（工具说明里写的 `data[].address` 实际不返回）。这些数据不存在，就一律不许编。
- **外送同样要过时段校验**：`delivery-query-stores` 也返回 `reservation` 与 `reservationTimeOptions`，不要把"外送"当成"不用管时段"。注意这些 option 通常只列**尚可预约**的时段，当前正在进行的时段可能不在其中——判断"现在能不能送"要用 `businessStatus` + `businessStartTime/EndTime`，判断"某时刻能否预约"才用 `reservationTimeOptions`。
- **团餐 create-order 需 `gmServiceCode`**（来自 `query-meal-assistance`）；外送需 `addressId`。
- 同一地址可能返回 0 / 1 / 多家门店，处理方法见 `delivery.md` 第一节。

## 三、门店与菜单数据形态

- query-meals 的 `data.categories` 是"推荐位"聚合，同一 code 会重复出现；真实商品详情在 `data.meals` map，`data.frequent` 是常点。
- **`data.frequent` 只在到店型菜单里稳定出现，外送场景实测可能不返回**（只有 categories + meals）。用户说"点我常点的"时，若该次 query-meals 没有 frequent，回退到 `order-list` 历史自行统计，并说明"这是按你的历史订单推的"，不得假装是接口给的常点。
- **`query-nearby-stores` 的 `distance` 单位是米，但不是"离用户多远"**：接口返回的 longitude / latitude 常为 0，距离基准是本次搜索的中心点（city + keyword），不保证是用户实时定位。展示措辞的三档规则见 `display.md`。
- **`delivery-query-stores` 字段**见本文件第二节。

## 四、品类实操（策略，不含具体商品）

- **早餐**：优先找组合类早餐套餐（内含主食与饮品可选位），通常比单品拼凑便宜；含薯饼的三件套更贵，只在用户明确要"吃饱"时才升级推荐。早餐供应窗口普遍很短（多在上午两三个小时内），时段外的"早餐"诉求必须补问日期，不得直接视作明天而不说。
- **健康 / 控卡**：用户提"健康一点"时，可先翻 categories 里有没有官方控卡类目（如"500 大卡套餐"一类，名称与内容会变）来**收缩候选范围**，但它只是入口、不是证据——类目名与商品名（含"蔬萃""轻食"）都不得当达标依据，最终仍须按 `nutrition.md` 用营养数据逐项核对。
- **极低预算（约 20 元以内）**：常规三件套套餐的起价普遍高于这一档，硬凑"汉堡＋薯条＋饮料"必然超支。优先找「随心配 / 1+1」这类"主食区选一件 + 饮品或小食区选一件"的两件组合——通常一个汉堡的钱就能拿到汉堡加饮料，且两个区内部同价自由换，先问清用户想要哪一类再定具体项。其次才是"单点汉堡 + 用券"。
- **餐品搭配规则**（query-meals 内置，团餐场景可参考）：20 元以下配小食；20-30 汉堡+小食或饮料；30-40 汉堡+薯条/小食+饮料；40-50 汉堡+薯条+小食+饮料；50 以上丰富组合、优先不重复小食。
- **多人（≥3 人）点餐**：先算人均上限（预算 ÷ 人数），再按"人数 × 单人套餐"配，不要按单品拼凑——多单加总最易超预算，而拼凑出来的份量还不均。忌口人数从整单里单独拆出来配（如 6 人里 2 人忌辣 → 4 份默认主食 + 2 份换主食），**不迁就全单**。同一 productCode 要用不同配置时，拆成多行 items（一行默认只传 productCode，另一行带 `roundList`），核价与下单都按行分别计价（实测 4 份默认 + 2 份换主食可各自计出不同单价）。
- **套餐 vs 随心配的取舍要报数**：预算紧时「随心配/1+1」比三件套便宜很多（常能省 30% 以上），代价是少了薯条这类正餐小食。两份都满足硬约束时，用价差百分比说明取舍，别只说"更划算"。
- **营养估算**：整餐营养用单品加总估算（按 `rounds` 拆项、乘数量），方法与匹配判据见 `nutrition.md`——**任一单品未匹配，整餐值只能标"待核实"**。展示必须标"估算值"，不宣称"达标"。
