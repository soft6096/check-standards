---
name: check-standards
description: Java 后端代码生成/修改完成后的关键规范兜底自检（check-standards，收敛闸门）。任何 Java 代码写完或改完后必须加载本 skill，用 grep/ast-grep 逐项核对关键规范——注释覆盖率（类/方法 Javadoc/步骤注释）、日志（@Slf4j/入口入参耗时/关键节点 INFO/异常 ERROR 带堆栈）、事务 rollbackFor、构造器注入、分层边界、SQL 在 XML、统一返回体、分页上限、密码加密等 12 项 HIGH + 代码规范组 C1-C6 + 注释组 N1-N4 + INFO + 场景化，每项附「文件:行号」证据；把未执行到位项（含 INFO，不分重要与否）统一提示用户确认是否补齐。**核对前先矫正中间产物命名与路径**（技术方案 3.x.1 / 接口清单 3.x.2 / 规范核对报告 5.2.x / 验收报告 5.3.x，去文件名中的任务 ID 前缀 T0xx，移入模块版本目录）。触发场景：写完代码、代码生成完毕、改完代码、提交前检查、代码规范核对/审查、注释日志核对、code review、规范自检、兜底检查、验收复核、产物文件命名矫正、文件改名归位。与 ai-dev-workflow 5.2 规范核对节点 / 5.3 验收配合使用，也可独立触发。
---

# 关键规范自动核对（Check Standards）

Java 代码生成 / 修改完成后的**兜底闸门**：规范条目多、分散在多个规范 skill，AI 编码可能漏执行。本 skill 用 **grep / ast-grep 实际扫描产出**，逐项核对关键规范，**禁止凭记忆答 ✅**。

> 本 skill 独立可触发，不依赖 ai-dev-workflow 全流程：无论走了完整 spec 流程、还是直接"写个接口/改段代码"，只要产出了 Java 代码，收尾都应加载本 skill 做兜底自检。

## 模式判定与核对依据（执行前必读，先判模式再核对）

**核对判定以项目约束为准，规范 skill 默认值仅兜底**：

| 模式 | 判定依据（先读哪个文件） | 选型类核对项怎么判 |
| :--- | :--- | :--- |
| **标准模式**（新项目 / 未走存量扫描） | `docs/<模块>V<版本>-<时间戳>/2.1-项目约束.md`（技术架构表含 ★ 人确认的选型）；**无该文件时** → 按规范 skill 默认值判定，并在报告中注明"未找到 2.1 约束，按规范默认值判定" | 按 2.1 人确认的选型判定（如 Log4j2 → 按 Log4j2 查，不强制 logback-spring.xml） |
| **存量适配模式**（老项目） | `docs/0.5-存量代码扫描.md` + `2.1-项目约束-存量适配.md`（老项目实际约定）；**无该文件时** → 若明显是老项目，向用户确认"是否按存量适配口径判定"，未确认前选型敏感项暂标"待确认" | 按老项目约定判定（如老项目用 R\<T\> 返回体 → 不判 Response\<T\>；用 Apifox 不写注解 → 按老约定） |
| 判定不了 | 停止并向人确认项目模式（禁止按默认值猜） | — |

**选型敏感核对项**（#1 接口文档 / #2 日志框架 / #3 数据访问层 / #10 返回体）：判定标准随项目模式变化，**先读 2.1 约束 / 0.5 扫描再判**；其余核对项（SQL 注释 / DDL 注释 / 事务 / 注入 / WHERE / 密码 / 分页等）为通用安全项，任何模式都按规范判定。

## 第 0 步：中间产物命名与路径矫正（核对前必做，防核对扫不到/验收引用断裂）

**为什么**：核对指令按规范命名扫描产物（如 `docs/*/3.*.1-技术方案*.md`、`3.*.2-接口清单*.md`）；若产物命名不规范（技术方案漏 `.1`、文件名带任务 ID 前缀 `T02-01`、5.2/5.3 报告漏功能序号）或路径不在模块版本目录下，**核对扫不到、验收引用断裂**。执行核对前先扫描矫正：

