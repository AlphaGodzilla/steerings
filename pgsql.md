# PostgreSQL 规约

原规约基线为 PostgreSQL 16+、HikariCP 6+、Hibernate 7+；实际版本以当前项目明确的基线为准。

## 规则索引

按索引定位小节，只读命中的部分，不必通读全文。

| 触发场景 | 读哪一节 |
| --- | --- |
| 写或改 Service 事务、并发与锁 | 事务与并发 |
| 建表、定主键或时间字段 | 表结构、主键与时间 |
| 写查询、分页或返回集合的查询 | 查询上限与分页 |
| 写 `JdbcTemplate` / `PreparedStatement` 的可空参数 | 原生 JDBC 可空参数类型 |
| 写 `JdbcTemplate` 回调或手动遍历 `ResultSet` | JdbcTemplate ResultSet 回调语义 |
| 映射 ID、计数或数值列 | JDBC 数值类型映射 |
| 加索引或排查慢查询 | 索引与慢查询 |
| 考虑数据库函数、触发器或存储过程 | 数据库函数、触发器与过程逻辑 |
| 新建或修改 Flyway 迁移脚本 | Flyway 迁移 |
| 写数据库集成测试或验收报告 | 测试与验收纪律 |

## 事务与并发

- 业务事务边界放在 Service 方法，禁止放在 Controller。
- 显式声明 `@Transactional(rollbackFor = Exception.class)`，确保 checked exception 回滚。
- 查询使用 `@Transactional(readOnly = true)`，包括经 Service 执行的单条查询；Repository 查询按项目约定声明只读属性，不代替 Service 事务边界。
- 事务内禁止 HTTP 请求、文件 IO、消息发送和复杂计算；异步操作通过 `TransactionSynchronization` 钩子触发。
- 跨 Service 调用明确 `Propagation`，禁止滥用默认传播行为。
- 同类方法直接调用 `@Transactional` 方法不会经过 Spring AOP 代理；优先将被调用方法抽到独立 Service，或在项目已支持的情况下通过 `AopContext.currentProxy()` 调用。
- JPA 乐观锁使用 `@Version`，类型为 `long`；冲突抛 `OptimisticLockException`。
- 多实例场景禁止依赖数据库行锁实现分布式互斥，避免高并发耗尽连接池；一般分布式互斥首选 Redisson，定时任务防重使用基于数据库表的 ShedLock。另见 [Redis 规约](./redis.md)。

## 表结构、主键与时间

- 所有表必须有主键，首选 Snowflake ID，数据库类型为 `BIGINT`。
- 禁止 UUID 作主键，避免 B-Tree 随机写入碎片化；禁止数据库自增，避免分布式扩展困难及业务规模暴露。
- Snowflake ID 由公共组件统一生成，禁止各模块自行实现；JPA 使用 `@Id` 映射。
- 时间字段统一使用 `TIMESTAMPTZ`，禁止 `TIMESTAMP`；数据库存储 UTC，时区转换交给客户端。
- Java 时间使用 `OffsetDateTime` 或 `Instant`，禁止 `LocalDateTime`。

## 查询上限与分页

- 所有返回 `List`、`Stream` 或其他集合的查询必须显式限制最大返回数量；无界 `findAll`、无上限的集合查询一律视为 bug。
- 可使用 SQL `LIMIT`、Spring Data `Pageable` / `Limit` 或 `TypedQuery#setMaxResults`；不能靠“业务数据不会很多”作为隐性上限。
- Spring Data 派生查询返回 `List<T>` 时必须带 `Pageable`（首选）或 `Limit` 参数。
- 单条结果返回 `Optional<T>` 或 `T`，禁止用 `List<T>` 接收理论上只有一行的数据。
- 原生 SQL 使用 `LIMIT :limit`；JPQL / JPA 同样必须显式限量，通过当前查询 API 支持的 `Pageable`、`Limit` 或 `setMaxResults` 落实，不把 SQL 方言语法直接套入不支持它的 JPQL。
- 控制层分页参数必须钳制最大值，例如 `pageSize ≤ 100`，不能让客户端任意指定上万条。
- 禁止深 OFFSET，必须使用 Keyset 游标分页，例如：

  ```sql
  SELECT id
  FROM records
  WHERE id < :last_seen_id
  ORDER BY id DESC
  LIMIT :limit;
  ```

