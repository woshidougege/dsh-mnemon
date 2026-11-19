---
id: "b3126a3d-15e3-424e-b74c-2f52f0880a37"
title: "Service Portfolio KPI valueType 统一为 COMPUTED：双端修复落地与待授权改库（ssg-sales-ops，2026-11-19）"
description: "方案B执行记录：后端 MR !595（resolveMetric 兼容 AUTO，mvn compile SUCCESS）与前端 MR !1116（KpiCardsConfig.vue:224 存 COMPUTED、kpiConfig.ts 归一历史 AUTO，vue-tsc 通过）；sp_sp_kpi_card 5 行改库 SQL 仍待用户授权（owner a_appconnect，p_renxp4 仅 SELECT，TST 10.62.79.148）；附主仓库 develop 未解决合并冲突遗留。"
status: "active"
created_at: "2026-11-19T08:57:19.599Z"
updated_at: "2026-11-19T08:57:19.599Z"
content_hash: "7d034e1ba2ad58019c75cb86b73b3fd4a7fd5c38f27f75f3334884960e98f15f"
source_paths:
  - "src/views/Portfolio/config/KpiCardsConfig.vue"
  - "src/views/Portfolio/config/kpiConfig.ts"
session_ids:
  - "mnemon-review-v1-bb2c848a-f691-495b-b76a-32390d745871"
memory_body_ids:
  []
---

# Service Portfolio KPI valueType 统一为 COMPUTED：双端修复落地与待授权改库（ssg-sales-ops，2026-11-19）

## 0. 状态摘要
根因与三步修复方案见既有文档 `a538574e-ee32-4c2d-b97c-85a1f34396bc`（KPI 配置页「保存即降级」缺陷与 Data Source 口径核查）。本次完成其中「用户选定方案B」后的代码执行部分：**前后端两处代码已改完、编译/类型检查通过、两个 MR 已开**；**数据库尚未修改**，等待用户授权使用一次服务账号。

## 1. 决策：方案B（双端都改 + 兼容历史值）
不改只改一侧的窄方案，而是在前端把取值方式枚举统一到后端的 `COMPUTED`，并在前端读取时把库里历史 `AUTO` 归一成 `COMPUTED`；同时后端 `resolveMetric` 兼容 `AUTO` 写法。理由：既消除「配置页 Save 即降级」的自我破坏路径，又不让历史行在灰度期变成手动空值。

## 2. 后端改动（dataagent/ssg-sales-ops，目标分支 admin-release）
- MR **!595** https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/595，sha `7a741f67c8bc9e58c01dee2440b29d49decc81dc`，1 个文件，state opened / can_be_merged / 无冲突。
- 关键逻辑（`ServicePortfolioReadService.resolveMetric`）：
```java
// 配置页历史版本把「取数」存成 AUTO，这里两种写法都认
boolean computed = "COMPUTED".equalsIgnoreCase(valueType) || "AUTO".equalsIgnoreCase(valueType);
if (!computed) {
    return manualValue;
}
```
- 验证：`mvn compile` BUILD SUCCESS，耗时 2m22s（2026-11-19 16:51:12 +08:00，EXITCODE=0）。

## 3. 前端改动（dataagent/ssg_sales_ops，目标分支 develop）
- MR **!1116** https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1116，sha `fb88a123cec6346acee678b59093a537f54b5e27`，2 个文件，state opened / can_be_merged / 无冲突。
- `src/views/Portfolio/config/KpiCardsConfig.vue:224`：下拉存值 `AUTO` → `COMPUTED`（`<el-option label="Manual value" value="MANUAL" />` / `label="From data source" value="COMPUTED"`）。
- `src/views/Portfolio/config/kpiConfig.ts`：`export type KpiValueType = 'MANUAL' | 'COMPUTED'`，`const valueTypes: KpiValueType[] = ['MANUAL', 'COMPUTED']`；读取归一 —— 注释「库里历史数据把取数写成 AUTO，读回来统一成 COMPUTED」，实现 `if (text === 'AUTO') return 'COMPUTED'`（关键点是**不再降级成 MANUAL**）。
- 验证：`vue-tsc --build` 通过（EXITCODE=0）。
- 提交链：`e1da4355` feat(portfolio): 展示页接入后端只读接口 → `fb88a123` fix(portfolio): KPI 卡取数方式统一成 COMPUTED。
- 做法：因主工作区被他人合并冲突占住，改动在临时 worktree `.tmp-fe-kpi`（分支 `feature/sp-display-wiring-develop`）中提交并 push，随后删除该 worktree，主仓库状态还原到进入前。

## 4. 待执行：改库（需用户授权）
表 `public.sp_sp_kpi_card`，owner 是 `a_appconnect`，`p_renxp4` 只有 SELECT 权限，改 5 行需借用一次服务账号；TST 库 10.62.79.148。

```sql
-- 1) 5 张卡恢复自动取数
UPDATE public.sp_sp_kpi_card
   SET value_type = 'COMPUTED', updated_by = 'renxp4', updated_time = now()
 WHERE metric_key IN ('OH_CODES','ACTIVE_PN','ODT_PROJECTS','REQUESTS','PN_TERMINATED');

-- 2) OH Codes 清掉残留的 1.6 和单位 K（真实值 1,618 条，不该带 K）
UPDATE public.sp_sp_kpi_card
   SET manual_value = '', unit = '', updated_by = 'renxp4', updated_time = now()
 WHERE metric_key = 'OH_CODES';
```

- 验证顺序：执行 SQL → 查库确认这 5 行 `value_type/manual_value/unit` → 请用户刷新页面（TST `https://ssgspark-tst.lenovo.com/spark/portfolio`）。
- 时效性：TST 后端现存版本已认 `COMPUTED`，**改库后页面立即恢复有数，不必等 !595 合并**；MR 的作用是防止以后再被配置页写坏。

## 5. 环境遗留问题（非本任务造成，需另行处理）
主仓库 `C:\Users\63588\lenovo\ssg_sales_ops` 当前在 `develop` 且处于**未解决的合并中状态**（`Unmerged paths: both modified: src/constants/feedbackMenuCatalog.ts`，分支落后 `origin/develop` 85 个提交），疑似其他任务 merge `tmp/review-chain-develop` 所留。该目录下另有未跟踪的 `.tmp-fe-sp/`。此状态会阻塞他人正常提交。
