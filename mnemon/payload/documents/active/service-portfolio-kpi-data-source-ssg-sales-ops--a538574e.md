---
id: "a538574e-ee32-4c2d-b97c-85a1f34396bc"
title: "Service Portfolio KPI 配置页「保存即降级」缺陷与 Data Source 口径核查（ssg-sales-ops，2026-11-19）"
description: "8 张 KPI 卡全部由 sp_sp_kpi_card 驱动；前端 valueType 枚举 MANUAL|AUTO 与后端 COMPUTED 不一致，kpiConfig.ts toValueType() 读时降级，导致配置页点一次 Save 就把 5 张 COMPUTED 卡改成手动空值（TST 2026-11-19 14:20:52 已发生）；Data Source 仅备注不参与取数；排序需强刷；修复三步待用户拍板。"
status: "active"
created_at: "2026-11-19T06:31:03.020Z"
updated_at: "2026-11-19T06:31:03.020Z"
content_hash: "2f54828fb72d2015882738989e5f0b7a82a8d8f07c93131dbe902fd9006b0617"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioConfigService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/dao/ServicePortfolioConfigMapper.java"
session_ids:
  - "mnemon-review-v1-e6b457e8-1597-4d93-a191-5ef22914093e"
memory_body_ids:
  []
---

# Service Portfolio KPI 配置页「保存即降级」缺陷与取数口径核查（ssg-sales-ops，2026-11-19）

页面：Business Insights / Big Events 顶部 8 张 KPI 卡（TST，`https://ssgspark-tst.lenovo.com/spark/portfolio`）。
相关工作区：`C:\\Users\\63588\\lenovo\\ssg-sales-ops`；分支 `feature/sp-api-fix-admin`。
前置调查见文档 `358e5584-8bcf-4543-b670-dad61175c405`（KPI 数值来源与接口权限现状核查，2026-11-18）与 `e64a20e2-b44a-4c2e-a9ba-dfd6aaf9ee20`（8 行目标配置与会议口径裁定）。

## 0. 结论摘要
- 8 张卡的**文案/排序/显隐/跳转/取值方式全部来自 PG 表 `public.sp_sp_kpi_card`**，代码里没有写死数字（`defaultsOfKpi()` 兜底已删）。
- 但存在一个**会自我破坏数据的枚举不一致缺陷**：前端 `valueType` 只用 `MANUAL | AUTO`，后端只认 `COMPUTED`；配置页读取时把 `COMPUTED` 降级成 `MANUAL`，因此**在配置页点一次 Save 就会把 5 张自动取数卡改成手动空值**。该破坏在 TST 已于 2026-11-19 14:20:52 实际发生（操作人 zhangcz1）。
- `Data Source` 输入框是纯备注字段，后端无任何代码读取，**不能写 SQL、也不是取数入口**。
- 排序问题的答案是「已保存成功，只是页面需 Ctrl+F5」，整条链路无 bug。
- 数据修复需用户点头后执行（按既有规矩不擅自动库）。

## 1. 数据来源判定
- 读取链路：`ServicePortfolioConfigMapper.selectKpiCards()` → `SELECT * FROM public.sp_sp_kpi_card WHERE visible = TRUE ORDER BY sort_no, id`（`ServicePortfolioConfigMapper.java:15`）。
- `ServicePortfolioReadService.kpiCards()` 注释明确：「KPI 卡完全由配置表驱动（管理员在配置页维护）… 配置为空就返回空列表，不在代码里放默认卡」，初始 8 行由 `V1.1.129` 种子迁移写入。
- 展示页无二次排序：`components/Kpis.vue:39` 直接 `v-for` 渲染接口返回数组；`index.vue:32` 的 `kpis` computed 只做字段映射。
- 取值：`ServicePortfolioReadService.resolveMetric`（`:84-96`）**只认 5 个 metric key**，其余一律回退 `manual_value`：

```java
private Object resolveMetric(String key, String valueType, Object manualValue) {
    if (!"COMPUTED".equalsIgnoreCase(valueType)) {
        return manualValue;
    }
    return switch (key) {
        case "OH_CODES" -> readMapper.countOhCodes();
        case "ACTIVE_PN" -> inThousands(readMapper.countActivePn());
        case "PN_TERMINATED" -> inThousands(readMapper.countObsoletePn());
        case "ODT_PROJECTS" -> readMapper.countConcordProjects();
        case "REQUESTS" -> hbbRequestMapper.countRows(new HbbRequestQuery(), null);
        default -> manualValue;
    };
}
```

- 新增任意 metric key 即使选了「From data source」也不会取数，会静默退回手动值。

