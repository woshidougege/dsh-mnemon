---
id: "cdf9332c-d834-409e-a325-7bd98a09249e"
title: "admin 启动耗时：8 份线程栈结论（SkyWalking 织入放大）与诊断回滚（ssg-sales-ops）"
description: "192~195s 启动的 8 份 main=RUNNABLE 栈证据、fat jar×SkyWalking 插件×AspectJ 切点三重放大机制、65s 空窗被看门狗配额错过的原因、用户叫停后的回滚（MR !527）与恢复办法、未实施的提速清单。"
status: "active"
created_at: "2026-11-12T15:34:37.616Z"
updated_at: "2026-11-12T15:34:37.616Z"
content_hash: "32ba7028a7693aa3dbbbcf58a9008462fea284c6da11ba04074a65c2f1badce1"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/AdminApplication.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/TrinoToPostgresSyncJobRecovery.java"
session_ids:
  - "mnemon-run-v1-05d74842-f54b-43c3-995b-1aed77b56a80"
memory_body_ids:
  []
---

# admin 启动耗时：8 份线程栈结论（SkyWalking 织入放大）与诊断回滚（ssg-sales-ops）

相关文档：`114173fc-0b85-4307-b84e-887ba2faaea3`（第一/二轮打点方案、分段耗时、MR !522/!523 的实现与部署后读日志判断树）。本篇只记录**第三轮 dump 的结论**与**用户叫停后的回滚结果**，不重复打点实现。

## 1. 结果：用户叫停，全部诊断代码已回滚

2026-11-12 用户决定（原话）：“把你加的都取消吧 现在感觉速度还可以了 没必要排查了”。

- 当时 **MR !523 已是 merged**（不是我关的），远端 `fix/startup-timing-admin` 已随合并自动删除（DELETE 接口 404 即因此），所以“取消”只能做回滚。
- 回滚分支 `chore/revert-startup-diagnostics-admin`，基于当时最新 `origin/admin-release` `9a2e344f0`：
  - `1cbb4e4f3` Revert 766f3640c（!523）→ 删除 `StartupWatchdog.java`、`StartupEventLogger.java`
  - `97aa9db10` Revert 61a12a6b6（!522）→ 删除 `StartupRunnerTailLogger.java`，撤销 Runner/数据源/批次恢复的耗时日志与 `@Order`
  - 净 -284 行、删 3 个文件、6 个文件恢复原样
- 验证：与诊断前基线 `4cf184979^1` 对比这 6 个文件 `git diff --stat` 为空；`mvn -o -T 4 -pl smartop-service/admin -am -DskipTests compile` → BUILD SUCCESS（合计 2:41，admin 1:29）；全模块 grep `StartupWatchdog|StartupEventLogger|StartupRunnerTailLogger|\[startup\]` 命中 0。
- MR **!527**（https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/527）opened / can_be_merged / 无冲突 / 9 files，sha `97aa9db102e9`，等待用户手动合并（约定：合并由用户点击）。
- 已清理本地 worktree `.tmp/worktrees/startup-timing` 与本地分支 `fix/startup-timing-admin`。
- **恢复诊断的办法**：原提交仍在历史里，直接 cherry-pick `61a12a6b6` / `766f3640c` 即可，不必重写打点。

## 2. 8 份 dump 的共同特征：main 全程 `RUNNABLE`，无等待

本次 `run()` 总耗时 **195.193s**（上一次 158.093s）。同一个包、同样的路径波动 37s，指向类加载/磁盘与 CPU 竞争，而不是固定等待。

| # | 相对 run() | 栈顶关键帧 | 含义 |
|---|---|---|---|
| 1 | +50.1s | `defineClass1` ← `FastJsonRedisSerializer.<clinit>` ← `RedisCacheConfig.<clinit>` ← cglib `ConfigurationClassEnhancer.enhance` | 生成配置类 CGLIB 子类触发配置类静态初始化，类加载落在启动关键路径 |
| 2 | +60.1s | `Server.doStart` ← `JettyWebServer.initialize` ← `createWebServer` | 创建 Jetty（21:40:21.718 打出 jetty-11.0.18） |
| 3 | +70.1s | `Arrays.binarySearch0` ← `JarFileEntries.getEntry` ← `JarURLConnection.get` ← `ClassLoader.getResource` ← bytebuddy `ClassFileLocator.locate` ← `ExecutingTransformer.transform` | SkyWalking 为找目标类原始字节码，回 fat jar 里再查一遍 |
| 4 | +80.1s | `AopUtils.canApply` ← `AspectJExpressionPointcut.matches` ← `InstMethodsInter.intercept` | 应用自身切点匹配也被 agent 插桩 |
| 5 | +100.2s | `SignatureParser.parseMethodTypeSignature` ← `Method.getGenericParameterTypes` ← `ResolvedMemberImpl.matches` ← `AspectJExpressionPointcut.matches$original$V4DgUpsf` / `$auxiliary$` | `$original$`/`$auxiliary$` 是 agent 改名证据：每个 Bean 每次匹配多一层拦截并重解析泛型签名 |
| 6 | +110.2s | `Inflater.inflateBytesBytes` ← bytebuddy `StreamDrainer.drain` ← `ClassFileLocator.locate` | 每个被织入的类都要从压缩包解压字节码 |
| 7 | +120.2s | `RandomAccessFile.readBytes` ← `JarFileEntries.getEntryData` ← `ClassLoader.getResource` ← bcel `ClassLoaderRepository.loadClass` ← `Java15AnnotationFinder.getAnnotations` ← `WithinAnnotationPointcut.fastMatch` | AspectJ 判注解切点又从 jar 随机读 class |
| 8 | +120.2s | 同 #7，配额用尽 | **+120s 之后没有任何 dump** |