### 目标命名规则（与 ai-dev-workflow「产物命名统一规则」一致，单一事实源）

| 产物 | 规范命名 | 常见错误 → 矫正 |
| :--- | :--- | :--- |
| 技术方案 | `3.<功能项序号>.1-<功能名>-技术方案.md` | `3.1-服务产品分类与BANNER-技术方案.md` → `3.1.1-…`（漏 `.1`）；`3.6-T02-01-前台商品列表与详情-技术方案.md` → `3.6.1-…`（漏 `.1` + 任务 ID 前缀） |
| 接口清单 | `3.<功能项序号>.2-<功能名>-接口清单（前后端通用）.md` | `3.6.2-T02-01-前台商品列表与详情-接口清单（前后端通用）.md` → `3.6.2-…`（任务 ID 前缀） |
| 任务拆解 / 契约测试 | `4.1-<功能名>-任务拆解.md` / `4.2-<功能名>-接口契约测试.md` | 带任务 ID 前缀 → 去掉 |
| 规范核对报告（本 skill 产物） | `5.2.<功能项序号>-<功能名>-规范核对报告.md` | `5.2-T02-01-前台商品列表与详情-规范核对报告.md` → `5.2.6-…`（**补功能序号**：从对应技术方案 `3.6.x-前台商品列表与详情-…` 推导 → 6） |
| 验收报告 | `5.3.<功能项序号>-<功能名>-验收报告.md` | `5.3-T02-01-前台商品列表与详情-验收报告.md` → `5.3.6-…`（同上） |

> **功能项序号怎么推**：从同一功能名的**技术方案文件名**（`3.<序号>.1-<功能名>-技术方案.md`）取序号；无技术方案 → 读 `1.1-功能清单.md` 功能项 # 列；仍无法确定 → 停下问用户，禁止猜。

### 执行步骤

1. **扫描**：`ls docs/`（模块目录）+ `find docs -name "*.md"` 列出全部中间产物，逐个对照上表核对文件名与路径
2. **判不合规**：① 技术方案漏 `.1`（`3.1-…`）；② 文件名带任务 ID 前缀（`T0\d+-` / `T02-01` 等）；③ 5.2/5.3 报告漏功能序号（`5.2-…` / `5.3-…`）；④ 路径不在 `docs/<模块名>V<版本号>-<YYYYMMDDHHMMSS>/` 下（散落 docs 根目录 / 其他目录）
3. **列出矫正清单**（`旧路径 → 新路径` + 原因）→ **先向用户确认**（本 skill 不改文件的原则同样适用：确认后才动手）
4. **矫正**：`mv`（或 `git mv`，git 仓库时保留历史）移入正确目录 + 改文件名；**改动引用**：若其他 md（约束/方案/验收报告）引用了旧文件名 → 同步更新引用
5. 矫正结果记入核对报告「产物矫正记录」小节（旧→新 + 原因），供 5.3 验收复核引用

> [!NOTE] 矫正范围边界
> 只矫正**中间产物 md 的文件名与目录**（docs/ 下产物），**不碰源码/测试代码**（src/ 由核对项检查）；无对应功能名 → 不强行改名（停下询问）。

## 执行方式（核心规则）