- 全表导出和批处理使用批量游标循环，每批不超过 1000 条，不一次性加载全表。

## 原生 JDBC 可空参数类型

- `JdbcTemplate`、`JdbcClient`、`PreparedStatement` 中禁止未定型的可空判定占位符，如 `(? IS NULL OR column = ?)`；参数为 `null` 时 PostgreSQL 可能无法推断类型。
- 按实际列类型显式转换判空占位符，例如 `?::BIGINT IS NULL`、`?::VARCHAR IS NULL`、`?::TIMESTAMPTZ IS NULL`；复合游标承担判空职责的参数也必须定型。
- 也可使用显式声明 JDBC 类型的绑定 API，但必须以真实 PostgreSQL 测试证明 null 路径可执行。
- 同一个 Java 变量绑定到多个 `?` 时，每个占位符仍是独立参数；后续比较表达式不会为前面的 `? IS NULL` 提供类型。
- 排查不能只改报错行或名为 cursor 的字段：必须扫描同一领域及全项目原生 SQL 的 `? IS NULL`、`COALESCE(?)`、可空比较和复合游标，逐项核对 BIGINT、文本、枚举文本、TIMESTAMPTZ 等实际类型。
- 新增或修改可选筛选、可空游标或时间范围查询，必须覆盖真实 PostgreSQL 的以下路径：
  1. 所有可选参数均为 null。
  2. 空结果集。
  3. 有数据时的筛选和分页。
- H2、mock 或只有非空参数的测试不能作为验收；完成后报告扫描范围、显式类型映射和测试结果。

## JdbcTemplate ResultSet 回调语义

- 修改前确认 `JdbcTemplate.query(...)` 实际选择的重载，不能凭 lambda 形状判断回调类型。
- `RowCallbackHandler` 是逐行回调，进入回调时 ResultSet 已在当前行；直接调用 `rs.getXxx(...)`，禁止在回调内调用 `rs.next()` 或 `while (rs.next())`。
- 需要手动 `while (rs.next())` 遍历时，必须使用 `ResultSetExtractor`，例如 `PreparedStatementCreator + ResultSetExtractor`，并用注释或方法名说明手动遍历语义。
- 混用这两类回调会静默跳过第一行，造成集合查询和分页数据丢失。
- 新增或修改 JDBC 集合查询必须覆盖 0 行、1 行、2 行结果；两行用例用于捕获跳过首行的问题。
- 提交前全项目扫描 `while (rs.next())`，逐处确认只用于 `ResultSetExtractor`；运行项目已有的 JDBC 回调语义契约测试。

## JDBC 数值类型映射

