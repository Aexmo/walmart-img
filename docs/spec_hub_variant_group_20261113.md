# HUB 大端改造规格：变体父子结构 + 前端父体折叠（2026-11-13）

> 提出方：HERMES_GIGABYTE（ZACK） · 执行方：YIMU_HUB（大端 / dropship-hub 服务端）
> 背景拍板：Aaron 2026-11-13 ——「每次上架要判断哪些产品是从同一个父体组弄下来的，产品端必须以多变体形式呈现」
> 关联任务：AUPlus 总线 `t_4a8de7926a`（`__probe__`，误发测试，请忽略并视作已关闭）

---

## 1. 问题（现状证据，已用 API 实测）

| 检查项 | 实测结果 |
|---|---|
| 父体行 `variants` 数据 | ✅ 已写入：`D0102HEGZ3X` = 36 条、`D0102HQE8AW` = 22 条，形状 `[{sku, options:{规格:"Design:Barack"}, price, stock}]` |
| `GET /api/products` 列表行 | ❌ 不返回 `variants`，也没有任何 `parent_id` / `variant_group_id` 字段 |
| HUB API 48 个端点 | ❌ 没有变体组/父体相关端点 |
| 后果 | 列表页把 1699 条当平铺行渲染 → 同一个父体的 36 个设计款在后台显示为 36 张独立商品卡，Aaron 看到「还是散的」 |

**结论：数据层已具备（父体行的 variants），缺的是「父子关系模型 + 列表折叠呈现」。**

---

## 2. 目标（三条，缺一不可）

1. **列表页折叠**：同一父体组在商品列表里只占**一张父体卡**，卡上显示「N 个变体」徽标 + 代表图（primary 变体）。
2. **详情页变体面板**：点父体卡进入详情，能列出全族子体（Design 下拉/缩略图），切换后图/价/库存/成本随之切换。
3. **子体仍是独立产品行**：每个子体保留自己的 `item_no / UPC / 价格 / 库存 / 洗图 / 上架状态`（下游 pending-deploy、lister、walmart_sync 都按 item_no 走），**不允许把子体合并成一行而丢掉 SKU 粒度**。

---

## 3. 数据模型（建议最小改动）

### 3.1 `products` 表新增字段（nullable，向后兼容）

```sql
ALTER TABLE products ADD COLUMN variant_group_id TEXT;      -- 父体组 ID，如 VG-pop-style-10-round-plate；非变体品为 NULL
ALTER TABLE products ADD COLUMN variant_role TEXT;          -- 'parent' | 'child' | NULL
ALTER TABLE products ADD COLUMN parent_item_no TEXT;        -- 子体 → 指向父体 item_no；父体自身为 NULL
ALTER TABLE products ADD COLUMN variant_label TEXT;         -- 子体在族内的标签值，如 'Barack'
ALTER TABLE products ADD COLUMN variant_attribute TEXT;     -- 变体维度名，如 'Design'
CREATE INDEX IF NOT EXISTS idx_products_vgroup ON products(variant_group_id);
```

> 说明：`variants`（已有 JSON 列）继续保留在**父体行**上，作为「族内清单快照」；新增列是给**列表/筛选/联查**用的结构化字段，避免每次列表都解析 JSON。

### 3.2 关系不变量（写入时必须校验）

- 一个 `variant_group_id` 下 **恰好一个** `variant_role='parent'`；
- 同族所有成员 `variant_attribute` 相同、`variant_label` **互不相同**；
- 父体本身也是一条真实产品行（有自己的 item_no/UPC/成本/库存），不是虚拟记录；
- `parent_item_no` 只允许指向同族父体，禁止跨族；
- 删除父体时，子体 `parent_item_no` / `variant_group_id` 置空（降级为独立品），**不级联删除**。

---

## 4. API 变更

### 4.1 `GET /api/products`（列表）

**返回新增字段**（每行）：`variant_group_id`、`variant_role`、`parent_item_no`、`variant_label`、`variant_attribute`、`variant_count`（父体行给族内总数，子体行给 NULL）。

