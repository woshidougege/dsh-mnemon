---
id: "358e5584-8bcf-4543-b670-dad61175c405"
title: "Service Portfolio KPI 数值来源与接口权限现状核查（ssg-sales-ops，2026-11-18）"
description: "8 张 KPI 卡片全部来自 sp_sp_kpi_card（defaultsOfKpi 已删）；5 张 COMPUTED 的 SQL/实测值（1618/278K/113/10/54K）与 3 张 MANUAL 手填值；ODT 113 vs PPT 30+、Requests 10 vs 12K+ 的口径偏差待确认；只读接口无 admin 校验、仅 6 处写操作有 requireAdmin（用户决定暂不加控制）；approvals 仅 requireLogin 的越权隐患；上线前置 V1.1.129 种子迁移。"
status: "active"
created_at: "2026-11-18T06:49:43.879Z"
updated_at: "2026-11-18T06:49:43.879Z"
content_hash: "f234fa6322cb05574cc18adca560e8da94a986c8cda6ec7182ff8223b7db6b73"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioConfigController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/mapper/ServicePortfolioReadMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/HbbRequestService.java"
session_ids:
  - "mnemon-run-v1-8d0d6b8d-9643-4fbc-ad85-2d6661c238fc"
memory_body_ids:
  []
---

# Service Portfolio KPI 数值来源与接口权限现状核查（ssg-sales-ops，2026-11-18）

## 0. 结论摘要
- 顶部 8 个 KPI 卡片的**标题/副标题/单位/排序/显隐/跳转**全部存在 PG 表 `public.sp_sp_kpi_card`，由管理员在 Portfolio configuration 页维护；接口只是遍历该表输出，代码里**已无写死数字**（旧的 `defaultsOfKpi()` 兜底已在 MR !576 删除）。
- 8 个数值中 **5 个实时算（value_type=COMPUTED）、3 个手填（MANUAL）**。
- 只读接口**没有** admin 限制；仅 6 处写操作有 `requireAdmin`。用户明确决策：**暂时不加 admin 控制**，在该口头确认前不动代码。
- 表为空时 `/kpi-cards` 返回空数组、页面 KPI 区空白 → PRE/PROD 上线前必须跑 `V1.1.129` 的种子迁移（这两个库该表仍 0 行）。

## 1. 卡片清单来源
`smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java:66-82`：
```java
for (Map<String, Object> card : configMapper.selectKpiCards()) {
    row.put("value", resolveMetric(key, str(card.get("value_type")), card.get("manual_value")));
}
```
表列：`metric_key / title / subtitle / value_type / manual_value / unit / sort_no / visible / link_type / link_url`。

## 2. 数值解析链路
`ServicePortfolioReadService.java:84-96`（`resolveMetric`）按 `value_type` 分派：
```java
case "ACTIVE_PN" -> inThousands(readMapper.countActivePn());
case "PN_TERMINATED" -> inThousands(readMapper.countObsoletePn());
case "ODT_PROJECTS" -> readMapper.countConcordProjects();
case "REQUESTS" -> hbbRequestMapper.countRows(new HbbRequestQuery(), null);
default -> manualValue;
```
`ServicePortfolioReadService.java:369`：`private static long inThousands(long value) { return Math.round(value / 1000.0); }` —— **K 是渲染时算的，不是库里的值**，所以 ACTIVE_PN 真值是 277,940 而非 278,000。

## 3. 8 张卡明细（TST 库实测，2026-11-18）
| sort | metric_key | type | 来源 / SQL | manual_value | unit | 跳转 | 实测 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | OH_CODES | COMPUTED | `gps_pn_lifecycle_overview`：`count(distinct offering_hierarchy_code)` where `in_scope='Y'` | — | — | LIBRARY | 1618 |
| 2 | ACTIVE_PN | COMPUTED | 同表：`count(distinct pn)` where `state='Released'` and upper(trim(in_scope))='Y'，再 /1000 | — | K | PBI | 277940 → 278 K |
| 3 | ODT_PROJECTS | COMPUTED | `gps_spark_concord_projectcr_detail`：`count(*)` | — | — | TAB | 113 |
| 4 | REQUESTS | COMPUTED | `sp_hbb_request`：`count(*)` where `deleted=false` | — | — | TAB | 10 |
| 5 | FAST_PASS | MANUAL | 无数据源 | 94 | % | PBI | 94% |
| 6 | CROSS_TEAM | MANUAL | 无数据源 | 9 | + | LIBRARY | 9+ |
| 7 | SLA | MANUAL | 无数据源 | 1.5 | WD | PBI | 1.5 WD |
| 8 | PN_TERMINATED | COMPUTED | `gps_pn_lifecycle_overview`：`count(distinct pn)` where `state='Obsolete'` and in_scope='Y'，再 /1000 | — | K | TAB | 54195 → 54 K |

