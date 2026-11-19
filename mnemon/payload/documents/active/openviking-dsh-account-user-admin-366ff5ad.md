---
id: "366ff5ad-c3f7-48f4-82ba-92f9bb9eb959"
title: "OpenViking 多租户隔离与 DSH 记忆后端接入结论（account/user 划分、admin 接口、测试环境实测）"
description: "openviking 0.4.6 的 account=租户/user=viking://user/{id} 隔离模型、admin 账号与用户接口、key 格式与标识符校验、PRESET_DIRECTORIES 目录树、renxiangpeng 新账号在测试环境的实测隔离结论与 DSH 四字段配置；含仅逻辑隔离与删除级联两个代价。"
status: "active"
created_at: "2026-11-19T03:04:55.743Z"
updated_at: "2026-11-19T03:04:55.743Z"
content_hash: "c1ac58a9fc3b61d737cb9a5d2539dfb0d4cc0a1765fec216834e7d8f4b1139da"
source_paths:
  - "openviking/core/directories.py"
  - "openviking/core/identifiers.py"
  - "openviking/core/user_id.py"
session_ids:
  - "mnemon-review-v1-03654087-ddac-4c79-bd7c-bc5e4b8f7931"
memory_body_ids:
  []
---

# OpenViking 多租户隔离与 DSH 记忆后端接入结论

2026-11-19 完成。用户诉求：确认 DSH 填写的 OpenViking 记忆后端配置「会不会影响测试环境里其他人的数据」，以及是否该新建 Account 而不是复用 `internal`。结论：新建独立 account 可行且隔离生效；更轻的替代是在 `internal` 下新增用户。

## 1. 服务端隔离模型（证据：本机 site-packages 内 openviking 0.4.6）

- 安装位置：`C:\Users\63588\AppData\Local\Programs\Python\Python312\Lib\site-packages\openviking`（另有 `openviking_cli`、`openviking_sdk` 0.1.4）。
- account 即租户；具体数据落在 `viking://user/{user_id}/...`。`viking://user` 只是 API 边界上的「当前用户」简写，其元数据属于 `viking://user/{user_id}`。
- `openviking/core/directories.py:162 initialize_account_directories` 只建账号级共享根 `viking://resources`（`scope_roots = {"resources": PRESET_DIRECTORIES["resources"]}`）；用户目录由 `openviking/core/directories.py:186 initialize_user_directories` 建，只展开 `PRESET_DIRECTORIES["user"]` 的根 + 一级子目录，更深层按写入懒创建（注释明确「keeps a new user root discoverable without materializing the full empty taxonomy」）。
- `openviking/core/directories.py:221 initialize_agent_directories` 已废弃，恒返回 0。
- 同一逻辑另有 `openviking/service/core.py:529 / :536` 的同名方法（服务层转发）。

## 2. PRESET_DIRECTORIES["user"] 目录树

`memories`（下分 preferences / entities / events / cases / patterns / tools / skills / trajectories / experiences）、`resources`（当前用户私有资源）、`privacy`（敏感配置快照）、`peers`（长期交互对象记忆）、`skills`（Claude Skills 协议定义的 skill registry）、`sessions`。`resources` 顶层 scope 则是「不绑定账号或 Agent 的全局共享知识库」，按项目/主题由用户自建。

## 3. 标识符与 key 规则

- `openviking/core/identifiers.py`：`_VALIDATION_PATTERN = re.compile(r"^[a-zA-Z0-9_.@-]+$")`；禁止 `.` / `..`；`@` 最多一个；`validate_account_id` 额外禁止以下划线 `_` 开头。
- `openviking/core/user_id.py` 的 `UserIdentifier(account_id, user_id)` 构造时就跑 `_validate_error()`，非法直接抛 `ValueError` 并打 `Invalid user identifier: ...` 日志。
- API key 形态：`base64(account_id).base64(user_id).base64(secret)`，因此解码 key 前两段即可自证「账号 + 用户」是否对得上。

## 4. 管理接口（测试环境）

- `GET /api/v1/admin/accounts` → 账号列表（含 `account_id` / `created_at` / `user_count`）。
- `POST /api/v1/admin/accounts` `{account_id, admin_user_id}` → 建租户，返回首个用户的 key；自动建 `viking://resources` 与 `viking://user/{admin_user_id}/...`。
- `POST /api/v1/admin/accounts/{account}/users` `{user_id, role}` → 在既有租户下加用户（不新增租户，对他人可见面更小）。
- `DELETE /api/v1/admin/accounts/{id}` → 级联清理该账号的 `viking://user/`、`viking://resources/` 与向量库记录，不可逆。
- 以上均需 root key；`system/status` 可看 `{initialized, user}`。

## 5. 测试环境实测（2026-11-19）

账号：`default`（0 用户，2026-09-03 建）、`internal`（`turing`, role=admin）、`renxiangpeng`（`default`, role=admin，2026-11-19T03:01:55Z 由用户新建）。

用 `renxiangpeng` 的 key 实测：`viking://user` → 只有 `["viking://user/default"]`（**看不到 `internal/turing`**，租户隔离生效）；`viking://user/default` → memories/peers/privacy/resources/sessions/skills；`viking://user/default/memories` → 200 且为空数组。`system/status` → `{"initialized": true, "user": "default"}`。

反例参考：早期用 `internal` 的 `turing` key 访问时，`viking://user/turing` 下是 longyu3 / memories（空）/ peers / privacy / resources / sessions / skills，与 `renxiangpeng` 命名空间完全不重叠。

## 6. DSH 侧四字段填法

地址 `https://ssg-openviking-tst.earth-us.xpaas.lenovo.com`；API Key 用该用户返回的 key（不入库、不落文档）；Account 填租户 id（`renxiangpeng` 或 `internal`）；User Key 所属用户填对应 user_id（`default` 或 `renxp4`/`turing`）。保存时 DSH 会校验 `viking://user/{user_id}/memories`，实测 200，可一次通过并同步出 1 个记忆空间。

## 7. 两个必须知道的代价

1. **只是逻辑隔离**：account 隔离的是命名空间与元数据，底层 S3 bucket、向量库、embedding 配额仍是同一套，写入照样消耗测试环境的解析与向量化资源。
2. **删除不可逆且会波及同一租户**：`DELETE /api/v1/admin/accounts/{id}` 级联清空该账号全部 user/resources/向量记录；另外首个用户必为 admin 角色，权限大于实际所需（只能管本租户，管不到 `internal`）。

## 8. 选型建议（当次给出、用户已按「新建 account」执行）

| | 新建 account | 在 `internal` 下加用户 |
|---|---|---|
| 隔离粒度 | 租户级，最彻底 | 用户级，通常够用 |
| 对他人可见改动 | 多一条 account | 多一条 user |
| 清理/交接 | 需 root 操作整租户 | 删单个用户 |

测试环境推荐后者；若希望整套东西将来能整体迁走，则建独立 account。用户最终新建了 `renxiangpeng` 租户、admin 用户为 `default`（创建时 `admin_user_id` 未改）。遗留可选收尾：在该租户下再注册同名 `renxiangpeng` 用户，让 account 与 user 同名更易读——用户未表态，未执行。
