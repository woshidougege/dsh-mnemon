---
id: "00abc05a-db17-4302-9eeb-6c47f99d9dc1"
title: "HW KPI 图卡 inset 三态改造：状态机、遮挡几何与双轨 MR 落地"
description: "Bookings/GPSUS 复用图卡的 ISG 角标小图改造实施记录：preview/expanded/collapsed 三态循环（收起态为标题行胶囊按钮）、inset bottom 由 26px 提到 48px 让开 40° 斜排 X 轴标签、v5 三处修正（Y 轴负刻度兜底 15%、单位字号与透明度、>6 季标签抽稀到 ≤20），以及 master !1070 / develop !1081 双轨提交与 develop 需补 v5 提交的原因。"
status: "active"
created_at: "2026-11-18T03:53:05.243Z"
updated_at: "2026-11-18T03:53:05.243Z"
content_hash: "cb6216343273659071b0c4f705a018d961b92db7cff8ff5bda817a3f55bd95f0"
source_paths:
  - "src/views/Bookings/components/miniTable.vue"
  - "src/components/SparkChart/index.vue"
session_ids:
  - "mnemon-run-v1-5af743d7-d104-44f7-acbe-914b65481810"
memory_body_ids:
  []
---

# HW KPI 图卡 inset 三态改造：实施结果与依据

> 前序阶段（问题诊断、方案对比、代码未动）见文档 `f5b15acb-8bb5-49fa-9f28-4958f4928c5f`。本文记录**实施落地后**的状态，取代其中"待用户拍板"的部分结论。

## 1. 用户诉求的三次演进

1. 起初：inset 小图"显示不全"，问能否自己拉图 → 从三个方案（A 点击放大 / B 加大框 / C zoom 插件）中选 A：卡片内原地放大，不弹 dialog、不引新依赖。
2. 用户指出："现在线上是默认态，点一下放大、再点一下隐藏就好了" → 确认交互不变，问题在**位置**：默认态小图压住了 X 轴季度标签（`FY2627Q1/Q2/Q3` 上半截被白卡盖掉）。
3. 用户澄清（关键）：**不是两态，要三态** —— 小图 → 大图 → 收起成一个图标按钮，再点才出来；同时确认"图没在右下角、压在 X 轴上"的判断是对的。

## 2. 三态状态机（`src/views/Bookings/components/miniTable.vue`）

```js
const INSET_MODES = ['preview', 'expanded', 'collapsed']
const insetMode = ref('preview')
const cycleInset = () => {
  const index = INSET_MODES.indexOf(insetMode.value)
  insetMode.value = INSET_MODES[(index + 1) % INSET_MODES.length]
}
```

| 态 | 呈现 | 触发 |
|---|---|---|
| preview（默认） | 168×88 角标，只画趋势不写数值 | 收起态点图标 |
| expanded | 320×196，带 K 轴、数值标签、Last 4/8/All 区间切换 | preview 态点小图任意位置 |
| collapsed | 卡片内无小图，标题行只剩 `〰 ISG` 胶囊（11px 折线 SVG，高 17px，`.bk-inset-toggle`） | expanded 态点 `✕` |

模板要点：`v-if="insetChart && insetMode === 'collapsed'"` 渲染标题行胶囊；`v-if="!showTable && insetChart && insetMode !== 'collapsed'"` 控制角标；`insetChart` 内部按 `insetMode === 'expanded'` 决定是否带轴与标签。

**设计理由（收起态入口放标题行而非图内）**：主图内任何位置都会盖住曲线或坐标轴；标题行是零遮挡区域，且与 "Show Data" 切换同排，视觉上同属卡片控制。

## 3. 遮挡几何：`bottom` 26px → 48px

- X 轴季度标签 40° 斜排，约占 46px 高；原 `bottom: 26px` 使角标底边正好切进标签带。
- 改为 `bottom: 48px`：底边落在斜标签上方约 2px，角标坐进绘图区右下角。
- 代价：仍会压住 Total 曲线尾部 1~2 季的标签，这是"角标小图"形态绕不开的取舍，用户已知悉。
- 该 48px 是按"9 字符 × 40° 斜排"估算的；Chart.js 轴高随标签长度动态计算，TST 实测若差几像素需微调。

## 4. 同批提交的三处修正（v5）

1. **Y 轴负刻度**：截图中出现 `-2.0` 等负值刻度。改为按最低点向下 15% 兜底，Bookings 轴底回到 0.5 附近，不再出负格。
2. **单位可读性**：单位 7px→8px、透明度 62%→75%；数值 9.5px→10px。
3. **标签密度**：季度数 > 6 时只标首季 + 最近 3 季 + 峰值，44 个标签降到最多 20 个，避免多线标签互相压。

## 5. 分支与 MR 现状

| 分支 | 目标 | MR | HEAD | 内容 |
|---|---|---|---|---|
| `dev/renxp4-hw-kpi-unit` | master | !1070（`can_be_merged`） | `f0fa50cf` | v5 + 三态 |
| `dev/renxp4-hw-kpi-inset-modes` | develop | !1081（`can_be_merged`） | `0e877d5f` | cherry-pick 了 v5 + 三态 |

- develop 侧 `!1071` 只合到 v4（`7db57a17`），所以 !1081 必须带两笔提交：`7b88aabe`（v5）+ `0e877d5f`（三态）。
- 改动仅 `src/views/Bookings/components/miniTable.vue` 一个文件，`npm run type-check` 通过；GPSSUS 页复用同组件，会一起生效。
- 提 MR 用 `.tmp/create-mr.ps1`：`git credential fill` 取 token → POST `/api/v4/projects/:id/merge_requests`；遇 409 时按 source/target 反查已存在的 opened MR 并打印，避免重复创建。

## 6. 教训

- 「上一笔已合并」不等于「已进目标分支」：develop 只收 v4，导致后续 MR 需要补提交，是本轮返工的根因。每次提 MR 前用 `git log origin/develop` 比对该文件的提交是否真的落地。
- 位置问题（是否遮挡）与交互问题（几态切换）要分开谈：用户说"这样是默认态"时，他讨论的是默认态的可用性，不是要改交互。

## 7. 待 TST 复核清单（合并后一起验）

1. 标签显示为 `15.0B` 这类"数值带单位、单位弱化"形态。
2. 左轴不再出现负刻度。
3. 角标小图不再压 X 轴季度标签。
4. 三态点击循环正常（preview → expanded → collapsed → preview），收起态胶囊能唤回。
