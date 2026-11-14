---
id: "00783d80-454e-422d-b457-52c4b3f14778"
title: "ssg_sales_ops：三块终态核对 + 设置页导航自动折叠修复（MR !1042）"
description: "settings 页左侧导航在生产不自动折叠的根因（SettingsNav.vue 的 storedChoice/forceCollapsed 差异）与 master 侧修复 MR !1042；Admin Control/通知中心/反馈中心 125 文件比对终态（仅剩 I&C 两处差异，上生产需同时打开两处开关）；tst 12 个接口与三页面的回归结果及 token 缓存 401 教训。"
status: "active"
created_at: "2026-11-14T17:55:29.711Z"
updated_at: "2026-11-14T17:55:29.711Z"
content_hash: "c10840757a331ab26781761a0d38f39c5874bbd28d75b7ea285bfb9b6db356d4"
source_paths:
  - "src/views/Setting/settings/components/SettingsNav.vue"
  - "src/constants/feedbackMenuCatalog.ts"
  - "src/views/BiAdmin/moduleTargets.ts"
  - "src/router/index.ts"
session_ids:
  - "mnemon-run-v1-14db3c67-c75a-4138-86a4-fc21e2ca696b"
memory_body_ids:
  []
---

# ssg_sales_ops：三块终态核对 + 设置页导航自动折叠修复（MR !1042）

日期：2026-11-15。仓库：`C:\Users\63588\lenovo\ssg-sales-ops`（Vue3 + vue-tsc）。
关联文档：**e473a76e-9ef6-443f-b7e6-fcbf4f544574**（11-13 那次三块一致性核对与 !1018 回放评审链路补齐）。本文只记 11-15 这轮的新结论：用户在生产发现的行为差异、根因、修复，以及 125 文件全量比对的终态。

## 1. 用户报的问题与真实根因

生产 https://ssgspark.lenovo.com/setting/settings 进入时**不再自动收起左侧设置导航**，测试环境正常。用户对先前"两环境一模一样"的结论提出异议——当时的判定标准是 `git diff` 有没有差异，把剩余差异逐个当作"与功能无关"就收尾，没有顺着差异去看**用户可见行为**。这是判断失误，不是部署问题。

根因在 `src/views/Setting/settings/components/SettingsNav.vue`，master 与 develop 的 `collapsed` 计算方式不同：

```ts
// master（有 bug）：有本地记录就完全按记录来
const collapsed = computed(() =>
  storedChoice.value === null
    ? Boolean(props.defaultCollapsed) || narrow.value
    : storedChoice.value === '1',
)

// develop（正确）：每次进入先强制收起，点击展开只生效本次
const forceCollapsed = ref(Boolean(props.defaultCollapsed))
const collapsed = computed(() =>
  forceCollapsed.value ||
  (storedChoice.value === null ? narrow.value : storedChoice.value === '1'),
)
```

关键点在 `NAV_STORAGE_KEY = 'settings-nav-collapsed'`：用户在设置页点过展开按钮后 localStorage 被写成 `'0'`，master 之后每次进设置页都读到这个 `'0'`，于是**永久保持展开**；develop 因为 `forceCollapsed` 只把本地记录当作"本次浏览内"的选择，所以看起来正常。

`toggleCollapsed()` 在 develop 里多了一步：`const wasCollapsed = collapsed.value; forceCollapsed.value = false; storedChoice.value = wasCollapsed ? '0' : '1'` —— 先解除强制收起，再记录用户这次的选择。

临时规避（无需发版）：F12 → Application → Local Storage 删掉 `settings-nav-collapsed` 再刷新。

## 2. 修复与交付

分支 `fix/settings-nav-autocollapse`（基于 origin/master），两个提交：

| commit | 说明 | 变更 |
|---|---|---|
| `0680415d` | fix(settings): 每次进入设置页都先收起左侧导航 | `SettingsNav.vue` 12 增 5 删 |
| `ed099f33` | docs(settings): 注明 I&C 上生产时需要同时打开的两处开关 | `feedbackMenuCatalog.ts` 注释 +3/-1 |

MR：**!1042** https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1042 → master，`can_be_merged`、无冲突、2 files。

## 3. 125 文件全量比对终态