0. **先矫正产物命名与路径**（见上「第 0 步」）：扫描 docs/ 中间产物，命名/路径不合规 → 列矫正清单 → 用户确认 → 矫正 → 再进入核对
1. **先判模式**（见上"模式判定与核对依据"），读对应约束文件，确定选型类核对项的判定口径
2. 对下方每一项，**实际执行「标准检查指令」**（grep/ast-grep），把**命令 + 命中行（文件:行号）粘贴为证据**
3. 无命中 → 记 `✅`；命中违规 → 记 `❌`（附证据 + 严重度）
4. **所有未执行到位项（HIGH / C/N / INFO / 场景化，不分重要与否）→ 统一列进「待用户确认清单」，一起向用户确认是否执行补齐**：把全部 ❌ 项证据汇总展示给人（**一次确认，不是逐项反复问**），问"以下 N 项未执行到位，是否按规范补齐？"——用户逐项确认（补齐 / 跳过）→ 确认"补齐"的项才执行补齐 → 重跑该项指令 + 新证据（已修复）；补齐后仍无法满足 → **升级人工核对**（标注"需人工核对"）。**禁止未经用户确认自动改代码、禁止把任何 ❌ 项当"不重要"默默跳过**（补齐全过程有人的确认闸门）
5. 输出《关键规范核对报告》（逐项 ✅/❌ + 证据 + 待用户确认清单），结果汇总进验收报告"关键规范落地核对表"

> [!WARNING] 证据强制 + 用户确认闸门（所有未执行到位项一体确认）
> **每项必须给出实际执行的 grep/ast-grep 命令与命中行**。只写 ✅/❌ 无证据 = 未核对，打回重跑。禁止"没做就声称做了"。
> **所有未执行到位项（含 INFO 参考项）→ 必须停下来一起让用户确认是否执行补齐**（人确认后才动手改代码），不自动默默补齐、不因"不阻塞"而跳过确认。

## 检查项清单

### HIGH（必须处理；❌ 补齐 → 重跑 → 仍 ❌ 升级人工核对）

