# Java 开发规约

适用于 Java / Spring 后端开发；技术版本以当前项目明确的基线为准，版本升级不得仅凭本文自动实施。

## 规则索引

按索引定位小节，只读命中的部分，不必通读全文。

| 触发场景 | 读哪一节 |
| --- | --- |
| 引入新框架或库、判断版本兼容性 | 技术基线 |
| 新建领域模块、安排包与目录、划分 Controller / Service / Repository | 架构与目录 |
| 定义 DTO 与校验注解，决定在哪里写注释 | 类型、校验与注释 |
| 使用虚拟线程、`@Async` 或多实例互斥 | 并发与虚拟线程 |
| 声明 `@Transactional`、传播行为或乐观锁 | 事务边界 |
| 新增或修改 REST 接口 | REST API 与文档 |
| 配置 Spring Security、MFA / Step-up、环境变量与密钥 | 安全与配置 |
| 写单元测试、集成测试或运行 Testcontainers | 测试与验收 |
| 格式化代码或创建 git 提交 | 格式化与变更纪律 |

## 技术基线

原规约基线：JDK 21+ LTS、Spring Boot 4.x / Spring Framework 7、Gradle Kotlin DSL 8.14+、Spring Data JPA / Hibernate 7+、JUnit 5.10+、Micrometer + OpenTelemetry。

- Spring Boot 4.x 项目统一使用 `jakarta.*`，禁止继续引入旧的 `javax.*` API。
- 涉及并发、事务、安全和可观测性时，说明设计选择及与当前框架版本的兼容性。
- 使用 PostgreSQL 时同时阅读 [PostgreSQL 规约](./pgsql.md)；使用 Redis 或分布式锁时同时阅读 [Redis 规约](./redis.md)。

## 架构与目录

- 采用垂直切片：基础设施横向共享，业务功能按领域或能力纵向组织。
- Java package 必须与目录一致；能力目录使用小写字母、数字和下划线，禁止连字符。
- 常规领域内部采用 `controller/`、`service/`、`service/impl/`、`repository/`、`entity/`、`dto/`。
- 已采用轻量垂直切片的能力包可将 Controller、Service 接口、ServiceImpl 放在能力根包，保留 `dto/`、`entity/`、`repository/` 子目录；沿用项目已明确的结构，不混用目录约定。
- Controller 只负责路由、参数校验和委托，不包含业务逻辑，不直接依赖外部系统底层客户端。
- Service 接口定义契约，实现层负责业务编排；Repository 负责持久化；Entity 只承载持久化映射，不承担业务编排。
- 外部 API 的 HTTP、签名、时间戳及协议 DTO 封装在独立集成层，业务 Service 不手写这些细节。
- 对外 DTO 使用业务语义，不泄漏密钥、签名、内部集成标识等实现细节。
- 新增领域模块同时提供 DTO、Service 接口、实现、Controller 和测试骨架；有目录契约测试时同步更新。

## 类型、校验与注释

- DTO 使用 `public record`，配合 `@Valid` 及 `jakarta.validation` 约束。
- 废弃的接口或方法必须使用 `@Deprecated` 标记。
- 公共工具类保持无状态，优先使用静态方法。
- 复杂规则、协议映射、并发控制、分布式锁、限频、事务边界、补偿任务、状态防回退和反直觉约束必须说明原因。
- 新增公共组件、跨域工具、定时任务、外部 API 适配和重要异常处理时，至少在类或关键方法处补充必要注释。
- 注释解释意图、约束和风险，不重复代码字面含义。

## 并发与虚拟线程

- Web 调度器和 `@Async` 默认使用 `Executors.newVirtualThreadPerTaskExecutor()`。
- 按原规约约束，禁止在虚拟线程中使用 `synchronized` 或阻塞 IO；这是项目工程约束，不应解释为所有 JDK 版本的固有技术限制。
- 多实例互斥与定时任务防重分开设计，具体遵循 [Redis 规约](./redis.md) 和 [PostgreSQL 规约](./pgsql.md)。

## 事务边界