- 映射以实际 PostgreSQL schema 及表达式返回类型为准，不能根据变量名或小数据测试推断。
- 排查同时核对迁移脚本、实体字段、SQL SELECT 列顺序以及 ResultSet / RowMapper / `queryForObject` 目标类型。
- Snowflake 主键和关联 ID / UID 的数据库类型为 `BIGINT`，Java 使用 `long` / `Long`；JDBC 使用 `getLong(...)`、`getObject(..., Long.class)` 或等价宽类型映射。
- 禁止用 `getInt(...)`、`Integer`、`Integer.class` 或未经检查的窄化转换接收上述 ID。
- PostgreSQL `count(*)`、`count(distinct ...)` 及窗口计数返回 `BIGINT`，默认用 `Long` / `getLong(...)` 接收。
- 接口明确要求有界 `int` 时，先读为 long / Number，再用 `Math.toIntExact(...)` 显式检查溢出。
- 仅判断存在性时使用 `SELECT EXISTS(...)`，不要使用 `count(*)`。
- 实际为 PostgreSQL `INTEGER` 的业务计数器不能机械改成 Long，必须逐列核对 schema。
- 发现一个转换错误时，扫描同一领域及全项目的 `getInt`、`Integer.class`、`intValue`、`count(*)`、`count(distinct ...)` 和所有 ID / UID 查询，逐项记录修复项与确认合法的项。
- 真实 PostgreSQL 回归至少用一个大于 `Integer.MAX_VALUE` 的 Snowflake ID 验证读写、相关流程和幂等路径；聚合映射验证 PostgreSQL 实际返回类型。
- mock 和源码字符串断言只能补充，不能作为唯一验收依据。

## 索引与慢查询

- 复合索引遵循最左前缀：等值列在前，范围列在后。
- 使用部分索引（`WHERE`）减小索引体积，使用覆盖索引（`INCLUDE`）减少回表。
- 不在普通索引列上套函数并期待仍命中原索引；需要函数查询时使用匹配的表达式索引，大小写不敏感文本也可评估 `CITEXT`。
- 生产环境启用 `pg_stat_statements`，定期审查慢查询。

## 数据库函数、触发器与过程逻辑

- 默认使用应用层 Service / Repository 事务、约束、索引及显式查询表达规则，尽量不采用数据库函数、触发器等过程逻辑。
- 只有数据库端实现具有明显且可证明的完整性、并发或性能收益时才可提出使用；不能仅因为 SQL 更短、实现方便或个人偏好采用。
- 创建或修改前必须说明问题、应用层替代方案、数据库端收益与代价、事务 / 锁 / 迁移 / 测试 / 运维影响，以及失败和回滚行为，并取得用户明确许可。
- 未经许可不得创建或修改函数、触发器、事件触发器、存储过程或相关 Flyway DDL。
- 现有数据库函数或触发器也不能擅自删除、替换或弱化，变更同样先说明影响并取得许可。
- 获得许可后，必须提供命名、幂等、状态防回退、性能、异常及迁移回滚说明。
- 使用真实 PostgreSQL 集成测试覆盖正常、异常、并发及迁移升级路径，禁止仅凭 mock、H2 或静态检查验收。

## Flyway 迁移

- 未获得用户明确指令前，禁止新建或修改任何 Flyway 迁移脚本，包括 `V*.sql`、`R*.sql`、`U*.sql`。
- 表结构调整先说明变更点并征求确认，得到明确指令后再实施。
- 已发布到生产或共享环境的迁移脚本不可变，后续变更通过新版本号脚本承载。
- 即使是开发期的 V1 初始脚本，也必须获得授权后才能直接覆写。
- 一般版本化迁移使用 `V{版本}__{描述}.sql` 命名。版本是递增整数

## 测试与验收纪律

- 数据库集成测试使用真实 PostgreSQL Testcontainers；Java / Spring 项目使用 `@SpringBootTest` 和项目容器配置或静态 `@Container`，测试类使用 `IT` 后缀。
- 使用项目规定的测试入口，一次只运行一个 Testcontainers 测试类、一个测试进程，完成后再运行下一个。
- 禁止全量运行、通配符批量启动、一次命令指定多个容器测试类或多个容器测试命令并行执行；`--max-workers=1` 不能替代逐类执行。
- 每个类单独报告命令和结果，不用全量结果替代验收。
- 容器生命周期由测试框架管理，不得删除长期运行的开发数据库容器；环境特有的 socket 和容器引擎参数遵循当前项目操作手册。
- 修复报告包含扫描范围、类型映射依据、已修复项、确认合法项及真实 PostgreSQL 回归结果。

Java 架构、配置、安全与格式化要求另见 [Java 开发规约](./java-development.md)。