| # | 核对项 | 标准检查指令（实际执行） | 判定标准（全部满足才 ✅） | 规范出处 |
| :---: | :--- | :--- | :--- | :--- |
| 1 | 接口文档支持（OpenAPI/Swagger）⚠️选型敏感 | `grep -n "springdoc\|knife4j\|springfox\|swagger" pom.xml`；`grep -rln "@Tag\|@Operation\|@Api\|@ApiOperation" src/main/java`；`grep -rln "@Schema\|@ApiModelProperty" src/main/java` | **按 2.1 选型/老约定判定**：标准模式 springdoc/knife4j → pom 有依赖 + Controller 有 @Tag/@Operation + DTO/VO 有 @Schema（Apifox 约定注解照写）；老项目 Swagger2 → 按 @Api/@ApiOperation/@ApiModelProperty 体系；老约定不写注解（纯 Apifox 在线文档）→ 按老约定，记"按老约定" | build-standards `dependency-standards.md` 4.6；java-code-standards `api-doc-standards.md` |
| 2 | 日志框架支持 ⚠️选型敏感 | `ls src/main/resources/logback*.xml src/main/resources/log4j2*.xml`；`grep -rln "@Slf4j" src/main/java`；`grep -rn "System.out" src/main/java`；`grep -n "log4j2\|logback" pom.xml` | **按 2.1 选型/老约定判定**：Logback → logback-spring.xml 存在（控制台+滚动文件+环境级 level）；Log4j2 → log4j2 配置存在且 pom 无 logback 并存；任何模式：全类 @Slf4j、无 System.out；**日志覆盖完善（人工抽查关键类）：请求入口有入参摘要+耗时日志、关键业务节点（状态变更/下单/消费完成/定时任务完成）有 INFO、异常处有 ERROR 带堆栈——大段逻辑零日志 → ❌（存量模式同样要求，不因老项目日志稀疏豁免）** | java-code-standards `00-common/04-logging-standards.md`；build-standards 4.7 |
| 3 | SQL 在 XML（手写 SQL 统一收拢）⚠️选型敏感 | `grep -rn "@Select\|@Insert\|@Update\|@Delete\|<script>" src/main/java`；`ls src/main/resources/mapper/*.xml` | **按 2.1/老约定判定**：标准模式 → Java 无注解 SQL，手写 SQL 全在 XML，namespace 一致；老项目 ORM/约定不同（注解 SQL/JPA）→ 按 0.5 扫描的数据访问约定判定（2.1 存量适配约束内要求的仍遵守） | database-standards `mybatis-plus/mybatis-xml-standards.md` |
| 4 | 详细设计 SQL 注释 | `grep -rn "CREATE TABLE\|SELECT \|INSERT INTO\|UPDATE " docs/*/3.*.1-技术方案*.md` | 技术方案《数据模型与 SQL》所有 SQL 带注释：DDL 每字段 COMMENT；查询/DML 每条 `--` 注释（用途+归属 Mapper+关键条件） | ai-dev-workflow `3.0-技术方案-通用骨架.md`；database-standards `sql-standards.md` 1.5 |
| 5 | DDL 字段注释 | `grep -A 25 "CREATE TABLE" src/main/resources/db/schema.sql docs/*/3.*.1-技术方案*.md` | CREATE TABLE **每个字段**带 `COMMENT`（含义/枚举/单位/时区）+ 表级 COMMENT；无裸字段无注释 | database-standards `table-design-standards.md` |
| 6 | JSON 入参/出参产物 | `ls docs/*/3.*.2-接口清单（前后端通用）.md`；`grep -n "入参\|出参" docs/*/3.*.2-接口清单*.md` | 每个 Controller 功能项的 `3.<功能项序号>.2-<功能名>-接口清单（前后端通用）.md` 已生成；每接口有 URL+方法+**JSON 入参示例**+**JSON 出参示例（成功/失败）**；与技术方案一致（无 HTTP 接口功能项标注跳过） | ai-dev-workflow `templates/3.4-接口清单-前后端通用.md` |
| 7 | 事务 rollbackFor | `grep -rn "@Transactional" src/main/java` | 每个 `@Transactional` 均带 `rollbackFor = Exception.class`（无 rollbackFor → ❌） | java-code-standards `service-impl-standards.md` |
| 8 | SQL 注入（拼接/`${}`） | `grep -rn '\${' src/main/resources/mapper/*.xml`；`grep -rn '"SELECT \|"INSERT \|"UPDATE \|"DELETE ' src/main/java` | XML 无 `${}` 拼接值（仅白名单排序/表名，需人工确认）；Java 无字符串拼接 SQL；无 `apply()/last()` 传用户输入 | database-standards `sql-standards.md`；java-code-standards `security-standards.md` |
| 9 | UPDATE/DELETE 带 WHERE | `grep -rn -i "^[[:space:]]*UPDATE \|^[[:space:]]*DELETE " src/main/resources/mapper/*.xml` | 所有 UPDATE/DELETE 语句带 WHERE（无 WHERE → HIGH ❌，全表变更风险） | database-standards `data-safety.md` |
| 10 | 统一返回体 ⚠️选型敏感 | `grep -rn "Map<" src/main/java/*/controller/*.java`；`grep -rln "class R\b\|class Result\b" src/main/java` | **按 2.1/老约定判定**：标准模式 → Controller 返回 `Response<T>`/`PageResult<T>`，无 Map 裸返回，无 R/Result 类；存量模式 → 按老项目返回体（如 `R<T>`）判定，无 Map 裸返回 | java-code-standards `00-common/01-naming-standards.md` / `01-java/controller-standards.md` |
| 11 | 密码加密 | `grep -rn -i "md5\|sha1\|DigestUtils" src/main/java` | 无 MD5/SHA1 用于密码存储（用 BCrypt 等慢哈希）；命中需人工确认场景 | java-code-standards `security-standards.md` |
| 12 | 分页上限 | `grep -rn "pageSize\|PageQuery" src/main/java/*/dto/*.java` | 分页入参 pageSize 有 `@Max`/上限校验 | java-code-standards `controller-standards.md` |

### INFO（❌ 同样列入待用户确认清单，不因"参考"而跳过）

| # | 核对项 | 标准检查指令 | 判定标准 |
| :---: | :--- | :--- | :--- |
| 13 | 敏感信息进日志 | `grep -rn "password\|token\|secret" src/main/java` | 日志/异常/响应不含密码/token 明文（命中核对是否脱敏） |
| 14 | 集合命名 | `grep -rn "List<.*> records\|List<.*> codes\|Set<.*> values" src/main/java` | 集合字段用 `xxxList/xxxSet/xxxMap` 后缀 |
| 15 | 魔法值/缓存 key | 抽查常量类 | 无裸魔法值；缓存 key 集中常量定义 |