**新增查询参数**：

| 参数 | 取值 | 语义 |
|---|---|---|
| `group` | `variant`（默认） \| `flat` | `variant` = 折叠视图，**只返回父体行**（子体不占行）；`flat` = 现状全量平铺（保持旧行为，兼容脚本） |
| `include_children` | `1` | 与 `group=variant` 搭配，把子体也返回（前端「展开变体」用） |
| `variant_group_id` | 任意 | 只看某一族 |

⚠️ **默认值必须是 `group=variant`**（Aaron 的诉求是默认看到父体），但**必须保留 `flat` 分支**——我们侧大批脚本（`walmart_sync.py` / `wm_inventory_patrol.py` / `hub_price_cost_patrol.py` / 每日同步）现在都是拉全量按 item_no 索引，走的必须是 `flat`。为免一次性改我们所有脚本，请**同时**满足：`group=flat` 与不传 `group` 但带 `all=1` 都返回全量平铺。

### 4.2 `GET /api/products/{pid}`（详情）

返回新增：

```json
{
  "product": { "...": "原有字段", "variant_group_id": "VG-...", "variant_role": "child", "parent_item_no": "D0102HEGZ3X", "variant_label": "Barack", "variant_attribute": "Design" },
  "variant_group": {
    "family_id": "VG-pop-style-10-round-plate",
    "attribute": "Design",
    "parent_item_no": "D0102HEGZ3X",
    "members": [ { "item_no": "...", "label": "Elon Musk", "is_primary": true, "price_cost": 13.64, "inventory": 500, "main_image": "..." } ]
  }
}
```

### 4.3 新增写端点 `POST /api/products/variant-group`（幂等，ZACK 侧调用）

```json
{
  "family_id": "VG-pop-style-10-round-plate",
  "attribute": "Design",
  "parent_item_no": "D0102HEGZ3X",
  "members": [
    {"item_no": "D0102HEGZ3X", "label": "Elon Musk"},
    {"item_no": "D0102HEGZ36", "label": "Barack"}
  ]
}
```
- 行为：按 item_no 查找 → 写 `variant_group_id/role/parent_item_no/variant_label/variant_attribute` → 若族已存在则**全量覆盖成员关系**（先清旧族的父子字段再写，避免残留）。
- 返回：`{"ok": true, "family_id": "...", "members": 36, "created": 1, "updated": 35, "removed": 0}`
- 校验失败返回 400 + 明确原因（父体不在成员里 / label 重复 / 缺 attribute）。
- 我们侧只需要这一个端点，就能把任意批次的分族结果灌进 hub（当前 2 族已就绪，后续每批 200 品会陆续灌）。

### 4.4 不变（严禁改动）

- `POST /api/inventory/update`（三字段轻量回传）、`POST /api/ingest/upload`（含 `?reprocess=1`）行为不变；
- `GET /api/rewrites/pending-deploy?platform=&operator=YIMU` 返回结构不变，**折叠逻辑不得影响上架队列**；
- `GET /api/listings`、media / wash / rewrite 系列不变。

---

## 5. 前端（商品列表页 + 详情页）

### 5.1 列表页

- 默认折叠：父体行渲染为**一张卡**，标题下方加徽标「36 个变体 · Design」；卡片主图用 primary 变体的主图。
- 子体不进默认列表；卡片右上角提供「展开变体」→ 原地展开 36 行缩略行（或抽屉）。
- 保留一个筛选器：`显示方式：折叠 / 平铺`（默认折叠）。
- 搜索/类目/状态筛选在折叠模式下按「父体 » 任意子体命中」判定（子体命中时把该父体整卡带出并高亮命中子体）。

### 5.2 详情页

- 父体详情：顶部变体选择器（Design 下拉 or 缩略图条），列出族内全部子体；选中后切换：主图/图集、`price_cost`、`inventory`、`UPC`、洗图版本、上架状态。
- 子体详情：顶部显示「所属父体：D0102HEGZ3X（36 个变体）」+ 一键跳父体。
- 变体选择器里**每项显示该子体自己的库存与成本**（Aaron 已定：变体定价按子体自身成本，不套母体价）。