- 业务事务边界放在 Service 方法，禁止放在 Controller。
- 显式声明 `rollbackFor = Exception.class`，确保 checked exception 触发回滚。
- 查询方法使用 `@Transactional(readOnly = true)`；单条查询经 Service 调用也需声明只读事务。
- Repository 查询按项目约定声明只读属性，不以 Repository 注解代替 Service 业务事务边界。
- 事务内禁止执行 HTTP 请求、文件 IO、消息发送和复杂计算；异步操作通过 `TransactionSynchronization` 钩子触发。
- 跨 Service 调用明确 `Propagation`，禁止滥用默认传播行为。
- 同类方法直接调用 `@Transactional` 方法不会经过 Spring AOP 代理；优先拆到独立 Service，或在项目已支持的情况下通过 `AopContext.currentProxy()` 调用。
- JPA 乐观锁使用 `@Version`，类型为 `long`；冲突按 `OptimisticLockException` 处理。

## REST API 与文档

- 对外 REST API 统一版本前缀，如 `/api/v1/`；错误响应遵循 RFC 7807 `ProblemDetail`。
- 新增、修改或拆分 HTTP 接口前，必须阅读并遵循 [API 文档规约](./apidoc.md)。
- SpringDoc 注解、字段与参数说明、Tag 合并、Knife4j 排序、认证标记及生成结果验收统一在 API 文档规约中维护，不在本文重复定义。

## 安全与配置

- Spring Security 按 Stateless + JWT/OAuth2 方式配置；纯 API 显式关闭 CSRF。
- 敏感操作复用项目统一的 MFA / Step-up 服务，不在各业务 Controller / Service 自行实现验证码验证、短期提升会话或令牌黑名单。
- 敏感请求通过请求体传递 access token，由 Service 校验用途、有效期及用户归属，并一次性消费；禁止设计有效期内可重复使用的 Step-up token。
- 已明确分离的安全域保持签名密钥、issuer、purpose 和 Redis 命名空间隔离，不得擅自合并认证体系。
- token 缺失、无效、过期、用途不匹配或重放返回 401；token 归属与当前用户不符返回 403。统一经异常处理器映射为 `ProblemDetail`。
- 配置走 `@ConfigurationProperties`，不硬编码；按项目基线启用绑定校验（原规约使用 `spring.config.validate-on-bound=true`）。
- 新增配置环境变量占位符时同步更新 `.env.example`，注明用途、示例、取值建议及本地/生产使用方式。
- 不生成或提交真实密钥、访问令牌、生产连接串及个人环境配置。
- traceId / spanId 通过 Micrometer + OpenTelemetry 自动注入，禁止手动拼接。

## 测试与验收

- 单元测试使用 JUnit 与 Mockito，Mock Repository，使用 `@ExtendWith(MockitoExtension.class)`。
- 集成测试使用 `@SpringBootTest` + Testcontainers，依赖真实数据库 / Redis，不以 mock 或 H2 替代真实基础设施验收。
- 新接口同步提供包含 `@Test` 和断言模板的测试骨架。
- 需要真实基础设施的集成测试类使用 `IT` 后缀，通过项目的 Testcontainers 配置或静态 `@Container` 管理容器。
- 运行测试必须使用项目规定的测试入口脚本，保留其环境加载和参数注入逻辑，不绕过脚本直接调用构建工具。
- 只要启动 Testcontainers，一次只运行一个测试类、一个测试进程，完成后再运行下一个。
- 禁止 Testcontainers 全量运行、通配符匹配多个类、一次命令指定多个测试类或多个测试命令并行执行；`--max-workers=1` 不能替代逐类执行。
- 纯单元测试且确认不启动 Testcontainers 时才可全量运行。
- 验收逐类报告命令与结果，不以一次全量运行结果替代。
- 容器生命周期交由测试框架管理；不得删除长期运行的开发数据库或 Redis 容器。运行环境特有的 socket、Ryuk 和参数传递方式遵循当前项目操作手册，不硬编码本机路径。

## 格式化与变更纪律

- 使用 Spotless；仅格式化当前 git 未提交的 modified / untracked 文件，通过 `ratchetFrom HEAD` 限定范围。
- 格式化前运行 `git status --short` 确认范围；产生无关历史文件格式化时，只撤销本次引入的无关差异，不覆盖用户已有改动。
- 修改前先读相关实现，沿用已有模式和命名；生成代码前校验包路径、类名和目录归属。
- 业务边界模糊、技术选型冲突时先确认，不自行假设。
- 未经用户明确同意不执行 git commit；提交消息遵循 [软件工程规约](./software_enginerring.md) 指定的流程。
