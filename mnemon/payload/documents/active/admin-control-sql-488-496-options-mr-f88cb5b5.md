---
id: "f88cb5b5-3f66-444e-9cc6-f27525525c52"
title: "Admin Control SQL 迁移：增删改查全量回归（488/496）、@Options 元数据缺陷与 MR 遗留口径修正"
description: "承接 419 条 XML 迁移的写操作回归：496 端点 488 实测 473 一致，15 差异逐条证伪；抓到 NotificationCustomRuleMapper 漏 useGeneratedKeys 致 id 恒为 0（文本比对盲区）；两个破坏性设计（rebuild 先清后建无事务、全删全插空 body 清表）；51 表恢复的 autocommit 与外键顺序坑；MR v6「已知遗留」数字口径修正（${} 105/48、List<Map> 621/114、startPage 14/9、lower(itcode) 6 处）与分层处置建议。"
status: "active"
created_at: "2026-11-18T20:17:17.919Z"
updated_at: "2026-11-18T20:17:17.919Z"
content_hash: "2a1a52b50c444fd90533ccf3436574c7dd0e8d1bcd62061de9c0cdcd433520d7"
source_paths:
  - "smartop-service/admin/src/main/resources/mapper/user/UserDao.xml"
  - "smartop-service/admin/src/main/resources/mapper/menu/MenusDao.xml"
session_ids:
  - "mnemon-review-v1-367263fc-8247-4f02-848a-c1e2ddbe0b76"
memory_body_ids:
  []
---

# Admin Control SQL 迁移：增删改查全量接口回归、两个破坏性设计缺陷与 MR 遗留口径修正

2026-11-19。承接 commit `0028a07b5`（分支 `feature/admin-control-permission-sql-admin`，MR !578 仍 opened、**未合并未部署**）。
前序文档：GET 端点比对与 419 条迁移记录见 `4ad84c6f-035c-4e07-9b52-19438e2f5f5b`；permission 包 MR3 实测见 `5ed1a09b-4e8b-4374-bd69-26e2021bf97a`；!578 评审与覆盖台账见 `fd202c70-993b-4181-b119-1c8e983999a9`。本文只记本轮**新增**的回归结果、缺陷与口径修正。

## 1. 全量回归覆盖（GET + 写操作 + DELETE 全覆盖）

| 类型 | 端点数 | 实测算 | 响应一致 | 备注 |
|---|---|---|---|---|
| 查 GET | 251 | 243 | 241 | 未测 8 个需真实文件上传 / 外部系统回调 |
| 增改 POST/PUT | 222 | 222 | 209 | 13 个差异全部核实为非 SQL 问题 |
| 删 DELETE | 23 | 23 | 23 | 双路径验证 |
| 合计 | 496 | 488 | 473 | 15 个差异逐个归因，**无一是 SQL 迁移缺陷** |

DELETE 为何要双路径：两侧删同一条会让第二次返回 404，所以拆成①目标不存在路径（`id=999999`，23/23 字节级一致）②造对象→真删（roles / groups / custom-rules / toolkits / toolkits-categories / trino-task，6/6 创建与删除两侧一致）。

环境：改动前 jar 起 18083、改动后起 18082，同一 tst 库、同一 `user` 请求头；测试结束两服务均已 stop（IDEA 里用户自起的 8082 全程未动）。

## 2. 15 个差异的归因（全部证伪）

- **时间戳类 4**：JWT `exp` 差 1 秒、`elapsedMs` 2395 vs 2470、`generatedTime`、`ttlSeconds` —— 同一服务连发两次同样会差。
- **顺序副作用 5**：`toggle` / `run` / `mark-read` / `read-all`，新侧先执行已改变状态，旧侧再执行自然不同。
- **已有任务在跑 3**：导出、Hive 性能巡检、缓存重建 —— 新侧先触发占了锁，旧侧收到 300。
- **随机数据 1**：`getRandomEfficiency`。
- **校验消息顺序 2**：`roleName is required` vs `enabled is required`（JSR-303 注解遍历顺序不同，两侧都失败，只是先报哪个字段）。