把 Admin Control、Notification Center、UsageIssueFeedback 三块（含配套 `services`、`constants`）共 **125 个文件**在 master 与 develop 逐份对比：**123 个完全一致**，剩余 2 个差异全部来自 I&C（改前）——

| 文件 | master | develop |
|---|---|---|
| `src/constants/feedbackMenuCatalog.ts` | I&C 菜单项被注释掉 | 打开（`key: 'ic'`、`value/path: '/spark/ic'`、`menuKey: 'businessInsight:ic'`） |
| `src/views/BiAdmin/moduleTargets.ts` | `PAGE_PATHS` 无 `ic` | 多 `ic: '/spark/ic'` |

用户确认"生产没有 I&C，其他都一样"，因此**保留这两处差异**（`src/views/IC/*` 与 `/spark/ic` 路由只在 develop 存在）。`feedbackMenuCatalog.ts` 那条注释已补全为：

```ts
// master 暂不展示 I&C：src/views/IC 与 /spark/ic 路由只在 develop，
// 反馈页也先不放，避免把生产上打不开的页面列成可选。
// I&C 上生产时要同时改两处：打开下面这段注释，并在 src/views/BiAdmin/moduleTargets.ts
// 的 PAGE_PATHS 里补上 ic: '/spark/ic'，否则标签能点但跳不过去。
```

**I&C 上生产 = 两处同时打开**，漏掉 `moduleTargets.ts` 会出现"标签可点但路由不存在"。

## 4. 测试环境（tst）接口回归记录

窗口 2026-10-15 至今，直连 https://ssgspark-tst.lenovo.com ，12 个接口全 200：

| 接口 | 关键返回 |
|---|---|
| `POST report/options` scope=query | users 63、bands 7、jobFamilies 20 |
| 同上 scope=feedback | userCounts 20（合计 70）、bandCounts `{7:3,8:34,9:17,10:4,ED:9}`、jiraStatusCounts 6 桶、ratingCounts `{1:28,-1:42}` |
| 同上 scope=pages | pages 12 |
| 同上 + `band=8`（级联） | userCounts 收窄到 21 人，bandCounts 仍全量 |
| `POST report/data` | total 54499 |
| `POST report/feedback-records` | total 70、review `{correct 0, wrong 17, pending 201}` |
| `GET issue/admin/feedback` | total 55、facets 8 维、review `{correct 0, wrong 6, pending 36}` |
| `GET issue/admin/list` | total 13 |
| `GET issue/admin/feedback/reporters` | 20 |
| `GET issue/admin/downvoted-answers` | total 42 |
| `POST report/export/count` / `GET issue/admin/export/count` | rows 54499 / 55 |

页面侧：Issue & Request（Wrong 6 / Pending 36、reason 13 项全列、8 个 0 条置灰、Jira `Creation failed 0` 灰）、User Logs → Data Inquiry Records（Band `6:3·7:74·8:220·9:79·10:17·Other:2·ED:38`、Success 450 / Failed 20）、User Logs → Feedback（`ED 9·8 34·7 3·9 17·10 4`、Jira status 8 项、`Grooming 0`/`Ready for Testing 0` 灰）。

## 5. 环境教训

- 第一轮全部 **401**：Redis 里缓存的测试环境 token 被换过一次（长度 77 → 205 字符），重新取 token 才通。**tst 全量 401 优先怀疑 token 缓存，而不是权限或后端**。
- `userCounts` 里没有 key 的人名（如 Adam Partaledis、Dana Radosinska）表示该人**在当前时间窗内没有反馈**，不是数据缺失；人名类下拉项不置灰，因此显示空白而非 0。
- Windows PowerShell 下 `New-Object IO.StreamReader($r.GetResponseStream())` 对 `HttpResponseMessage` 无效（该类无 `GetResponseStream()`）；另有一处 `git credential fill` 脚本因中文字符编码触发 `Missing closing '}' in statement block`，改用现成凭据即可。

## 6. 待办

- !1042 合并 → 生产部署后，三块代码层面仅剩 I&C 那 2 行差异，其余零差异。
- 若要防复发：两环境共享代码的模块可考虑 CI 加 diff 守护（此前核对时已提出，尚未落地）。