---

## 6. 数据契约（ZACK 侧已产出，可作迁移输入）

`/api/products/variant-group` 的请求体与本文件 `families_20261113.json` 完全同源。样例（真实数据，2 族 58 子体）：

```json
{
  "family_id": "VG-pop-style-10-round-plate",
  "attribute": "Design",
  "parent_item_no": "D0102HEGZ3X",
  "members": [
    {"item_no": "D0102HEGZ3X", "label": "Elon Musk"},
    {"item_no": "D0102HEGZ36", "label": "Barack"},
    {"item_no": "D0102HQEZYV", "label": "Mona"}
  ]
}
```

完整清单见随本任务上传的 `families_20261113.json`（`families[]` 每项含 `family_id / attribute / parent_item_no / members[{item_no,label,price,inventory,is_primary}]`，`singles[]` 为不成族的单件）。

**分族口径（ZACK 侧算法，供参考，不需要你在服务端实现）**：同供应商 + 同形态尺寸（如 `10" Round Plate`）= 同一父体；标题差异段 = 变体标签；主题词（Gift Idea / Political / Artist）不参与分组。

---

## 7. 验收标准（可机器核对，逐条给证据）

1. `GET /api/products?group=variant` 返回行数 = 单件数 + 族数（当前应为 42 + 2 = **44**），父体行带 `variant_count=36/22`。
2. `GET /api/products`（不传参或 `group=flat`）仍返回 **1699** 全量行（不破坏下游脚本）。
3. `GET /api/products?group=variant&include_children=1` 返回 1699 行，且每行 `parent_item_no/variant_label` 正确。
4. `POST /api/products/variant-group` 重复调用幂等：第二次返回 `created=0`，成员关系不变。
5. 前端：列表默认看到 2 张父体卡 + 42 张单件卡；父体卡徽标显示 36 / 22；展开能看到全族子体；详情页切换 Design 时主图与库存变化。
6. `GET /api/rewrites/pending-deploy?platform=walmart&operator=YIMU` 返回内容与改造前**完全一致**（diff = 0）。

---

## 8. 兼容与回滚

- 迁移只加列不改列，**旧代码不读新列也不受影响**；前端折叠为纯展示层。
- 我们侧所有脚本改用 `group=flat` 之前，请保持 `flat` 行为与今日一致（这点是硬要求，改错会打断沃尔玛每日同步与库存巡检）。
- 回滚：`variant_group_id/variant_role/parent_item_no/variant_label/variant_attribute` 全部置 NULL + 前端默认切回平铺，即恢复今日状态；不改任何 products 主数据。

---

## 9. 分工与交付物

| 项 | 归属 |
|---|---|
| DB 迁移 + 写端点 + 列表/详情字段 | **YIMU_HUB（大端）** |
| 前端折叠/变体面板 | **YIMU_HUB（大端）** |
| 分族结果灌入（调 `POST /api/products/variant-group`） | ZACK（本机脚本 `build_variant_families.py`，已就绪） |
| 沃尔玛侧 `variantGroupId` / `isPrimaryVariant` / `variantAttributeNames` 上架 | ZACK（lister 改造，等端点上线后同步做） |

**交付物**：迁移脚本 + 端点实现 + 前端改动 + 一份「验收 6 条」的自测输出（贴命令与返回）。

**优先级建议**：4.1 列表字段 + 4.3 写端点（ZACK 侧被阻塞在这里）→ 5.1 列表折叠 → 5.2 变体面板。前两项上线后我可以立刻灌数据并给出验收证据。

---

## 10. 联系与上下文

- 提出方：HERMES_GIGABYTE（ZACK，Windows 本机）
- 相关真实数据：父体 `D0102HEGZ3X`（36 变体）、`D0102HQE8AW`（22 变体），均已在 hub 里写入 `variants`
- 待办批次：ZACK 正在按同规则再采 200 品，会产生更多族（预计以 Pop Style / Smart Finds / Design Living 为主），端点上线后批量灌入