### 场景化（对应类型文件存在时才核）

| # | 核对项 | 触发条件 | 判定标准 |
| :---: | :--- | :--- | :--- |
| 16 | Job 防重入 + 批处理 | `grep -rln "@Scheduled" src/main/java` 有命中 | 有分布式锁/状态位防重入；批处理带 LIMIT；无长事务 |
| 17 | Listener 幂等 + 死信 | `grep -rln "@RabbitListener\|@KafkaListener" src/main/java` 有命中 | 消费幂等（唯一键/状态位）；重试有上限 + 死信队列；无 catch 静默 |
| 18 | 文件上传安全 | `grep -rln "MultipartFile" src/main/java` 有命中 | 扩展名+MIME 双白名单；UUID 重命名；大小限制 |
| 19 | 写接口幂等（HTTP） | Controller 存在 POST/PUT 写接口（下单/支付回调/批量提交等） | 写接口有幂等方案：唯一键约束 / 幂等令牌（防重提交） / Redis SETNX；幂等键持久化与业务同事务；无重复双写（自动 grep 难查 → **人工抽查**，标"需人工核对"；见 java-code-standards `distributed-standards.md`） |

### 代码规范组（所有代码必核，❌ 先向用户确认再补齐）

> 规范出处：java-code-standards `00-common/*` + `01-java/*`（命名/分层/注入/异常/实体边界）。质量类项（重复代码）需人工抽查，机器只扫可查信号。
> **存量适配模式**：C1-C4 按老项目约定判定（如老项目统一用 `@Autowired` 字段注入 → 按老约定记录，不强制构造器注入；确认时提供"对齐老项目"选项）。

| # | 核对项 | 标准检查指令（实际执行） | 判定标准 | 严重度 |
| :---: | :--- | :--- | :--- | :--- |
| C1 | 构造器注入 | `grep -rn "@Autowired" src/main/java`（**应为空**） | 无字段注入（`@Autowired` 字段）；统一构造器注入（`@RequiredArgsConstructor` + final 字段）（存量模式按老约定） | HIGH |
| C2 | 分层边界 | `grep -rn "Mapper" src/main/java/*/controller/*.java`；Controller 方法体抽查 | Controller 不注入 Mapper、不写业务逻辑/SQL/事务（只收参→调 Service→返回）（存量模式按老约定） | HIGH |
| C3 | Entity 不暴露 | `grep -rn "Entity\|@TableName" src/main/java/*/controller/*.java`；接口方法出入参抽查 | 接口出入参用 DTO/VO，不暴露 Entity（禁 Entity 直接作请求/响应） | HIGH |
| C4 | 异常处理 | `grep -rn "throw new RuntimeException\|catch (.*) {}" src/main/java` | 无裸 `RuntimeException`（用 `BusinessException(ErrorCode, msg)`）；无空 catch 吞异常（存量模式按老项目异常体系） | HIGH |
| C5 | 命名单字母/泛称 | `grep -rn "catch (.* e)\|throw new.*(.* e)" src/main/java`；抽查类/变量名 | 无单字母类名（R/Result）、异常参数非 `e`、无泛称变量（dto/data/obj） | INFO |
| C6 | 公共组件复用 | 扫描 `common/util`、`common/base` 与业务代码重复方法体（抽查 2-3 处） | Phase 0.5 公共组件已实现且被复用；无 ≥2 处相同/相似方法体（发现 → 提示抽公共，需人工确认） | INFO |

### 注释组（所有代码必核，❌ 先向用户确认再补齐）