## 3. 三重放大机制（100 多秒的真正来源）

1. **Spring Boot fat jar 类读取很贵**：`getResource` 走 `JarURLConnection` → 二分查找条目 → inflate 解压。单次不贵，但 SkyWalking 对**每个类**都要再定位一次原始字节码（#3），有时解压（#6），AspectJ 还会再读一遍（#7）。
2. **SkyWalking 8.16 注册 100+ 插件**：每个类都过 `ProtectiveShieldMatcher → HasSuperTypeMatcher → TypePool` 完整匹配链，`TypePool` 解析类又回到第 1 条的 jar 读取路径 → 互相放大。
3. **应用侧 AspectJ 表达式切点偏宽**：栈中出现 `couldMatchJoinPointsInType`、`getGenericParameterTypes`，每个 Bean 的每个候选方法都要签名级匹配；而 `matches` 本身又被插桩（#4、#5），开销翻倍以上。

## 4. 66 秒空窗：这次仍被漏掉，但边界已锁定

```
+128.363s  WebServerInitializedEvent（端口 80 就绪）
+129.197s  ContextRefreshedEvent  context=AnnotationConfigApplicationContext@688e2e8d（子上下文）
   ……65.7 秒，无任何日志、无 dump……
+194.882s  ContextRefreshedEvent  context=ssg-sales-ops--admin-1（主上下文 refresh 完成）
+194.973s  ApplicationStartedEvent
+195.193s  run() 返回
```

看门狗 `MAX_DUMPS=8`、间隔 10s → 配额在 **+120.194s** 用尽，而窗口从 +129.2s 才开始，正好错过。若以后还要查这段：

- 看门狗参数改为 `MAX_DUMPS=40`、`FIRST_DUMP_AFTER_MS=40000`、`DUMP_INTERVAL_MS=10000`，`run()` 返回即停，可覆盖到 6 分钟；栈指纹相同则打 “stuck” 标记。
- 再注册 `SmartLifecycle(phase=Integer.MAX_VALUE)` 与 `SmartInitializingSingleton` 打点，把 `finishRefresh` 段切开。
- 另：Runner 名单里有第 4 个 `ddlApplicationRunner`，仓库内搜不到（来自依赖 jar），是否执行 DDL 尚未查证。

## 5. 当时给出的提速清单（未实施，按预期收益排序）

| 项 | 做法 | 预期 |
|---|---|---|
| 精简 SkyWalking 插件（最大项） | 只保留实际使用的：`apm-springmvc-annotation-commons`、tomcat/jetty-thread-pool、`apm-jdbc-commons`、postgresql、httpclient、okhttp、kafka、spring-kafka、xxl-job、spring-scheduled、toolkit-logback、trace-ignore；其余从镜像 plugins 目录裁掉，或用 `plugin.exclude_plugins` | -30~60s |
| AspectJ 切点收窄 | 把 `execution(* com.lenovo..*.*(..))` 这类宽表达式改成 `@annotation(...)` / `@within(...)` | -10~20s |
| 去掉 fat jar | 解压目录或 thin jar + lib（`-Dloader.path`），让 `getResource` 从“查找+解压”变直接文件读 | -10~20s |
| AppCDS | `-XX:ArchiveClassesAtExit` 打归档后带 `-XX:SharedArchiveFile` 启动 | -5~10s |

用户判断当前速度可接受，以上均**未做**；若日后启动时间再次成为问题，先按此表动手，不必先重跑 dump。

## 6. 涉及文件

- `smartop-service/admin/src/main/java/com/lenovo/admin/AdminApplication.java`
- `smartop-service/admin/src/main/java/com/lenovo/admin/conf/StartupWatchdog.java`（已在 !527 删除）
- `smartop-service/admin/src/main/java/com/lenovo/admin/conf/StartupEventLogger.java`（已在 !527 删除）
- `smartop-service/admin/src/main/java/com/lenovo/admin/conf/StartupRunnerTailLogger.java`（已在 !527 删除）
- `smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/TrinoToPostgresSyncJobRecovery.java`
- `smartop-service/admin/src/main/java/com/lenovo/admin/coc/schedule/CocFlagSyncTaskRegistrar.java`