Mapper 位置：`ServicePortfolioReadMapper.java`（`countObsoletePn` 在 :289-291、`countConcordProjects` 在 :293-294）、`HbbRequestMapper.countRows`（:99 起的 dynamic SQL，含 projectIds/status/changeType/requestor/team/mine/hbb/keyword 过滤）。

3 个 MANUAL 值是按需求 PPT 写进 TST 的，业务尚未给正式口径；会议中业务也承认这几项仍在人工统计，需在配置页按实际口径改。

## 4. 两个口径需要业务确认（上线风险）
- **ODT Projects 数的是 Concord 明细表全部 113 行**，需求 PPT 写「30+」。
- **Requests 数的是 HBB Change Request 的 10 条**，PPT 写「12K+」。12K+ 不可能是刚上线的 HBB 变更请求条数，业务大概率想要历史变更请求总量或另一张源表；不澄清则上线后业务第一眼就看出不对。

## 5. 接口权限现状（用户决策：暂时不加控制）
判定函数：PG 函数 `public.ssg_is_active_admin(#{itcode})`，见 `ServicePortfolioReadMapper.java:296`；Java 侧 `ServicePortfolioReadService.java:644` `isAdmin(itcode)`。

**24 个只读接口全部只要求登录，无 admin 判断**：`/kpi-cards`、`/roadmap`、`/page-links`、`/concord`、`/concord/options`、`/lifecycle/{overview,utilization,offering-trend}`、`/oh`、`/oh/tree`、`/oh/codes`、`/project-cards`、`/overview-modules`、`/eol-panels`、`/config/*` 的 4 个 GET，以及 HBB 的 projects/requests/detail/dashboard/export/comments/approvals/attachments 等查询类。

**仅 6 处写操作有 requireAdmin**：
| 接口 | 位置 |
| --- | --- |
| `PUT /config/kpi-cards` | `ServicePortfolioConfigController.java:48` |
| `PUT /config/project-cards` | `ServicePortfolioConfigController.java:65` |
| `PUT /config/overview-modules` | `ServicePortfolioConfigController.java:82` |
| `PUT /config/eol-panels` | `ServicePortfolioConfigController.java:99` |
| `POST /hbb/projects` | `HbbRequestService.java:78`（`saveProject`） |
| `POST /hbb/requests/status` | `HbbRequestService.java:336`（`changeStatus`） |

`ServicePortfolioConfigController.java:104-114` `requireAdmin`：`DataRefreshController.resolveItcode(request)` → 空则 `NEED_LOGIN` → `readService.isAdmin` 为假则 `FORBIDDEN_ERROR`。`HbbRequestService.java:785-790` 是同义的私有版本。

保留这 6 处的原因（仅一条）：4 个 PUT 是**整份覆盖保存**，业务误点一次即可清空配置页。若业务要能自行改 KPI/项目卡文案，只需把这 4 个 PUT 的 `requireAdmin` 降为 `requireLogin`（4 行改动）。

## 6. 已知未修漏洞（按"暂时不加控制"搁置）
`POST /hbb/requests/{id}/approvals`（`HbbRequestService.java:459` `addApproval`）只做 `requireLogin`：**任何登录用户都能对任意变更请求批准/拒绝**，既非 admin 校验，也无"当前用户是该请求 assigned approver"校验。审批流正式上线前必须补一条判定，否则等于人人可审批。

## 7. 上线前置
- PRE/PROD 执行 `V1.1.129` 种子迁移写入 8 行 KPI 配置（含 3 个 MANUAL 值），否则页面 KPI 区空白。
- 迁移遵守仓库"迁移只建表不塞数据"的既有规矩的例外已被接受（本卡系配置种子），按已定方案执行。

## 8. 相关文档
- `e64a20e2-b44a-4c2e-a9ba-dfd6aaf9ee20`：KPI 卡配置落地的会议口径裁定、8 行目标配置与 TST 写权限阻塞（表 owner 为服务账号 `a_appconnect`，个人账号只有 SELECT）。
- `5b52aa68-ce8b-4e5c-9d25-575abd544793`：早期排查状态——`sp_sp_kpi_card`/`sp_sp_roadmap_item`/`sp_sp_page_link` 三表在 TST 均 0 行、中文文案源自 `defaultsOfKpi()`、`docs/` 被 `.gitignore:49` 忽略。**注意该文档所述 defaultsOfKpi 兜底已在本轮后删除（!576），其"代码兜底"结论已过期**。

## 9. 未闭合
1. ODT Projects / Requests 两个口径待业务确认（是否改源表）。
2. 3 个 MANUAL 值的正式口径与维护责任人未定。
3. 4 个 config PUT 是否降级为 requireLogin，等用户点头。
4. approvals 越权问题待审批流上线前修。
