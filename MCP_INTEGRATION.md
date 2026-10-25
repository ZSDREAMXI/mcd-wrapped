# MCP 集成说明

## 使用的 MCP Server

| 项 | 值 |
| --- | --- |
| Server | 麦当劳中国官方远程托管 MCP Server（[M-China/mcd-mcp-server](https://github.com/M-China/mcd-mcp-server)） |
| 接入地址 | `https://mcp.mcd.cn` |
| 传输方式 | Streamable HTTP |
| 鉴权 | `Authorization: Bearer <MCP Token>`（Token 于 mcd.mcd.cn 控制台申请） |

## 使用的工具

| 工具 | 用途 | 角色 |
| --- | --- | --- |
| `order-list` | 拉取当前用户全部历史订单（订单/商品/金额/门店/时间） | **核心数据源** |
| `available-coupons` | 查询麦麦省当前可领优惠券 | 个性化建议（可选） |
| `query-my-coupons` | 查询用户卡包中的券 | 个性化建议（可选） |
| `campaign-calendar` | 查询当月营销活动日历 | 个性化建议（可选） |

说明：下单类工具（`create-order` 等）在本 skill 中**默认不调用**——报告类场景只读不写，除非用户明确要求下单。

## 调用流程

```
用户："生成我的麦当劳年度报告"
        │
        ▼
① order-list ──► 全量历史订单（含套餐子项、门店、时间、实付金额）
        │
        ▼
② 读取本地 mcd-wrapped-records.json（站外团购补录）
   └─ 无文件且存在 0 元单 → 询问用户是否补录（一次）
        │
        ▼
③ 清洗：剔除已取消订单；0 元单匹配补录记录（整单核销 / 混合单）
        │
        ▼
④ 指标计算：真实总花费 = 应用内实付 + 站外实付；
   最常点 TOP5 / 招牌组合 / 时段 / 城市画像
        │
        ▼
⑤ 人格判定（固定规则表，顺序匹配）→ 年度"麦门人格" + 数据判词
        │
        ▼
⑥ 渲染 assets/wrapped-template.html → 独立 HTML 年报
        │
        ▼
⑦ 个性化建议（可调用 available-coupons / campaign-calendar 落到真实优惠）
```

## 实测数据契约（踩坑记录）

以下结论来自对 `order-list` 真实返回的实测，写入了 `SKILL.md` 以保证可复现：

- `realTotalAmount` 为**元**为单位的字符串（如 `"13.5"`）；`"0"` 表示应用内实付 0 元，可能是券/积分兑换，也可能是站外团购券核销；
- 套餐子项位于 `orderProductList[].comboItemList[]`，字段名是 `name`（不是 `productName`）；
- 接口无分页参数，一次返回全部；
- 城市名嵌在 `storeName` 中（紧随"麦当劳"）。

## 业务价值

- **对用户**：订单流水 → 可分享的年度记忆（人格认证 + 判词），并把站外团购支出还原进真实账本，看清"到底花了多少钱"；
- **对生态**：示范了 MCP 在"数据洞察 + 内容生成"方向的用法——不止能点餐，还能把账户数据变成情感化内容；严格的只读边界（默认不下单）也是对 MCP 安全使用的一种实践。