> 规范出处：comment-standards `standards/comment-standards.md`（全量注释：所有类/变量/方法注释 + 步骤注释 + // WHY: + 禁翻译式）。机器查"覆盖率"，"质量"（业务含义/不翻译式）需人工抽查——覆盖率 ❌ 与质量可疑项都先向用户确认再处理。

| # | 核对项 | 标准检查指令（实际执行） | 判定标准 | 严重度 |
| :---: | :--- | :--- | :--- | :--- |
| N1 | 类注释覆盖率 | `grep -rn "public class\|public interface" src/main/java`，逐个核对类前有 `/** */` 类注释 | 所有类（Controller/Service/Impl/Mapper/Entity/DTO/VO/Config/Utils）有类注释，第一句概述职责 | HIGH |
| N2 | 方法 Javadoc 覆盖率 | `grep -rn "public " src/main/java`，逐个核对方法前有 `/** */`（含 @param/@return 业务含义） | 所有方法（含测试方法）有 Javadoc：功能 + @param/@return 写业务含义（不重复参数名） | HIGH |
| N3 | 步骤注释 + WHY（**大段代码必须有注释**） | `grep -rn "// 1\." src/main/java` 抽查；对照方法体 | 方法体 ≥2 个逻辑步骤有编号注释（`// 1.`）；复杂逻辑有 `// WHY:`；注释与代码一致；**无 ≥10 行连续逻辑代码零注释**；**长方法（>20 行）逐段核对：每一段逻辑（含方法后半段）都有步骤注释；深层嵌套（≥3 层 if/for/try）分支前必须有注释**（存量模式同样要求，不因老项目注释稀疏豁免） | HIGH |
| N4 | 禁翻译式注释 | 抽查注释（对照代码） | 无逐行翻译式注释（`// 价格乘以数量`）；注释写业务含义非复述代码 | INFO（人工） |

> [!WARNING] N2/N3 必须逐方法通读核对（防"长代码丢焦点"）
> grep 只能证明"哪里**有**注释"，证明不了"哪里**缺**注释"。N2/N3 除跑 grep 外，**必须对本轮生成/修改的每个 Java 文件逐方法通读核对**：
> - 从方法第一行读到最后一行，重点盯**方法后半段**与**深层嵌套分支**——这是 LLM 生成长代码时注意力衰减、注释最易被漏的高发区（典型漏法：写 9 步只注释前 5 步、嵌套 if/for 里整段零注释）
> - 长方法（>20 行）逐段确认：编号步骤注释覆盖到最后一步；嵌套 ≥3 层的分支前有注释
> - 禁止只数 `// 1.` 的 grep 命中数就判 ✅；未通读的方法按"未核对"处理，打回重核

## 输出格式

```text
=== 关键规范核对报告（5.2.<功能项序号>-<功能名>-规范核对报告）===
项目: <路径>    时间: <时间戳>    模式: 标准/存量适配
产物矫正记录（第 0 步，如无 → "无，命名/路径已合规"）:
  docs/3.1-服务产品分类与BANNER-技术方案.md → docs/<模块>V<版本>-<时间戳>/3.1.1-服务产品分类与BANNER-技术方案.md（漏 .1）
  docs/<模块>…/5.2-T02-01-前台商品列表与详情-规范核对报告.md → …/5.2.6-前台商品列表与详情-规范核对报告.md（补功能序号 + 去任务 ID）
[✅] 1 OpenAPI/Swagger    证据: pom.xml:12 springdoc-openapi; UserController.java:3 @Tag
[❌] 7 事务 rollbackFor  证据: OrderServiceImpl.java:45 @Transactional（无 rollbackFor）(HIGH)
[❌] C1 构造器注入       证据: OrderServiceImpl.java:12 @Autowired（字段注入）(HIGH)
[❌] N2 方法 Javadoc     证据: ProductServiceImpl.java:30 public 方法无 /** 注释 (HIGH)
...
结论: 12 HIGH + C/N 组：通过 X / 未通过 Y（INFO 命中 Z 条同样列入待确认）
⚠️ 待用户确认清单（全部未执行到位项，一次确认，按严重度分组）:
  HIGH/C/N: #7 事务 rollbackFor、C1 构造器注入、N2 方法 Javadoc …
  INFO:     #13 敏感信息、#14 集合命名 …
  （逐项：补齐/跳过）
```