## 3. 本轮唯一抓到的真实迁移缺陷：`@Options` 元数据丢失

`NotificationCustomRuleMapper.xml` 的 `<insert>` 漏写：

```xml
useGeneratedKeys="true" keyProperty="rule.id" keyColumn="id"
```

基线注解上有 `@Options(useGeneratedKeys = true, keyProperty = "rule.id")`，迁移时丢了。症状：新增通知规则后返回的 id **永远是 0**。已修，commit `0028a07b5` 即「补回 NotificationCustomRuleMapper 的 useGeneratedKeys 映射」。

**教训（与「同源解析器假阴性」并列的第二类盲区）**：SQL 文本一致不代表行为一致 —— `@Options`/`@Results`/`keyProperty` 这类**映射元数据不在 SQL 文本里**，文本比对与起始动词扫描都必然漏掉。凡是迁移 `@Insert`/`@Update` 且带 `@Options` 的 statement，需要单独列一张元数据核对表。

## 4. 测试暴露的两个破坏性设计缺陷（未改，待决策）

这 4 张表在测试中被清空，根因是接口自身设计，不是迁移问题：

**4.1 `ChatbiQuestionClusterService.rebuild()` 先清空后重建、无事务**

```java
// ChatbiQuestionClusterService.java:154
public Map<String, Object> rebuild() {
    List<Map<String, Object>> samples = mapper.listQuestionSamples(null, null);
    int cleared = mapper.clearAll();        // DELETE FROM ssg_chatbi_question_cluster (9964 行)
    int dailyCleared = mapper.clearDaily(); // DELETE FROM ssg_chatbi_question_daily (12008 行)
    // ... 然后逐条 insert
}
```

入口 `POST /chatbi/events/report/questions/rebuild`。中途抛异常即两表为空。实测后 `cluster` 9964→426、`daily` 12008→0。
建议：先建临时表或先插后清理。

**4.2 `deleteProfileGroups` / `deleteInternalUsers` 全删全插保存**

`ChatbiEventQueryMapper.xml:121` 与 `:137` 是裸 `DELETE FROM`（无 WHERE），配合后续 insert 做覆盖式保存。请求 body 为空时只删不插，表直接清空（`ssg_chatbi_profile_group` 11→0、`ssg_chatbi_internal_user` 62→0）。
建议：加「空列表拒绝执行」守卫。两处均涉及业务语义，需单独 MR。

## 5. 数据恢复：51 张写表快照回放（含两个坑）

做法：`.tmp/backup/` 全量快照（27844 行）→ 每表 `DELETE` 全表 → 批量 `INSERT` → `setval` 重置序列。校验 `=== 全量 diff === OK=49 DIFF=0`（49 张 `public.ssg_*` 逐表行数与备份一致）。

两个坑：
1. `DbRestore2` 用 `setAutoCommit(false)` 并在循环内反复切换事务，打印「ok」但数据没落地 —— 改为 autocommit 逐批提交 + **commit 后立即在同一连接 `SELECT count(*)` 自校验**（`DbRestore3`）。教训：批量恢复的「成功」必须以同连接回读为准，不能信那句打印。
2. 51 张表循环被打断时 `ssg_user_menu`(1004) / `ssg_user_role`(59) 只执行了 DELETE，需二次补恢复；`ssg_toolkit_category` 因外键 `fk_ssg_toolkit_category`（被 `ssg_toolkit` 引用）删不掉，改为「清父表 → 重建分类 → 重建工具」顺序。

## 6. MR 描述「已知遗留」数字口径修正

MR 描述 v6 里那节数字混用了**主工作区 admin+auth+common 全域**（1076 个 Java 文件）的扫描结果，却放在只改 admin 模块的 MR 里，读者会误读为 admin 内数量：

| MR v6 原写 | admin 单模块实测 | 说明 |
|---|---|---|
| `${}` 150 处 / 54 文件 | **105 处 / 48 文件**（java + xml） | 全域数字 |
| `List<Map<String,Object>>` 619 处 | **621 处 / 114 文件** | 基本准确，差 2 是正则宽松度 |
| `PageHelper.startPage` 23 处 | **14 处 / 9 文件** | 全域数字 |
| permission 包 `lower(itcode)=lower(?)` 4 处 | **6 处** | 手工数漏 |