## 2. `Data Source` 字段的真相（用户提问②）
- 前端 `PortfolioKpiConfig.dataSource` 类型为 `string`，placeholder `Source name, table or report`（`config/KpiCardsConfig.vue`），保存到 `sp_sp_kpi_card.data_source` 列。
- 后端保存逻辑 `ServicePortfolioConfigService.saveKpiCards()`（`:97-140`）只是 `entity.setDataSource(text(card.get("dataSource")))` 落库；`resolveMetric` 完全不读该列。
- 结论：**业务不需要、也无处录入 SQL**；表名/报表名写进去也只是一段备注文本。

## 3. 排序「改了不生效」的结论（用户提问①）
链路全部正常：配置页移动后重编号（`config/KpiCardsConfig.vue:128`、`:132-139`，保存时 `:144` 重新赋 `sortNo`）→ 后端 `saveKpiCards` 写 `sort_no` → 查询 `ORDER BY sort_no, id` → 前端按数组顺序渲染。
TST 库实测 14:20:52 那次保存已把顺序写入（`ACTIVE_PN` 被移到 `sort=1`），说明**排序生效、只是浏览器未刷新**；若强刷仍不对才需要怀疑部署版本。
（注：`KpiCardsConfig.vue` 行号来自 checkpoint 中摘要化的 grep 命中，`:128/:132-139/:144` 为移动与重编号相关行。）

## 4. 核心缺陷：前端 `AUTO` vs 后端 `COMPUTED`
- 前端类型：`export type KpiValueType = 'MANUAL' | 'AUTO'`（`config/kpiConfig.ts:4-5`），下拉选项 `<el-option label="From data source" value="AUTO" />`（`KpiCardsConfig.vue`）。
- 转换函数 `toValueType()`（`config/kpiConfig.ts:37-39`）：`valueTypes.includes(value) ? value : 'MANUAL'` —— 后端返回的 `COMPUTED` 不在白名单，被**降级成 `MANUAL`**；`toKpiCard()`（`:46-66`）再用它构造表单，`toValueType(card.valueType)` 在 `:53`。
- 反向 `toBackendCard()`（`:69-86`）原样回传 `valueType`，于是 `MANUAL` 被写回库。
- 双重后果：
  1. 选「From data source」保存 → 存成 `AUTO`，后端 `!"COMPUTED"` 判定为手动 → 卡片变空。
  2. 仅打开配置页点 Save（不做任何修改）→ 5 张 `COMPUTED` 卡被改写为 `MANUAL` + 空 `manual_value` → 展示页对应卡全空。

## 5. TST 实际破坏状态（2026-11-19 14:20:52，updatedBy=zhangcz1）
```
sort | metric        | valueType | manual | unit | visible
1    | ACTIVE_PN     | MANUAL    |        | K    | true
2    | OH_CODES      | MANUAL    |        |      | true
3    | ODT_PROJECTS  | MANUAL    |        |      | true
4    | REQUESTS      | MANUAL    |        |      | true
5    | FAST_PASS     | MANUAL    | 94     | %    | true
6    | CROSS_TEAM    | MANUAL    | 9      | +    | true
7    | SLA           | MANUAL    | 1.5    | WD   | true
8    | PN_TERMINATED | MANUAL    |        | K    | true
```
前 5 张（`ACTIVE_PN`/`OH_CODES`/`ODT_PROJECTS`/`REQUESTS`/`PN_TERMINATED`）原本应为 `COMPUTED`，现全部无手动值 → 展示页会显示空白；`FAST_PASS`/`CROSS_TEAM`/`SLA` 三张 MANUAL 卡正常。

## 6. 建议修复顺序（截至记录时用户尚未拍板）
1. **数据侧（1 分钟）**：把 5 张卡 `value_type` 改回 `COMPUTED`，刷新即恢复数值；不改库则展示页持续空白。
2. **代码侧（根治）**：前端下拉值改为 `COMPUTED`（同时保留 `AUTO` 兼容历史数据），`toValueType()` 不得把 `COMPUTED` 降级；否则下次任何人点 Save 都会再次破坏。
3. **文案侧**：把 `Data Source` 输入框标注为「仅备注，不参与取数」，避免业务误以为可写 SQL/表名。

## 7. 未闭合事项
1. 是否立即修库（把 5 张卡改回 `COMPUTED`）待用户答复——按规矩不擅自修改数据。
2. 代码修复未实施、未建 MR；需同时处理前端枚举与历史 `AUTO` 数据兼容。
3. PRE/PROD 上线前仍需 `V1.1.129` 种子迁移把该表填满（两库该表仍 0 行，见文档 `358e5584-8bcf-4543-b670-dad61175c405`）。