- 报告**落盘**：`docs/<模块名>V<版本号>-<YYYYMMDDHHMMSS>/5.2.<功能项序号>-<功能名>-规范核对报告.md`（走 ai-dev-workflow 流程时，**功能项序号与 3.x 技术方案一致**，如功能项 6 → `5.2.6-前台商品列表与详情-规范核对报告.md`，文件名禁止拼任务 ID；**独立触发无 docs 目录时** → 直接在对话里输出报告 + 提示用户是否落盘）；12 项 HIGH + C/N 组 + INFO + 场景化结论汇总进**验收报告**「关键规范落地核对表」
- **全部未执行到位项（含 INFO）→ 一次向用户确认是否补齐**（展示证据 → 人逐项确认补齐/跳过 → 确认的才执行 → 重跑）

> [!NOTE] 两个"人"的动作区分
> - **用户确认闸门**（执行前）：发现未执行到位项 → 向用户确认"是否补齐"（补齐/跳过），确认后 AI 才动手——决定"要不要补"
> - **升级人工核对**（执行后）：补齐后重跑仍 ❌、或用户选择跳过 → 该项标注"需人工核对"，由人介入判断——决定"怎么办"
> 两者都需人在场，但时机不同：确认在动手前，人工核对在动手后仍失败时。

## 完成标准

- [ ] **产物命名与路径已矫正**（第 0 步）：docs/ 中间产物命名符合规范（技术方案 3.x.1 / 接口清单 3.x.2 / 核对报告 5.2.x / 验收报告 5.3.x，无任务 ID 前缀），路径在模块版本目录下；矫正清单已向用户确认，矫正记录写入报告
- [ ] **模式已判定**（标准/存量适配，读对应约束文件；无约束文件时已在报告中注明口径），选型敏感项（#1/#2/#3/#10）与代码规范 C1-C4 按项目约束/老项目约定判定，非规范默认值一刀切
- [ ] 12 项 HIGH + 代码规范组（C1-C6）+ 注释组（N1-N4）+ INFO + 场景化全部实际执行检查指令并附证据（文件:行号）
- [ ] **N2/N3 已逐方法通读核对**（本轮生成/修改的每个 Java 文件从方法头读到方法尾，重点核过方法后半段与深层嵌套分支的注释覆盖，未只凭 grep 命中数判 ✅）
- [ ] **所有未执行到位项（不分重要与否）已一起向用户确认是否补齐**（非自动默默补、非只确认 HIGH）——确认"补齐"的项已补齐重跑通过；用户选择"跳过"或补齐后仍 ❌ → 已标注"需人工核对"（不静默吞掉）
- [ ] 核对报告已输出（走流程时落盘 `docs/<模块名>V<版本号>-<YYYYMMDDHHMMSS>/5.2.<功能项序号>-<功能名>-规范核对报告.md`）
- [ ] 12 项 HIGH + C/N 组 + INFO + 场景化结果已汇总进验收报告核对表

## 与其他 skill 的关系

| skill | 关系 |
|---|---|
| **ai-dev-workflow** | 本 skill 承担其 5.1 收尾 / 5.3 验收的"关键规范自动核对"兜底闸门；ai-dev-workflow 的 `/check-standards` 命令加载本 skill 执行。本 skill 也可脱离 ai-dev-workflow 独立触发 |
| **java-code-standards** | 核对项的规范出处（命名/分层/注入/异常/日志/事务/安全/性能） |
| **comment-standards** | 注释组 N1-N4 的规范出处（全量注释/步骤注释/禁翻译式） |
| **database-standards** | SQL/DDL/索引/数据安全核对项的规范出处 |
| **build-standards** | 依赖（springdoc/knife4j、logback）核对项的规范出处 |