必须补进描述的一句：迁移把 SQL 从 Java 注解搬进 XML 后 `${}` **没有减少**，只是换了位置（105 处里 XML 约 30 处，其余仍在 Java 的 Wrapper/拼接代码）—— 搬家解决的是「SQL 可读、可审、可复用」，**不是**「消除注入面」。

真实注入面位置（已核准）：
- `smartop-service/admin/src/main/resources/mapper/user/UserDao.xml`：`:125` `${id}`（`batchDelUser` 的 IN 列表）、`:130` `${id}, ${roleId}`（`batchDelUserRole`）、`:144` `${bean.userId}, ${bean.roleId}`（`batchInsertUserRole` 批量插入）、`:157` `${uId}`（`batchUpdateUser` 的 where）。
- `smartop-service/admin/src/main/resources/mapper/menu/MenusDao.xml`：`:14` `and cm.access_level != ${accessLevel}`、`:246` `${param.orderBy},`。
- permission 包 6 处：`EffectivePermissionDao.xml:15`、`EffectivePermissionQueryDao.xml:13`/`:29`、`PermissionManagementDao.xml:91`/`:97`、`RoleManagementDao.xml:246`。

## 7. 推荐处置（分层，未执行）

1. **本 MR 收尾**：改 MR 描述该节（口径标注 + 准确数字 + 点破「搬家不消除注入面」）；`UserDao.xml` 4 处 `${}`→`#{}` 独立 commit。安全依据：`UserDao.java:117` `batchDelUser(@Param("ids") List<Integer> ids)`、`:140`/`:146` `List<BatchUserRole>`、`:148` `List<Integer> updateUIdList`，全是整型 id，`${}` 只是图省事；同文件 `:136` 的 `userIdList` 已用正确的 `LOWER(#{itcode})`，说明作者知道正确写法。改后行为一致，但 `"1) or (1=1"` 不再可注入。
2. **下个 MR（不能机械替换）**：`MenusDao.xml` 2 处 —— `${param.orderBy}` 在 ORDER BY 位置，换 `#{}` 会变成带引号字面量导致排序失效，正解是 Controller/Service 收敛成白名单枚举 + XML `<choose>`；`MenuController:315` 现硬编码 `setOrderBy("lv4.sequence")` 暂时兜住，但 `ReportMenuParam` 是 `@RequestBody` 反序列化，只要改成透传或出现别的调用方即成注入点。`${accessLevel}` 需先确认是枚举还是数字。
3. **权限分页 count**：`PermissionManagementService` 的 `listUsers` 自动 count 会把 4 个 `string_agg` 子查询再算一遍（20 行触发 81 次子查询）；正解是独立 `COUNT(*)` count SQL + `startPage(p, s, false)`。其余 13 处 `startPage` 无证据，不动。
4. **降级为规范，不排期**：permission 包 `lower(itcode)` —— 早先列为「P0-1 索引失效」属**过度诊断**：`ssg_user` 仅 285 行，`Seq Scan` + `Rows Removed by Filter: 284` 只花 0.29 ms，建函数索引收益可忽略；且库里 285 行 itcode 全部小写（`count(*) FILTER (WHERE itcode <> lower(itcode))` = 0），新代码直接写 `itcode = lower(?)` 即可。真正拖慢 `listUsers` 的是 `string_agg` 相关子查询，但「拆 3 条 SQL」实测仅 0.98~1.05x，无提升，**已回退**。
5. **不做**：`List<Map<String,Object>>` 621 处 —— 本次保证 SQL 文本不变、Map 的 key 就不变，前端零改动才是正确状态；除非出现明确 DTO 化需求（如接 OpenAPI 生成文档）。

## 8. 代码状态

`feature/admin-control-permission-sql-admin` 5 个提交全部已推送、工作区干净；admin 注解 SQL 命中 0，mapper XML 121 个（799 条 statement）。MR !578 仍 opened，未合并。
