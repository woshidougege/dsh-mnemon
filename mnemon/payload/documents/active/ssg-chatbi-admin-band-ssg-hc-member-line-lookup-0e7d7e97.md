---
id: "0e7d7e97-e70b-4e74-ba13-472d50742cd3"
title: "SSG ChatBI Admin Band/汇报线口径修复：改用 ssg_hc_member_line_lookup 点查函数"
description: "Vendor/外部顾问 Band 与 L3-L5 为空或「—」的根因（070b4145f 漏改两处 HC_JOIN）与修复：改用 ssg_hc_member_line_lookup 点查函数，含 346ms 视图 join vs 18µs/行实测量化、41 人 0 差异验证、own_level=2 遗留边界、MR !563。"
status: "active"
created_at: "2026-11-17T02:45:03.306Z"
updated_at: "2026-11-17T02:45:03.306Z"
content_hash: "4e0709b9d429d701627174b504c8f11a18194cb79c9a9a3c24bffd5435c42ca7"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiFeedbackRecordMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiHcMemberOverrideMapper.java"
session_ids:
  - "mnemon-run-v1-61de97b1-8e88-47c7-92e8-6d2e8b65f589"
memory_body_ids:
  []
---

# SSG ChatBI Admin Band / 汇报线口径修复：改用 ssg_hc_member_line_lookup 点查函数

状态：已实现并推送，MR **!563**（`fix/vendor-reporting-line-admin` → `admin-release`，`can_be_merged`、`has_conflicts=False`、3 files changed）。**合并/部署仍待用户授权**，页面效果未做浏览器验证。
基线提交：`9327b8f1c`（admin-release 最新），实现提交：`5f146fa9a`（55 insertions / 68 deletions）。

## 1. 症状

问数记录 / 点赞点踩页面里，Vendor、外部顾问（非 HC 清单人员）的 **Band、Job Family、L3/L4/L5 汇报线全部为空/「—」**；而同一个人的汇报线在**筛选下拉里能选到**——即「下拉能选到、明细查不出值」。

## 2. 根因（不是数据链路断，是读错了表）

- 汇报线推导链路已落在视图 `public.ssg_hc_member_line`（= 清单 ⟕ `ssg_hc_member_override` ⟕ 顾问汇报线推导），生产 41 个非清单人员里 40 个有 L3–L5（例：`ZHANGWQ15` → `Chhavi Arora / Mary Goodwin / Rebecca Liang`）。
- 但提交 `070b4145f`（「优化 Admin Control 接口响应速度」，11-12）只把**推送**那一处换成点查函数，**问数记录和点赞点踩两处漏了**，其 `HC_JOIN` 仍是旧写法「`ssg_hc_snapshot` ⟕ `ssg_hc_member_override`」。这两张表只有正式员工，Vendor 查不到行 → 三列与 Band 全 NULL。
- 下拉候选走视图、明细走快照，于是口径分叉。

结论：**「Vendor / 外部顾问也必须出值」的展示与筛选，一律读 `public.ssg_hc_member_line_lookup(...)`，不要直连 `ssg_hc_snapshot`。**

## 3. 修复内容

点查函数签名（`V1.1.116__Optimize_hc_member_lookup.sql:52`）：

```sql
CREATE OR REPLACE FUNCTION public.ssg_hc_member_line_lookup(p_itcode text, p_period text)
```

返回 `itc, band, job_family, l3, l4, l5`。内部结构（`V1.1.116__Optimize_hc_member_lookup.sql` 约 118–150 行）：
`seed`（规范化 itcode）→ `me` / `anc1`（自身在快照中）/ `anc2`（按经理向上找一层的 chain）→ `best`（`ORDER BY pref LIMIT 1`）→ `drv`（按 `own_level` 决定 l3/l4/l5 取谁的 nm）→ `out`（`eff` 有行取快照值；无 eff 但有 seed 时以 `key` 兜底返回全 NULL 列，保证不丢行）。

改动位置：

| 文件 | 改动 |
|---|---|
| `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java` | `HC_JOIN` 改为 `LATERAL ssg_hc_member_line_lookup(...)`；Band、Job Family 筛选同步换成点查函数 |
| `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiFeedbackRecordMapper.java` | 明细 `HC_JOIN` + 6 处筛选 `EXISTS` 统一换点查函数 |
| `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiHcMemberOverrideMapper.java` | 同步调整（清单/补充表相关查询） |

## 4. 为什么用点查函数而不是直接 join 视图（性能实测，生产只读）

视图是 4.2 万行宽表，整表 join 正是上一轮要优化掉的写法：

| 场景 | Execution Time |
|---|---|
| 1000 行整表视图 join（被优化掉的写法） | **346.563 ms** |
| 1000 行旧写法（直连 `ssg_hc_snapshot`） | 1.099 ms（另一次 0.886 ms） |
| 1000 行 lookup（外层列参数） | 18.390 ms（另一次 5.989 ms） |
| 20 次 lookup（一页明细量级） | 5.145 ms（wall 241 ms，含连接开销） |
| 明细 20 行 lookup（反馈表） | 10.823 ms |
| count 带 l3 筛选：lookup vs 旧 effective 视图 | 60.312 ms vs 90.269 ms |
| count 基线（无筛选） | 0.290 ms |

点查走索引 `idx_ssg_hc_snapshot_itcode_norm`，约 **18 µs/行**，一页 20 行 0.36 ms，与直连快照同一量级，不会把上次的接口提速优化退回去。`result_feedback` 表仅 227 行。

## 5. 验证证据（生产只读，个人账号）

- 点查函数 vs 视图，在 41 个非清单人员上 L3/L4/L5 **逐行一致，0 差异**（`n=[41] diff=[0]`，无差异明细行）。
- 42,738 个清单 itcode 的 band **全部非空**，与快照覆盖一致；`dup_groups=[1] band_conflict=[0]`。
- `mvn -o -T 4 -pl smartop-service/admin -am compile` 通过。
- 未做页面/浏览器验证；三列出值、按汇报线能筛到 Vendor 需合并部署后确认。

## 6. 已知边界（不是 bug）

`CARORA1` 这类「**直属经理本身就在第 2 层**」的人仍是空：视图口径按 `own_level=2` 只取 `l3` 列，没有可用的更高层经理。清单里的 `arorac` 是同一处理方式，一致即可。

## 7. 踩坑记录（本机核对时）

- 探测反馈数据时表名写错：`ERROR: relation "public.ssg_chatbi_feedback_record" does not exist`（位置 22）。反馈事实表实际是 **`agent.result_feedback`**，`ChatbiFeedbackRecordMapper.FROM_RECORD` 也明确注释「刻意不关联 `public.ssg_chatbi_event`，结果反馈表才是赞踩状态/原因/评论的事实来源」。
- PowerShell 里把 `-Dfile.encoding=UTF-8` 传给 java 会被 shell 吃掉，报「找不到或无法加载主类 .encoding=UTF-8 / ClassNotFoundException: /encoding=UTF-8」（乱码显示）。传 JVM 参数要用引号包整串或用参数数组。

## 8. 相关文档

- `25fb29f1-f532-44dc-b110-618cf4796fdc`：ChatBI 问数记录 Source report 轮次级改造（同一 mapper 的另一处口径改造）。
- `53c4bc18-6900-4cfa-9b75-d36a82197291`：问数记录「问题轮」口径契约与 clarify/补录轮处置决策。
