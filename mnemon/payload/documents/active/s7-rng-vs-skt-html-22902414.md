---
id: "22902414-4674-492f-a835-6949ca313918"
title: "S7 RNG vs SKT 五放加里奥复盘：数据与反证（HTML + 超高清截图交付）"
description: "2017 世界赛 RNG-SKT 五局 BP 与决赛对照的调研结论：25 个 ban 位 0 给加里奥、发条/蛇女/妖姬全开而 Faker 未选、Inven 一塔率 6/6 vs 3/10、SKT 决赛自 ban 加里奥仍 0-3，附页面结构与来源路径。"
status: "active"
created_at: "2026-11-17T04:01:17.334Z"
updated_at: "2026-11-17T04:01:17.334Z"
content_hash: "ea15fae1aa86a8552dd1a2f4f144169ceb6ed421584e6a487ef3baa9f3b5ed23"
source_paths:
  - "rng-galio/index.html"
  - "rng-galio/rng-galio-s7.png"
  - "rng-galio/shot.mjs"
session_ids:
  - "mnemon-run-v1-e23c2e5d-d188-4083-bce4-863d390df8e6"
memory_body_ids:
  []
---

# S7 RNG vs SKT“五放加里奥”复盘：数据与反证（2026-11-17 交付）

## 任务与交付物
用户要求：做一个 HTML 页面并给出超高清截图，用公开数据反驳“RNG 该 ban 加里奥”与“给 Faker 发条/蛇女/妖姬就赢”两套说法。已完成。

- HTML 源文件：`rng-galio/index.html`（35,273 字节，静态页，可直接改）
- 超高清长图：`rng-galio/rng-galio-s7.png`（2880×12900，2 倍渲染）
- 截图脚本：`rng-galio/shot.mjs`（2,809 字节；先量页面 HEIGHT 再以 deviceScaleFactor=2 全页截取）

页面结构（六段）：五局 BP 全表 → 反事实检验 → Inven 的一塔数据 → 决赛对照组 → 7.18 版本逻辑 → 结账。

## 四条核心结论（附证据）
1. **RNG 五局 25 个 ban 位，0 个给加里奥；SKT 五局中单全是加里奥。** 而 RNG 赢下的 G1、G3 恰是放了加里奥仍赢——所以“放加里奥必输”先不成立。
2. **发条五局开了四局（仅 G2 被 RNG ban），蛇女五局全开，妖姬五局全开，Faker 一次没选。** 妖姬在整个 S7 世界赛 0 ban 0 pick。故“给他妖姬就赢”连前提都不存在——不是他不想赢，而是这三个英雄在 7.18 都不是答案。
3. **加里奥是 SKT 的止血带，不是矛。** Inven 统计：Faker 拿加里奥时 SKT 一塔率 6/6，不拿时 3/10；SKT 丢一塔的 7 局，7 次全在下路。加里奥用来兜住 Bang 与 Wolf 的对线崩盘。
4. **最硬的反证是决赛（S7 决赛 SSG 3-0 SKT）。** 是 SKT 自己三局 ban 加里奥（不是 SSG ban），Faker 改拿蛇女、瑞兹、卡尔玛，仍 0-3；第一局 SKT 0-7 被零封。

## 结论定位
“不禁加里奥输得更快”的问题不在结论本身，而在于把一次可讨论的 BP 判断包装成不可证伪的神话；且它与“Faker 拿发条就赢”互相打架——两句不能同时为真。RNG 不是死在加里奥手里，是死在第五局插眼的盲僧、那条大龙和女警的速推上。

## 关键 BP 事实备查
- S7 决赛三局，SSG 掷硬币拿到 1/3/5 局选边权，三局全在蓝色方；SKT 红方，perma ban 加里奥 + 卡莉丝塔 + 慎，这等于把关键 ban 位全花在止血上，腾不出手处理 Crown 的玛尔扎哈。
- 第二局 SSG 蓝方 ban：杰斯、锐空（洛）、史瓦妮、星朵拉、吶儿；pick 珍娜、剎雅、马尔札哈、嘉文四世、吶儿。SKT 红方 ban：克黎思妲、崔斯塔娜、加里欧、慎、札克；pick 法洛士、古拉格斯、露璐、雷兹、犽宿。
- 第三局 SSG 蓝方 ban：杰斯、洛、卡萨丁、岩雀、纳尔；pick 猪妹、维鲁斯、露露、蚂蚱、大虫子。SKT 红方 ban：加里奥、风女、复仇之矛、慎、凯南；pick 小炮、酒桶、日女、扇子妈、巨魔。

## 来源
- 虎扑逐局 BP 帖（m.hupu.com/bbs/628282638）
- Inven Global 2017-11-03《Why did Faker pick Galio five times》(invenglobal.com/articles/3416) — 一塔率 6/6 vs 3/10 数据源
- gol.gg（S7 英雄数据 / 逐场对局，World Championship 2017 英雄榜）
- 17173 与游民星空 S7 决赛三局战报、EXP.GG 第二场 BP 表、sohu 决赛综述

## 环境备注
本次 DeepSeek web_search / web_fetch 通路在会话中返回 HTTP 401（API key 无效），故改用其它已启用搜索通道与直接 fetch 取得证据；若后续同类抓取再次 401，需用户确认 provider 的 base URL 与 searchApiKeyEnv，而非自行更换端点。
